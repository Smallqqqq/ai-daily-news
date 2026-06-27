---
tags:
  - agent
  - function-calling
  - tool-use
  - llm
day: 3
series: agent-dev-30days
title: "Day 03：Function Calling 深度解析——LLM 如何'使用工具'"
created: 2026-06-23
updated: 2026-06-23
---

# 从零学会Agent开发（三）：Function Calling 深度解析——LLM 如何"使用工具"

![[illustration-1.png]]

## 前言

上一节我们学会了调用 LLM API 进行文本对话。但一个只会"说话"的模型无法查询实时天气、不能读取本地文件、更没法操作数据库。**Function Calling**（函数调用，Anthropic 称为 Tool Use）就是让模型突破这些限制的关键技术。

> **一句话定义**：Function Calling 不是 LLM 在执行函数，而是 LLM 输出一段结构化的"函数调用请求"，由你的代码执行函数并将结果回传给模型，模型再基于真实数据继续推理。

这是 Agent 的**底层原语（primitive）**。理解它的完整生命周期，是构建可靠 Agent 系统的第一步。

---

## 一、Function Calling 的本质：拆穿"魔法"

很多初学者以为 LLM 真的"调用"了函数。事实是：

```
你想象中的：  LLM → 执行 get_weather("北京") → 返回 "25°C"

实际发生的：  LLM → 返回 JSON {"name":"get_weather","arguments":{"city":"北京"}}
              ↓
              你的代码 → 解析 JSON → 调用真实的 get_weather("北京") → 得到 "25°C"
              ↓
              你的代码 → 把 "25°C" 回传给 LLM → LLM 生成自然语言回复
```

**三个关键认知**：

1. **LLM 只做决策**：它决定调用哪个函数、传什么参数，但绝不执行代码。Anthropic 的官方文档强调："The model never executes the function. It decides what to call and with what arguments."（模型从不执行函数，它只决定该调用什么、用什么参数）[1]
2. **你的代码是执行器**：解析 LLM 返回的 JSON，真正调用函数，拿到结果。这是安全性边界的核心——所有有副作用的操作都在你的控制之下[2]
3. **结果回传是闭环**：把函数返回值传回 LLM，它才能基于真实数据继续推理。没有这一步，LLM 就只是一个返回 JSON 的"翻译器"

这个模式被各大 LLM 提供商原生支持。OpenAI 称之为 **Function Calling**，Anthropic 称之为 **Tool Use**，Google 称之为 **Function Declarations**——本质完全相同，仅 API 格式有差异[3]。

---

## 二、完整生命周期：从 Schema 到最终回复

一次完整的 Function Calling 经过七个阶段。下面这张流程图覆盖了整个链路：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Function Calling 完整生命周期                       │
├─────────────┬───────────────────────────────────────────────────────┤
│   步骤       │   发生了什么                                            │
├─────────────┼───────────────────────────────────────────────────────┤
│  ① 定义工具   │   你：编写 JSON Schema，描述函数名、用途、参数类型和约束      │
│  ② 注入请求   │   你：将工具定义 + 系统提示词 + 用户消息一起发送给 LLM       │
│  ③ 模型决策   │   LLM：判断是否需要调用工具；若需要，生成 tool_call JSON     │
│  ④ 解析参数   │   你：从响应中提取 tool_call，解析函数名和参数              │
│  ⑤ 执行函数   │   你：用解析出的参数调用真正的函数，获取结果                  │
│  ⑥ 结果回传   │   你：将函数返回值作为新消息发送给 LLM                      │
│  ⑦ 继续推理   │   LLM：基于函数返回的真实数据，生成最终的自然语言回复         │
│  ⑧ 循环判断   │   你：检查 finish_reason，若仍是 tool_calls 则回到步骤④    │
└─────────────┴───────────────────────────────────────────────────────┘
```

![[个人/agent-dev-30days/day-03/illustration-2.png]]

### 2.1 阶段一：定义工具 Schema

这是整个链路中容易被低估的环节。Schema 写得好不好，直接决定 LLM 能不能正确"理解"你的工具。

一个最小化的工具定义包含三个核心要素[4]：

| 要素 | 作用 | 重要性 |
|------|------|--------|
| `name` | 工具的唯一标识，LLM 用它"点名"调用 | 高 |
| `description` | 用自然语言描述工具功能和适用场景 | **极高**（最重要的字段） |
| `parameters` | JSON Schema 格式的参数定义 | 高 |

### 2.2 阶段二：注入 Prompt

工具定义不是单独发送的，而是随 system prompt 和用户消息一起发给 LLM。不同平台的具体字段略有差异[3]：

| 平台 | 工具定义字段 | 参数 Schema 字段名 | Strict 模式 |
|------|-------------|-------------------|-------------|
| OpenAI | `tools`（外包 `type: "function"` 层） | `parameters` | `strict: True` |
| Anthropic | `tools`（直接顶级定义） | `input_schema` | `strict: True` |
| Google Gemini | `tools` → `function_declarations` | `parameters` | 不支持 |

关于 `tool_choice`，各平台的关键取值[1][5]：

- `"auto"`：LLM 自行决定调不调工具、调哪个（**推荐默认值**）
- `"none"`：强制不调用工具
- `"required"`（OpenAI）/ `{"type": "any"}`（Anthropic）：强制至少调用一个工具
- 指定工具名：强制调用某个特定工具，适合需要确保结构化输出的场景

### 2.3 阶段三至七：LLM 返回 tool_call → 解析 → 执行 → 回传 → 推理

LLM 返回的不是普通文本，而是一个 **tool_calls 数组**（OpenAI）或包含 **tool_use content blocks** 的响应（Anthropic）：

```json
// OpenAI 格式 —— 注意 arguments 是 JSON 字符串（需要 json.loads 解析）
{
  "id": "call_abc123",
  "type": "function",
  "function": {
    "name": "get_weather",
    "arguments": "{\"city\": \"北京\"}"
  }
}

