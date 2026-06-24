---
tags: [agent, tutorial, 30days]
day: 1
---

# 从零学会Agent开发（一）：什么是 AI Agent？从 Chatbot 到 Agent 的进化



Agent 是一个**会自己动手干活**的 AI 系统——它能用工具、有记忆、会规划，在循环中自主完成多步任务。Chatbot 动嘴，Agent 动手。

![[杂记/agent-dev-30days-release/day-01/illustration-2.png]]

## 进化三部曲：从问答到执行

2022 年底 ChatGPT 横空出世时，所有人都在惊叹"它居然能理解我说的话"。你写一句，它回一段，像魔法一样。但很快你会发现一个尴尬的事实——它能告诉你"去三亚的机票怎么买"，却没法真的帮你买。它能写出完美的 SQL 查询语句，却没法连接数据库执行它。它是个知识渊博的"嘴"，没有"手"。

这个 gap——理解任务和完成任务之间的距离——是整个 Agent 浪潮的起点。

### 第一阶段：纯 Chatbot（一问一答）

这是最早也是最基本的形态。用户发一条消息，LLM 基于训练数据和上下文返回一条回复。每次对话都是无状态的——模型不记得上一轮你说了什么（除非你手动把历史塞进 prompt），更不可能操作任何外部系统。

架构极简：`用户输入 → LLM 推理 → 文本输出`。这类系统适合闲聊、信息查询、文案润色，但面对 "帮我查一下今天北京飞上海的机票" 这种需要连接外部数据的请求时，直接哑火。

### 第二阶段：工具增强型 LLM（一次调用一个工具）

2023 年下半年，OpenAI 推出了 Function Calling，Anthropic 推出了 Tool Use。模型不再只能输出文字——它可以输出一个结构化的"函数调用请求"，告诉你的代码："调用 `search_flights` 函数，参数是 `{from: '北京', to: '上海', date: '2026-06-23'}`。"

你的代码执行这个函数，把结果塞回对话上下文，模型再基于结果生成人类可读的回复。模型第一次有了"手"。但局限性也明显：它只会用一次工具，用完就停。如果搜索结果需要过滤、排序、二次查询，它不会自己继续。

### 第三阶段：Agent（自主循环决策）

到了 2024-2025 年，大家逐渐意识到：把一次工具调用变成**循环**，让模型在每轮看到工具结果后继续决策"下一步做什么"，直到任务完成——这就是 Agent。

这就是 ReAct（Reasoning + Acting）模式的核心：模型**推理**当前状态 → 决定**执行**什么动作 → **观察**执行结果 → 再次**推理**……如此循环，直到目标达成或触发停止条件。

一个典型的 Agent 循环：
```
目标："帮我查一下张三的订单状态，如果是待发货就取消它"
→ 推理：需要先查到张三的用户ID
→ 动作：调用 query_user(name="张三")
→ 观察：得到 user_id=12345
→ 推理：现在查订单
→ 动作：调用 query_orders(user_id=12345)
→ 观察：订单 #6789，状态=待发货
→ 推理：符合取消条件
→ 动作：调用 cancel_order(order_id=6789)
→ 观察：取消成功
→ 完成
```

## 什么是 Agent？一个正式定义

综合 Anthropic、IBM、Google 等机构的定义以及 2026 年行业的共识，可以这样定义 AI Agent：

> **AI Agent 是一个以 LLM 为推理核心的自主软件系统，它能够感知环境、制定计划、调用工具执行动作、观察执行结果，并在循环中持续调整策略，直到完成既定目标或达到停止条件。**

拆开来看，Agent 具备四个核心特征：

- **自主性（Autonomy）**：Agent 不是按固定脚本执行，而是在目标驱动下自行决定每一步做什么。给它"帮我预订明天下午的会议室"，它会自己规划"查日程→找空闲会议室→预约"这个流程，不需要你逐步指导。
- **反应性（Reactivity）**：Agent 能感知环境变化并做出响应。工具调用失败？网络超时？返回结果不符合预期？它应该检测到这些情况并调整行为，而非闷头走到底。
- **主动性（Proactivity）**：Agent 不仅被动响应，还可以主动采取行动推进目标。比如在发现第一次查询返回空结果时，它可能主动换一个搜索关键词再试，而不是直接回复"没找到"。
- **社交能力（Social Ability）**：Agent 可以跟人类交互（请求澄清、汇报进度）、跟其他 Agent 协作（多 Agent 系统）、跟外部系统交互（通过工具层）。这不只是"聊天"，而是有目的的信息交换。

