# Agent 工具调用与协作生态面试题

## 1. 什么是 Function Calling？它的工作流程是什么？

Function Calling（函数调用）是一种让 LLM 以**结构化方式选择并调用外部工具**的机制。开发者先向模型描述可用函数的名称、用途和参数结构，模型根据用户意图判断是否需要调用函数；如果需要，模型会生成函数名和符合约定格式的参数，再由宿主程序执行真正的函数。

需要特别注意：**LLM 本身通常不会直接执行函数。**它负责理解意图并生成调用请求，真正访问数据库、调用 API、运行代码或修改外部系统的是 Agent、应用程序或工具运行时。

一个典型的 Function Calling 流程如下：

1. **定义函数**：开发者使用 JSON Schema 等方式描述函数名、功能、参数类型、必填字段和取值约束。
2. **发送请求**：应用将用户问题、对话上下文和可用函数定义一起发送给 LLM。
3. **选择函数**：LLM 判断是直接回答，还是生成一个或多个 Function Call。
4. **生成参数**：LLM 按照函数 Schema 输出函数名和结构化参数。
5. **校验与执行**：宿主程序校验参数、权限和调用风险，然后执行本地函数或外部 API。
6. **返回结果**：应用将函数执行结果作为 Tool Result 放回对话上下文。
7. **生成最终回答**：LLM 根据用户问题和工具返回值，整理出自然语言答案；如果信息不足，也可以继续调用其他函数。

完整过程可以概括为：

**用户请求 → LLM 选择函数并生成参数 → 应用校验并执行函数 → 执行结果返回 LLM → LLM 生成最终答案。**

例如，系统向模型提供一个查询天气的函数：

```json
{
  "name": "get_weather",
  "description": "查询指定城市的天气",
  "parameters": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称"
      },
      "date": {
        "type": "string",
        "description": "日期，格式为 YYYY-MM-DD"
      }
    },
    "required": ["city", "date"]
  }
}
```

当用户问“明天北京天气怎么样”时，模型可能生成：

```json
{
  "name": "get_weather",
  "arguments": {
    "city": "北京",
    "date": "2026-07-20"
  }
}
```

应用收到调用请求后，校验日期和城市参数，调用真实天气服务，再把查询结果返回给模型。模型最后生成“北京明天有小雨，最高气温 30℃”之类的自然语言回答。

Function Calling 与普通文本生成的主要区别是：普通文本生成只返回自然语言，格式可能不稳定；Function Calling 通过函数定义和参数 Schema 约束模型输出，更方便程序解析并连接外部系统。但是，Schema 只能提高结构正确率，不能保证参数在业务上一定正确，也不能保证工具结果可信。

实际使用时需要注意：

- **参数校验**：不能直接信任模型生成的参数，应检查类型、范围、格式和必填项。
- **权限控制**：根据当前用户身份限制可见工具及可操作的数据，不能让模型绕过鉴权。
- **高风险操作确认**：转账、删除、发送消息等有副作用的操作，应进行二次确认或审批。
- **错误与超时处理**：工具可能失败、超时或返回空结果，需要设置重试、降级和最大调用次数。
- **提示注入防护**：外部内容和工具返回值都可能包含恶意指令，不能把它们当成高优先级系统指令。
- **幂等性**：对于创建订单、付款等写操作，应使用幂等键，避免重试导致重复执行。
- **可观测性**：记录函数选择、参数、执行结果、耗时和错误，便于审计与问题定位，同时注意脱敏。

Function Calling 是很多 Tool-Using Agent 的基础，但二者不完全等价：**Function Calling 解决“如何把模型的工具调用意图变成结构化请求”，Agent 则进一步负责规划、记忆、循环执行、错误恢复和完成条件判断。**

简单来说：**Function Calling 让 LLM 决定“调用哪个函数、传什么参数”，由应用程序负责“安全地执行函数并返回结果”；它是模型连接外部数据和真实业务能力的重要桥梁。**

## 2. 模型是如何学会工具调用的？

模型的工具调用能力通常不是由某一个环节单独产生的，而是由**预训练、工具调用数据微调、偏好对齐和运行时约束**共同形成的。

### 1. 预训练提供基础理解能力

LLM 在大规模文本和代码上进行预训练后，已经具备了一些工具调用所需的基础能力，例如：

- 理解自然语言指令和函数说明；
- 理解 API 文档、代码、JSON 和参数类型；
- 从用户问题中提取实体、时间、地点等参数；
- 根据上下文推断下一步应该做什么。

但只经过预训练的模型通常不能稳定遵守特定的 Function Calling 协议，也可能输出格式错误的 JSON，因此还需要针对工具使用进行训练和对齐。

### 2. 监督微调让模型学习调用格式和流程

训练数据中会加入大量工具调用样例。一个典型样例包含：

1. 可用工具的名称、描述和参数 Schema；
2. 用户提出的问题；
3. 模型应该直接回答还是调用工具；
4. 如果调用工具，应该选择哪个工具并生成哪些参数；
5. 工具执行结果；
6. 模型如何根据工具结果生成最终答案。

例如：

```text
工具：get_weather(city, date)
用户：明天北京天气怎么样？
模型：调用 get_weather(city="北京", date="2026-07-20")
工具：小雨，最高气温 30℃
模型：北京明天有小雨，最高气温 30℃。
```

通过大量类似数据，模型逐渐学会：**什么时候需要工具、选择哪个工具、如何填写参数，以及如何使用工具结果回答问题。**

训练数据除了人工标注，也可以来自真实调用日志、规则生成和模型合成，再经过程序校验或人工筛选。为了提高鲁棒性，数据中还会包含缺少参数、多个相似工具、并行调用、工具报错、结果为空和需要重试等情况。

### 3. 偏好对齐让模型学会更合理地使用工具

仅仅学会生成调用格式还不够，模型还需要判断什么时候**不应该**调用工具。例如，用户问“1 加 1 等于多少”时通常不需要搜索；用户要求查询实时天气时则不应该依赖模型记忆直接回答。

训练阶段可以使用人工反馈、AI 反馈、偏好优化或强化学习，让模型倾向于：

- 在需要外部实时数据、私有数据或实际操作时调用工具；
- 能直接回答时避免不必要的调用；
- 在信息不足时先询问用户，而不是编造参数；
- 工具失败时解释问题、重试或选择替代工具；
- 对高风险操作请求确认，而不是直接执行。

### 4. 推理时动态提供具体工具

模型通常不需要在训练阶段见过业务中的每一个具体工具。应用在推理时会把当前可用工具的名称、描述和参数 Schema 放入模型上下文，模型根据训练得到的通用工具使用能力理解这些新工具。

例如，模型训练时可能没有见过公司内部的 `query_employee_leave_balance`，但如果工具描述清楚地说明“查询员工剩余年假”，并定义好 `employee_id` 和 `year` 参数，模型仍然可能正确使用它。

因此，模型真正学到的不是一张固定的工具列表，而是：

**阅读工具说明 → 将用户意图与工具能力匹配 → 按 Schema 生成调用参数 → 根据返回结果继续处理。**

这是一种基于上下文的泛化能力。不过，如果工具名称含糊、描述不完整、多个工具功能高度重叠，模型的选择准确率仍会下降。

### 5. 结构化输出约束提高格式可靠性

有些系统还会在解码阶段使用 JSON Schema、Grammar 或 Constrained Decoding，限制模型只能生成符合格式要求的函数名和参数。这可以减少括号缺失、字段拼错和类型错误。

但需要区分：

- **训练和对齐**解决的是模型是否知道“何时调用、调用什么、参数语义是什么”；
- **结构化解码**解决的是输出能否满足语法和 Schema；
- **应用层校验**解决的是参数在业务上是否合法、用户是否有权限以及操作是否安全。

即使输出完全符合 JSON Schema，也可能出现城市选错、日期理解错误或调用了不合适的工具，因此不能只依赖格式约束。

### 6. 工具执行与模型训练是两个不同环节

训练样例可以只包含预先准备好的工具调用轨迹，模型不一定在训练时真的连接并执行了每个 API。到了运行阶段，模型仍然只是生成调用请求，真正的函数由外部程序执行。

如果系统希望模型从实际执行结果中持续改进，可以收集成功率、参数错误、用户纠正和工具异常等日志，经过脱敏、筛选和标注后形成新的训练或评测数据，而不是让线上模型自动修改自身参数。

