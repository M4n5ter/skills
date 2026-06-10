---
name: ubuntu-gnome-wayland-rdp
description: Set up, verify, and troubleshoot Ubuntu GNOME Wayland Remote Login over RDP with GNOME Remote Desktop system mode and GDM. Use when the user wants lock-screen or logged-out RDP access to Ubuntu GNOME, insists on Wayland instead of xrdp/Xorg, connects from macOS Windows App or Microsoft Remote Desktop, sees server redirection failures, black screens, login loops, blurry or offset resolution, stale handover sessions, or wants cleanup after remote desktop experiments.
---

# Ubuntu GNOME Wayland RDP

## 目标架构

使用 GNOME Remote Desktop 的 system-mode remote login：

- 服务端入口：`gnome-remote-desktop.service`，system bus，监听 `3389`。
- 登录管理器：`gdm3` / `gdm.service`，创建 remote greeter 和 remote user session。
- 桌面会话：GNOME Wayland，不走 xrdp/Xorg/XFCE。
- 客户端：macOS Windows App / Microsoft Remote Desktop，必须能跟随 GNOME RDP server redirection。

不要把 user-mode remote desktop 当成 remote login。user-mode 只能共享已登录且未锁屏的当前会话；锁屏、未登录、GDM 登录界面都应使用 system-mode。

## 操作原则

- 先通过 SSH 取证，再改配置。RDP 黑屏和登录循环经常是会话状态问题，不一定是配置问题。
- 不要安装或建议 `xrdp`、`xorgxrdp`、XFCE、lightdm，除非用户明确接受 Xorg fallback。
- 不要随手重启 `gdm.service`。它会结束本地和远程图形会话；只有用户接受图形会话中断时才重启。
- 不要在客户端正处于登录或黑屏 handover 期间重启 `gnome-remote-desktop-handover.service`。这会打断当前握手，可能把“黑屏”变成“回到选用户循环”。
- 不要点击 GNOME 顶栏橙色远程会话指示器里的停止/断开控件，除非就是要终止当前 RDP 虚拟显示器。它可能让客户端失去交互并留下残留连接。
- 不要在回复里泄露 `grdctl --show-credentials` 输出或用户密码。

## 初始检查

```bash
lsb_release -a
grdctl --help
zcat /usr/share/doc/gnome-remote-desktop/README.md.gz 2>/dev/null | sed -n '1,220p'
systemctl status gdm.service gnome-remote-desktop.service --no-pager
dpkg -l | grep -E 'gnome-remote-desktop|gdm3|gnome-shell|mutter|gnome-session|ubuntu-session'
```

确认没有误装的 Xorg remote stack：

```bash
dpkg -l | grep -Ei 'xrdp|xorgxrdp|xfce|lightdm'
```

## 配置 system-mode RDP

安装组件：

```bash
sudo apt update
sudo apt install -y gnome-remote-desktop gdm3
cat /etc/X11/default-display-manager
```

创建 system daemon TLS 文件。优先使用 `gnome-remote-desktop` 用户的 home 布局：

```bash
sudo install -d -o gnome-remote-desktop -g gnome-remote-desktop -m 0700 \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop

sudo -u gnome-remote-desktop HOME=/var/lib/gnome-remote-desktop \
  openssl req -x509 -nodes -newkey rsa:4096 -days 3650 \
  -subj '/CN=ubuntu-remote-login' \
  -keyout /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key \
  -out /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt

sudo chmod 0600 /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key
sudo chmod 0644 /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt
```

配置 RDP。`set-credentials` 是 RDP dispatcher 入口凭据；redirection 到 GDM 后仍会用 Ubuntu 用户密码登录桌面。

```bash
sudo grdctl --system rdp set-tls-key /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key
sudo grdctl --system rdp set-tls-cert /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt
sudo grdctl --system rdp set-auth-methods credentials
sudo grdctl --system rdp set-credentials <rdp-username>
sudo grdctl --system rdp enable
sudo systemctl enable --now gnome-remote-desktop.service
sudo systemctl enable display-manager.service 2>/dev/null || sudo systemctl enable gdm.service
sudo systemctl set-default graphical.target
```

验证：

```bash
sudo grdctl --system status
sudo ss -ltnp | grep ':3389'
systemctl is-active gdm.service gnome-remote-desktop.service
journalctl -u gnome-remote-desktop.service --since '5 minutes ago' --no-pager | tail -120
```

## 开机自启验证

remote login 依赖 GDM 进入 `graphical.target`，以及 system-mode `gnome-remote-desktop.service` 开机监听 `3389`。

```bash
systemctl get-default
systemctl is-enabled display-manager.service gnome-remote-desktop.service gnome-remote-desktop-configuration.service
systemctl is-active gdm.service gnome-remote-desktop.service gnome-remote-desktop-configuration.service
ls -l /etc/systemd/system/display-manager.service 2>/dev/null || true
```