## 四大核心组件：拆开一个 Agent

每个 Agent 无论多复杂，底层都由四个组件构成。

![[杂记/agent-dev-30days-release/day-01/illustration-3.png]]

### 大脑（LLM —— 推理引擎）

LLM 是 Agent 的认知中枢，负责所有需要"理解"和"判断"的环节：解析用户意图、拆解复杂目标为子步骤、在每一步决定调用哪个工具、评估工具返回的结果是否满足预期、生成最终的回复文本。

LLM 本身不执行任何工具、不存储记忆、不管理状态。它是一台推理机器——给它输入（上下文），它就输出（文本或函数调用请求）。Anthropic 在《Building Effective Agents》中强调：LLM 需要被增强（augmented LLM），包装好工具接口、放进循环里，才构成 Agent。

选择 LLM 时需要考虑三个维度：推理能力（能否处理多步复杂逻辑）、工具调用准确率（能否正确选择工具和参数）、上下文窗口大小（能容纳多少历史信息）。2026 年的前沿模型在 5-10 个工具场景下的函数调用准确率可达 85-92%，但工具数量涨到 20+ 时会掉到 65-78%，这是实际开发中需要重点关注的瓶颈。

### 手脚（Tools —— 执行层）

工具是 Agent 连接外部世界的接口。没有工具，Agent 就是纯文本生成器；有了工具，它能查询数据库、调用 API、读写文件、执行代码、发送邮件、搜索网页。工具本质上就是一个带 Schema 的函数——你告诉模型函数的名称、用途和参数格式，模型决定何时调用及传什么参数。

工具分为两大类：
- **读取型工具**：只获取信息，不改变外部状态。比如搜索网页、查询数据库、读取文件。风险较低，适合做 Agent 的第一个工具。
- **写入型工具**：会改变外部状态。比如发送邮件、修改数据库、提交代码。这类工具需要额外的安全把控——权限校验、人工确认、操作日志、回滚机制缺一不可。

Anthropic 的建议是：花精力打磨工具的文档描述（ACI，Agent-Computer Interface），因为模型对工具的理解完全依赖你的描述质量。一个好的工具描述应该包含：功能说明、每个参数的含义和约束、返回值格式、典型使用场景。写得越清晰，模型用错的概率越低。

### 记忆（Memory —— 状态管理）

记忆决定了一个 Agent 能"记住"什么。LLM 本身是无状态的——每轮 API 调用都像第一次见面，你不在 prompt 里告诉它的它就不知道。记忆系统就是把这个状态显式管理起来的方案。

按生命周期分三层：
- **短期记忆（Working Memory）**：当前对话的完整历史。每次和 LLM 交互时，你需要把之前的对话轮次（包括工具调用和结果）全部放进消息列表。这是最基础、也最消耗上下文窗口的记忆形式。
- **长期记忆（Long-term Memory）**：跨会话的持久化信息。用户偏好、历史决策、学到的经验，通常存在向量数据库或关系数据库中，按需检索注入上下文。比如 Agent 记住了"张三是 VIP 客户，不要向他推荐打折商品"。
- **情节记忆（Episodic Memory）**：记录了"我们之前做这个任务时发生了什么"——成功了还是失败了？哪个工具参数好使？这对于自我改进型 Agent 尤其关键。

记忆系统的核心问题是**什么该记、什么该忘**。全记会爆上下文，全忘就退化成 Chatbot。只记录对后续决策有影响的信息，其余丢掉。

### 规划（Planning —— 决策层）

规划组件负责把一个高层目标拆成可执行的步骤，并在执行过程中动态调整。纯 LLM 输出是"想到哪说到哪"，而规划要求结构化地思考"先做什么、再做什么、如果失败怎么办"。