简单来说：**预训练让模型看懂语言、代码和 API，监督微调让它学会工具调用格式与流程，偏好对齐让它学会何时调用，运行时 Schema 告诉它当前有哪些工具，结构化解码和应用校验则保证格式与执行安全。模型学到的是通用的“使用工具的方法”，而不是记住所有具体工具。**

## 3. 什么是 MCP？它的架构和工作流程是什么？

MCP（Model Context Protocol，模型上下文协议）是一套用于连接 AI 应用与外部工具、数据源和提示模板的开放协议。它规定了客户端和服务端如何发现能力、交换消息、调用工具及返回结果，使同一个 MCP Server 可以被多个支持 MCP 的 AI 应用复用。

在没有 MCP 时，每个 AI 应用通常都要分别适配数据库、文件系统、GitHub、搜索服务等系统。如果有 `N` 个 AI 应用和 `M` 个外部系统，可能产生大量定制连接。MCP 在两者之间提供统一接口，目标是把“每个应用分别适配每个系统”变成“应用和外部系统分别适配 MCP”。因此，MCP 经常被类比为 AI 应用连接外部能力的“USB-C”，但它统一的是通信协议，并不会自动统一底层业务 API、权限和数据语义。

### 一、MCP 的三个核心角色

MCP 采用 Host—Client—Server 架构：

1. **MCP Host**：面向用户的 AI 应用，例如 IDE、桌面助手或 Agent 平台。它负责管理 LLM、对话上下文、权限和多个 MCP 连接，并决定哪些能力可以交给模型使用。
2. **MCP Client**：Host 内部的协议客户端，负责与某个 MCP Server 建立连接、协商能力和收发 MCP 消息。一个 Host 可以管理多个 Client，分别连接不同 Server。
3. **MCP Server**：向 AI 应用暴露工具、资源或提示模板的程序。它可以是 Host 启动的本地进程，也可以是部署在网络上的远程服务。

例如，IDE 是 MCP Host；IDE 内部创建文件系统 MCP Client 和 GitHub MCP Client；对应的文件系统 MCP Server 与 GitHub MCP Server 分别提供读写文件、查询仓库和创建 Issue 等能力。

### 二、MCP Server 可以提供哪些能力？

MCP Server 主要可以暴露三类原语（Primitives）：

| 原语 | 作用 | 一般由谁控制 | 示例 |
| --- | --- | --- | --- |
| **Tools** | 可以执行的函数，既可以查询数据，也可以产生外部副作用 | 模型根据上下文选择，Host 负责审批和执行编排 | 查询数据库、发送消息、创建日程 |
| **Resources** | 供应用读取并放入模型上下文的数据 | 应用选择和管理 | 文件内容、数据库 Schema、API 文档 |
| **Prompts** | 可复用的提示模板或工作流入口 | 通常由用户显式选择 | 总结代码、分析日志、生成周报 |

常见协议方法包括：

- `tools/list`：发现 Server 提供的工具；
- `tools/call`：调用指定工具；
- `resources/list`、`resources/read`：发现和读取资源；
- `prompts/list`、`prompts/get`：发现和获取提示模板。

除此之外，MCP 还定义了一些客户端能力，例如让 Server 请求 Host 使用 LLM 的 Sampling、向用户请求补充信息的 Elicitation，以及通知、进度跟踪等机制。具体能力需要根据协议版本和初始化时的协商结果使用。

### 三、MCP 使用什么方式通信？

MCP 的数据层基于 **JSON-RPC 2.0**，定义请求、响应、通知、错误、生命周期和各种原语的消息格式。传输层负责消息通过什么通道发送，当前稳定规范的标准传输主要包括：

1. **stdio**：Host 将 MCP Server 作为本地子进程启动，通过标准输入和标准输出交换消息，适合本地文件、代码和命令行工具。
2. **Streamable HTTP**：Client 通过 HTTP 与独立部署的远程 Server 通信，并可以按需使用 SSE 接收流式消息，适合企业服务和云端 API。

MCP 不是只能用于远程网络服务。Server 指的是提供 MCP 能力的程序，与它运行在本机还是远程无关。

### 四、一次 MCP 工具调用的典型流程

以当前稳定协议的典型流程为例：

1. Host 创建 MCP Client，并连接配置好的 MCP Server。
2. Client 与 Server 初始化连接，协商协议版本和双方支持的能力。
3. Client 通过 `tools/list` 等方法发现 Server 提供的工具，并把适合当前用户使用的工具注册到 Host。
4. Host 将工具名称、描述和输入 Schema 提供给 LLM。
5. 用户提出问题，LLM 判断需要调用某个工具，并生成结构化参数。
6. Host 进行权限、参数和风险校验，再由 Client 向 Server 发送 `tools/call`。
7. Server 调用真实数据库、文件系统或业务 API，并将执行结果返回 Client。
8. Host 把工具结果加入模型上下文，LLM 继续调用其他工具或生成最终答案。

完整过程可以概括为：

**Host 连接 Server → 发现能力 → 将工具提供给 LLM → LLM 选择工具 → Host 校验 → Client 调用 Server → 结果返回 LLM → 生成答案。**

例如，一个数据库 MCP Server 可以同时提供：

- Tool：`query_database`，执行经过限制的数据库查询；
- Resource：数据库表结构和字段说明；
- Prompt：指导模型按照业务口径分析数据的提示模板。

Host 可以先读取 Schema 作为上下文，再让模型调用查询工具，最后根据查询结果生成分析报告。

### 五、MCP 和 Function Calling 有什么区别？

二者处在不同层次，并且经常组合使用：

| 对比项 | Function Calling | MCP |
| --- | --- | --- |
| **主要解决的问题** | 模型如何用结构化格式表达“调用哪个函数、传什么参数” | AI 应用如何以统一协议连接、发现和调用外部能力 |
| **关注范围** | 函数定义、工具选择和参数生成 | Host、Client、Server、生命周期、能力发现、传输和消息格式 |
| **是否规定工具来自哪里** | 通常不规定，由应用自行注册和执行 | Server 可以通过标准协议动态暴露工具、资源和 Prompts |
| **是否由模型直接执行** | 否，外部应用执行 | 否，MCP Server 或其下游系统执行 |

在实际系统中，Host 经常先通过 MCP 的 `tools/list` 获得工具 Schema，再把它转换成当前 LLM 支持的 Function Calling 格式。模型生成 Function Call 后，Host 再将调用路由为 MCP 的 `tools/call`。

因此：**Function Calling 更像模型与应用之间的调用接口，MCP 更像应用与工具提供方之间的标准连接协议。MCP 不替代 Function Calling，也不负责模型的工具选择推理。**

### 六、MCP 和普通 API 有什么区别？

MCP Server 底层仍然经常调用 REST、GraphQL、数据库 SDK 或本地函数。MCP 并不是要替代所有业务 API，而是在它们上面增加一个面向 AI 应用的标准适配层，提供统一的能力描述、动态发现、调用格式和上下文交换方式。

普通 API 主要面向确定性程序，调用方通常提前知道接口；MCP 更强调让 AI Host 在运行时发现有哪些 Tools、Resources 和 Prompts，并把它们安全地提供给模型。

### 七、MCP 的优势和局限是什么？

主要优势包括：

- **降低集成成本**：Server 实现一次后，可以被多个兼容的 AI Host 复用。
- **动态发现能力**：Client 可以查询当前有哪些工具、资源和提示模板。
- **解耦模型与外部系统**：Server 不需要绑定某个特定 LLM 厂商。
- **本地与远程统一**：同一种协议可以连接本地进程和云端服务。
- **能力边界清晰**：不同 Server 可以分别管理自己的数据、工具和权限。

主要局限包括：

- MCP 只统一协议，不能保证 Server 的工具描述、返回数据和业务语义正确；
- 工具过多会增加上下文 Token、工具选择错误率和路由复杂度；
- 远程调用仍然存在网络延迟、超时、重试和服务可用性问题；
- Client、Server 和不同协议版本之间可能出现兼容性差异；
- 引入第三方 Server 会扩大权限、数据泄漏、Prompt Injection 和供应链攻击面。

### 八、MCP 有哪些安全注意事项？

Host 不能因为一个 Server 使用了 MCP 就默认信任它。生产环境至少需要：