不要过度解读 `gdm.service` 显示 `disabled`。Ubuntu 常通过 `display-manager.service` alias 和 `graphical.target` 拉起 GDM。关键判断是：

- `systemctl get-default` 是 `graphical.target`。
- `display-manager.service` 指向 GDM。
- `gdm.service` 当前 `active`。
- `gnome-remote-desktop.service` 是 `enabled` 且 `active`。
- `gnome-remote-desktop-configuration.service` 是 `enabled`；当前 `inactive` 通常正常。

## Windows App / Microsoft Remote Desktop

普通“添加电脑”卡片可能无法处理 GNOME 的 RDP server redirection。若服务端日志出现 `Sending server redirection` 随后 `ERRINFO_LOGOFF_BY_USER`，优先导入 `.rdp` 文件。

推荐最小 `.rdp`：

```text
full address:s:192.168.1.101:3389
username:s:m4n5ter
use redirection server name:i:1
authentication level:i:0
prompt for credentials on client:i:1
enablecredsspsupport:i:0
screen mode id:i:2
desktopwidth:i:0
desktopheight:i:0
dynamic resolution:i:1
smart sizing:i:1
session bpp:i:32
use multimon:i:0
redirectclipboard:i:1
audiomode:i:0
```

显示设置：

- 分辨率使用“此显示器的默认值”或客户端动态分辨率。
- 开启全屏启动、适应窗口、调整大小时更新会话分辨率。
- 避免固定 Retina 物理分辨率，例如 `desktopwidth:i:2560`、`desktopheight:i:1664`、`desktopscalefactor:i:200`、`dynamic resolution:i:0`。
- 画面偏到右下角、有滚动条或轻微模糊时，先修客户端显示设置，不要先改 GNOME 服务端。

## 诊断入口

先确认网络和 system-mode 服务：

```bash
nc -vz <host> 3389
ssh <user>@<host> 'date; systemctl is-active gdm.service gnome-remote-desktop.service; ss -ltnp | grep ":3389" || true'
ssh <user>@<host> 'sudo grdctl --system status'
ssh <user>@<host> 'journalctl -u gnome-remote-desktop.service --since "10 minutes ago" --no-pager | tail -160'
```

再看会话拓扑：

```bash
loginctl list-sessions --no-legend
for s in $(loginctl list-sessions --no-legend | awk '{print $1}'); do
  echo "--- session $s ---"
  loginctl show-session "$s" -p Id -p Name -p User -p Class -p Type -p State \
    -p Service -p Remote -p RemoteHost -p Leader -p LockedHint -p Active -p Desktop
done
```

常见 session 形态：

- remote greeter：`Class=greeter`，`Service=gdm-launch-environment`。
- remote desktop：`Service=gdm-password`，`Type=wayland`，`Remote=yes`。
- SSH：`Service=sshd`，`Type=tty`；不要误判为 RDP session。

## 故障分支

### 0x207 或连接后立刻断开

如果 system GRD 日志有：

```text
[RDP] Sending server redirection
ERRINFO_LOGOFF_BY_USER
```

通常是客户端没有跟随 redirection。使用包含 `use redirection server name:i:1` 的 `.rdp` 导入连接；不要继续调 Ubuntu 服务。

### 输入密码后回到选用户

先判断认证是否成功：

```bash
journalctl -u gdm.service --since '10 minutes ago' --no-pager | tail -220
```

若看到 `pam_unix(gdm-password:session): session opened` 和 `gkr-pam: unlocked login keyring`，密码已经通过，问题在 session/handover，不是账号密码。

检查旧 remote session：

```bash
loginctl list-sessions --no-legend
loginctl show-session <session-id> -p Service -p Type -p Remote -p LockedHint -p State -p Leader
```

如果存在旧的 `Service=gdm-password`、`Type=wayland`、`Remote=yes`，且 `LockedHint=yes` 或用户一直回到选用户，通常是旧 RDP session 卡住。经用户确认会关闭该远程 GUI session 后，清理：

```bash
loginctl terminate-session <session-id>
sudo systemctl restart gnome-remote-desktop.service
```

### 输入密码后黑屏、不进入桌面

先确认桌面是否已经起来：

```bash
ps -eo pid,user,stat,comm,args | grep -E 'gnome-shell|gnome-session|gnome-remote|gdm-wayland|Xwayland' | grep -v grep
journalctl _UID=<uid> --since '10 minutes ago' --no-pager | \
  grep -Ei 'gnome-shell|gnome-session|mutter|remote-desktop|handover|wayland|fail|error|warning' | tail -240
```