三种主流规划模式：

| 模式 | 工作方式 | 适用场景 |
|------|----------|----------|
| **ReAct** | 推理一步 → 执行一步 → 观察结果 → 再推理。步进式决策 | 大多数通用场景，不确定性强 |
| **Plan-Execute** | 先一次性生成完整计划，再逐步执行 | 结构化任务，步骤间依赖明确 |
| **Hierarchical** | 拆分多级子目标，可能委派给子 Agent | 复杂大型任务，需要多角色协作 |

入门从 ReAct 开始最合适：简单直观、框架支持最好，每一步的推理过程都可观测——你能清楚看到 Agent 当时"想了什么"，调试方便。

## Agent vs 普通 LLM 调用：一张表说清

| 维度 | 普通 LLM 调用 | AI Agent |
|------|-------------|----------|
| **交互模式** | 一问一答，单轮 | 多轮循环，自主推进 |
| **输出类型** | 纯文本 | 文本 + 工具调用 + 状态变更 |
| **能否操作外部系统** | 不能，只管生成 | 能，通过工具层调用 API、数据库、文件系统 |
| **记忆** | 仅当前对话（手动管理） | 短期 + 长期 + 情节记忆，系统化管理 |
| **决策方式** | 基于单次 prompt 推理 | 基于循环中的持续推理 + 环境反馈 |
| **任务复杂度** | 单步回答或生成 | 多步分解、执行、验证 |
| **何时停止** | 输出即停 | 目标达成 / 达到最大步数 / 触发错误 / 人工中断 |
| **典型用例** | 文案润色、代码解释、闲聊 | 自动化运维、智能客服、代码生成+测试+PR |

本质差异：LLM 调用是"描述世界"，Agent 是"改变世界"。

## 一个 Agent 循环的伪代码

看完理论，用一段伪代码直观感受 Agent 的核心循环：

```python
def agent_loop(goal, tools, max_steps=10):
    """
    Agent 主循环：推理 → 行动 → 观察 → 再推理，直到目标达成。
    
    参数:
        goal: 用户给定的目标，如 "帮我查一下北京今天的天气"
        tools: 可用的工具列表，每个工具包含名称、描述和参数 schema
        max_steps: 最大循环步数，防止 Agent 陷入死循环
    """
    # 初始化消息历史，系统提示词设定 Agent 的行为边界
    messages = [
        {"role": "system", "content": "你是一个能使用工具的 AI 助手。分析任务，选择工具，逐步执行，直到任务完成。"},
        {"role": "user", "content": goal}
    ]
    
    step = 0
    while step < max_steps:
        step += 1
        
        # ---- 第1步：推理 ----
        # 将当前消息历史发给 LLM，让它决定下一步做什么
        response = llm.chat(messages, tools=tools)
        
        # ---- 第2步：判断 ----
        # 如果模型认为任务已完成，直接返回文本回复
        if response.has_text and not response.has_tool_call:
            return response.text
        
        # 如果模型请求调用工具，解析出工具名和参数
        if response.has_tool_call:
            tool_name = response.tool_name
            tool_args = response.tool_args
            
            # ---- 第3步：行动 ----
            # 在实际环境中执行这个工具调用
            tool_result = execute_tool(tool_name, tool_args)
            
            # ---- 第4步：观察 ----
            # 把工具调用请求和结果都追加到消息历史中
            messages.append({"role": "assistant", "tool_call": response.tool_call})
            messages.append({"role": "tool", "name": tool_name, "content": tool_result})
            # 循环继续：下一轮 LLM 会看到工具结果，推理下一步
    
    # 达到最大步数仍未完成，返回异常
    raise Exception(f"Agent 在 {max_steps} 步内未能完成任务")
```

关键设计点：
- **`max_steps` 是安全阀**：没有它，一个困惑的 Agent 会永远循环下去，烧光你的 API 预算。
- **每条信息都计入历史**：工具调用的请求和结果都要写回 `messages`，否则 LLM 不知道刚才发生了什么。
- **停止条件不只有"完成"**：超步数、工具报错、用户中断都要兜底。