// Anthropic 格式 —— input 已经是解析好的 dict（不需要再 json.loads）
{
  "type": "tool_use",
  "id": "toolu_01Abc123",
  "name": "get_weather",
  "input": {"city": "北京"}
}
```

你的代码从 tool_call 中取出函数名和参数，路由到真正的业务函数执行，然后将结果回传。**关键细节**：必须保留完整对话历史——原始用户消息 + LLM 的 tool_call 响应 + 工具执行结果——全部放入 messages 数组。缺少任何一环，LLM 都可能"失忆"[1][5]。

---

## 三、JSON Schema 设计实战

Schema 写得好，LLM 调用工具的准确率可以从 60% 提升到 95% 以上。Anthropic 官方文档强调："Provide extremely detailed descriptions. This is by far the most important factor in tool performance."[4]

### 3.1 四条铁律

1. **description 比类型约束更重要**：LLM 靠自然语言描述来理解你的意图，类型约束只是辅助
2. **枚举值用 `enum` 约束**：不要只在 description 里写"可选值为 A/B/C"，用 `enum` 字段硬约束，LLM 输出稳定性大幅提升[6]
3. **必填参数放到 `required` 数组**：不要让 LLM 去"猜"哪些参数是必须的
4. **嵌套不超过两层**：超过两层的嵌套对象，LLM 填充的准确性会明显下降。如果业务确实需要复杂结构，考虑拆成多个工具[6]

### 3.2 天气查询工具（基础版）

```json
{
  "name": "get_weather",
  "description": "查询指定城市的实时天气信息，返回温度、湿度、风速和天气状况。当用户询问某地天气时使用此工具，不要用于查询未来天气预报。",
  "input_schema": {
    "type": "object",
    "properties": {
      "city": {
        "type": "string",
        "description": "城市名称，使用中文全称如'北京市'、'上海市'，不要使用简称或英文名"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "温度单位：celsius 为摄氏度，fahrenheit 为华氏度"
      }
    },
    "required": ["city"]
  }
}
```

设计要点：
- `city` 的 description 明确要求"全称"和"不接受缩写"，避免 LLM 传 "BJ" 之类无法识别的值
- `unit` 用 `enum` 硬约束，杜绝 LLM 自由发挥出 "℃"、"degrees" 等非标准值

### 3.3 股票查询工具（进阶版：数组 + 枚举 + 嵌套）

```json
{
  "name": "get_stock_price",
  "description": "查询指定股票的实时价格和日内涨跌幅。支持美股和A股市场，支持批量查询。当用户询问股票行情时使用。",
  "input_schema": {
    "type": "object",
    "properties": {
      "symbols": {
        "type": "array",
        "description": "股票代码列表。美股直接用 ticker（如 AAPL、TSLA），A 股使用 6 位数字代码（如 600519、000001）。",
        "items": {"type": "string"},
        "minItems": 1,
        "maxItems": 10
      },
      "market": {
        "type": "string",
        "description": "股票所属市场，决定价格查询的数据源。",
        "enum": ["US", "CN"]
      },
      "include_extended_hours": {
        "type": "boolean",
        "description": "是否包含盘前/盘后交易价格（仅美股有效）。默认 false。"
      }
    },
    "required": ["symbols", "market"]
  }
}
```

四个关键设计：
- `symbols` 用 `type: "array"` + `items` 定义数组元素类型，`minItems: 1` + `maxItems: 10` 约束数量
- `market` 用 `enum: ["US", "CN"]` 硬约束，比自然语言描述可靠约 15-20%[6]
- description 中给出格式示例（"如 AAPL、TSLA"）。实践表明这非常有效，LLM 会参照示例生成更规范的值
- `include_extended_hours` 的 description 注明了业务约束"仅美股有效"

### 3.4 反例对照：糟糕的 Schema

```json
// ❌ 糟糕的 Schema —— 不要这样写
{
  "name": "search",
  "description": "搜索",
  "input_schema": {
    "type": "object",
    "properties": {
      "q": {"type": "string"},
      "type": {"type": "string"}
    }
  }
}
```

这个定义的问题：
- description 只有一个"搜索"，LLM 完全不知道何时用它，和 GPT 的搜索功能有什么区别
- 参数名 `q`、`type` 没有说明，LLM 猜不出该填什么
- 没有 `required`，LLM 可能不填关键参数
- 没有 `enum`，`type` 的值完全不可控

---

## 四、完整代码实战：双重 API 并行演示

下面用 Python 写一个完整的可运行的示例，同时演示 **OpenAI API** 和 **Anthropic API** 两种实现方式，覆盖单工具调用和多工具并行调用。

### 4.1 工具定义 + 模拟业务函数

```python
"""
Function Calling 完整示例 —— 天气查询 + 股票查询双工具
同时演示 Anthropic API 和 OpenAI API 两种调用方式
"""

