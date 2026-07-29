# 从零学会Agent开发（六）：LangChain 与 LangGraph 入门——当手写不够用了

> **目标读者：** 昨天手写完 ReAct Agent，今天引入框架。理解 LangChain 提供什么、LangGraph 解决什么、什么时候该用它。

---

![[illustration-1.png]]

## 1. 回顾：手写 Agent 三个真实的痛点

Day 05 我们一笔一画写了一个完整的 ReAct Agent。~130 行代码，跑起来没问题。但当你开始加第二个工具、支持多轮持久化、处理各种异常时，代码开始膨胀。

痛点不是凭空说的，是真实踩出来的。

### 痛点一：消息管理越来越像手搓协议

你需要手动拼 message 列表、手动判断 `role`、手动区分 `tool_calls` 和 `tool_result`：

```python
# 手写版：每次都要手动拼消息
messages.append({"role": "assistant", "content": None, "tool_calls": [...]})
messages.append({"role": "tool", "tool_call_id": "...", "content": "..."})
```

两个工具还好，十个工具的时候，这段代码散落在循环的每个分支里。更棘手的是，不同 LLM provider（OpenAI、Anthropic、DeepSeek）对消息格式的要求还不一样。

### 痛点二：状态追踪靠全局变量

Day 05 的 Agent 只有一个 `messages` 列表。但实际场景中你可能需要追踪：
- 当前迭代轮数（防止 LLM 死循环）
- 中间搜索结果（后面节点需要引用）
- 用户偏好（记住"我人在北京"）

手写方案下，这些都靠函数传参或全局变量。加到 5 个变量时，你已经不知道哪个函数改了哪个。

### 痛点三：错误处理分散在多个位置

工具调用可能失败。LLM 可能返回格式错误。网络超时需要重试。手写版里 try-except 散落在 `call_tool()`、`llm_think()`、主循环三个地方。加一种错误类型，要改三处。

框架要解决的不是"能不能写"，而是"改了还能不能维护"。

---

## 2. LangChain 的角色：统一接口层

LangChain 和 LangGraph 的关系经常被搞混。用一句话定位：

> LangChain 是工具箱（零件），LangGraph 是流水线（传送带）。你用 LangChain 的零件，搭在 LangGraph 的流水线上。

LangChain 提供了三样东西，让 Day 05 的代码能砍掉一半。

### 2.1 统一的 LLM 接口

不管你用 OpenAI、Anthropic、DeepSeek 还是本地模型，调用方式都一样：

```python
from langchain_openai import ChatOpenAI

# 换模型只需改初始化和 base_url
model = ChatOpenAI(model="gpt-4o-mini", temperature=0)
response = model.invoke([HumanMessage(content="你好")])
```

Agent 代码零改动换模型提供商。