- 只连接可信或经过审核的 Server，并校验安装来源和版本；
- 根据用户和任务动态限制可见工具，遵循最小权限原则；
- 对文件写入、删除、发送消息和支付等操作进行用户确认；
- 在 Host 和 Server 两端校验参数，防止命令注入、路径穿越、SSRF 和越权访问；
- 将 Tool、Resource 的内容视为不可信数据，防御其中的 Prompt Injection；
- 对远程连接使用认证、授权和安全传输，并避免把凭证暴露给 LLM；
- 设置超时、速率限制、最大返回大小和审计日志，敏感字段需要脱敏。

### 九、用一个最小代码示例理解 Host、Client 和 Server

下面使用官方 Python SDK 和本地 `stdio` 传输演示三者的关系。示例有两个文件：

```text
用户
  ↓
host.py                    ← MCP Host：管理 LLM、上下文和权限
  └── ClientSession        ← MCP Client：负责 MCP 协议通信
        ↓ stdio / JSON-RPC
      server.py            ← MCP Server：暴露并执行 get_weather 工具
```

先安装 SDK：

```bash
uv add "mcp[cli]>=1.27,<2"
```

以下代码使用当前稳定的 Python SDK v1.x API；v2 仍处于预发布阶段，接口有所不同。

`server.py` 是 MCP Server。它定义并暴露一个天气查询 Tool：

```python
from mcp.server.fastmcp import FastMCP


mcp = FastMCP("weather-server")


@mcp.tool()
def get_weather(city: str) -> dict:
    """查询指定城市的天气。"""
    # 实际项目中，这里会调用天气 API。
    fake_weather = {
        "北京": {"condition": "晴", "temperature": 30},
        "上海": {"condition": "小雨", "temperature": 28},
    }
    weather = fake_weather.get(
        city,
        {"condition": "未知", "temperature": None},
    )
    return {"city": city, **weather}


if __name__ == "__main__":
    # Server 从 stdin 读取 MCP 消息，通过 stdout 返回结果。
    mcp.run(transport="stdio")
```

`host.py` 是面向用户的 MCP Host。它在内部创建一个 `ClientSession` 作为 MCP Client，并由 Client 启动及连接 Server：

```python
import asyncio
import sys
from pathlib import Path

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


async def main() -> None:
    server_file = Path(__file__).with_name("server.py")
    server_params = StdioServerParameters(
        command=sys.executable,
        args=[str(server_file)],
    )

    # Host 创建 MCP Client，并通过 stdio 启动和连接 Server。
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as client:
            await client.initialize()

            # Client 通过 tools/list 动态发现 Server 的工具。
            response = await client.list_tools()
            print("发现的工具：", [tool.name for tool in response.tools])

            # 真实 Host 会把工具 Schema 交给 LLM。
            # 这里省略具体模型 SDK，直接模拟 LLM 生成的 Function Call。
            llm_tool_call = {
                "name": "get_weather",
                "arguments": {"city": "北京"},
            }

            # Client 将调用转换成 MCP 的 tools/call 请求并发给 Server。
            result = await client.call_tool(
                llm_tool_call["name"],
                arguments=llm_tool_call["arguments"],
            )

            tool_result = result.structured_content or result.content
            print("工具结果：", tool_result)

            # 真实 Host 接下来会把 tool_result 放回模型上下文，
            # 再调用一次 LLM 生成面向用户的自然语言答案。


if __name__ == "__main__":
    asyncio.run(main())
```

将两个文件放在同一目录，运行：

```bash
uv run python host.py
```

可能看到类似输出：

```text
发现的工具： ['get_weather']
工具结果： {'city': '北京', 'condition': '晴', 'temperature': 30}
```

这个例子中各部分的职责是：

- **Host 是整个 `host.py` 应用**：负责管理用户请求、LLM、工具暴露、权限校验和最终答案；
- **Client 是 `ClientSession`**：它属于 Host，负责 `initialize`、`tools/list`、`tools/call` 等协议通信；
- **Server 是 `server.py` 进程**：它声明工具 Schema，接收调用请求，并执行真正的业务代码；
- **LLM 不是 MCP Client**：LLM 只看到 Host 提供的工具定义并生成调用意图，Host 才是连接模型和 MCP Client 的协调者。

SDK 在底层完成 JSON-RPC 消息的序列化与传输。这个示例使用 `stdio`，所以 Host 会自动启动本地 Server；如果改成 Streamable HTTP，Server 会独立部署，Client 改为通过 URL 连接，但 Host—Client—Server 的职责划分基本不变。

需要注意，MCP 规范演进较快。截至 2026 年 7 月 19 日，最新正式发布的稳定规范仍是 `2025-11-25`；官方已经发布 `2026-07-28` 候选规范，计划将协议核心改为无状态并调整生命周期等机制，但在正式发布前不应把候选规范当作稳定行为。实际开发和面试回答都应说明所依据的协议版本。

简单来说：**MCP 为 AI 应用和外部能力之间定义了统一的连接协议；Server 暴露 Tools、Resources 和 Prompts，Host 负责模型、上下文、权限与编排，Client 负责协议通信。Function Calling 解决模型如何提出工具调用，MCP 解决工具如何被标准化发现、连接和执行。**