import json
import time
import random
from typing import Any, Dict

# ============================================================
# 第一部分：工具定义（JSON Schema）
# 注意：Anthropic 和 OpenAI 的 Schema 包裹格式不同
# ============================================================

# --- 通用 JSON Schema（核心部分，两种 API 复用） ---
WEATHER_SCHEMA = {
    "type": "object",
    "properties": {
        "city": {
            "type": "string",
            "description": "城市名称，使用中文全称如'北京市'，不要用拼音"
        },
        "unit": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "description": "温度单位"
        }
    },
    "required": ["city"]
}

STOCK_SCHEMA = {
    "type": "object",
    "properties": {
        "symbol": {
            "type": "string",
            "description": "股票代码，如'AAPL'、'600519.SH'（贵州茅台）"
        },
        "period": {
            "type": "string",
            "enum": ["1d", "5d", "1mo", "3mo", "1y"],
            "description": "查询周期：1d当日、5d近五日、1mo近一月等"
        }
    },
    "required": ["symbol"]
}

# 通用描述文本
WEATHER_DESC = (
    "查询指定城市的实时天气信息，返回温度、湿度、风速和天气状况。"
    "当用户询问某地天气时使用此工具。不包含未来天气预报。"
)
STOCK_DESC = (
    "查询指定股票的最新价格和历史走势。返回当前价、涨跌幅、成交量等数据。"
    "当用户询问股票行情时使用。支持美股（如 AAPL）和 A 股（如 600519.SH）。"
)

# --- Anthropic 工具定义格式 ---
# 注意：Anthropic 使用 input_schema 字段，直接顶级定义，无需外层包裹
ANTHROPIC_TOOLS = [
    {
        "name": "get_weather",
        "description": WEATHER_DESC,
        "input_schema": WEATHER_SCHEMA,
        "strict": True  # 启用严格模式，保证输出 100% 符合 Schema
    },
    {
        "name": "get_stock_price",
        "description": STOCK_DESC,
        "input_schema": STOCK_SCHEMA,
        "strict": True
    }
]

# --- OpenAI 工具定义格式 ---
# 注意：OpenAI 需要外包 type: "function" + function 层，参数用 parameters 字段
OPENAI_TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": WEATHER_DESC,
            "parameters": WEATHER_SCHEMA,
            "strict": True
        }
    },
    {
        "type": "function",
        "function": {
            "name": "get_stock_price",
            "description": STOCK_DESC,
            "parameters": STOCK_SCHEMA,
            "strict": True
        }
    }
]


# ============================================================
# 第二部分：模拟业务函数
# 生产环境中应替换为真实的 API 调用
# ============================================================

# 模拟天气数据
MOCK_WEATHER = {
    "北京市": {"temp": 28, "humidity": 65, "wind": "东南风 3级", "condition": "晴"},
    "上海市": {"temp": 32, "humidity": 78, "wind": "南风 2级",   "condition": "多云"},
    "深圳市": {"temp": 35, "humidity": 85, "wind": "西南风 4级", "condition": "雷阵雨"},
}


def get_weather(city: str, unit: str = "celsius") -> Dict[str, Any]:
    """查询城市天气（生产环境应替换为真实 API 调用）"""
    time.sleep(0.2)  # 模拟网络延迟

    if city not in MOCK_WEATHER:
        available = ", ".join(MOCK_WEATHER.keys())
        return {
            "success": False,
            "error_type": "city_not_found",
            "message": f"未找到'{city}'的天气数据",
            "available_cities": available
        }

    data = MOCK_WEATHER[city]
    temp = data["temp"]
    if unit == "fahrenheit":
        temp = round(temp * 9 / 5 + 32, 1)

    return {
        "success": True,
        "city": city,
        "temperature": temp,
        "unit": unit,
        "humidity": data["humidity"],
        "wind": data["wind"],
        "condition": data["condition"]
    }