## Agent 当前能力边界（2026 年真实状况）

知道 Agent 不能做什么往往比知道它能做什么更关键——这直接决定了能不能上线、会不会出事故。以下基于 2026 年行业数据和真实项目经验整理。

### 能做（可靠部署，ROI 已验证）

- **结构化数据提取**：从邮件、PDF、网页中提取结构化字段（如发票金额、合同日期），准确率可稳定在 95%+。
- **工单分级和路由**：自动分类客服工单并按紧急程度分配，多个团队已大规模投产。
- **轻量研究辅助**：结合搜索工具在多信息源之间交叉验证，生成带引用来源的综述报告。
- **代码审查辅助**：扫描 diff、标注潜在 bug、风格问题和安全隐患，如 Claude Code 和 Cursor 的代码审查功能。
- **多步 API 编排**：按固定业务规则串联多个 API 调用（如查用户→查订单→发通知），在条件清晰的场景下表现稳定。

### 做不好（需要人工监督和兜底）

- **长链路自主任务**：超过 10-15 步的复杂任务容易出现错误累积——前一步的小偏差传播到后面会被放大。SWE-Bench Verified 和 GAIA L2-L3 级别的任务完成率仍在 35-55% 区间，长程自主性离可靠还差一截。
- **跨系统数据对账**：在不同系统间做数据一致性校验时，Agent 对 schema 差异的适应性不够，容易出现"看起来对了但实际漏了"的静默失败。
- **模糊任务的自主澄清**：当用户说"帮我优化一下这个系统"这种高度模糊的指令时，Agent 要么贸然执行（可能做错），要么陷入分析瘫痪（什么都不做）。它不像人一样能精准反问关键澄清问题。
- **不可逆操作**：发邮件、提交代码、修改线上数据库——这类操作的"爆炸半径"很大，Agent 当前还不具备足够可靠的风险判断能力。最佳实践是加入人工确认节点。

### 做不了（研究阶段，别指望投产）

- **真正的因果推理**：Agent 本质上是模式匹配，不是因果理解。在训练数据之外的新颖场景中，它的"推理"可能完全偏离正确逻辑。GAIA 基准测试和大规模现实世界泛化数据都清楚地显示了这一上限。
- **开放式跨领域规划**：让一个 Agent 同时处理"安排出差行程 + 准备技术汇报 PPT + 跟进项目风险"这种跨角色、跨领域的组合任务，目前只有研究 demo，不具备生产可靠性。
- **无监督高风险决策**：医疗诊断、法律判决、大额金融交易——这些领域的错误成本太高，Agent 不应该也不被允许在没有人类把关的情况下做出最终决定。ArXiv 上 2026 年的综述论文明确指出，当决策边界模糊时，Agent 的可靠性和可问责性仍是核心挑战。

一个实用的判断框架：评估任务的**新颖性、风险等级、常识依赖度、容错时间窗口**四个维度。任务越常规、风险越低、规则越清晰、容错时间越充裕，就越适合交给 Agent。

## 动手实践：第一个可调用工具的 Agent

下面是一个**完整可运行**的 Python 脚本，演示了 Agent 的核心循环。它调用 Anthropic Claude API，定义了两个工具（获取天气和计算器），展示了 ReAct 模式的完整执行流程。