参考：[MCP 架构说明](https://modelcontextprotocol.io/docs/learn/architecture)、[MCP Server 核心能力](https://modelcontextprotocol.io/docs/learn/server-concepts)、[MCP 2025-11-25 传输规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[MCP Python SDK v1.x](https://github.com/modelcontextprotocol/python-sdk/tree/v1.x)、[MCP 2026-07-28 候选规范公告](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)

## 4. 什么是 Skill？Skill 的工作原理是什么？

Skill（技能）是一种向 Agent 提供**可复用的专业知识、操作步骤和任务资源**的能力包。它将某类任务的“做事方法”整理成 Agent 可以按需加载的说明，使模型不需要重新训练，也能更稳定地执行代码审查、数据分析、文档处理或企业内部流程等任务。

按照 Agent Skills 开放规范，一个 Skill 至少是一个包含 `SKILL.md` 的目录，还可以携带脚本、参考资料和模板：

```text
pdf-report/
├── SKILL.md          # 必需：元数据和核心操作说明
├── scripts/          # 可选：可以执行的 Python、Shell 等脚本
├── references/       # 可选：详细文档、规范和领域知识
└── assets/           # 可选：模板、图片和其他静态资源
```

`SKILL.md` 由 YAML Frontmatter 和 Markdown 正文组成。例如：

```markdown
---
name: pdf-report
description: 提取并总结 PDF，生成结构化报告。处理 PDF、扫描件或报告总结任务时使用。
---

# PDF 报告处理流程

1. 检查文件是否存在以及是否受密码保护。
2. 使用 `scripts/extract.py` 提取正文和表格。
3. 按照 `references/report-rules.md` 总结内容。
4. 使用 `assets/report-template.md` 生成报告。
5. 检查页码、数字和引用是否与原文一致。
```

其中：

- `name` 用于标识 Skill；
- `description` 说明 Skill 能做什么以及什么情况下使用，是 Skill 发现和触发的重要依据；
- Markdown 正文描述输入要求、执行步骤、分支判断、输出格式和检查规则；
- `scripts/` 提供可以重复、确定性执行的代码；
- `references/` 保存只在特定场景下才需要读取的详细知识；
- `assets/` 保存报告模板、配置样例、图片等产物资源。

### 一、Skill 的核心原理：渐进式披露

如果 Agent 在启动时把所有 Skill 的完整内容都放入上下文，会浪费大量 Token，也容易造成指令互相干扰。因此，Skill 通常采用 Progressive Disclosure（渐进式披露），分三层加载：

1. **发现（Discovery）**：Agent 启动时只读取所有已安装 Skill 的 `name` 和 `description`，形成一个占用较小的技能目录。
2. **激活（Activation）**：用户显式指定某个 Skill，或者当前任务与某个 `description` 匹配时，Agent 才读取该 Skill 的完整 `SKILL.md` 正文。
3. **执行（Execution）**：Agent 按照说明完成任务，并只在需要时读取 `references/`、运行 `scripts/` 或使用 `assets/`。

执行流程可以概括为：

```text
用户提出任务
    ↓
Agent 根据 name 和 description 匹配 Skill
    ↓
加载完整 SKILL.md 到模型上下文
    ↓
模型理解步骤、约束和输出要求
    ↓
按需读取 references / assets，运行 scripts 或调用工具
    ↓
检查结果并向用户返回产物
```

这种设计兼顾了两个目标：Agent 可以安装很多 Skill，但每次任务只为少量相关 Skill 支付上下文 Token；同时，复杂细节又能在真正需要时被加载。

### 二、Skill 是如何被触发的？

Skill 通常有两种触发方式：

1. **显式触发**：用户直接指定“使用 `pdf-report` Skill 处理这个文件”。
2. **自动触发**：Agent 根据用户任务与各 Skill 的 `description` 判断哪个 Skill 适用。

具体匹配算法不是 Skill 文件格式强制规定的。不同 Agent 可以使用 LLM 判断、关键词规则、Embedding 检索或单独的 Router。无论采用哪种方式，`description` 都应该同时写清楚“能做什么”和“什么时候使用”；描述过窄会漏触发，描述过宽则会误触发。

如果多个 Skill 同时适用，Agent 可以组合使用，例如先用 PDF Skill 提取文档，再用数据分析 Skill 计算指标，最后用演示文稿 Skill 生成幻灯片。多个 Skill 的指令出现冲突时，应由 Host 根据系统指令、用户要求、权限和优先级进行处理。

### 三、Skill 为什么不需要重新训练模型？

Skill 的说明会在推理时被加入模型上下文，本质上属于 Context Engineering。LLM 已经具备阅读说明、执行步骤和调用工具的通用能力，Skill 只是把某个领域的程序性知识临时提供给它。

因此，安装 Skill 通常不会修改模型参数，也不会让模型永久“学会”这项能力：

- 当前任务激活 Skill 后，模型按照 Skill 内容行动；
- 未激活时，完整说明不会进入上下文；
- 删除或修改 Skill 后，后续任务会使用新的说明；
- 任务执行日志可以用于改进 Skill，但不会自动更新模型权重。

Skill 可以包含可执行脚本。此时 LLM 负责判断何时以及如何使用脚本，而确定性的解析、转换和校验逻辑由脚本完成。这样可以把模型擅长的语义理解与程序擅长的稳定执行结合起来。

### 四、Skill 与 Prompt、Function Calling、MCP、RAG 有什么区别？

| 概念 | 主要解决的问题 | 典型内容 |
| --- | --- | --- |
| **System Prompt** | Agent 在整个会话中应遵守哪些全局规则 | 身份、总体行为、安全边界 |
| **Skill** | 某一类任务应该按照什么流程完成 | 操作步骤、判断规则、示例、脚本、模板 |
| **Function Calling** | 模型如何结构化表达一次工具调用 | 函数名和参数 |
| **MCP** | AI 应用如何标准化发现、连接和调用外部能力 | Tools、Resources、Prompts 和通信协议 |
| **RAG** | 如何从外部知识库找回与当前问题相关的事实 | 文档检索、Rerank 和上下文构造 |
| **Fine-tuning** | 如何通过训练改变模型自身的行为和能力 | 训练数据和模型参数更新 |

它们可以组合使用。例如，一个“生成销售周报”Skill 可以规定完整工作流程：先通过 MCP 连接销售数据库，使用 Function Calling 查询指标，再通过 RAG 找到业务规则，最后套用 Skill 自带的模板生成报告。

可以把它们简单理解为：

- **Skill 告诉 Agent 怎么做；**
- **Function Calling 让模型表达要调用什么；**
- **MCP 让应用连接到可以执行的工具；**
- **RAG 为任务补充相关知识。**

### 五、Skill 的优势和局限是什么？

主要优势包括：

- **复用性**：将一次成功经验整理成可重复使用的工作流程；
- **一致性**：固定关键步骤、输出格式和检查规则，减少模型自由发挥；
- **Token 效率**：通过渐进式披露，只加载当前任务需要的内容；
- **可维护性**：Skill 是普通文件，可以进行版本控制、Code Review、测试和回滚；
- **可移植性**：遵循同一开放格式的 Skill 可以在兼容的 Agent 产品之间迁移；
- **可组合性**：多个小 Skill 可以组合完成更复杂的任务。

主要局限包括：

- Skill 仍然依赖 LLM 正确理解和遵循说明，不能保证每次执行完全一致；
- `description` 写得不好会造成漏触发或误触发；
- 多个 Skill、系统指令和用户要求之间可能产生冲突；
- Skill 内容和依赖资源会占用上下文，过长的说明可能降低模型注意力；
- 不同 Agent 对自动触发、工具权限和可选字段的支持可能不同；
- 外部 Skill 可能包含恶意指令、危险脚本或被篡改的依赖，存在供应链风险。

因此，安装第三方 Skill 前应审查 `SKILL.md` 和脚本，限制文件、网络和命令权限，对写入、删除、发送及支付等操作保留用户确认，并对 Skill 进行版本固定、测试和来源校验。

简单来说：**Skill 是给 Agent 使用的可复用“任务说明书和资源包”。它不修改模型权重，而是通过渐进式披露，在任务匹配时把操作流程加载到上下文，再让 Agent 按需使用脚本、参考资料、模板和外部工具完成任务。**

参考：[Agent Skills 概览](https://agentskills.io/home)、[Agent Skills 格式规范](https://agentskills.io/specification)、[Agent Skills 客户端实现原理](https://agentskills.io/client-implementation/adding-skills-support)、[OpenAI Skills 说明](https://help.openai.com/en/articles/20001066-skills-in-chatgpt)

## 5. 什么是 A2A 协议？

A2A（Agent2Agent，Agent-to-Agent）是一套用于**独立 AI Agent 之间发现能力、交换消息、委派任务和返回结果**的开放协议。它最初由 Google 提出，后来交由 Linux Foundation 托管。

A2A 主要解决不同团队、框架、语言或厂商开发的 Agent 如何互相协作的问题。通信双方不需要知道对方内部使用了什么模型、Prompt、Memory 或工具，只需要了解对方公开的能力和交互接口，因此它强调 **Opaque Agent（内部实现不透明的 Agent）之间的互操作**。

### 一、A2A 的核心概念

1. **A2A Client**：发起请求或委派任务的一方，通常是当前正在为用户服务的 Agent 或其应用程序。
2. **A2A Server / Remote Agent**：接收请求并完成任务的一方。它本身是一个具有推理、工具和任务管理能力的独立 Agent，而不仅是一个普通函数。
3. **Agent Card**：Agent 的能力说明书，描述 Agent 的名称、地址、协议版本、认证要求、支持的交互方式和公开能力。标准发现地址通常是 `/.well-known/agent-card.json`。
4. **Message**：Agent 之间交换的一轮消息，可以包含文本、文件和结构化数据等 Part。
5. **Task**：需要持续跟踪的任务实体，具有唯一 ID 和执行状态，适合长耗时、多轮或需要人工补充信息的任务。
6. **Artifact**：Remote Agent 执行任务后产生的正式结果，例如报告、代码、表格或结构化数据。

需要注意，Agent Card 中的 `skills` 表示该 Agent 对外声明的能力，例如“预订机票”或“生成财务报告”；它与上一题中以 `SKILL.md` 为核心的 Agent Skills 能力包不是同一个概念。

### 二、A2A 的典型工作流程

假设一个“旅行规划 Agent”需要委派“机票 Agent”查询航班：

1. 旅行规划 Agent 获取机票 Agent 的 Agent Card。
2. 根据 Agent Card 判断它是否支持航班搜索、流式响应，以及需要什么认证方式。
3. A2A Client 向机票 Agent 发送 Message，例如“查询 8 月 1 日北京到上海的航班”。
4. 如果任务可以立即完成，Remote Agent 可以直接返回结果；如果需要较长时间，则创建一个 Task。
5. Remote Agent 在执行过程中更新 Task 状态；需要日期或舱位等额外信息时，可以请求客户端补充。
6. Client 通过同步响应、流式事件、轮询或异步通知获取进度。
7. 任务完成后，Remote Agent 返回包含航班列表的 Artifact，旅行规划 Agent 再继续完成整体行程。

流程可以概括为：

```text
发现 Agent Card
    ↓
确认能力、协议和认证方式
    ↓
发送 Message 或创建 Task
    ↓
Remote Agent 独立推理并执行
    ↓
返回状态、消息和 Artifact
    ↓
调用方 Agent 继续完成用户目标
```

### 三、A2A 如何通信？

A2A 复用成熟的 Web 技术。当前 1.0 规范定义了统一的数据模型和操作，并支持 JSON-RPC、gRPC、HTTP + JSON/REST 等协议绑定。常见实现使用 HTTP(S) 和 JSON-RPC；需要实时返回进度时，可以使用流式响应。

A2A 原生考虑了三类交互：

- **同步请求—响应**：适合能够快速完成的任务；
- **流式交互**：持续返回生成内容和任务状态；
- **异步长任务**：任务在请求结束后继续执行，Client 后续查询或接收通知。

消息内容不局限于自然语言，还可以携带文件引用和结构化 JSON，因此两个 Agent 可以交换对程序友好的数据，而不是只能互相“聊天”。

### 四、A2A 与 MCP 有什么区别？

| 对比项 | MCP | A2A |
| --- | --- | --- |
| **连接对象** | Agent 与工具、数据源 | 独立 Agent 与独立 Agent |
| **能力暴露** | Tools、Resources、Prompts | Agent Card 中的能力和交互方式 |
| **调用粒度** | 通常调用一个明确的工具操作 | 委派一个可能需要自主规划的任务 |
| **内部实现** | Tool 通常是确定性函数或 API | Remote Agent 可以拥有自己的模型、记忆和工具 |
| **长任务协作** | 可以扩展支持，但不是最初的核心定位 | Task、状态、流式和异步协作是核心能力 |

二者可以配合使用：一个主 Agent 通过 A2A 将“分析客户流失原因”委派给数据分析 Agent；数据分析 Agent 内部再通过 MCP 查询数据库、读取文档和执行代码。

可以简单理解为：

```text
MCP：Agent 如何使用工具
A2A：Agent 如何与另一个 Agent 合作
```

### 五、什么时候适合使用 A2A？

A2A 适合跨团队、跨服务、跨框架或跨厂商的独立 Agent 协作，特别是需要能力发现、任务委派、长任务状态和结果交付的场景。

如果只是单 Agent 调用一个确定性 API，使用 REST、Function Calling 或 MCP 通常更简单；如果多个子 Agent 都运行在同一个应用和同一个编排器中，直接使用框架内部接口也可能比引入 A2A 更合适。

A2A 统一了通信方式，但不会自动解决 Agent 是否可信、授权范围、任务结果是否正确以及费用由谁承担等问题。生产环境仍需要身份认证、细粒度授权、输入与文件校验、超时和配额、审计追踪，以及高风险操作确认。

简单来说：**A2A 是独立 Agent 之间的标准协作协议。调用方通过 Agent Card 发现对方能力，通过 Message 和 Task 委派工作，并接收状态与 Artifact；双方可以保持内部模型、记忆和工具实现不透明。**

参考：[A2A 官方项目](https://github.com/a2aproject/A2A)、[A2A 1.0 协议规范](https://a2a-protocol.org/latest/specification/)、[A2A 与 MCP 的关系](https://github.com/a2aproject/A2A/blob/main/docs/specification.md#appendix-b-relationship-to-mcp-model-context-protocol)

## 6. MCP 的消息格式是什么？

MCP 的数据层基于 **JSON-RPC 2.0**。每条消息可以分成两部分：外层是 JSON-RPC 通用信封，负责标识请求、响应和通知；内层的 `method`、`params`、`result` 则由 MCP 定义具体语义。

例如：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

其中各字段的含义是：

| 字段 | 含义 |
| --- | --- |
| `jsonrpc` | 固定为 `"2.0"`，表示使用 JSON-RPC 2.0 |
| `id` | 请求标识，用于将响应与请求对应；可以是字符串或整数，不能为 `null` |
| `method` | 要执行的 MCP 方法，例如 `tools/list`、`tools/call` |
| `params` | 方法参数，可选，通常是一个 JSON Object |
| `result` | 请求成功时的返回数据 |
| `error` | 协议请求失败时的错误对象，与 `result` 互斥 |
| `_meta` | 可选扩展元数据，例如进度 Token、关联任务和厂商自定义信息 |

### 一、Request：请求消息

请求可以由 Client 发给 Server，也可以由 Server 发给 Client。它必须包含 `id`，接收方处理后需要返回相同 `id` 的响应。

例如，Client 请求 Server 返回可用工具：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list",
  "params": {}
}
```

例如，Client 调用天气工具：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "北京"
    }
  }
}
```

Client 常用的 Server 方法包括 `tools/list`、`tools/call`、`resources/read` 和 `prompts/get`。MCP 是双向协议，Server 也可以请求 Client 执行 `sampling/createMessage`、`elicitation/create` 或 `roots/list` 等操作，但前提是双方已经声明相应能力。

### 二、Result Response：成功响应

成功响应必须携带与原请求相同的 `id`，并使用 `result` 返回结果：

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "查询指定城市的天气",
        "inputSchema": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string"
            }
          },
          "required": ["city"]
        }
      }
    ]
  }
}
```

工具调用成功后，可以同时返回供模型阅读的非结构化内容和供程序处理的结构化结果：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "北京当前晴，气温 30℃"
      }
    ],
    "structuredContent": {
      "city": "北京",
      "condition": "晴",
      "temperature": 30
    },
    "isError": false
  }
}
```