def get_stock_price(symbol: str, period: str = "1d") -> Dict[str, Any]:
    """查询股票价格（生产环境应替换为真实行情 API）"""
    time.sleep(0.3)  # 模拟网络延迟

    mock_prices = {
        "AAPL": 198.56, "GOOGL": 178.23, "MSFT": 432.15,
        "600519.SH": 1680.00, "000001.SZ": 12.34
    }

    if symbol.upper() not in mock_prices:
        return {
            "success": False,
            "error_type": "symbol_not_found",
            "message": f"未找到股票代码'{symbol}'，请检查代码是否正确"
        }

    base_price = mock_prices[symbol.upper()]
    change = round(random.uniform(-3, 3), 2)
    change_pct = round(change / base_price * 100, 2)

    return {
        "success": True,
        "symbol": symbol.upper(),
        "price": base_price,
        "change": change,
        "change_percent": f"{change_pct:+.2f}%",
        "period": period,
        "currency": "CNY" if (".SH" in symbol or ".SZ" in symbol) else "USD"
    }


# 函数路由表：函数名 → 执行函数
TOOL_EXECUTORS = {
    "get_weather": get_weather,
    "get_stock_price": get_stock_price,
}
```

### 4.2 Anthropic API 完整调用循环

```python
# ============================================================
# 第三部分-A：Anthropic Messages API 的完整 Agent 循环
# ============================================================

import os
import anthropic


def anthropic_agent_loop(user_query: str, max_iterations: int = 10) -> str:
    """
    使用 Anthropic Messages API 的完整 Agent 循环。

    Anthropic 与 OpenAI 的核心差异：
    - 工具调用存在于 response.content 中的 tool_use blocks
    - 参数已自动解析为 dict（不需要 json.loads）
    - 工具结果用 role="user" + tool_result blocks 回传
    - stop_reason 用 "end_turn" / "tool_use"（不是 "stop" / "tool_calls"）

    Args:
        user_query: 用户输入的自然语言问题
        max_iterations: 最大迭代次数，防止无限循环（安全上限）
    Returns:
        Claude 的最终文本回复
    """
    client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
    messages = [{"role": "user", "content": user_query}]

    for iteration in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            system="你是一个有用的助手，可以查询天气和股票信息。当需要实时数据时，请使用提供的工具。",
            tools=ANTHROPIC_TOOLS,
            messages=messages,
        )

        # 情况1：模型直接给出文本回复（无需调用工具）
        if response.stop_reason == "end_turn":
            for block in response.content:
                if block.type == "text":
                    return block.text
            return "无法生成回复"

        # 情况2：模型请求调用工具
        if response.stop_reason == "tool_use":
            # 将 assistant 响应（含 tool_use blocks）加入对话历史
            messages.append({
                "role": "assistant",
                "content": response.content
            })

            # 提取所有 tool_use blocks，并行执行
            tool_use_blocks = [
                block for block in response.content
                if block.type == "tool_use"
            ]

            tool_results: list = []
            for tool_block in tool_use_blocks:
                func_name = tool_block.name
                func_args = tool_block.input  # Anthropic 已自动解析 JSON 为 dict

                executor = TOOL_EXECUTORS.get(func_name)
                if executor is None:
                    result = {
                        "success": False,
                        "error_type": "unknown_tool",
                        "message": f"未知工具: {func_name}"
                    }
                else:
                    try:
                        result = executor(**func_args)
                    except Exception as e:
                        # 捕获执行异常，返回结构化错误（不给 LLM 看 traceback）
                        result = {
                            "success": False,
                            "error_type": "execution_error",
                            "message": f"工具执行异常: {str(e)}",
                            "retryable": False
                        }

                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": tool_block.id,  # 必须匹配 tool_use block 的 id
                    "content": json.dumps(result, ensure_ascii=False)
                })

            # 将工具结果作为 user 消息回传（Anthropic 专有格式）
            messages.append({
                "role": "user",
                "content": tool_results
            })
            continue  # 继续循环，让 Claude 基于结果回复

        # 情况3：其他 stop_reason
        print(f"[警告] 异常 stop_reason: {response.stop_reason}")
        break

    return "抱歉，处理超时，请简化您的问题后重试。"
```

### 4.3 OpenAI API 完整调用循环

```python
# ============================================================
# 第三部分-B：OpenAI Chat Completions API 的完整 Agent 循环
# ============================================================

from openai import OpenAI