### 2.2 Tool 抽象：Python 函数即工具

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息"""
    weather_data = {"北京": "32°C，晴", "上海": "28°C，小雨"}
    return weather_data.get(city, f"未找到 {city} 的天气数据")
```

`@tool` 一行装饰器——函数名即工具名，docstring 即工具描述，类型注解自动生成 JSON Schema。手写版那几十行参数定义，一行不用写了。

### 2.3 消息格式标准化

```python
from langchain_core.messages import (
    HumanMessage,    # 用户消息
    AIMessage,       # 模型回复（含 tool_calls）
    ToolMessage,     # 工具执行结果
    SystemMessage,   # 系统提示
)
```

四种消息类型覆盖所有场景，框架内部自动处理 OpenAI / Anthropic 序列化差异。

LangChain 有一个根本局限：它不提供执行循环。什么时候调 LLM、什么时候调工具、结果怎么喂回 LLM——这套 Agent 循环需要你自己写。这就是 LangGraph 要做的事。

---

## 3. LangGraph 核心理念：State + Node + Edge

LangGraph 给了你三个概念（来源：[LangGraph Graph API 文档](https://docs.langchain.com/oss/python/langgraph/graph-api)）：

> State（状态）→ Node（节点）→ Edge（边）

![[个人/agent-dev-30days/day-06/illustration-2.png]]

### 3.1 State：Agent 的工作台

State 用 TypedDict 定义，是整个 Agent 在任何时刻的完整快照。每个 key 可以带 reducer 控制合并策略：

```python
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # add_messages reducer：追加而不覆盖，消息历史自动累积
    messages: Annotated[list, add_messages]
    # 普通字段：谁写谁覆盖
    iteration_count: int
```

State 是全局唯一的，每个 Node 接收完整 State，只返回要更新的字段，框架自动合并。

### 3.2 Node：纯函数

节点就是 Python 函数，签名 `(State) -> dict`。返回值是 State 的部分更新：

```python
def call_model(state: AgentState) -> dict:
    """调用 LLM，返回的 message 由 add_messages 自动追加"""
    response = model_with_tools.invoke(state["messages"])
    return {"messages": [response]}  # 只返回改了的部分
```

每个节点只做一件事：调 LLM、执行工具、做路由判断。手写版里裹在大循环中的逻辑，被拆成独立可测试的函数。

### 3.3 Edge：决定下一步

普通边（`add_edge`）：固定流转，A 之后必定到 B。

条件边（`add_conditional_edges`）：根据 State 决定去哪，这是 Agent 循环的核心：

```python
def should_continue(state: AgentState) -> str:
    """检查 LLM 输出是否有 tool_calls，决定下一步"""
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"    # → 去执行工具
    return "__end__"      # → 结束
```

ReAct Agent 的状态机：

```
START → [agent 节点]
              │
              ▼
     should_continue 判断
        ╱            ╲
   有 tool_calls   无 tool_calls
       │                │
       ▼                ▼
   [tools 节点]        END
       │
       └──→ 回到 [agent 节点]（形成循环）
```

这个"判断 → 分支 → 循环"的结构，在 LangGraph 里由 4 行 `add_node` + `add_edge` 表达。对比手写版 20 行的 `while True`，控制流从隐式变成了显式。

---

## 4. 完整代码：用 LangGraph 重写 Day 5 的 Agent

安装依赖：

```bash
pip install langgraph langchain-openai langchain-core
```

### 4.1 LangGraph 版（核心逻辑 ~70 行）

```python
"""
Day 06: 用 LangGraph 重写 Day 05 的 ReAct Agent
核心逻辑 ~70 行 vs 手写版 ~130 行，减少约 46%
"""
import os
from typing import TypedDict, Annotated, Literal
from langgraph.graph.message import add_messages

from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from langchain_core.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.memory import MemorySaver

# ==================== 1. 定义工具 ====================
# @tool 装饰器自动从函数签名和 docstring 生成 tool schema
# 不需要手写 JSON Schema

@tool
def get_weather(city: str) -> str:
    """获取指定城市的天气信息"""
    weather_data = {
        "北京": "32°C，晴",
        "上海": "28°C，小雨",
        "深圳": "30°C，多云",
        "杭州": "29°C，阴",
    }
    return weather_data.get(city, f"未找到 {city} 的天气数据")

@tool
def calculator(expression: str) -> str:
    """计算数学表达式，支持 +、-、*、/ 和括号"""
    try:
        # 安全计算：用空 __builtins__ 限制可调用函数
        result = eval(expression, {"__builtins__": {}}, {})
        return f"计算结果：{result}"
    except Exception as e:
        return f"计算出错：{e}"

tools = [get_weather, calculator]

# ==================== 2. 初始化 LLM 并绑定工具 ====================
model = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY"),
)
# bind_tools 将工具的 JSON Schema 注入 LLM 调用上下文
model_with_tools = model.bind_tools(tools)

# ==================== 3. 定义 State ====================
class AgentState(TypedDict):
    """Agent 共享状态。messages 使用 add_messages reducer 追加而非覆盖。"""
    messages: Annotated[list, add_messages]

# ==================== 4. 定义节点函数 ====================
SYSTEM_PROMPT = SystemMessage(content="你是一个有用的助手。可以使用工具来回答问题。")

def call_model(state: AgentState) -> dict:
    """Agent 节点：调 LLM 推理下一步。返回的消息经 add_messages 追加到历史。"""
    messages = [SYSTEM_PROMPT] + state["messages"]
    response = model_with_tools.invoke(messages)
    return {"messages": [response]}

# ToolNode 是 LangGraph 内置的工具执行节点
# 自动读取最后一条 AIMessage 中的 tool_calls，执行对应的 Python 函数
tool_node = ToolNode(tools)

# ==================== 5. 定义路由器（条件边） ====================
def should_continue(state: AgentState) -> Literal["tools", "__end__"]:
    """如果 LLM 要调工具 → tools 节点；否则 → 结束。"""
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return "__end__"

# ==================== 6. 组装图 ====================
workflow = StateGraph(AgentState)

workflow.add_node("agent", call_model)
workflow.add_node("tools", tool_node)

workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {"tools": "tools", "__end__": END},
)
workflow.add_edge("tools", "agent")  # ← 工具结果回到 LLM，形成循环

# 编译图（不带 checkpointer 的最简版）
agent = workflow.compile()

# ==================== 7. 运行 ====================
if __name__ == "__main__":
    result = agent.invoke(
        {"messages": [HumanMessage(content="北京今天天气怎么样？")]}
    )
    print(f"🤖 {result['messages'][-1].content}")
```

### 4.2 逐段对比：手写版 vs LangGraph 版

| 模块 | 手写版（Day 05） | LangGraph 版（Day 06） | 省了什么 |
|------|-----------------|----------------------|---------|
| 工具定义 | 手写 JSON Schema（~15 行/工具） | `@tool` 装饰器（~8 行/工具） | Schema 自动生成，参数校验自动 |
| 消息管理 | 手写 `{"role": ...}` 字典拼接 | `HumanMessage` / `AIMessage` / `ToolMessage` | 跨 provider 兼容，reducer 自动合并 |
| 主循环 | 手写 `while True` + `break`（~20 行） | 条件边 `should_continue`（5 行） | 循环逻辑从业务代码中解耦 |
| 工具执行 | 手写 if-elif 分支调用（~15 行） | `ToolNode(tools)`（1 行） | 自动匹配工具名、错误处理、结果封装 |
| 状态管理 | `messages = []` 全局变量 | `AgentState(TypedDict)` | 类型安全，checkpoint 就绪 |
| 加新工具 | 改 3 处（schema + 函数 + 映射表） | 添 `@tool` 函数 + 加入 tools 列表 | 2 处 vs 3 处，且不涉及循环逻辑 |
| 总代码量 | ~130 行 | ~70 行 | 减少约 46% |

核心差异不全在代码量。手写版的 `while` 循环需要读完整个函数体才能理解跳转逻辑；LangGraph 版把跳转提取为 `should_continue` 路由函数，一眼就能看懂状态机。

来源：[LangGraph ReAct Agent 官方示例](https://ai.google.dev/gemini-api/docs/langgraph-example) 和 [LangGraph Tutorial v1.0 API](https://agentsindex.ai/blog/langgraph-tutorial)。

---

## 5. Checkpointing 入门：对话中断后从断点恢复

每次 super-step（节点执行完成的时刻）LangGraph 自动保存 State 快照（来源：[LangGraph Persistence 文档](https://docs.langchain.com/oss/python/langgraph/persistence)）。

### 5.1 基础用法

```python
from langgraph.checkpoint.memory import MemorySaver

# 加上 checkpointer 再看区别
memory = MemorySaver()
agent_with_memory = workflow.compile(checkpointer=memory)

# 每个对话用唯一的 thread_id 标识
config = {"configurable": {"thread_id": "user-session-001"}}

# 第一次对话
agent_with_memory.invoke(
    {"messages": [HumanMessage(content="我叫张三，在北京工作")]},
    config,
)

# 第二次对话 —— 同一个 thread_id，自动恢复消息历史
result = agent_with_memory.invoke(
    {"messages": [HumanMessage(content="我刚才说我叫什么？")]},
    config,
)
print(result["messages"][-1].content)
# → "你刚才说你是张三，在北京工作。"  ← 记住了，没有 extra 代码
```

你不需要写任何序列化/反序列化。每次 `invoke` 结束后，框架自动把完整 State（包括 messages 列表）存到 checkpointer。下次用相同 `thread_id` 调用时，自动从最近的 checkpoint 恢复。

### 5.2 三种 Checkpointer 怎么选

| Checkpointer | 持久化 | 适用场景 |
|-------------|--------|---------|
| `MemorySaver` | 进程内存，重启消失 | 开发/调试 |
| `SqliteSaver` | 本地 .db 文件 | 本地持久化，单机服务 |
| `PostgresSaver` | PostgreSQL 数据库 | 生产环境，多进程共享，崩溃恢复 |

```python
# 生产环境示例
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:pass@localhost:5432/agent_db"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()  # 自动建表
    graph = workflow.compile(checkpointer=checkpointer)
    # 之后所有 invoke 都自动写入 checkpoint
```

### 5.3 状态回溯（Time Travel）

```python
# 查看对话的所有历史状态
for snapshot in agent_with_memory.get_state_history(config):
    step = snapshot.metadata.get("step", "?")
    last_msg = snapshot.values["messages"][-1]
    print(f"Step {step}: {last_msg}")

# 回到之前的某个状态，重新执行
agent_with_memory.update_state(config, previous_state.values)
```

这个能力对调试极其有用。你可以回退到 Agent 做某个错误决定之前的状态，修改输入，看它会走什么不同的路径。

---

## 6. LangGraph 与其他框架的定位对比

以下是基于多份独立分析整理的选型参考（来源：[DevToolLab](https://devtoollab.com/blog/langgraph-vs-crewai-vs-autogen)、[ODSEA](https://odsea.com/blog/langgraph-vs-crewai-vs-autogen-production)、[DecodeTheFuture](https://decodethefuture.org/en/langgraph-vs-crewai-vs-autogen-2026/)）。

### 6.1 一句话定位

| 框架 | 一句话 | 类比 |
|------|-------|------|
| LangChain | 工具箱：LLM 接口 + 消息格式 + Tool 抽象 | 瑞士军刀 |
| LangGraph | 控制流引擎：状态机图的执行器，内置持久化 | 流水线传送带 |
| CrewAI | 角色协作框架：用"团队"隐喻组织 Agent | 项目经理分配任务 |
| AutoGen（AG2） | 对话式多 Agent：Agent 间通过消息协商 | 圆桌会议 |
| MS Agent Framework | AutoGen 的正式继任者，Azure 原生 | 微软全家桶底座 |

### 6.2 什么时候该用谁

| 场景 | 推荐 | 原因 |
|------|------|------|
| 简单问答/单次 LLM 调用 | LangChain LCEL | 图是多余的 |
| 多步推理 + 工具调用 | LangGraph | 条件边天然支持 Tool ↔ LLM 循环 |
| 对话断点恢复 | LangGraph | 内置 checkpointing，零额外代码 |
| 人工审批节点 | LangGraph | `interrupt()` 原生支持暂停-继续 |
| 快速原型 / 内部工具 | CrewAI | 角色隐喻直观，最快 30 行出 demo |
| .NET / Azure 技术栈 | MS Agent Framework | 唯一支持 Python + .NET 的一线框架 |
| 新项目 | 不要碰 AutoGen 原版 | 已进入维护模式，功能冻结 |

### 6.3 一个判断标准

> 如果你的 Agent 控制流画出来是一条直线，用 LangChain。如果画出来有菱形分叉，用 LangGraph。（来源：[Abstract Algorithms](https://www.abstractalgorithms.dev/from-langchain-to-langgraph-when-agents-need-state-machines)）

LangChain 和 LangGraph 不是替代关系，而是同一团队维护的上下层关系。LangGraph 的每个节点内部都可以用 LangChain 的 LLM 接口、消息格式、Retriever。两者是互补的。

---

## 7. 小结

今天完成了从"手写"到"框架"的转变。核心收获：

| 概念 | 一句话 |
|------|-------|
| LangChain | 统一 LLM 接口、Tool 抽象、消息标准化 |
| LangGraph 三要素 | State（TypedDict）+ Node（纯函数）+ Edge（条件流转） |
| Checkpointing | 每次 super-step 后自动存快照，断点恢复 |
| 什么时候用 LangGraph | 流程图里有菱形分叉、需要状态持久化、需要人机协作 |
| LangGraph 与其他框架的关系 | LangGraph 是底层控制引擎，CrewAI/MSAF 是高层封装，不是竞品是互补 |

明天是 Day 07，第一周总结。我们会回顾 Day 01 ~ 06 的核心脉络，做一次完整的知识地图梳理。

---

## 参考资料

1. [LangGraph Tutorial: Build a Working ReAct Agent with the v1.0 API](https://agentsindex.ai/blog/langgraph-tutorial) — AgentsIndex, 2026-04
2. [LangGraph 101: Building Your First Stateful Agent](https://www.abstractalgorithms.dev/langgraph-101-building-your-first-stateful-agent) — Abstract Algorithms, 2026-03
3. [From LangChain to LangGraph: When Agents Need State Machines](https://www.abstractalgorithms.dev/from-langchain-to-langgraph-when-agents-need-state-machines) — Abstract Algorithms, 2026-03
4. [LangGraph ReAct Agent Pattern in LangGraph](https://www.abstractalgorithms.dev/langgraph-react-agent-pattern) — Abstract Algorithms, 2026-03
5. [ReAct agent from scratch with Gemini and LangGraph](https://ai.google.dev/gemini-api/docs/langgraph-example) — Google AI for Developers
6. [LangGraph Persistence 文档](https://docs.langchain.com/oss/python/langgraph/persistence) — LangChain 官方文档
7. [Graph API overview](https://docs.langchain.com/oss/python/langgraph/graph-api) — LangChain 官方文档
8. [LangGraph vs CrewAI vs AutoGen 2026 Comparison](https://devtoollab.com/blog/langgraph-vs-crewai-vs-autogen) — DevToolLab, 2026-06
9. [LangGraph vs. CrewAI vs. AutoGen: Which Ships to Production?](https://odsea.com/blog/langgraph-vs-crewai-vs-autogen-production) — ODSEA, 2026-05
10. [LangGraph vs CrewAI vs AutoGen (2026): Which to Pick](https://decodethefuture.org/en/langgraph-vs-crewai-vs-autogen-2026/) — DecodeTheFuture, 2026-06