`content` 是 Content Block 数组，可以包含文本、图片、音频、资源链接和嵌入式资源；`structuredContent` 是可选的结构化 JSON，如果工具声明了 `outputSchema`，它应该满足该 Schema。

### 三、Error Response：协议错误响应

如果请求格式错误、方法不存在或参数不符合协议要求，接收方返回 `error`：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "field": "city"
    }
  }
}
```

`error` 至少包含整数类型的 `code` 和字符串类型的 `message`，还可以使用 `data` 携带错误详情。常见 JSON-RPC 错误包括：

| Code | 含义 |
| --- | --- |
| `-32700` | JSON 解析失败 |
| `-32600` | 请求结构不合法 |
| `-32601` | 方法不存在 |
| `-32602` | 参数不合法 |
| `-32603` | 服务端内部错误 |

需要区分**协议错误**和**工具执行错误**：

- 请求本身无法被 MCP 正常处理时，返回 JSON-RPC `error`；
- 请求合法，但工具调用 API 失败、业务校验不通过或没有查询到结果时，通常仍返回成功的 JSON-RPC `result`，并在工具结果中设置 `isError: true`。

例如：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "不支持城市：火星"
      }
    ],
    "isError": true
  }
}
```

这样模型能够看到工具的业务错误，并决定修正参数、选择其他工具或向用户询问，而不是把它当成连接协议已经损坏。

### 四、Notification：通知消息

通知是单向消息，与 Request 的区别是**没有 `id`，接收方也不能返回 Response**。

例如，初始化完成通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

例如，长耗时请求的进度通知：

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "job-123",
    "progress": 50,
    "total": 100,
    "message": "已处理 50%"
  }
}
```

其他通知还包括工具列表变化、资源更新、日志消息和请求取消等。通知适合发送“状态已经变化”之类的信息；如果发送方必须知道处理是否成功，则应该使用带 `id` 的 Request。

### 五、当前稳定版的初始化消息

在 MCP `2025-11-25` 稳定规范中，Client 与 Server 建立连接后需要先进行版本和能力协商。

Client 发送：

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "initialize",
  "params": {
    "protocolVersion": "2025-11-25",
    "capabilities": {},
    "clientInfo": {
      "name": "demo-host",
      "version": "1.0.0"
    }
  }
}
```

Server 返回：

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "result": {
    "protocolVersion": "2025-11-25",
    "capabilities": {
      "tools": {
        "listChanged": true
      }
    },
    "serverInfo": {
      "name": "weather-server",
      "version": "1.0.0"
    }
  }
}
```

Client 随后发送不带 `id` 的 `notifications/initialized`，双方才进入正常通信阶段。由于 MCP 的候选新规范正在调整生命周期机制，实际实现时必须以双方协商的协议版本为准。

### 六、消息格式与传输方式的关系

JSON-RPC 定义的是消息内容，`stdio` 和 Streamable HTTP 决定消息如何传输：

- **stdio**：每条 JSON-RPC 消息占一行，通过进程标准输入和标准输出传输。Server 的普通日志必须写入 `stderr`，否则会破坏 MCP 消息流。
- **Streamable HTTP**：Client 通常使用 HTTP POST 发送一条 JSON-RPC 消息，Server 可以返回一个 `application/json` 响应，也可以通过 SSE 流返回消息和最终响应。

因此，同一个 `tools/call` JSON-RPC 对象既可以通过本地 `stdio` 传输，也可以通过远程 HTTP 传输；改变传输方式通常不需要改变 MCP 方法本身。

简单来说：**MCP 消息是在 JSON-RPC 2.0 信封中放入 MCP 方法和数据。带 `id` 的 Request 必须收到相同 `id` 的 Result 或 Error；不带 `id` 的 Notification 不需要响应。工具业务失败通常通过 `result.isError` 返回，而协议和消息格式错误通过 JSON-RPC `error` 返回。**

参考：[MCP 基础消息格式](https://modelcontextprotocol.io/specification/2025-11-25/basic)、[MCP Tools 消息](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、[MCP 传输规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[MCP Schema Reference](https://modelcontextprotocol.io/specification/2025-11-25/schema)

## 7. MCP 采用哪些通信方式？

截至 2026 年 7 月 19 日，MCP 当前正式稳定规范是 `2025-11-25`。它使用 **JSON-RPC 2.0** 编码消息，并定义了两种标准传输方式：

1. **stdio**：通过标准输入和标准输出在本地进程之间传输消息；
2. **Streamable HTTP**：通过 HTTP 在远程 Client 和 Server 之间传输消息，并可以按需使用 SSE 实现流式通信。

需要区分：JSON-RPC 2.0 规定“消息长什么样”，stdio 和 Streamable HTTP 规定“消息通过什么通道传输”。同一个 `tools/call` JSON-RPC 对象既可以通过 stdio 发送，也可以通过 HTTP 发送。

### 一、stdio

stdio 主要用于本地 MCP Server：

```text
MCP Host
   ↓ 启动子进程