def openai_agent_loop(user_query: str, max_iterations: int = 10) -> str:
    """
    使用 OpenAI Chat Completions API 的完整 Agent 循环。

    OpenAI 与 Anthropic 的核心差异：
    - tool_calls 在 message.tool_calls 列表中（不是 content blocks）
    - arguments 是 JSON 字符串（需要 json.loads 手动解析）
    - 工具结果用 role="tool" + tool_call_id 回传（不是 user 消息）
    - finish_reason 用 "stop" / "tool_calls"（不是 "end_turn" / "tool_use"）

    Args:
        user_query: 用户输入的自然语言问题
        max_iterations: 最大迭代次数
    Returns:
        GPT 的最终文本回复
    """
    client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
    messages = [{"role": "user", "content": user_query}]

    for iteration in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=OPENAI_TOOLS,
            tool_choice="auto",  # 让模型自行决定是否调用工具
            # 可选：parallel_tool_calls=False 强制串行调用
        )
        choice = response.choices[0]

        # 情况1：模型直接给出文本回复
        if choice.finish_reason == "stop":
            return choice.message.content or "无法生成回复"

        # 情况2：模型请求调用工具（可能一次返回多个 tool_call）
        if choice.finish_reason == "tool_calls":
            # 将 assistant 消息加入对话历史
            messages.append(choice.message.model_dump())

            # 遍历所有 tool_calls（支持并行调用）
            for tool_call in choice.message.tool_calls:
                func_name = tool_call.function.name

                # OpenAI 的 arguments 是 JSON 字符串，需要手动解析
                try:
                    func_args = json.loads(tool_call.function.arguments)
                except json.JSONDecodeError:
                    # 参数解析失败 → 返回结构化错误让 LLM 自行修正
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": json.dumps({
                            "success": False,
                            "error_type": "invalid_arguments",
                            "message": "参数 JSON 格式错误，请检查后重试",
                            "raw": tool_call.function.arguments[:200]
                        }, ensure_ascii=False)
                    })
                    continue

                # 路由执行
                executor = TOOL_EXECUTORS.get(func_name)
                if executor is None:
                    result = {
                        "success": False,
                        "error_type": "unknown_tool",
                        "message": f"未知工具: {func_name}"
                    }
                else:
                    try:
                        result = executor(**func_args)
                    except TypeError as e:
                        result = {
                            "success": False,
                            "error_type": "type_error",
                            "message": f"参数类型错误: {str(e)}",
                            "retryable": True
                        }
                    except Exception as e:
                        result = {
                            "success": False,
                            "error_type": "execution_error",
                            "message": f"工具执行异常: {str(e)}",
                            "retryable": False
                        }

                # 将工具结果以 role="tool" 回传
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,  # 必须匹配，否则 OpenAI 返回 400
                    "content": json.dumps(result, ensure_ascii=False)
                })

            continue  # 继续循环

    return "抱歉，处理超时，请简化您的问题后重试。"


# ============================================================
# 第四部分：测试运行
# ============================================================

if __name__ == "__main__":
    # 单工具测试
    print("=== 单工具测试：天气查询 ===")
    # result = anthropic_agent_loop("北京今天天气怎么样？")
    # print(result)

    # 多工具并行测试
    print("\n=== 多工具并行测试：天气 + 股票 ===")
    # result = openai_agent_loop("北京天气怎么样？顺便查一下苹果股价")
    # print(result)
```

### 4.4 关键差异速查表

| 维度 | Anthropic Claude | OpenAI GPT-4o |
|------|------------------|---------------|
| 工具定义字段 | `input_schema`（直接顶级） | `parameters`（外包 `function` 层） |
| 工具调用位置 | `response.content` 中的 `tool_use` blocks | `message.tool_calls` 数组 |
| 参数 JSON | `tool_block.input`（已自动解析为 dict） | `tool_call.function.arguments`（字符串） |
| 结果回传方式 | `role="user"` + `tool_result` blocks | `role="tool"` + `tool_call_id` |
| 停止原因 | `"end_turn"` / `"tool_use"` | `"stop"` / `"tool_calls"` |
| 强制调用工具 | `tool_choice={"type": "any"}` | `tool_choice="required"` |
| Strict 模式 | `strict: True` | `strict: True` |

---

## 五、错误处理三层防御策略

在 Function Calling 的生产环境中，每个环节都可能出错。你需要三层防御[7][8]：

```
┌──────────────────────────────────────────────────────────────────┐
│                       错误处理三层策略                              │
├──────────┬───────────────────┬──────────────────────┬────────────┤
│   层级    │     错误类型       │      处理策略          │  LLM 感知  │
├──────────┼───────────────────┼──────────────────────┼────────────┤
│ 第1层    │ 参数 JSON 解析失败  │ 返回结构化错误 +       │    是      │
│          │ / 类型不匹配       │ 具体修正提示给 LLM      │            │
├──────────┼───────────────────┼──────────────────────┼────────────┤
│ 第2层    │ 函数执行异常       │ try-catch 包裹 +       │    是      │
│          │ / 下游 API 失败    │ 错误分类（可重试/不可   │            │
│          │                   │ 重试）+ 结构化反馈      │            │
├──────────┼───────────────────┼──────────────────────┼────────────┤
│ 第3层    │ LLM API 超时      │ 指数退避 + 随机抖动 +   │    否      │
│          │ / 429 频率限制     │ idempotency key +      │            │
│          │                   │ 全局重试预算上限        │            │
└──────────┴───────────────────┴──────────────────────┴────────────┘
```

### 5.1 第一层：参数解析失败

LLM 生成的参数 JSON 可能格式错误，尤其是在使用某些开源模型时[7]：

```python
def safe_parse_arguments(tool_call) -> tuple[bool, dict]:
    """
    安全解析 LLM 返回的参数 JSON。

    OpenAI 的 arguments 是字符串，需 json.loads；
    Anthropic 的 input 已经是 dict，直接使用。

    Returns:
        (success, result) —— success=True 时 result 为解析后的 dict；
        success=False 时返回结构化错误信息
    """
    # OpenAI 格式：arguments 是 JSON 字符串
    if hasattr(tool_call, 'function'):
        raw_args = tool_call.function.arguments
        try:
            return True, json.loads(raw_args)
        except json.JSONDecodeError as e:
            return False, json.dumps({
                "success": False,
                "error_type": "invalid_json",
                "message": f"参数 JSON 格式错误: {str(e)}，请确保参数是合法的 JSON 对象",
                "received": raw_args[:200]
            }, ensure_ascii=False)
    # Anthropic 格式：input 已经是 dict
    elif isinstance(tool_call, dict) or hasattr(tool_call, 'input'):
        args = tool_call.input if hasattr(tool_call, 'input') else tool_call.get('input', {})
        return True, args
    else:
        return False, json.dumps({
            "success": False,
            "error_type": "unexpected_type",
            "message": f"参数类型异常，期待 JSON 字符串或字典"
        }, ensure_ascii=False)