```python
"""
Day 01 动手实践：一个最简单的 Agent 循环
依赖：pip install anthropic
运行前需要设置环境变量 ANTHROPIC_API_KEY
"""

import json
import os
import textwrap

from anthropic import Anthropic

# ============================================================
# 第1步：初始化 API 客户端
# ============================================================
client = Anthropic(
    api_key=os.environ.get("ANTHROPIC_API_KEY", "your-api-key-here")
)

# ============================================================
# 第2步：定义工具（Tools）
# 每个工具是一个 Python 函数 + 对应的 JSON Schema 描述
# ============================================================

# 工具1：查询天气
def get_weather(city: str) -> str:
    """模拟查询天气的 API（实际项目中替换为真实 API 调用）"""
    # 模拟数据
    weather_data = {
        "北京": "晴天，25°C，湿度40%",
        "上海": "多云，28°C，湿度65%",
        "深圳": "阵雨，30°C，湿度80%",
    }
    return weather_data.get(city, f"未找到 {city} 的天气数据")


# 工具2：简单计算器
def calculate(expression: str) -> str:
    """安全的数学表达式求值"""
    try:
        # eval 有安全风险，这里仅用于演示
        # 生产环境应使用专门的表达式解析库
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"计算出错: {e}"


# 工具的 JSON Schema 描述 —— LLM 靠这些信息来理解和调用工具
TOOLS = [
    {
        "name": "get_weather",
        "description": "查询指定城市的当前天气信息。返回天气状况、温度和湿度。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如 北京、上海、深圳",
                }
            },
            "required": ["city"],
        },
    },
    {
        "name": "calculate",
        "description": "执行数学计算。支持加减乘除和括号。",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 2+3*4",
                }
            },
            "required": ["expression"],
        },
    },
]

# 工具名 → 实际函数的映射表
TOOL_EXECUTORS = {
    "get_weather": get_weather,
    "calculate": calculate,
}

# ============================================================
# 第3步：Agent 主循环 —— 这就是一个最简 ReAct Agent
# ============================================================
def run_agent(user_goal: str, max_steps: int = 8) -> str:
    """
    Agent 主循环。
    
    流程：用户目标 → LLM推理 → 决定调用工具 or 回复
         → 如果调用工具：执行工具 → 结果反馈给 LLM → 再次推理
         → 重复直到 LLM 认为任务完成 或 达到最大步数
    """
    # 构建初始消息列表
    messages = [
        {
            "role": "user",
            "content": user_goal,
        }
    ]

    step = 0
    while step < max_steps:
        step += 1
        print(f"\n{'='*50}")
        print(f"  Step {step}：LLM 推理中...")
        print(f"{'='*50}")

        # 调用 Claude API，附带工具定义
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system=textwrap.dedent("""\
                你是一个能使用工具的 AI 助手。
                分析用户的问题，选择合适的工具来获取信息，然后综合工具结果给出最终答案。
                如果当前信息不够，不要猜测——调用工具获取准确数据。
            """),
            messages=messages,
            tools=TOOLS,
        )

        # ---- 处理响应 ----
        # Anthropic API 的响应中包含多个 content block
        # text block = 文本回复，tool_use block = 工具调用请求
        has_text = False
        has_tool_call = False

        # 先把模型的响应内容加入消息历史
        assistant_content = []
        for block in response.content:
            if block.type == "text":
                assistant_content.append({"type": "text", "text": block.text})
                has_text = True
            elif block.type == "tool_use":
                assistant_content.append({
                    "type": "tool_use",
                    "id": block.id,
                    "name": block.name,
                    "input": block.input,
                })
                has_tool_call = True

        messages.append({"role": "assistant", "content": assistant_content})

        # 情形A：模型只输出文本，没有调用工具 → 任务完成
        if has_text and not has_tool_call:
            print(f"\n  [Agent 完成] 最终回复：")
            final_text = "".join(
                b.text for b in response.content if b.type == "text"
            )
            print(f"  {final_text}")
            return final_text

        # 情形B：模型调用了工具 → 执行工具，把结果反馈回去
        if has_tool_call:
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    tool_name = block.name
                    tool_input = block.input
                    print(f"\n  [工具调用] {tool_name}")
                    print(f"  参数: {json.dumps(tool_input, ensure_ascii=False)}")

                    # 实际执行工具函数
                    executor = TOOL_EXECUTORS.get(tool_name)
                    if executor:
                        exec_result = executor(**tool_input)
                    else:
                        exec_result = f"错误：未知工具 {tool_name}"

                    print(f"  结果: {exec_result}")

                    # 工具执行结果作为 tool_result 消息加入历史
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": exec_result,
                    })

            messages.append({"role": "user", "content": tool_results})
            # 循环回到顶部，LLM 会基于工具结果继续推理

    # 达到最大步数仍未完成
    return f"错误：Agent 在 {max_steps} 步内未能完成任务，请检查目标或增加步数限制。"


# ============================================================
# 第4步：运行 Agent
# ============================================================
if __name__ == "__main__":
    # 示例1：单工具任务
    print("\n" + "=" * 60)
    print("  🧪 测试1：查询天气")
    print("=" * 60)
    result = run_agent("北京今天天气怎么样？")

    # 示例2：多步骤任务
    print("\n" + "=" * 60)
    print("  🧪 测试2：计算+天气（多步骤）")
    print("=" * 60)
    result = run_agent("北京的温度和上海的温度加起来是多少？")
```