MCP Server
   ↕ stdin / stdout
JSON-RPC 消息
```

它的工作方式是：

1. MCP Client 根据配置启动 MCP Server 子进程；
2. Client 将 JSON-RPC 消息写入 Server 的标准输入 `stdin`；
3. Server 从 `stdin` 读取请求，处理后将 JSON-RPC 响应写入 `stdout`；
4. Client 从 `stdout` 读取响应，并交给 Host 继续处理。

stdio 传输的主要特点包括：

- 每条消息是独立的 JSON-RPC Request、Response 或 Notification；
- 消息以换行符分隔，一条消息不能包含未转义的换行；
- `stdout` 只能输出合法的 MCP 消息；
- 普通日志必须写入 `stderr`，否则会污染协议消息流；
- 不需要开放网络端口，适合访问本地文件、Git、数据库客户端和命令行工具；
- Server 生命周期通常由 Host 管理，Host 退出时会关闭输入流并终止子进程。

例如，在本地 IDE 中接入文件系统 MCP Server 时，IDE 可以直接启动 Server 进程，通过 stdio 发送 `resources/read` 或 `tools/call`，不需要部署 HTTP 服务。

### 二、Streamable HTTP

Streamable HTTP 主要用于独立部署的远程 MCP Server：

```text
MCP Client ── HTTP POST ──> /mcp
           <── JSON 或 SSE ── MCP Server
```

Server 通常提供一个同时支持 POST 和 GET 的 MCP Endpoint，例如：

```text
https://example.com/mcp
```

Client 向 Server 发送消息时：

1. 每条 JSON-RPC 消息使用一个新的 HTTP POST 请求发送；
2. POST Body 中包含一个 JSON-RPC Request、Response 或 Notification；
3. Client 在 `Accept` 中声明同时支持 `application/json` 和 `text/event-stream`；
4. 如果发送的是 Request，Server 可以直接返回一个 JSON 响应，也可以建立 SSE 流后返回多条消息和最终响应；
5. 如果发送的是 Notification 或 Response，Server 接收成功后通常返回 HTTP `202 Accepted`，不返回 JSON-RPC Body。

一个简化的请求示例是：

```http
POST /mcp HTTP/1.1
Host: example.com
Content-Type: application/json
Accept: application/json, text/event-stream
MCP-Protocol-Version: 2025-11-25

{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "city": "北京"
    }
  }
}
```

Server 可以直接返回：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "北京当前晴，气温 30℃"
      }
    ]
  }
}
```

如果 Server 需要流式返回进度、通知或中间消息，则可以将响应类型设置为 `text/event-stream`。Client 也可以使用 HTTP GET 打开 SSE 通道，接收 Server 主动发送的消息。

当前稳定规范还支持可选的有状态 Session：Server 可以在初始化响应中返回 `MCP-Session-Id`，Client 在后续请求中携带该 Header。HTTP Client 还需要携带双方协商得到的 `MCP-Protocol-Version`。

远程 MCP Server 还应使用 TLS、认证和授权，校验 `Origin` Header，并设置超时、速率限制和消息大小限制；本地 HTTP Server 应尽量只监听 `127.0.0.1`，防止 DNS Rebinding 和未授权访问。

### 三、SSE 是第三种标准传输吗？

不是。**SSE 是 Streamable HTTP 内部可选的流式消息机制，不是当前独立的第三种 MCP 标准传输。**

早期 `2024-11-05` 规范使用过单独的 `HTTP + SSE` 传输方式，当前已经由 Streamable HTTP 替代。Client 和 Server 可以为了兼容旧实现继续支持旧版 HTTP + SSE，但新项目通常应优先使用 Streamable HTTP。

### 四、MCP 支持 WebSocket 吗？

WebSocket 当前不是 MCP 正式定义的标准传输。MCP 允许 Client 和 Server 实现自定义 Transport，只要保持 JSON-RPC 消息格式和对应版本要求，但自定义 Transport 需要双方分别适配，跨产品互操作性通常不如 stdio 和 Streamable HTTP。

### 五、应该如何选择？

| 场景 | 推荐传输方式 |
| --- | --- |
| 本地 IDE、桌面助手调用本地工具 | stdio |
| 访问本地文件、Git 或命令行程序 | stdio |
| 跨机器或跨网络调用 | Streamable HTTP |
| 云端、企业服务和多租户系统 | Streamable HTTP |
| 需要流式进度和 Server 消息 | Streamable HTTP + SSE |
| 必须兼容早期 MCP Server | 根据版本回退到旧 HTTP + SSE |

### 六、为什么 MCP 不把 WebSocket 作为标准传输？

WebSocket 可以在一个长连接上进行全双工通信，从技术上完全能够承载 MCP 的 JSON-RPC 消息。MCP 也允许双方实现自定义 WebSocket Transport。它没有被选为标准传输，主要不是因为能力不足，而是 MCP 的典型调用模式和工程目标更适合 stdio 与 Streamable HTTP。

#### 1. 大部分 MCP 调用不需要持续全双工连接

MCP 最常见的流程是 Client 发起 `tools/call`、`resources/read` 等请求，Server 返回结果，本质上仍然以请求—响应为主。只有进度通知、流式结果和 Server-to-Client 请求等少数场景需要反向通信。

Streamable HTTP 已经可以使用：

- HTTP POST 完成 Client → Server 消息；
- HTTP Response 或 SSE 完成 Server → Client 消息；
- GET + SSE 在需要时建立 Server → Client 的消息通道。

因此，它已经能够满足 MCP 所需的双向消息交换，不一定要为每个连接长期维护一个全双工 WebSocket。

#### 2. HTTP 更容易复用现有基础设施

Streamable HTTP 可以直接使用成熟的：

- API Gateway、反向代理和普通负载均衡器；
- OAuth、Bearer Token、企业身份系统和标准 HTTP 状态码；
- WAF、限流、访问日志、Tracing 和审计系统；
- Serverless、容器平台、CDN 与多级网络代理；
- 按 URL、Header 和 Method 进行路由及策略控制。

WebSocket 虽然也能接入这些系统，但通常需要额外配置 Upgrade、连接保持、心跳、空闲超时和连接级限流。一些企业代理和 Serverless 环境对长连接的支持也比普通 HTTP 请求更复杂。

#### 3. WebSocket 会增加有状态连接的运维成本

长连接通常与某个 Server 实例绑定。进行水平扩容时，系统可能需要 Sticky Session、共享 Session 状态或专门的连接网关。Server 滚动发布、实例故障和网络切换时，还要处理断线重连、消息重放、重复消息与顺序恢复。

普通 HTTP 请求可以由任意实例处理，更适合无状态服务和弹性扩缩容。这也是 `2026-07-28` 候选规范计划移除协议级 Session、强化无状态 HTTP 调用的重要原因。

#### 4. HTTP 的权限和可观测边界更自然

每个 HTTP 请求都可以独立携带身份、协议版本、方法信息和 Trace Context，网关能够按具体请求进行鉴权、限流、路由和审计。

WebSocket 通常在连接建立时完成一次认证，后续多条逻辑消息共享同一连接。系统仍然可以对每条消息授权，但需要在应用层额外实现 Token 更新、细粒度权限检查、流量统计和审计关联。使用 WebSocket 也不会自动解决 JSON-RPC 的请求关联、超时、取消、重试和幂等性问题。