```

### 5.2 第二层：函数执行异常 -- 结构化错误反馈

生产环境中，你的工具函数对接外部 API、数据库、文件系统，任何一个环节都可能失败。关键是**不让 LLM 看到裸的 Python 异常堆栈**[8]：

```python
import functools
import logging

logger = logging.getLogger(__name__)


def tool_error_handler(func):
    """
    工具函数装饰器：自动捕获所有异常，返回结构化错误。

    设计原则：
    - LLM 永远不会看到 Python traceback
    - 不同错误有不同 error_type，方便 LLM 决定下一步策略
    - retryable 字段告诉 LLM 是否应该重试
    - suggested_action 给 LLM 明确的引导
    """
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except ValueError as e:
            # 参数值无效（如传入的城市不存在）—— 不可重试
            logger.warning(f"[{func.__name__}] 参数值错误: {e}")
            return {
                "success": False,
                "error_type": "invalid_value",
                "message": f"参数值无效: {str(e)}",
                "retryable": False,
                "suggested_action": "请用户提供正确的参数值"
            }
        except ConnectionError as e:
            # 网络错误 —— 可重试，但需退避
            logger.error(f"[{func.__name__}] 网络连接失败: {e}")
            return {
                "success": False,
                "error_type": "network_error",
                "message": f"外部服务连接失败: {str(e)}，请稍后重试",
                "retryable": True,
                "suggested_action": "等待片刻后重试，或告知用户服务暂时不可用"
            }
        except TimeoutError as e:
            # 超时 —— 可重试一次
            logger.error(f"[{func.__name__}] 请求超时: {e}")
            return {
                "success": False,
                "error_type": "timeout",
                "message": f"请求超时（{str(e)}），服务响应过慢",
                "retryable": True,
                "suggested_action": "可尝试重试一次，若仍超时应告知用户"
            }
        except Exception as e:
            # 未知异常 —— 不重试，记录详细日志供排查
            logger.exception(f"[{func.__name__}] 未预期的异常")
            return {
                "success": False,
                "error_type": "unknown_error",
                "message": "工具内部异常，请告知用户稍后重试",
                "retryable": False,
                "suggested_action": "告知用户当前无法完成此操作"
            }
    return wrapper
```

### 5.3 第三层：API 超时与限流

LLM API 本身的调用也可能出错。指数退避 + 随机抖动（jitter）防止"重试风暴"[9]：

```python
import asyncio


class LLMRetryHandler:
    """
    LLM API 调用的重试处理器。

    核心设计：
    - 指数退避：base_delay * (2 ** attempt)
    - 随机抖动：每个间隔加 0%~25% 随机偏移，防止多个客户端"同步"重试
    - 错误分类：429/5xx/超时可重试；4xx（除 429）不可重试
    - 全局重试预算：限制单次 Agent 运行的总重试次数，防止死循环
    """

    # 可重试的 HTTP 状态码
    RETRYABLE_STATUS = {429, 500, 502, 503, 504}

    def __init__(self, max_retries: int = 3, base_delay: float = 1.0,
                 max_run_retries: int = 10):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_run_retries = max_run_retries
        self.run_retry_count = 0

    def should_retry(self, error: Exception) -> bool:
        """判断错误是否值得重试"""
        if hasattr(error, 'status_code'):
            return error.status_code in self.RETRYABLE_STATUS
        # 网络超时等底层传输错误可重试
        if isinstance(error, (ConnectionError, TimeoutError)):
            return True
        return False

    def get_delay(self, attempt: int, retry_after: float = None) -> float:
        """
        计算退避延迟。
        优先使用服务端返回的 Retry-After 头；
        否则使用指数退避 + 随机抖动。
        """
        if retry_after is not None:
            return retry_after  # 服务端说了算

        exponential = self.base_delay * (2 ** attempt)
        jitter = exponential * random.uniform(0, 0.25)  # 0%~25% 随机抖动
        return exponential + jitter

    def budget_exhausted(self) -> bool:
        return self.run_retry_count >= self.max_run_retries
