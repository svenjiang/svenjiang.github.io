+++
date = '2026-05-15T14:55:06+08:00'
draft = false
title = 'Claude Code 深度讲解（七）：MCP 生态'
categories = ["AI 工程"]
tags = ["Claude Code"]
series = ["Claude Code 深度讲解"] 
+++
# 工具标准化：MCP 如何让 Agent 连接一切

Claude Code 的内置工具只是起点。Model Context Protocol（MCP）是一套开放标准，让任何外部服务都能以统一方式接入 Agent——从 GitHub 到数据库，从浏览器到 Slack。这一章讲清楚 MCP 的架构设计、集成实践，以及如何开发自定义 MCP server。

## MCP 架构图

{{< mcp-architecture-diagram >}}

## MCP 配置与调用演示

{{< mcp-config-demo >}}

---

## 7.1 MCP 解决的核心问题

在 MCP 出现之前，给 AI Agent 接入外部工具是一件碎片化的工作：每个工具有自己的 API 格式、认证方式、错误处理规范，每次集成都要重新造轮子。更糟的是，不同 Agent 框架之间的工具无法复用——为 Claude 写的集成代码，无法直接用在其他模型上。

MCP 的目标是成为"AI 工具的 USB-C"——一个统一的标准接口，工具只需实现一次，就能被所有支持 MCP 的 Agent 使用。

### 类比：为什么需要标准

想象没有 USB 标准的世界：每台设备有自己的接口，打印机用 A 口，鼠标用 B 口，显示器用 C 口。换一台电脑，所有外设都要重新适配。USB 的出现让"任何设备，任何电脑"成为可能。

MCP 对 AI 工具做了同样的事：定义了工具如何声明自己的能力、如何接收调用请求、如何返回结果。Claude Code 作为"电脑"，任何 MCP server 作为"外设"，二者通过标准协议通信，互不依赖具体实现。

---

## 7.2 架构设计：Server / Client 模型

MCP 采用标准的 Client-Server 架构，Claude Code 是 Client，外部工具是 Server。

### 三个核心概念

**Tools（工具）**

Server 声明它提供哪些可调用的函数。每个 Tool 有名称、描述、参数 schema（JSON Schema 格式）。Claude 看到这些声明后，就知道可以调用什么、需要传什么参数。

**Resources（资源）**

Server 可以暴露可读取的数据资源——文件、数据库记录、API 响应等。Resources 是"可以读的东西"，Tools 是"可以做的事情"，二者分工明确。

**Prompts（提示模板）**

Server 可以提供预定义的 prompt 模板，供 Agent 在特定场景下使用。比如 GitHub MCP server 可以提供"代码审查"的标准 prompt 模板。

### 两种传输方式

**stdio（本地进程）**

MCP server 作为本地子进程启动，通过标准输入输出与 Claude Code 通信。这是最常见的方式，延迟低，适合本地工具（文件系统操作、本地数据库、命令行工具封装）。

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"]
    }
  }
}
```

**SSE（远程服务）**

MCP server 作为独立的 HTTP 服务运行，通过 Server-Sent Events 与 Claude Code 通信。适合需要持久连接的远程服务（云端 API、团队共享工具）。

```json
{
  "mcpServers": {
    "github": {
      "type": "url",
      "url": "https://api.githubcopilot.com/mcp/v1"
    }
  }
}
```

### 工具发现与动态注册

Claude Code 启动时会连接所有配置的 MCP server，拉取它们声明的工具列表，将这些工具注入到当前会话的可用工具集里。Agent 在 Thought 阶段就能看到这些工具，并在需要时调用它们——与内置工具（Bash、Read、Write 等）完全平等，无需特殊处理。

---

## 7.3 配置与管理：三级作用域

MCP server 的配置遵循三级作用域，与 CLAUDE.md 的设计哲学一致——内层覆盖外层，越靠近项目越优先。

| 级别     | 配置文件位置                        | 作用范围               | 适合场景                           |
| -------- | ----------------------------------- | ---------------------- | ---------------------------------- |
| 用户全局 | `~/.claude/claude.json`             | 当前用户所有项目       | 个人常用工具（GitHub、本地数据库） |
| 项目级   | `.claude/claude.json`（项目根目录） | 当前 repo，随 git 提交 | 团队共享工具，写入版本控制         |
| 企业级   | 系统级策略配置                      | 组织内所有用户         | 强制接入的合规工具、内部 API       |

### 最小权限原则在 MCP 中的体现

每个 MCP server 只声明它实际需要的工具——不应该暴露"以防万一"的接口。Claude Code 在调用 MCP 工具前，同样遵循权限分级：读取类工具自动放行，写入类工具展示确认，不可逆操作强制确认。MCP 扩展的是工具集，不扩展权限规则。

---

## 7.4 MCP 生态：常见 server 一览

截至目前，MCP 生态已有数百个官方和社区维护的 server。以下是最常用的几类：

### 代码与版本控制

**GitHub MCP Server**

提供 PR 管理、Issue 操作、代码搜索、仓库内容读取等工具。让 Agent 能直接操作 GitHub，而不必通过 Bash 调用 `gh` CLI。

```bash
# 典型工具：
# - create_pull_request
# - list_issues
# - search_code
# - get_file_contents
```

**GitLab MCP Server**

类似 GitHub server，面向 GitLab 用户，支持 MR、CI pipeline 状态查询等。

### 数据库

**PostgreSQL / SQLite MCP Server**

让 Agent 直接查询数据库，读取 schema、执行 SELECT 查询。写操作通常被限制或需要额外确认，防止数据误修改。

### 浏览器自动化

**Puppeteer MCP Server**

让 Agent 控制无头浏览器，截图、填表、抓取动态渲染页面。对于需要操作 Web UI 的任务（E2E 测试、竞品信息采集）非常有用。

### 文件与知识库

**Filesystem MCP Server**（官方）

对特定目录提供更细粒度的文件操作权限控制。可以限制 Agent 只能操作某个目录，比内置的 Read/Write 工具有更严格的边界。

**Memory / Knowledge Graph Server**

为 Agent 提供持久化的键值存储或图结构知识库，突破单次对话的记忆限制。

### 团队协作

**Slack MCP Server**

读取频道消息、发送通知、搜索历史。让 Agent 能在完成任务后自动通知相关人员。

**Linear / Jira MCP Server**

Issue 追踪系统集成。Agent 可以创建 bug report、更新任务状态、关联 PR 与 Issue。

---

## 7.5 开发自定义 MCP Server

当现有生态里没有你需要的工具时，开发一个自定义 MCP server 是正确的选择。MCP SDK 目前支持 TypeScript 和 Python，实现一个基本的 server 只需要几十行代码。

### TypeScript 实现示例

以下是一个最简单的 MCP server，提供一个"查询内部 API"的工具：

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  { name: "internal-api", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// 声明工具
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "query_deployment_status",
      description: "查询指定服务的最新部署状态",
      inputSchema: {
        type: "object",
        properties: {
          service_name: {
            type: "string",
            description: "服务名称，如 api-gateway、user-service",
          },
          environment: {
            type: "string",
            enum: ["staging", "production"],
            description: "目标环境",
          },
        },
        required: ["service_name", "environment"],
      },
    },
  ],
}));

// 实现工具
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === "query_deployment_status") {
    const { service_name, environment } = request.params.arguments as {
      service_name: string;
      environment: string;
    };

    // 调用内部 API
    const response = await fetch(
      `https://internal.company.com/deploy/${environment}/${service_name}`
    );
    const data = await response.json();

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify(data, null, 2),
        },
      ],
    };
  }

  throw new Error(`Unknown tool: ${request.params.name}`);
});