#### 5. 本地场景使用 stdio 更简单

本地 MCP Server 通常由 Host 直接启动，不需要开放端口、处理网络认证或维护 Socket。stdio 已经提供双向字节流，部署成本和攻击面都更小，因此没有必要再使用 WebSocket。

#### WebSocket 什么时候仍然适合？

如果系统需要高频、低延迟、长时间持续的双向事件，例如实时协作、远程设备控制或大量交互式流数据，并且 Client 与 Server 都由同一团队控制，可以实现 WebSocket 自定义 Transport。

代价是双方必须约定连接、认证、重连、心跳和消息恢复机制，而且其他只支持标准 stdio 或 Streamable HTTP 的 MCP Client 无法直接连接，通用互操作性会下降。

简单来说：**WebSocket 能承载 MCP，但 MCP 的多数操作是 Client 主动的请求—响应，Streamable HTTP 加可选 SSE 已经能够满足双向和流式需求，同时更容易复用 HTTP 的鉴权、网关、负载均衡和无状态扩缩容能力。因此 WebSocket 被保留为可选的自定义 Transport，而不是标准默认方案。**

### 七、候选新规范的变化

官方已经发布 `2026-07-28` 候选规范，但截至 2026 年 7 月 19 日尚未正式发布。候选规范计划将远程 MCP 改成无状态模式：

- 删除 `initialize` / `initialized` 握手；
- 删除协议层的 `MCP-Session-Id`；
- 将协议版本、Client 信息和必要能力放在每次请求中；
- Server-to-Client 交互改为可以重新提交的多轮 HTTP 请求，减少对长连接和粘性 Session 的依赖。

这并不意味着 HTTP 被替换，而是 Streamable HTTP 的交互方式从协议层有状态转向无状态。正式发布和 SDK 完成适配前，生产实现仍应以双方实际支持的协议版本为准。

简单来说：**本地 MCP Server 通常使用 stdio，远程 MCP Server 通常使用 Streamable HTTP；JSON-RPC 2.0 是两者共同承载的消息格式，SSE 只是 Streamable HTTP 中可选的流式机制。**

