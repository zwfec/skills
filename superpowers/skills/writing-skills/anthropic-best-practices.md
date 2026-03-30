# 技能编写最佳实践

> 学习如何编写 Claude 可以成功发现和使用的有效技能。

好的技能是简洁、结构良好，并经过实际使用测试的。本指南提供实用的编写决策，帮助你编写 Claude 可以有效发现和使用的技能。

有关技能如何工作的概念背景，请参阅 [技能概述](/en/docs/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows) 是公共资源。你的技能与 Claude 需要知道的其他所有内容共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他技能的元数据
* 你的实际请求

并非技能中的每个令牌都有立即成本。启动时，只预加载所有技能的元数据（名称和描述）。Claude 只在技能变得相关时读取 SKILL.md，并只在需要时读取其他文件。然而，在 SKILL.md 中保持简洁仍然很重要：一旦 Claude 加载它，每个令牌都与对话历史和其他上下文竞争。

**默认假设**：Claude 已经非常聪明

只添加 Claude 还没有的上下文。挑战每条信息：

* "Claude 真的需要这个解释吗？"
* "我可以假设 Claude 知道这个吗？"
* "这段话是否证明其令牌成本合理？"

**好的例子：简洁**（约 50 个令牌）：

```markdown
## 提取 PDF 文本

使用 pdfplumber 进行文本提取：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
```

**坏的例子：冗长**（约 150 个令牌）：

```markdown
## 从 PDF 文档中提取文本内容

PDF 文件广泛用于文档分发，但从中提取文本可能具有挑战性。本节解释如何使用 pdfplumber 库从 PDF 文件中提取文本内容。

pdfplumber 是一个强大的 Python 库，允许你从 PDF 文件中提取文本、表格和其他内容。它是建立在 pdfminer.six 之上的，并提供更友好的 API。

要使用 pdfplumber，首先需要安装它：

```python
import pdfplumber

# 打开 PDF 文件
with pdfplumber.open("file.pdf") as pdf:
    # 从第一页提取文本
    text = pdf.pages[0].extract_text()
    print(text)
```
```

坏的例子提供了 Claude 已经知道或可以推断的上下文。好的例子直接进入有用的内容。

### 遵循标准结构

一致的技能结构有助于 Claude 高效地找到信息。使用此标准结构：

```markdown
---
name: skill-name
description: Use when [triggering conditions]
---

# 技能名称

## 概述
[1-2 句核心原则]

## 何时使用
[触发条件]

## [特定于技能的部分]
[主要内容]

## 常见错误
[要避免什么]
```

**为什么此结构有效：**
* **概述** 在详细说明前建立心智模型
* **何时使用** 帮助 Claude 决定是否加载技能
* **主体** 提供可操作内容
* **常见错误** 防止误用

### 触发时机，而非实现细节

描述中的 `description` 字段是 Claude 用来决定是否加载技能的主要信号。它应该回答"何时使用此技能？"而不是"此技能做什么？"。

**好的描述示例：**
* `description: Use when working with PDF files and need to extract text, tables, or images`
* `description: Use when implementing authentication flows that need to handle token refresh`
* `description: Use when debugging race conditions in async code`

**坏的描述示例：**
* `description: A skill for working with PDF files`（太模糊）
* `description: Implements OAuth 2.0 token refresh logic`（描述实现，不是触发时机）
* `description: Help with async debugging`（缺少具体触发器）

## 编写 SKILL.md

### 概述部分

概述应在 1-2 句话中建立核心心智模型。不要解释背景 - 直接陈述原则。

**好的概述：**
> 在尝试修复之前始终找到根本原因。症状修复是失败。

**坏的概述：**
> 调试是软件开发的关键部分。当代码行为不正确时，你需要系统地调查以找到根本原因。本技能提供调试工作流。

### 何时使用部分

要具体但简洁。使用项目符号列出触发条件。