```

### 5.4 区分"工具错误"和"业务正常结果"

这是一个容易被忽视的细节[8]：

```python
# ❌ 模糊 —— 调用方分不清是 API 挂了还是真的没数据
return json.dumps({"error": "No data found"})

# ✅ 明确 —— 用结构化字段区分状态
return json.dumps({
    "status": "empty",   # 不是 "error"
    "message": "该城市暂无天气数据，可尝试查询周边城市"
})

# 真正的错误用不同的 status + error_type
return json.dumps({
    "status": "error",
    "error_type": "api_timeout",
    "message": "上游天气 API 超时，请稍后重试"
})
```

---

## 六、并行 Function Calling：性能翻倍的秘密

现代 LLM（GPT-4o、Claude 3.5 Sonnet+）能在**单次响应中返回多个 tool_call**，让运行时代码并行执行这些独立调用。这是 Agent 性能的关键优化[10]。

### 6.1 LLM 何时并行调用？

LLM 判断以下条件**同时满足**时，会一次返回多个 tool_call：

1. **调用之间无数据依赖**：调用 B 的参数不需要调用 A 的结果
2. **所有参数当前已知**：每个调用的参数都可以从对话上下文中直接确定

```
依赖判断速查：
┌───────────────────────────────┬────────┬─────────────────────┐
│            场景                │  依赖   │      调用模式        │
├───────────────────────────────┼────────┼─────────────────────┤
│ 查东京 + 巴黎 + 开罗天气       │  无    │ 并行：3 个 tool_call  │
│ 查用户 ID → 查该用户订单       │  有    │ 串行：先 ID 再订单    │
│ 查天气 + 查股价（同一用户提问） │  无    │ 并行：2 个 tool_call  │
│ 搜索产品 → 加入购物车          │  有    │ 串行：先搜索后加购    │
└───────────────────────────────┴────────┴─────────────────────┘
```

**快速判断法**：你能在第一个函数返回之前写出第二个函数的所有参数吗？能 -- 并行；不能 -- 串行[10]。

### 6.2 异步并行执行（asyncio.gather）

```python
import asyncio


async def execute_tool_async(name: str, args: dict) -> dict:
    """异步执行单个工具调用"""
    executor = TOOL_EXECUTORS.get(name)
    if executor is None:
        return {"success": False, "error_type": "unknown_tool",
                "message": f"未知工具: {name}"}
    # 在线程池中运行同步函数
    loop = asyncio.get_running_loop()
    try:
        return await loop.run_in_executor(None, lambda: executor(**args))
    except Exception as e:
        return {"success": False, "error_type": "execution_error",
                "message": str(e)}


async def execute_tools_parallel(tool_calls: list) -> list[dict]:
    """
    并行执行多个工具调用。

    关键技术点：
    - asyncio.gather 同时启动所有协程
    - return_exceptions=True 确保单个工具失败不中断其他任务
    - 总耗时 ≈ max(所有单独调用的耗时)，而非 sum
    """
    tasks = []
    for tc in tool_calls:
        # 兼容 OpenAI 和 Anthropic 格式
        name = tc.function.name if hasattr(tc, 'function') else tc.name
        args = tc.function.arguments if hasattr(tc, 'function') else tc.input
        if isinstance(args, str):
            args = json.loads(args) if args else {}
        tasks.append(execute_tool_async(name, args))

    # return_exceptions=True 是关键：单个失败不取消其他任务
    results = await asyncio.gather(*tasks, return_exceptions=True)

    # 将异常转换为结构化错误
    return [
        {"success": False, "error_type": "execution_error", "message": str(r)}
        if isinstance(r, Exception) else r
        for r in results
    ]
```

### 6.3 并发控制

当 LLM 一次返回 10+ 个 tool_call 时，需要对下游服务做并发保护[10]：

```python
# 限制最大并发数为 5，保护下游 API
SEMAPHORE = asyncio.Semaphore(5)

async def execute_with_limit(name: str, args: dict) -> dict:
    """带并发限制的工具执行"""
    async with SEMAPHORE:
        return await execute_tool_async(name, args)