// 启动
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Python 实现示例

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent
import json, httpx

server = Server("internal-api")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="query_deployment_status",
            description="查询指定服务的最新部署状态",
            inputSchema={
                "type": "object",
                "properties": {
                    "service_name": {"type": "string"},
                    "environment": {"type": "string", "enum": ["staging", "production"]},
                },
                "required": ["service_name", "environment"],
            },
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "query_deployment_status":
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"https://internal.company.com/deploy"
                f"/{arguments['environment']}/{arguments['service_name']}"
            )
        return [TextContent(type="text", text=json.dumps(resp.json(), indent=2))]
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

### 配置接入 Claude Code

```json
{
  "mcpServers": {
    "internal-api": {
      "command": "node",
      "args": ["/path/to/internal-api-server/dist/index.js"],
      "env": {
        "INTERNAL_API_TOKEN": "${INTERNAL_API_TOKEN}"
      }
    }
  }
}
```

### 设计自定义 server 的几条原则

**工具描述要精准**

Claude 通过 `description` 字段决定何时调用某个工具。描述模糊会导致工具被误用或被忽略。好的描述包含：工具做什么、什么时候该用、返回什么格式的数据。

**参数 schema 要严格**

用 `required` 字段标明必填参数，用 `enum` 限制枚举类型，用 `description` 说明每个参数的含义。schema 越清晰，Agent 调用时出错的概率越低。

**错误信息要可读**

当工具执行失败时，返回清晰的错误描述——包含出错原因和可能的修复建议。Agent 会把这个错误信息作为 Observation，用于决策下一步操作。

**保持工具粒度适中**

一个工具做一件事。不要把"查询+更新+删除"塞进同一个工具，也不要把一个操作拆成十个微步骤。参考内置工具的粒度：`Read` 读文件，`Bash` 执行命令——语义边界清晰，容易预测。

> **MCP 的长远意义**：它把"Agent 能做什么"从模型本身解耦出来。模型提供推理能力，MCP server 提供操作能力，两者独立演进。今天为 Claude Code 写的 MCP server，明天可以被任何支持 MCP 协议的 Agent 复用——这是真正意义上的工具生态。

---

## 小结

模块七的核心是"扩展性的标准化"：

- **MCP 解决的问题**：工具集成的碎片化，让每个工具只需实现一次
- **Server / Client 架构**：Claude Code 作为 Client，外部服务作为 Server，通过 stdio 或 SSE 通信
- **三级配置作用域**：用户全局、项目级、企业级，内层覆盖外层
- **丰富的生态**：GitHub、数据库、浏览器、协作工具，覆盖主要开发场景
- **自定义 server**：几十行代码即可接入任意内部系统

MCP 让 Claude Code 从"一个固定工具集的 Agent"变成了"可以连接一切的 Agent 平台"。