**好的例子：**
```markdown
## 何时使用

**使用时机：**
- 测试失败且原因不明显
- 生产中出现意外行为
- 需要找到哪个代码路径触发问题

**不使用时机：**
- 错误消息已经告诉你确切问题
- 只是运行命令查看结果
```

### 主体内容

专注于 Claude 不会知道的内容：

**包含：**
* 项目或组织特定的约定
* 经过验证的特定于领域的模式
* 从经验中学到的常见陷阱
* 此上下文独有的决策

**排除：**
* 通用编程知识
* Claude 可以从代码推断的内容
* 重复文档的背景解释
* "教程式"介绍

### 示例

好的示例展示具体用法，没有不必要的设置：

**好的例子：**
```typescript
// 等待条件而非猜测时序
await waitFor(() => machine.state === 'ready');
```

**坏的例子：**
```typescript
// 首先，导入必要的模块
import { waitFor } from './helpers';

// 然后，定义你的测试
test('machine becomes ready', async () => {
  // 创建机器实例
  const machine = new Machine();

  // 启动机器
  machine.start();

  // 等待它准备好
  await waitFor(() => machine.state === 'ready');

  // 断言它已准备好
  expect(machine.state).toBe('ready');
});
```

### 常见错误部分

这是最有价值的部分之一。列出你从经验中学到的具体陷阱：

**好的例子：**
```markdown
## 常见错误

**❌ 轮询太快：** `setTimeout(check, 1)` - 浪费 CPU
**✅ 修复：** 每 10ms 轮询

**❌ 没有超时：** 如果条件从未满足则永远循环
**✅ 修复：** 始终包含带清晰错误的超时
```

## 元数据最佳实践

### 名称字段

* 使用小写字母、数字和连字符
* 要描述性但简洁
* 使用动词-名词模式：`extract-pdf-text`、`debug-race-conditions`

### 描述字段

* 以 "Use when" 开始
* 保持在一行（通常 < 100 个字符）
* 专注于触发条件
* 具体但简洁

## 测试技能

好的技能经过实际使用测试。当你编写技能时：

1. **先编写技能** - 文档你学到的东西
2. **在新上下文中测试** - 在新会话中尝试
3. **观察 Claude 是否找到它** - 检查是否在适当时加载
4. **观察 Claude 是否遵循它** - 检查是否正确应用
5. **迭代** - 根据观察优化

**常见问题：**
* 技能太长 → Claude 跳过部分
* 描述太模糊 → Claude 不加载
* 示例太复杂 → Claude 复制不必要的设置
* 缺少触发器 → Claude 不知道何时使用

## 文件组织

### 何时拆分文件

**保持在一个文件：**
* 概念适合 < 500 行
* 所有内容都相关于核心概念
* Claude 通常一次需要所有内容

**拆分为多个文件：**
* 参考材料（API 文档、配置选项）
* 长示例或模板
* 不同用例的独立部分

### 目录结构

```
skill-name/
  SKILL.md           # 主技能文件
  reference.md       # 可选：详细参考
  examples/          # 可选：示例文件
    example1.md
    example2.md
```

## 关键要点

1. **简洁为王** - 每个令牌都与其他上下文竞争
2. **触发时机，而非实现** - 描述何时使用，不是如何工作
3. **结构一致** - 帮助 Claude 高效导航
4. **测试实际使用** - 真实使用揭示问题
5. **专注于独特价值** - Claude 不知道的内容
6. **迭代优化** - 好的技能从经验演进

## 示例技能结构

```markdown
---
name: example-skill
description: Use when [specific triggering conditions]
---

# 示例技能

## 概述
[核心原则 1-2 句]

## 何时使用
[触发条件]

## 核心模式
[主要内容 - 保持简洁]

## 快速参考
[表格或项目符号用于常见操作]

## 常见错误
[要避免什么]
```

记住：好的技能是 Claude 实际使用并发现有帮助的技能。当有疑问时，用新上下文测试并观察。