### 运行说明

1. **安装依赖**：`pip install anthropic`
2. **设置 API Key**：`export ANTHROPIC_API_KEY="your-key"` (Linux/Mac) 或 `set ANTHROPIC_API_KEY=your-key` (Windows)
3. **运行**：`python agent_day01.py`

### 代码要点解读

- **工具描述是 Agent 的"说明书"**：TOOLS 列表中每个工具的 `description` 字段直接决定了模型能否正确理解和使用它。描述写糊了，Agent 就瞎用。
- **工具结果必须写回消息历史**：很多初学者会漏掉这一步。模型不会"自动知道"工具执行了什么，你必须显式地把结果放进 `messages`。
- **最大步数是安全阀**：`max_steps=8` 防止模型陷入无限循环。实际项目中可以根据任务复杂度调整，但一定设上限，否则你可能一觉醒来发现 API 账单多了几千块。
- **为什么不用 LangChain/框架**：Anthropic 官方建议——先用原生 API 把 Agent 循环跑通，理解了底层机制之后，再根据需求选择框架。框架的价值是帮你管理状态、路由和重试，不是替你理解 Agent 怎么工作。

## 常见踩坑

- **坑1：工具描述写得像 API 文档而不是使用说明**。工具描述是写给 LLM 看的 prompt，不是写给开发者看的。它需要说清"什么情况下用这个工具"而不是"这个工具底层怎么实现"。典型错误："调用 /api/v1/weather 接口，method=GET"——模型不需要知道你的 HTTP 细节，它需要知道"查询指定城市的天气"。
- **坑2：不设最大步数**。Agent 在遇到工具返回异常、信息不完整、或者自身理解偏差时，可能会无限循环。`max_steps` 不只保护钱包，也是防止生产环境中一个 Agent 挂了整个服务跟着挂。
- **坑3：忘了把工具结果写回消息列表**。这是最常见的无声 bug——Agent 调用了工具，下一轮 LLM 却看不到工具返回了什么，于是重复调用同一工具，像个失忆症患者反复问同一个问题。
- **坑4：一把梭上框架**。LangChain、CrewAI、AutoGen 这些框架是帮你省事的，但如果你不理解底层循环，出了 bug 根本无从调试。先手写 50 行原生的 ReAct 循环，吃透之后再上框架，你会比 90% 的开发者更清楚自己的 Agent 在干什么。

## 参考来源
- [Anthropic - Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)
- [Scrimba - How to Build AI Agents: A Developer's Guide in 2026](https://scrimba.com/articles/how-to-build-ai-agents/)
- [AI Agent Rank - AI Agent vs LLM in 2026: What's the Actual Difference?](https://aiagentrank.io/blog/ai-agent-vs-llm-2026)
- [MasterPrompting - Agent Components: Memory, Tools, Planning, and Perception](https://masterprompting.net/learn/agents/agent-components)
- [Gravity - What Can an AI Agent Actually Do? Capabilities and Boundaries in 2026](https://gravity.fast/blog/what-can-an-ai-agent-actually-do/)
- [IBM - The 2026 Guide to AI Agents](https://www.ibm.com/think/ai-agents)
- [AnyCap - What AI Agents Can't Do in 2026: Honest Developer Guide](https://anycap.ai/page/en-US/news/what-ai-agents-cant-do-in-2026)
- [ArXiv - The Path Ahead for Agentic AI: Challenges and Opportunities](https://arxiv.org/pdf/2601.02749v1)