如果看到 `gnome-shell --mode=ubuntu`、`gnome-session` 已启动，`loginctl` 中对应 session 是 `Remote=yes`、`Service=gdm-password`、`LockedHint=no`，但客户端仍黑屏，检查 handover：

```bash
busctl --system tree org.gnome.RemoteDesktop
gdbus introspect --system --dest org.gnome.RemoteDesktop \
  --object-path /org/gnome/RemoteDesktop/Rdp/Dispatcher
gdbus introspect --system --dest org.gnome.RemoteDesktop \
  --object-path /org/gnome/RemoteDesktop/Rdp/Handovers/session<id>

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  journalctl --user -u gnome-remote-desktop-handover.service --since '10 minutes ago' --no-pager | tail -160
```

若 system tree 存在 `/org/gnome/RemoteDesktop/Rdp/Handovers/session<id>`，且属性 `HandoverIsWaiting = true`，同时 user handover 日志有：

```text
[DaemonHandover] Failed to start handover: ... UnknownObject: Failed to start handover
```

这是半完成 handover：GDM 已创建 remote user session，system dispatcher 正等 handover，但用户态 handover 没有成功接管客户端。不要重启 user handover 来“救活”当前连接；先让客户端断开，然后清理失败 session 和用户态 GNOME targets：

```bash
loginctl terminate-session <session-id>

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  systemctl --user stop gnome-remote-desktop-handover.service \
    gnome-session@ubuntu.target gnome-session.target graphical-session.target 2>/dev/null || true

busctl --system tree org.gnome.RemoteDesktop
ss -ltnp | grep ':3389'
```

预期：`Handovers` 目录为空，`3389` 仍监听。让用户关闭 RDP 窗口并重新连接。

### 点击橙色远程会话指示器后失去交互

GNOME 顶栏的橙色远程会话指示器包含停止/断开当前远程会话的控件。误点后常见日志：

```text
gnome-shell: Removed virtual monitor Meta-0
gnome-remote-desktop-daemon: Disconnected by EIS
ERRINFO_RPC_INITIATED_DISCONNECT
```

这是 GNOME 主动移除了 RDP 虚拟显示器或输入通道，不是密码错误，也不是 `3389` 没监听。典型状态：

- `gnome-remote-desktop.service` 仍 active。
- `ss -ltnp` 仍显示 `*:3389` LISTEN。
- `ss -tnp` 可能残留一条 `:3389` ESTAB。
- `loginctl` 仍有 `Service=gdm-password`、`Type=wayland`、`Remote=yes` 的 user session。
- system handover tree 可能仍有 `/Rdp/Handovers/session<id>`，但 `HandoverIsWaiting` 可能是 `false`。

清理顺序：

```bash
loginctl terminate-session <session-id>

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  systemctl --user stop gnome-remote-desktop-handover.service \
    gnome-session@ubuntu.target gnome-session.target graphical-session.target 2>/dev/null || true

sudo systemctl restart gnome-remote-desktop.service

busctl --system tree org.gnome.RemoteDesktop
ss -ltnp | grep ':3389'
ss -tnp | grep ':3389' || true
```

预期：`Handovers` 目录为空，`3389` 只有 LISTEN，没有 ESTAB。让用户关闭旧 RDP 窗口后重新连接。不要为了这个故障重启 GDM，除非 session 无法被正常终止。

如果 Ubuntu 侧已经没有 remote user session，但仍有 `:3389` ESTAB/CLOSE-WAIT，并且 GRD 日志没有新的 redirection/GDM 登录事件，检查客户端进程是否残留连接：

```bash
lsof -nP -iTCP@<ubuntu-ip>:3389 || true
netstat -anv -p tcp | grep '<ubuntu-ip>.3389' || true
```

在 macOS Windows App 中，残留连接通常归属于 `/Applications/Windows App.app/Contents/MacOS/Windows App`。先正常退出客户端：

```bash
osascript -e 'tell application "Windows App" to quit' || true
sleep 2
lsof -nP -iTCP@<ubuntu-ip>:3389 || true
```

只有正常退出失败时才考虑杀进程。退出后再次确认 Ubuntu 侧只有 LISTEN：

```bash
ssh <user>@<ubuntu-ip> 'ss -tnp | grep ":3389" || true; ss -ltnp | grep ":3389" || true'
```

长期规避误点：安装一个用户级 GNOME Shell extension，把 `ScreenSharingIndicator._stopSharing()` patch 成 no-op。不要改 `/usr/lib/gnome-shell/libshell-*.so` 或系统 gresource；系统包更新会覆盖且风险更高。

扩展目录：

```text
~/.local/share/gnome-shell/extensions/disable-remote-sharing-stop@m4n5ter.local/
```

`metadata.json`：

