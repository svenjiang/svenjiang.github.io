+++
date = '2026-05-29T08:41:13+08:00'
draft = false
title = 'Claude Code 深度讲解（九）：评估观测'
categories = ["AI 工程"]
tags = ["Claude Code"]
series = ["Claude Code 深度讲解"] 
+++

# 怎么知道 Agent 做得好不好？

评估一个 Coding Agent 比评估一个问答模型难得多。单次问答有标准答案，但 Agent 的执行是一条轨迹——过程、中间决策、最终结果，三者都可能出问题。这一章讲清楚评估 Agent 的方法论，以及 Claude Code 内置了哪些可观测性工具。

## Agent 评估框架图

{{< evaluation-framework-diagram >}}

## 可观测性工具演示

{{< observability-tools-demo >}}

---

## 9.1 为什么 Agent 评估比问答评估难

传统 LLM 评估的范式很简单：给模型一个问题，对比它的答案和标准答案，计算准确率。这套方法对 Coding Agent 完全失效，原因有三。

### 原因一：输出不是文本，是行动序列

Agent 的"答案"不是一段话，而是一系列工具调用——读了哪些文件、执行了哪些命令、写入了什么代码。两个 Agent 可以用完全不同的路径到达同一个正确结果，也可以走相同的路径却在最后一步出错。单纯比较最终输出，丢失了大量有价值的信息。

### 原因二：任务是开放性的

"修复这个 bug"的正确解法可能有十种。哪种是"标准答案"？这个问题本身就没有答案。传统评估依赖的"ground truth"在开放性任务中难以定义。

### 原因三：非确定性

同一个 prompt，Claude Code 两次执行可能走不同的路径，调用不同的工具，产生不同但都正确的结果。这让"重复实验取平均"这种统计方法的成本极高——每次运行都要真实执行工具调用，代价昂贵。

> **核心困境**：真实评估需要真实执行，真实执行代价高昂，所以评估频率被迫降低；评估频率低，系统退化就难以被及时发现。这个循环是 Agent 工程化面临的根本挑战之一。

---

## 9.2 评估维度：轨迹、结果与效率

既然单一维度不够，就需要多维度评估。一个完整的 Agent 评估框架应该覆盖三个层次。

### 维度一：结果正确性

最终产物是否符合预期？

**硬性验证（可自动化）**

- 测试套件通过率：修复 bug 后，原有测试是否全部通过？有没有引入新的失败？
- 构建成功率：生成的代码能否正常编译和构建？
- Lint 通过率：代码风格是否符合项目规范？
- 功能验证：给定特定输入，输出是否正确？

**软性验证（需人工或 LLM-as-judge）**

- 代码质量：可读性、可维护性、是否遵循最佳实践？
- 完整性：任务的所有子要求是否都被满足？
- 副作用：是否有意外改动了不相关的文件或逻辑？

### 维度二：轨迹质量

从起点到终点走的路对不对？

**工具调用效率**

完成任务用了多少次工具调用？调用次数越少，通常意味着规划更准确、信息利用更高效。一个需要 40 轮才能完成的任务，如果另一个 Agent 用 12 轮就做到了，后者的轨迹质量更高。

**不必要的操作**

Agent 是否读了大量最终没有用到的文件？是否重复执行了相同的工具调用？是否尝试了错误方向并在浪费若干轮后才调头？

**安全性**

轨迹中是否包含了不必要的高风险操作？是否在应该询问用户的地方自行猜测？

### 维度三：效率与成本

| 指标           | 说明                                       |
| -------------- | ------------------------------------------ |
| Token 消耗     | Input + Output tokens，直接对应 API 成本   |
| 挂钟时间       | 从任务开始到完成的真实时间，含工具执行等待 |
| 工具调用次数   | 每次调用都有延迟和成本开销                 |
| 上下文压缩次数 | 压缩越少，信息损失越少，通常质量越高       |

效率维度在生产环境中尤其重要。一个在 CI 里每次 PR 都跑 Claude Code 的团队，如果单次任务消耗 50,000 tokens，累积成本会非常可观。

---

## 9.3 SWE-bench：行业基准的设计与局限

SWE-bench 是目前 Coding Agent 领域最广泛使用的评估基准，理解它的设计有助于理解行业对 Agent 能力的衡量标准。

### SWE-bench 的设计

SWE-bench 从 GitHub 上真实的开源项目（Django、scikit-learn、Flask 等）中收集 issue + pull request 对。每个样本包含：

- **任务描述**：原始 GitHub issue 的文字描述
- **代码库快照**：issue 提出时的 repo 状态
- **验证测试**：对应 PR 里包含的测试用例

Agent 的任务是：给定 issue 描述和代码库，生成能让验证测试通过的代码修改。评分标准是"测试通过率"。

### 为什么这个设计是合理的

**任务来自真实世界**：不是人工构造的玩具题，而是真实开源项目里发生过的 bug 修复和功能实现。

**验证是客观的**：测试通过/不通过是二值判断，没有主观评分的模糊性。

**覆盖范围广**：300+ 个不同的真实 repo，涵盖不同规模和不同类型的工程问题。

### SWE-bench 的局限

**分布偏差**：全部是 Python 项目，且以知名开源库为主。在其他语言、私有代码库中，基准的代表性有限。

**测试覆盖不完整**：通过测试不等于"正确修复"。如果原有测试本身不够全面，Agent 可能找到通过测试但本质上是错误的解法（Goodhart's Law：当测试变成目标，它就不再是好的衡量指标）。

**单次成功率掩盖了轨迹差异**：两个 Agent 都通过了 40% 的样本，但一个平均用了 8 次工具调用，另一个用了 35 次——SWE-bench 的单一分数看不出这种差异。

