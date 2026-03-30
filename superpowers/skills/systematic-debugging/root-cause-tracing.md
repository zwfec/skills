# 根本原因追踪

## 概述

错误通常在调用栈深处表现出来（在错误目录中 git init、在错误位置创建文件、用错误路径打开数据库）。你的本能是在错误出现的地方修复，但那是治疗症状。

**核心原则：** 向后通过调用链追踪直到找到原始触发器，然后在源头修复。

## 何时使用

```dot
digraph when_to_use {
    "错误出现在栈深处?" [shape=diamond];
    "可以向后追踪?" [shape=diamond];
    "在症状点修复" [shape=box];
    "追踪到原始触发器" [shape=box];
    "更好：也添加深度防御" [shape=box];

    "错误出现在栈深处?" -> "可以向后追踪?" [label="是"];
    "可以向后追踪?" -> "追踪到原始触发器" [label="是"];
    "可以向后追踪?" -> "在症状点修复" [label="否 - 死胡同"];
    "追踪到原始触发器" -> "更好：也添加深度防御";
}
```

**使用时机：**
- 错误在执行深处发生（不在入口点）
- 堆栈跟踪显示长调用链
- 不清楚无效数据从哪里来
- 需要找到哪个测试/代码触发问题

## 追踪流程

### 1. 观察症状
```
错误：git init 在 /Users/jesse/project/packages/core 中失败
```

### 2. 找到直接原因
**什么代码直接导致这个？**
```typescript
await execFileAsync('git', ['init'], { cwd: projectDir });
```

### 3. 问：谁调用了这个？
```typescript
WorktreeManager.createSessionWorktree(projectDir, sessionId)
  → 被 Session.initializeWorkspace() 调用
  → 被 Session.create() 调用
  → 被 Project.create() 的测试调用
```

### 4. 继续向上追踪
**传递了什么值？**
- `projectDir = ''`（空字符串！）
- 空字符串作为 `cwd` 解析为 `process.cwd()`
- 那是源代码目录！

### 5. 找到原始触发器
**空字符串从哪里来？**
```typescript
const context = setupCoreTest(); // 返回 { tempDir: '' }
Project.create('name', context.tempDir); // 在 beforeEach 之前访问！
```

## 添加堆栈跟踪

当你无法手动追踪时，添加仪器：

```typescript
// 在有问题的操作之前
async function gitInit(directory: string) {
  const stack = new Error().stack;
  console.error('DEBUG git init:', {
    directory,
    cwd: process.cwd(),
    nodeEnv: process.env.NODE_ENV,
    stack,
  });

  await execFileAsync('git', ['init'], { cwd: directory });
}
```

**关键：** 在测试中使用 `console.error()`（不是 logger - 可能不显示）

**运行并捕获：**
```bash
npm test 2>&1 | grep 'DEBUG git init'
```

**分析堆栈跟踪：**
- 查找测试文件名
- 找到触发调用的行号
- 识别模式（同一个测试？同一个参数？）

## 查找哪个测试导致污染

如果在测试期间出现某些东西但你不知道是哪个测试：

使用此目录中的二分脚本 `find-polluter.sh`：

```bash
./find-polluter.sh '.git' 'src/**/*.test.ts'
```

逐个运行测试，在第一个污染者处停止。参见脚本了解用法。

## 真实示例：空 projectDir

**症状：** `.git` 在 `packages/core/` 中创建（源代码）

**追踪链：**
1. `git init` 在 `process.cwd()` 中运行 ← 空 cwd 参数
2. WorktreeManager 被调用时 projectDir 为空
3. Session.create() 传递了空字符串
4. 测试在 beforeEach 之前访问了 `context.tempDir`
5. setupCoreTest() 最初返回 `{ tempDir: '' }`

**根本原因：** 顶层变量初始化访问空值

**修复：** 使 tempDir 成为 getter，如果在 beforeEach 之前访问则抛出错误

**还添加了深度防御：**
- 第1层：Project.create() 验证目录
- 第2层：WorkspaceManager 验证不为空
- 第3层：NODE_ENV 守卫拒绝在 tmpdir 之外的 git init
- 第4层：git init 前的堆栈跟踪日志

## 关键原则

```dot
digraph principle {
    "找到直接原因" [shape=ellipse];
    "可以向上追踪一级?" [shape=diamond];
    "向后追踪" [shape=box];
    "这是源头吗?" [shape=diamond];
    "在源头修复" [shape=box];
    "在每层添加验证" [shape=box];
    "错误不可能" [shape=doublecircle];
    "永远不要只修复症状" [shape=octagon, style=filled, fillcolor=red, fontcolor=white];

    "找到直接原因" -> "可以向上追踪一级?";
    "可以向上追踪一级?" -> "向后追踪" [label="是"];
    "可以向上追踪一级?" -> "永远不要只修复症状" [label="否"];
    "向后追踪" -> "这是源头吗?";
    "这是源头吗?" -> "向后追踪" [label="否 - 继续"];
    "这是源头吗?" -> "在源头修复" [label="是"];
    "在源头修复" -> "在每层添加验证";
    "在每层添加验证" -> "错误不可能";
}
```

**永远不要只修复错误出现的地方。** 追溯找到原始触发器。

## 堆栈跟踪技巧

**在测试中：** 使用 `console.error()` 不是 logger - logger 可能被抑制
**操作前：** 在危险操作前记录，而不是在它失败后
**包含上下文：** 目录、cwd、环境变量、时间戳
**捕获堆栈：** `new Error().stack` 显示完整调用链

## 现实世界影响

来自调试会话（2025-10-03）：
- 通过5级追踪找到根本原因
- 在源头修复（getter 验证）
- 添加了4层防御
- 1847 个测试通过，零污染