```json
{
  "uuid": "disable-remote-sharing-stop@m4n5ter.local",
  "name": "Disable Remote Sharing Stop Button",
  "description": "Prevents accidental clicks on GNOME's remote sharing indicator from stopping the active RDP virtual monitor.",
  "shell-version": ["50"]
}
```

`extension.js`：

```js
import {Extension} from 'resource:///org/gnome/shell/extensions/extension.js';
import {ScreenSharingIndicator} from 'resource:///org/gnome/shell/ui/status/remoteAccess.js';

export default class DisableRemoteSharingStopExtension extends Extension {
    enable() {
        this._originalStopSharing = ScreenSharingIndicator.prototype._stopSharing;

        ScreenSharingIndicator.prototype._stopSharing = function () {
            console.log('disable-remote-sharing-stop: ignored remote sharing stop click');
        };
    }

    disable() {
        if (this._originalStopSharing)
            ScreenSharingIndicator.prototype._stopSharing = this._originalStopSharing;

        delete this._originalStopSharing;
    }
}
```

启用：

```bash
XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  gsettings get org.gnome.shell enabled-extensions

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  gsettings set org.gnome.shell enabled-extensions "['existing@uuid', 'disable-remote-sharing-stop@m4n5ter.local']"
```

它会在下一次 GNOME session 启动时生效。登录后点击橙色远程共享按钮应只产生日志 `ignored remote sharing stop click`，不应再出现 `Removed virtual monitor Meta-0`。

### 多个 remote greeter 残留

多次失败 redirection/handover 可能留下多个 `gdm-greeter-*` session。它们通常是运行态残留。优先清理失败的 `gdm-password` remote user session；只有用户接受图形会话中断时才重启 GDM：

```bash
sudo systemctl restart gdm.service
sudo systemctl restart gnome-remote-desktop.service
```

## 用户态 D-Bus 快速探针

在 remote user session 存在时，用这些命令确认 GNOME Shell/Mutter 对象是否存在：

```bash
XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  busctl --user list | grep -Ei 'gnome|mutter|remote|shell'

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  gdbus introspect --session --dest org.gnome.Mutter.RemoteDesktop \
    --object-path /org/gnome/Mutter/RemoteDesktop | head -160

XDG_RUNTIME_DIR=/run/user/<uid> DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  gdbus introspect --session --dest org.gnome.Shell \
    --object-path /org/gnome/Shell | head -160
```

`Cannot load libcuda.so.1`、`Cannot load libnvidia-encode.so.1` 在 AMD/iGPU 机器上通常只是硬件编码探测失败；如果 Vulkan 初始化成功，不要把它当主因。

## 清理最终状态

只保留 system-mode remote login：

- 保留 `/etc/gnome-remote-desktop/grd.conf`。
- 保留 `/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.*`。
- 保留 system `gnome-remote-desktop.service` enabled/active。
- 保留 `gdm3` / `display-manager.service`。

清理 user-mode/headless 试验：

```bash
sudo -u <user> XDG_RUNTIME_DIR=/run/user/<uid> \
  DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<uid>/bus \
  dconf reset -f /org/gnome/desktop/remote-desktop/

sudo -u <user> XDG_RUNTIME_DIR=/run/user/<uid> \
  systemctl --user disable --now gnome-remote-desktop.service gnome-remote-desktop-headless.service 2>/dev/null || true

sudo rm -rf /home/<user>/.local/share/gnome-remote-desktop
sudo rm -f /etc/gnome-remote-desktop/rdp-tls.key /etc/gnome-remote-desktop/rdp-tls.crt
```

清理 xrdp/XFCE/lightdm 试验：

```bash
sudo apt purge -y xrdp xorgxrdp xfce4 lightdm dbus-x11
sudo apt autoremove --purge -y
dpkg -l | grep -Ei 'xrdp|xorgxrdp|xfce|thunar|lightdm'
dpkg -l | grep '^rc' | grep -Ei 'xrdp|xorgxrdp|xfce|thunar|lightdm|dbus-x11'
```

macOS 侧：

- 删除临时 `.rdp` 文件，除非用户想保留备份。
- 删除调试中创建的 Windows App 数据库备份。
- 用 Windows App UI 删除旧失败设备卡片，只保留导入后可工作的卡片。
- 不要直接编辑 Windows App 内部 SQLite/Core Data 数据库。

最终核验：

```bash
sudo grdctl --system status
sudo ss -ltnp | grep ':3389'
systemctl get-default
systemctl is-active gdm.service gnome-remote-desktop.service
systemctl is-enabled display-manager.service gnome-remote-desktop.service gnome-remote-desktop-configuration.service
sudo find /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop -maxdepth 1 -type f -ls
sudo find /etc/gnome-remote-desktop -maxdepth 1 -type f -ls
```
