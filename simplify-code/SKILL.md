---
name: "simplify-code"
description: "Reviews changed files for reuse, code quality, and efficiency, then fixes concrete issues."
---

# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency. Fix any issues found.

## Phase 1: Identify Changes

Run `git diff` or `git diff HEAD` if there are staged changes. If there are no git changes, review the most recently modified files that the user mentioned or that you edited earlier in this conversation.

## Phase 2: Launch Three Review Agents In Parallel

Launch all three agents concurrently. Pass each agent the full diff so each one sees the same context.

### Agent 1: Code Reuse Review

For each change:

1. Search for existing utilities and helpers that could replace newly written code.
2. Flag any new function that duplicates existing functionality and point to the existing function.
3. Flag inline logic that should use an existing utility instead of hand-rolled behavior.

### Agent 2: Code Quality Review

Review the same changes for:

1. redundant state
2. parameter sprawl
3. copy-paste with slight variation
4. leaky abstractions
5. stringly-typed code
6. unnecessary JSX nesting
7. unnecessary comments

### Agent 3: Efficiency Review

Review the same changes for:

1. unnecessary work
2. missed concurrency
3. hot-path bloat
4. recurring no-op updates
5. unnecessary existence checks
6. memory issues
7. overly broad operations

## Phase 3: Fix Issues

Wait for all three agents to complete. Aggregate their findings and fix each issue directly.

If a finding is a false positive or not worth addressing, note it briefly and move on.

When done, briefly summarize what was fixed, or confirm the code was already clean.
