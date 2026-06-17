+++
date = '2026-04-10T11:14:39+08:00'
draft = false
title = 'Claude Code 深度讲解（二）：核心架构'
categories = ["AI 工程"]
tags = ["Claude Code"]
series = ["Claude Code 深度讲解"] 
+++
# Agent 在内部是怎么"思考和行动"的？

拆开 Claude Code 的引擎盖，看清 ReAct 循环、工具系统、上下文窗口管理三层机制如何协同工作。

## 2.1 ReAct 循环：Agent 的心跳

如果你只能理解 Agent 的一个概念，那就是 ReAct 循环。它是整个系统运转的核心节律——每一次"呼吸"都包含三个阶段：**Reason（推理）、Act（行动）、Observe（观察）**。

```
用户输入
    ↓
[Thought] 理解目标，规划下一步操作
    ↓
[Action]  调用工具：Bash / Read / Write…
    ↓
[Observation] 工具返回结果，写入上下文
    ↓
完成？ → 是 → 输出结果
  ↓ 否
返回 [Thought]（继续迭代）
```

### 真实执行轨迹示例

以"帮我找出所有未处理的 TODO 注释并整理成报告"为例：

**第 1 轮**

- `Thought`：用户要找 TODO 注释。需要先了解项目结构，知道有哪些源文件。
- `Action`：`Bash: find . -type f -name "*.py" -o -name "*.ts" | head -50`
- `Observation`：返回 43 个文件路径，主要在 src/、tests/、scripts/ 三个目录下。

**第 2 轮**

- `Thought`：用 grep 在所有源文件里搜索 TODO/FIXME/HACK 关键词，比逐文件读取高效得多。
- `Action`：`Bash: grep -rn "TODO\|FIXME\|HACK" src/ tests/ --include="*.py"`
- `Observation`：找到 17 条匹配，包含文件名、行号、注释内容。

**第 3 轮**

- `Thought`：17 条较多，应该分类整理：按优先级（FIXME 最紧急）和模块分组，才有实用价值。
- `Action`：`Read: src/auth/login.py`（验证某条 FIXME 的上下文）
- `Observation`：确认该 FIXME 与 JWT 刷新逻辑有关，属于安全相关问题，优先级高。

**第 4 轮**

- `Thought`：信息已足够，生成报告。用 Markdown 格式，按 FIXME / TODO / HACK 三类分组，每条附文件链接。
- `Action`：`Write: TODO_report.md`
- `Observation`：文件写入成功。任务完成，向用户汇报。

> **ReAct 的精髓不是"循环"本身，而是每个 Observation 都在修改 Agent 下一次 Thought 的内容。这让 Agent 能从错误中恢复、根据实际情况调整策略，而不是机械执行预设脚本。**

### 循环什么时候终止？

终止条件有三种：

1. Agent 判断任务已完成
2. 达到工具调用次数上限（防止无限循环）
3. 遇到无法自行解决的歧义，主动向用户提问（Minimal Footprint 原则的体现）

---

## 2.2 工具系统：Agent 的双手

### 四类工具及其设计逻辑

**文件操作类**：`Read` / `Write` / `Edit` / `MultiEdit`

注意有专门的 `Edit` 工具——它做的是精准的字符串替换，而不是整个文件重写。这避免了大文件时"读全文 → 改一行 → 写全文"的巨大浪费。

**代码执行类**：`Bash`

核心就是 `Bash`，支持任意 shell 命令。这一个工具解锁了整个操作系统：git、npm、pytest、docker……任何已安装工具都可驱动。内置超时机制防止挂起。

**搜索探索类**：`Glob` / `Grep` / `LS`

三者组合是理解代码库的核心手段，让 Agent 按需精准取用信息，而非全量加载。

**网络与协作类**：`WebFetch` / `WebSearch` / `Task`

`Task` 工具是多 Agent 协作的基础设施——将子任务委托给独立 Agent 实例并行执行。

### 工具粒度的设计权衡

- **粒度太粗**（只有一个"执行任意操作"工具）：权限控制无法精细化，安全性差，也无法针对不同工具设置不同的确认策略。
- **粒度太细**（ReadLine、ReadRange、ReadFile 三个不同的读取工具）：Agent 每次调用前都要做元决策"该用哪个工具"，消耗推理资源，也增加出错概率。

> **工具粒度的设计原则：以"操作的语义边界"为单位，而不是以"技术实现"为单位。`Read` 就是读一个文件，`Bash` 就是执行一条命令。简单直觉，容易预测。**

### 工具调用在架构上的结构

每次工具调用对 Claude 来说是一次结构化的 JSON 输出：

```json
{
  "tool": "Bash",
  "input": {
    "command": "pytest tests/ -x --tb=short",
    "timeout": 30000
  }
}
```

模型输出这段 JSON，运行时环境真正执行命令，把 stdout/stderr 作为 Observation 插回 conversation history。对模型来说，工具调用和普通对话轮次在结构上是对称的——都是"我说一句，你回一句"，只不过"你"是真实的系统而不是人。

---

## 2.3 上下文窗口管理：Agent 的工作记忆

### 上下文里装着什么

| 内容           | 占比（估算） | 说明                                            |
| -------------- | ------------ | ----------------------------------------------- |
| System prompt  | ~8%          | Claude Code 的行为规则、工具定义                |
| CLAUDE.md      | ~6%          | 项目记忆（命令、约定、架构说明）                |
| 对话历史       | ~50%         | Thought + Action + Observation 累积，随任务增长 |
| 当前读入的文件 | ~15%         | 按需读取                                        |
| 余量           | ~21%         | 留给下一轮生成                                  |

对话历史是上下文消耗的最大变量。每轮的 `Bash` 输出如果很长（比如一次 test run 输出了几百行），会急速压缩可用空间。

### 两种压缩策略

**自动压缩**：Claude 主动总结历史对话，用摘要替换原始内容，保留任务目标和关键 Observation，丢弃中间过程的细节。

**`/compact` 命令**：用户手动触发，效果比自动压缩更彻底。适合在大任务的阶段性完成点使用。

> **压缩是有代价的**：被压缩掉的细节无法找回。如果某个 Observation 里包含后续任务需要的信息，被压缩后 Agent 可能会重复执行相同的工具调用来"重新发现"它。这是当前架构的根本局限。

### 如何读代码库而不溢出

Agent 不会"先把所有文件读进来再思考"。实际策略更像一个经验丰富的工程师接手新项目：

```
先看目录结构（LS）
→ 读 README 和 package.json
→ 搜索入口文件（Glob）
→ 按需 Grep 关键符号
```

始终是"拉取（pull）"而不是"推送（push）"——需要什么才加载什么。

---

## 2.4 多轮对话状态管理：任务不漂移

### 任务目标如何被保持

用户最初的 prompt 会被保留在 conversation history 的最顶部，永远不会被压缩掉。这是一个简单但有效的锚点——每轮 Thought 阶段，模型都能回头看"我要做什么"。

### Session 持久化

- `--continue`：自动接续上次未完成的对话
- `--resume`：列出历史会话供选择恢复

底层是将 conversation history 序列化到磁盘，重启时重新加载。

> **实用建议**：对于非常长的任务，定期用 `/compact` 整理 + 写一段文字总结当前状态到 CLAUDE.md，比单纯依赖 history 持久化效果更稳定。恢复的 history 已经是"冷"的——模型对其中细节的"熟悉度"不如刚执行时高。
