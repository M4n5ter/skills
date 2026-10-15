# TypeScript 类型建模

在 TypeScript 中落实[类型系统纪律](type-system-discipline.md)时使用。优先让数据形状和编译器承担约束；只在外部数据进入时解析、校验，并在确有需要的使用点强化类型。

## 用可辨识联合表达状态

让每个变体都有相同名称、不同字面量值的判别字段，避免布尔值与可选字段拼出矛盾状态。

```ts
type DiffState =
  | { kind: "loading" }
  | { kind: "ready"; diff: string }
  | { kind: "error"; error: string };
```

统一使用 `kind`、`type` 或 `tag` 中的一种。不要让同一对象可以同时表示“加载中”和“结果已就绪”。

## 以构造表达约束

```ts
type NonEmpty<T> = [T, ...T[]];
type Pairs<T> = [T, T][];

function isNonEmpty<T>(xs: T[]): xs is NonEmpty<T> {
  return xs.length > 0;
}

function head<T>(xs: NonEmpty<T>): T {
  return xs[0];
}

function sum(xs: number[]): number {
  return xs.reduce((total, x) => total + x, 0);
}
```

非空列表由一个头元素加其余元素构成；偶数个元素用成对的集合表达。普通列表进入时，只收窄一次，让事实随类型传播。`head` 需要非空输入，或将返回类型改为 `T | undefined`；`sum([])` 有合理结果 0，就不必强化输入。

`!`、`arr[0] as T` 或“这不应该发生”的抛错，是使用点需要更强类型或更宽结果类型的信号。不要为了描述得最精确而强化一切。

时间区间可以用起点与时长表达，并在需要时计算终点。但普通 `number` 仍允许负数和非有限值；如果契约要求非负有限时长，就在边界校验这个约束。形状本身没有保证的性质，不能宣称已经由类型保证。

## 为语义不同的基本类型加标记

```ts
type UserId = string & { readonly __brand: "UserId" };
type OrderId = string & { readonly __brand: "OrderId" };

function parseUserId(input: unknown): UserId {
  if (typeof input !== "string" || input.length === 0) {
    throw new Error("UserId must be a non-empty string");
  }
  return input as UserId;
}
```

这里的契约仅是非空字符串；实际项目应校验其真实 ID 规格。断言只出现在已完成校验的构造边界。下游信任类型。`UserId` 与 `OrderId` 不应互换，采用 `readonly __brand` 形状，不要额外发明标记约定。只有语义混淆确有风险时才加标记。

## 外部数据先作为 unknown

RPC、JSON、IPC、文件、环境变量和数据库结果在解析前都视为 `unknown`。不要靠 `any` 或未经证明的 `as` 绕过边界。

优先寻找仓库已有的运行时模式定义与解析库，由同一模式负责校验并派生类型。不要同时手写模式、平行接口和一份可能漂移的守卫，也不要仅为一个守卫新添库依赖。失败属于正常分支时，使用已有解析库的非抛异常接口。

没有可复用解析器时，守卫必须验证其完整声明：

```ts
type User = { id: string; name: string };

function isUser(input: unknown): input is User {
  return typeof input === "object"
    && input !== null
    && "id" in input
    && typeof input.id === "string"
    && "name" in input
    && typeof input.name === "string";
}

function parseUser(input: unknown): User {
  if (!isUser(input)) throw new Error("Invalid user");
  return input;
}
```

只检查存在 `id` 却声称得到完整 `User` 的守卫，比裸断言更隐蔽。守卫使用 `isX` 或 `hasX` 命名。已有领域类型先于解析器存在时，应让解析器的类型约束检验它确实证明了该领域类型的所有字段。

收窄优先级是：判别字段的 switch／if、`in`、`typeof`／`instanceof`、用户定义的类型守卫，最后才是经过校验的 `as`。移除旧断言时，先找原因：缺少判别字段、来源类型过宽、边界未解析，或确实无法表达的语义。

## 让编译器检查穷尽性

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "rect"; width: number; height: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "rect":
      return shape.width * shape.height;
    default: {
      const exhaustive: never = shape;
      return exhaustive;
    }
  }
}
```

新增变体却不处理它，必须编译失败。返回值函数可以返回 `never` 绑定；无返回值的 switch 可以在 default 中使用 `void exhaustive`。不要用默认兜底掩盖遗漏。

## 校验值并从权威类型派生

```ts
type Config = { theme: "dark" | "light"; cols: number };
const config = { theme: "dark", cols: 3 } satisfies Config;

type UserSummary = Pick<User, "id" | "name">;
```

`satisfies` 校验表达式符合目标类型，同时保留适用的字面量信息；它不是运行时校验。已有协议、模式或生成类型定义数据形状时，从它们派生，不要另写可能漂移的接口。先考虑 `Pick`、`Omit`、`Parameters`、`ReturnType`、`Awaited` 和 `typeof`。

在边界把输入转为命名的领域类型，`Record<string, unknown>` 之类的宽泛输入表示应止于解析处。内部不重复校验。具体边界安排见[边界纪律](boundary-discipline.md)。