参考：[MCP 2025-11-25 传输规范](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、[MCP 2026-07-28 候选规范](https://blog.modelcontextprotocol.io/posts/2026-07-28-release-candidate/)

## 8. 实时语音对话采用什么传输协议？为什么？

实时语音对话通常不会使用 MCP 传输音频，而是根据接入场景选择 WebRTC、WebSocket 或 SIP + RTP：

| 场景 | 常用协议 |
| --- | --- |
| 浏览器、移动端实时语音 | WebRTC |
| 后端服务器连接实时语音模型 | WebSocket |
| 电话呼入、呼出 | SIP + RTP/SRTP |
| 录音转写、非实时 TTS | 普通 HTTP 上传或流式 HTTP |

### 一、浏览器和移动端：WebRTC

典型链路是：

```text
麦克风
  ↓
WebRTC
  ↓
实时语音模型或语音网关
  ↓
WebRTC
  ↓
扬声器
```

WebRTC 的底层媒体通常使用 RTP/SRTP，并优先通过 UDP 传输；同时使用 ICE、STUN 和 TURN 完成连接协商及 NAT 穿透。

WebRTC 适合实时语音的主要原因包括：

- 支持真正的双向实时音频；
- UDP 不会因为一个数据包丢失而阻塞后续音频；
- RTP 提供序列号和时间戳，便于处理乱序、丢包和连续播放；
- 支持网络抖动处理和自适应拥塞控制；
- 浏览器通常提供回声消除、降噪和自动增益等音频能力；
- 支持用户在模型说话时进行打断，即 Barge-in；
- 使用 SRTP 加密媒体数据；
- 可以通过 STUN 和 TURN 穿越 NAT 及防火墙。

实时语音通常每隔约 10～20 毫秒生成一个音频块。偶尔丢失一小段声音，通常比为了重传而造成明显停顿更容易接受，因此实时媒体更强调低延迟和连续播放，而不是保证每一个字节都成功到达。

### 二、服务端到服务端：WebSocket

后端 Agent 连接实时语音模型时，经常使用 WebSocket：

```text
后端 Agent ←── WebSocket ──→ 实时语音服务
             音频帧 + 控制事件
```

WebSocket 建立在 TCP 之上，连接建立后双方可以主动发送数据，适合在同一个连接中传输：

- 用户音频帧和模型生成的音频；
- 语音识别的中间文本和最终文本；
- 开始、停止、静音和打断事件；
- Function Call、Tool Result 和状态信息；
- 错误、用量和其他控制消息。

WebSocket 的接口相对简单，也容易穿过常见的 HTTPS 网络环境。但它本身不提供 WebRTC 的媒体能力，例如音频设备管理、回声消除、抖动缓冲、NAT 穿透和专门的丢包处理。

由于 WebSocket 使用 TCP，数据丢失时需要重传，后续数据可能被阻塞，也就是 Head-of-Line Blocking。因此，浏览器或移动端直接进行低延迟语音时通常优先 WebRTC；可信后端连接实时模型时，WebSocket 往往更容易开发和控制。

### 三、电话场景：SIP + RTP/SRTP

电话机器人通常采用：

```text
电话运营商
   ↓ SIP：建立、挂断和转接通话
语音网关
   ↓ RTP/SRTP：传输实际音频
语音 Agent
```

SIP 负责呼叫建立、振铃、挂断和转接等信令；RTP 负责传输实时音频，SRTP 则增加媒体加密。二者分别对应“控制通话”和“传输声音”。

### 四、为什么语音使用长连接，而普通 MCP 调用不需要？

两者的通信特征不同：

| 对比项 | MCP 工具调用 | 实时语音 |
| --- | --- | --- |
| 数据形态 | 离散的 JSON 请求和结果 | 连续产生的音频帧 |
| 通信频率 | 相对较低 | 通常每 10～20 毫秒一个音频块 |
| 交互方向 | 主要是请求—响应 | 持续双向传输 |
| 核心目标 | 正确性、可审计和工具互操作 | 低延迟、连续播放和及时打断 |
| 常用传输 | stdio、Streamable HTTP | WebRTC、WebSocket、RTP |

语音对话过程中，用户和模型可能同时产生事件，系统还要快速检测用户打断模型，因此维持一个全双工长连接很有价值。MCP 工具调用通常是偶尔发生的离散操作，没有必要使用专门的实时媒体通道。

### 五、语音 Agent 与 MCP 如何配合？

一套完整架构通常是：

```text
用户
  ↕ WebRTC
实时语音 Agent
  ↕ stdio / Streamable HTTP
MCP Server
  ↓
数据库、搜索、日历等外部系统
```

各部分的职责是：

- WebRTC 或 WebSocket 负责实时音频、转写事件和打断；
- 实时模型负责理解语音、推理和决定是否调用工具；
- MCP 负责发现和调用后端工具；
- 普通 HTTP 和 OAuth 可以负责认证、配置及非实时管理。

不建议将连续音频切成大量 JSON 后通过 MCP `tools/call` 传输。这样会增加 Base64、JSON 序列化、请求调度和模型工具循环的开销，而且 MCP 没有定义音频抖动缓冲、丢包恢复、回声消除和媒体拥塞控制等能力。

简单来说：**浏览器和移动端实时语音通常使用 WebRTC，服务端连接实时语音模型常用 WebSocket，电话系统使用 SIP + RTP/SRTP。它们负责低延迟的连续音频流，MCP 则继续负责语音 Agent 背后的离散工具调用。**

参考：[W3C WebRTC 标准](https://www.w3.org/TR/webrtc/)、[WebRTC 连接机制](https://webrtc.org/getting-started/peer-connections)、[RTP RFC 3550](https://www.rfc-editor.org/info/rfc3550/)、[WebSocket RFC 6455](https://www.rfc-editor.org/info/rfc6455/)

## 9. 大模型网关使用什么通信方式？为什么需要 LLM Gateway？

LLM Gateway（大模型网关）位于业务应用和一个或多个模型服务之间，为上层提供统一的模型调用入口，并集中处理路由、认证、限流、成本、安全和可观测性等横切能力。

一个典型链路是：

```text
客户端 / Agent
      ↓ HTTPS + JSON
    LLM Gateway
      ↓ 供应商或自建模型的 API
OpenAI / Anthropic / Gemini / 自建模型
      ↑
    JSON 或 SSE 流式响应
```

### 一、大模型网关通常采用什么通信方式？

不同场景通常采用：

| 场景 | 常用通信方式 |
| --- | --- |
| 普通文本、Embedding 和图片请求 | HTTPS + JSON |
| 流式文本生成 | HTTPS + SSE |
| 实时语音或持续双向事件 | WebRTC 或 WebSocket |
| 企业内部服务和推理集群 | HTTP/2、gRPC 或内部 RPC |
| 多供应商统一调用 | OpenAI-compatible API 格式较常见 |

很多企业会对业务提供统一接口，例如：

```http
POST /v1/chat/completions
POST /v1/responses
POST /v1/embeddings
```

业务只使用逻辑模型名：

```json
{
  "model": "company-general",
  "messages": [
    {
      "role": "user",
      "content": "解释一下 MCP"
    }
  ]
}
```

网关再将 `company-general` 路由到 OpenAI、Gemini、自建 vLLM 或其他实际部署，并完成供应商请求及响应格式转换。

网关没有统一的实现方式。常见选择包括 LiteLLM Proxy、Kong AI Gateway、Cloudflare AI Gateway、Envoy AI Gateway，以及基于 Nginx、Envoy、APISIX、Higress 或企业基础设施构建的自研网关。

### 二、为什么不让业务直接调用模型？

小型 Demo 可以直接调用模型 API，但当系统进入多模型、多业务和多租户的生产环境时，分散直连会产生接口耦合、密钥泄漏、成本失控、重复治理和故障难以统一处理等问题。LLM Gateway 提供一个集中控制点。

### 三、统一模型接口与凭证

不同供应商的请求格式、模型名称、认证方式、错误码和流式事件可能不同。网关可以：

- 对业务提供统一的请求和响应格式；
- 将统一格式转换成各供应商格式；
- 集中保存并轮换供应商 API Key；
- 避免每个业务分别实现供应商 SDK；
- 降低模型切换和供应商迁移成本。

统一接口不等于必须抹平所有模型特性。对于推理强度、Prompt Cache、Realtime、原生工具等供应商特有能力，网关还需要允许扩展字段或提供专用接口，否则容易被“最小公共能力”限制。

### 四、模型路由、负载均衡和故障降级

网关可以根据任务、模型能力、成本、延迟、区域和服务状态动态选择模型：

```text
简单分类任务 → 小模型
复杂推理任务 → 推理模型
敏感数据任务 → 私有部署模型
主模型限流或故障 → 备用模型
欧洲用户 → 欧洲区域部署
```

常见能力包括：

- 多个模型部署之间的负载均衡；
- 按租户、业务或数据区域路由；
- 按成本、质量或延迟选择模型；
- 灰度发布、A/B 测试和模型版本切换；
- 超时、有限重试、熔断和 Fallback；
- 根据 Context Length 和并发容量选择部署。

需要注意，流式响应开始返回后通常不能随意切换到另一个模型重试，否则可能向用户返回重复、缺失或互相矛盾的半段答案。安全的重试通常发生在首 Token 返回之前，或者由上层明确丢弃原流并重新开始。

### 五、Token 级限流与预算控制

普通 API Gateway 常按 QPS 限流，但两个模型请求的成本可能相差几个数量级：

```text
请求 A：输入和输出共 100 Token
请求 B：输入和输出共 100,000 Token
```

因此，LLM Gateway 还需要根据以下维度进行限制：

- 每分钟输入或输出 Token 数；
- 同时进行的模型生成数量；
- 单次请求最大输入、输出和上下文长度；
- 用户、Agent、项目和租户预算；
- 指定模型或供应商的调用额度；
- 单次 Agent Run 的总 Token、总次数和总费用；
- 重试与 Fallback 的额外次数及成本上限。

网关可以向业务发放 Virtual Key，将多个业务的权限、配额和费用归属与底层供应商 Key 解耦。

### 六、成本统计

大模型计费通常与模型、输入 Token、输出 Token、缓存 Token、图片、音频和工具调用有关。网关可以统一记录：

- 用户、项目、部门和租户的模型用量；
- 每个模型、供应商和区域的成本；
- Prompt Cache 命中及节省的费用；
- 重试和 Fallback 产生的额外成本；
- 每个 Agent 任务的端到端费用；
- 月度预算、异常消费和费用趋势。

由于不同供应商价格及计费口径会变化，网关的价格表需要版本化并持续更新，不能只依赖静态常量。

### 七、可观测性

普通 API 主要观察 QPS、错误率和响应时间；模型调用还需要关注：

- **TTFT（Time to First Token）**：首 Token 延迟；
- **TPOT（Time per Output Token）**：输出一个 Token 的平均耗时；
- **TPS（Tokens per Second）**：模型生成速度；
- 输入、输出、推理和缓存 Token；
- Finish Reason、截断、拒答和内容审核结果；
- 工具调用次数和 Agent 迭代次数；
- 超时、429、重试、Fallback 和熔断次数；
- 每次请求及整个任务的费用。

网关是统一采集这些指标、生成 Trace ID 和关联业务请求的合适位置。但 Prompt、响应和工具结果可能包含敏感信息，日志应进行脱敏、权限隔离和保留期限控制。

### 八、安全、权限和合规

LLM Gateway 可以集中实施：

- API Key、JWT、OAuth、mTLS 和服务身份认证；
- 按用户、租户和业务限制可使用的模型；
- 敏感信息检测、脱敏和 DLP；
- 输入和输出内容审核；
- 数据区域、供应商和模型白名单；
- 审计日志、数据保留和合规策略；
- 隐藏供应商凭证，避免业务代码直接持有密钥。

但网关不能单独解决所有 Prompt Injection、工具越权和结果真实性问题。Agent Runtime 仍然需要对工具权限、外部内容、用户确认和业务操作进行控制。

### 九、缓存与性能优化

网关可以提供：

- 完全相同请求的精确缓存；
- Prompt Cache 的统一接入和统计；
- Embedding 结果缓存；
- 谨慎使用的语义缓存；
- 请求合并、批量 Embedding 和连接复用；
- Prompt 压缩和上下文大小限制。

语义缓存不能只比较问题相似度，还必须考虑用户身份、权限、System Prompt、模型版本、工具定义、知识库版本和时效性，否则可能返回错误或越权答案。

### 十、LLM Gateway 与普通 API Gateway 有什么区别？

| 普通 API Gateway | LLM Gateway |
| --- | --- |
| 主要按请求数限流 | 还按 Token、费用和生成并发限流 |
| 观察响应时间和状态码 | 还观察 TTFT、TPOT、TPS 和 Token |
| 普通 JSON Response | 需要正确代理 SSE 和实时流 |
| 主要根据 URL 路由 | 还根据模型能力、成本和任务路由 |
| 普通响应缓存 | 需要处理 Prompt Cache 和语义缓存 |
| 统计请求次数和流量 | 统计模型、Token、租户和任务成本 |
| 普通请求重试 | 流式输出开始后的重试需要特别谨慎 |
| 不理解 Prompt 和模型 | 需要理解模型参数、Context Length 和供应商差异 |

LLM Gateway 通常是在传统 API Gateway 或反向代理能力之上，增加模型语义、供应商适配和 Token 成本治理，而不是完全替代普通网关。

### 十一、所有项目都需要大模型网关吗？

- **单模型、低流量 Demo**：可以直接调用模型 API，不必过早引入网关。
- **多业务、多模型或多租户系统**：适合使用统一网关。
- **有预算、安全、审计和高可用要求的企业**：通常需要网关。
- **自建大规模推理集群**：需要网关完成路由、负载均衡和容量保护。

网关自身处于所有模型请求的关键路径，也可能成为性能瓶颈和单点故障。因此应尽量无状态部署、运行多个副本，配置超时、熔断和降级，并保证在网关异常时有明确的故障处理策略。

简单来说：**当前 LLM Gateway 对外通常提供 HTTPS + JSON 或 OpenAI-compatible API，流式文本使用 SSE，实时语音使用 WebRTC 或 WebSocket。它位于业务和模型服务之间，统一处理接口适配、模型路由、Token 限流、成本、可观测性、安全和故障降级，是大模型从 Demo 进入多模型、多租户生产环境后的重要基础设施。**

参考：[LiteLLM Gateway](https://docs.litellm.ai/)、[Kong AI Gateway](https://docs.konghq.com/gateway/latest/ai-gateway/)、[Cloudflare AI Gateway](https://www.cloudflare.com/developer-platform/products/ai-gateway/)