**不测试交互质量**：SWE-bench 是全自动化的，无法评估 Agent 在何时提问、如何处理歧义上的质量。

---

## 9.4 Claude Code 内置的可观测性工具

评估不只是跑 benchmark，更重要的是在日常使用中持续观察 Agent 的行为。

### --verbose：完整执行轨迹

`--verbose`（或 `-v`）模式输出每一轮 ReAct 循环的完整内容：

```bash
claude --verbose "fix the failing tests"
```

输出内容包含：

- 每轮的完整 Thought 推理文本
- 每次工具调用的名称和完整参数
- 每次工具返回的完整 Observation
- 每轮消耗的 token 数量

这是理解"Agent 为什么这么做"的最直接工具。当 Agent 做出了出乎意料的决策，加上 `--verbose` 重跑，通常能在 Thought 里找到原因。

### Token 消耗追踪

每次会话结束后，Claude Code 输出本次消耗的 token 统计：

```
Session complete.
Input tokens:  12,847
Output tokens: 1,203
Cache read:    8,420
Total cost:    ~$0.087
```

如果某类任务的 token 消耗异常高，通常意味着：上下文里加载了太多无关内容，或者 Agent 走了很多弯路。

### --output-format json：程序化处理

在 CI 场景里，`--output-format json` 让监控系统能捕获每次执行的结构化数据：

```json
{
  "result": "Fixed 2 failing tests in src/auth/token.ts",
  "cost": {
    "input_tokens": 12847,
    "output_tokens": 1203,
    "cache_read_tokens": 8420
  },
  "duration_ms": 34521,
  "session_id": "sess_01XYZ",
  "num_turns": 8
}
```

把这些数据写入时序数据库，就能建立 Agent 性能的长期趋势监控——token 消耗是否在增加？特定类型任务的成功率是否在下降？

### 执行日志与审计

Claude Code 在 `~/.claude/logs/` 下保留完整的执行日志，包含所有工具调用和返回值：

```bash
# 查看最近一次会话的完整日志
cat ~/.claude/logs/$(ls -t ~/.claude/logs/ | head -1)
```

---

## 9.5 测试 Agent 的方法论

### 分类：确定性任务 vs 开放性任务

**确定性任务**（有明确的正确答案）

```python
# 示例：测试 Agent 能否正确实现一个排序函数
def test_agent_sorting():
    result = run_agent("实现归并排序，签名为 merge_sort(arr: list) -> list")
    assert merge_sort([3,1,4,1,5]) == [1,1,3,4,5]
    assert merge_sort([]) == []
```

这类测试可以完全自动化，适合作为 regression test。

**开放性任务**（没有唯一正确解法）

```python
# 示例：测试 Agent 能否合理重构代码
def test_agent_refactor():
    result = run_agent("重构 UserService，提取共用的数据库操作到基类")
    # 验证结构性要求，而不是具体实现
    assert file_exists("src/services/base.py")
    assert class_inherits("UserService", "BaseService")
    assert all_tests_pass()
    # 人工审查代码质量（不可完全自动化）
```

### 回归测试：测试语义，不测试文本

Agent 的非确定性让传统回归测试面临困境：同一个任务，两次运行可能产生不同但都正确的代码。

```python
# ❌ 错误做法：比较输出字符串
assert agent_output == expected_output  # 脆弱，几乎总会失败

# ✅ 正确做法：验证语义属性
assert all_tests_pass()                       # 功能正确
assert no_new_lint_errors()                   # 代码质量
assert no_unrelated_files_modified()          # 无副作用
assert git_diff_touches_only(["src/auth/"])   # 改动范围符合预期
```

### 用 Claude Code 测试 Claude Code（自举测试）

让 Claude Code 自己生成测试用例，然后用这些测试用例评估 Claude Code 的输出。在文档生成、代码审查等"输出难以客观评估"的任务中特别有用：

```bash
# 步骤一：让 Agent 生成评估标准
claude "给'实现用户登录 API'这个任务，
       生成一个包含 10 个验证点的评估检查表，
       每个点应该是可以程序化验证的"

# 步骤二：让 Agent 完成任务
claude "实现用户登录 API，需要支持邮箱+密码认证，返回 JWT"

# 步骤三：用步骤一的检查表评估步骤二的输出
claude "根据这个检查表，评估刚才实现的登录 API：[检查表内容]"
```

### 建立 Golden Dataset

对于核心任务场景，建立一个"黄金数据集"——精心设计的测试任务，包含明确的验收标准。每次模型升级或 CLAUDE.md 修改后，跑一遍 golden dataset，对比新旧结果：

```
golden_dataset/
├── task_001_fix_null_pointer/
│   ├── setup.sh          # 准备测试环境
│   ├── prompt.txt        # 任务描述
│   ├── validate.sh       # 验证脚本
│   └── expected_diff.md  # 预期改动范围（参考，非精确匹配）
├── task_002_add_api_endpoint/
│   └── ...
└── task_003_refactor_service/
    └── ...
```

> **评估的本质是反馈循环**。SWE-bench 告诉你模型的上限在哪里；`--verbose` 告诉你某次具体执行哪里出了问题；golden dataset 告诉你你的配置（CLAUDE.md、prompt 工程）是变好了还是变差了。三者结合，才能对 Agent 的真实能力有清醒的认知。

---

## 小结

模块九的核心是"用数据说话，而不是凭感觉"：

- **评估困难的根源**：Agent 输出是行动序列，任务是开放性的，执行是非确定性的
- **三个评估维度**：结果正确性、轨迹质量、成本效率
- **SWE-bench**：行业基准，有代表性但有偏差，通过率不是唯一指标
- **内置可观测性**：`--verbose`、token 追踪、JSON 输出、执行日志
- **测试方法论**：测试语义而非文本、golden dataset、自举测试

---