```

### 6.4 性能收益

假设每个工具调用耗时 2 秒：

| 调用模式 | 3 个独立工具 | 总耗时 |
|---------|------------|-------|
| 串行执行 | weather(2s) → stock(2s) → news(2s) | **6 秒** |
| 并行执行 | weather + stock + news 同时启动 | **~2 秒** |
| 性能提升 | — | **3x** |

这就是现代 Agent 框架在底层都使用并行执行的原因。它不是可选的优化，是生产环境的必须项[10]。

---

## 七、生产环境 Best Practice Checklist

综合多家厂商文档和生产经验，以下清单覆盖工具设计、执行安全和错误处理三个维度。

### 7.1 工具设计

- [ ] 每个工具 description 至少 2-3 句话，说明"做什么"、"何时用"、"何时不用" [4]
- [ ] 每个参数都有清晰 description，不依赖参数名猜测
- [ ] 用 `enum` 约束有限选项，用 `minimum/maximum` 约束数值范围 [6]
- [ ] 必填参数列在 `required` 中，无歧义
- [ ] 每个工具参数不超过 5-7 个，嵌套不超过 2 层
- [ ] 启用 `strict: true` 模式（OpenAI 和 Anthropic 都支持）[5][4]
- [ ] 工具数量控制在 20 个以内（单次请求）。超过则按场景分组或使用工具搜索
- [ ] 相关操作合并为一个工具 + action 参数，而非拆分多个（如 `pr_action: create|review|merge`）[4]

### 7.2 执行安全

- [ ] **绝不让 LLM 直接执行有副作用的操作**（发邮件、扣款、写数据库）[1]
- [ ] 所有外部 API 调用必须经过你的函数执行层，在此层做权限校验和频率限制
- [ ] 写操作需要 human-in-the-loop 确认或至少记录审计日志
- [ ] 区分"读"和"写"操作——读操作可以积极重试，写操作必须谨慎
- [ ] 非幂等的写操作必须带 idempotency key，防止重复执行 [8]

### 7.3 错误处理

- [ ] LLM 永远不应看到 Python traceback——只返回结构化错误对象 [7]
- [ ] 结构化错误包含 `error_type`、`message`、`retryable`、`suggested_action` 四个字段
- [ ] 区分可重试错误（429、5xx、网络超时）和不可重试错误（400、401、403、404）[9]
- [ ] 重试策略使用指数退避 + 随机抖动 [9]
- [ ] 设置全局重试预算上限（如单次运行最多重试 10 次）
- [ ] 记录每次工具调用的日志（输入参数、输出结果、耗时），便于排查

### 7.4 性能优化

- [ ] 使用并行执行（asyncio.gather）处理无依赖的多工具调用
- [ ] 使用 prompt caching 缓存工具定义（Anthropic: `cache_control: {"type": "ephemeral"}` 放在最后一个工具定义上）
- [ ] 清理上下文中过期的工具结果，避免 token 膨胀
- [ ] 设置最大迭代次数，防止 Agent 陷入死循环

---

## 参考来源

1. [Anthropic - Tool Use Concepts](https://github.com/anthropics/skills/blob/main/skills/claude-api/shared/tool-use-concepts.md) — 官方工具使用概念指南，Manual Agentic Loop 标准写法、Tool Runner beta API
2. [Dataquest - Python Function Calling: How to Give LLMs Access to Real-World Tools (2026)](https://www.dataquest.io/blog/python-function-calling/) — Function Calling 生命周期详解，含 finish_reason 判断和消息拼接注意事项
3. [AgentPatterns.ai - Tool Calling Schema Standards for AI Agent Development](https://agentpatterns.ai/standards/tool-calling-schema-standards/) — 跨平台工具调用 Schema 规范对照，OpenAI/Anthropic/Gemini/MCP 四平台差异
4. [Anthropic - Define Tools (Claude API Docs)](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools) — 工具定义的官方最佳实践，description 撰写指南、strict 模式、input_examples 用法
5. [OpenAI - Function Calling Guide](https://developers.openai.com/api/docs/guides/function-calling) — OpenAI 官方函数调用文档，strict 模式、tool_choice 参数说明、最佳实践
6. [KitLab - JSON Schema for Function Calling: Practical Guide with Claude and GPT (2026)](https://kitlab.app/en/blog/json-schema-function-calling-guide) — JSON Schema 三平台差异对比、嵌套对象处理、常见陷阱
7. [Samanvya Tripathi - Function Calling Patterns for Production LLM Agents (2026)](https://samanvya.dev/blog/function-calling-production) — 生产环境三支柱：权限分级、结构化错误处理、人机协同
8. [Ninad Pathak - What Nobody Tells You About Error Handling in Production AI Agents (2026)](https://ninadpathak.com/blog/production-ai-agent-errors/) — 工具失败导致的级联错误、idempotency key 最佳实践、提示检查点
9. [AgentMarketCap - Tool Call Reliability Patterns for Production AI Agents in 2026](https://agentmarketcap.ai/blog/2026/04/11/tool-call-reliability-patterns-production-agents-2026) — 重试分类（瞬态/永久/模糊）、指数退避公式、Retry-After 优先原则
10. [CallSphere - Parallel Tool Execution: Running Multiple Tools Simultaneously in Agents (2026)](https://callsphere.ai/blog/parallel-tool-execution-running-multiple-tools-simultaneously) — asyncio.gather 并行模式、return_exceptions 用法、信号量并发控制
