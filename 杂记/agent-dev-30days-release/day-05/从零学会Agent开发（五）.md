---
series: Agent 开发 30 天
day: 5
title: 不依赖框架，手写你的第一个 Agent
tags: [agent, react, llm, tool-calling, from-scratch, anthropic-sdk]
date: 2026-06-23
---

# 从零学会Agent开发（五）：不依赖框架，手写你的第一个 Agent

前面四天你理解了 Prompt、Function Calling 和 ReAct 模式的理论。今天把这些概念全部串起来，从零实现一个完整可运行的 Agent。

手写一遍之后，再用 LangChain、CrewAI 等框架时，你会清楚地知道框架在替你做什么。正如 [pguso/agents-from-scratch](https://github.com/pguso/agents-from-scratch) 所说：

> "Agents are not personalities. They are loops, state, and constraints."

---

![[illustration-1.png]]

## 1. 整体架构设计

先理清 Agent 由哪些部件组成。

```
┌─────────────────────────────────────────────────────────────┐
│                       Agent 主循环                           │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │ System Prompt │───▶│  LLM Client  │───▶│ Tool Registry │  │
│  │ 组装           │    │ (Anthropic   │    │ ┌───────────┐ │  │
│  │               │    │  SDK)        │    │ │ calculator│ │  │
│  └──────────────┘    └──────┬───────┘    │ │ web_fetch │ │  │
│                             │            │ └───────────┘ │  │
│                             ▼            └───────┬───────┘  │
│                     ┌───────────────┐            │          │
│                     │ Message       │◀───────────┘          │
│                     │ History       │                       │
│                     └───────┬───────┘                       │
│                             │                               │
│                             ▼                               │
│                     ┌───────────────┐                       │
│                     │ 判断是否继续   │                       │
│                     │ (stop_reason) │                       │
│                     └───────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

![[个人/agent-dev-30days/day-05/illustration-2.png]]

五个核心组件：

| 组件 | 职责 | 关键点 |
|------|------|--------|
| System Prompt 组装 | 告诉 LLM 它是谁、有什么工具、怎么工作 | 工具描述要具体到参数级别 |
| LLM Client | 封装 Anthropic Messages API，处理 tool_use 内容块 | 支持自动重试和指数退避 |
| Tool Registry | 维护工具名到函数的映射，安全执行工具调用 | 参数校验、异常兜底 |
| Message History | 维护完整对话历史，正确处理 assistant/user 的 content 结构 | Anthropic 的内容块格式与 OpenAI 完全不同 |
| 主循环 | 驱动 ReAct 流程：调用 LLM → 解析响应 → 执行工具 → 回传结果 → 判断终止 | 必须有 max_steps 安全阀 |

数据流（一轮 Reasoning + Acting）：

```
用户问题
   │
   ▼
┌─────────────────┐
│ 1. 组装消息       │  system + user query + 历史工具结果
└────────┬────────┘
         ▼
┌─────────────────┐
│ 2. 调用 LLM      │  发送 messages + tools 定义
└────────┬────────┘
         ▼
    ┌────────────┐
    │ stop_reason │── "tool_use" ──→ 3a. 解析 tool_use 块 → 执行工具
    │   是什么？   │                       │
    └────────────┘                        ▼
         │                         结果追加到 messages
         │ "end_turn"                      │
         ▼                                ▼
    4. 返回文本答案（任务完成）      回到步骤 2
```

---

## 2. 完整代码实现

### 环境准备

```bash
pip install anthropic requests
```

设置 API Key：

```bash
# Windows PowerShell
$env:ANTHROPIC_API_KEY = "sk-ant-api03-xxxxxxxx"

# Mac / Linux
export ANTHROPIC_API_KEY="sk-ant-api03-xxxxxxxx"
```

---

### Step 1：System Prompt 组装

System Prompt 是 Agent 的大脑设定，需要包含三件事：角色定义、工具描述、行为规范。

```python
import json

# ============================================================
# Step 1: System Prompt 组装
# 将角色定义 + 工具描述 + 输出格式指令拼接为完整提示词
# ============================================================

def build_system_prompt() -> str:
    """
    构建 Agent 的 System Prompt。
    包含角色定义、可用工具详细说明和工作原则。
    """
    tools_description = """
## 可用工具

你可以使用以下工具来完成复杂任务。每个工具都有明确的输入参数要求。

### 1. calculator
- **功能**：执行数学计算
- **参数**：
  - expression (string, 必填): 数学表达式，如 "2 + 3 * 4"、"sqrt(16) + 10"、"pow(2, 8)"
- **支持运算**：加(+)、减(-)、乘(*)、除(/)、幂(**)、取余(%)、开方(sqrt)、绝对值(abs)以及常用 math 函数

### 2. web_fetch
- **功能**：获取指定 URL 的网页文本内容
- **参数**：
  - url (string, 必填): 要获取的网页 URL，如 "https://news.ycombinator.com"
- **注意**：返回的是从 HTML 中提取的纯文本摘要，不是原始 HTML

## 工作原则

1. **先分析再行动**：先理清用户要什么，确定解决步骤，再按顺序调用工具
2. **工具优先**：能用工具获取的数据不要自己编造，调工具拿真实结果
3. **逐步执行**：每次调用当前需要的工具，拿到结果再决定下一步
4. **简洁准确**：最终答案要完整、精确，包含关键数据和依据
"""
    return f"""你是一个能干的 AI 助手，能够通过调用工具来完成复杂的任务。

你的工作方式：
1. 分析用户的问题，确定解决步骤
2. 按顺序调用需要的工具
3. 基于工具返回的真实数据来回答，不要编造信息
4. 如果工具调用失败，分析原因并尝试替代方案

{tools_description}
"""
```

工具描述要具体到参数名、类型、示例，这样模型才能正确构造 tool_use 请求。明确写"不要编造信息"是防幻觉的关键——很多模型在不确定时倾向自己编数据。System Prompt 控制在 500-1500 字，太短模型不知道怎么做，太长会稀释关键信息。

---

### Step 2：LLM 调用封装

使用 Anthropic Python SDK 的 `messages.create` API。传入 `tools` 参数后，模型会返回 `tool_use` 类型的内容块。

```python
import os
import time
from anthropic import Anthropic, APIError, APITimeoutError, RateLimitError

# ============================================================
# Step 2: LLM 调用封装
# 封装 Anthropic Messages API，支持工具调用和自动重试
# ============================================================

class LLMClient:
    """
    封装 Anthropic Messages API 调用。
    内置指数退避重试，对 429（限流）和 5xx（服务器错误）自动重试。
    """

    DEFAULT_MODEL = "claude-sonnet-4-5-20250929"
    MAX_TOKENS = 4096
    MAX_RETRIES = 3           # 最大重试次数
    RETRY_BASE_DELAY = 1.0    # 基础等待秒数，每次翻倍

    def __init__(self, model: str = None):
        self.client = Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))
        self.model = model or self.DEFAULT_MODEL

    def call(
        self,
        system_prompt: str,
        messages: list[dict],
        tools: list[dict],
    ) -> "LLMResponse":
        """
        调用 LLM，带自动重试。

        Args:
            system_prompt: 系统提示词
            messages: 消息历史列表（Anthropic 格式）
            tools: 工具定义列表（Anthropic tool schema 格式）

        Returns:
            LLMResponse 对象，包含 stop_reason 和 content 列表
        """
        last_error = None
        for attempt in range(self.MAX_RETRIES):
            try:
                response = self.client.messages.create(
                    model=self.model,
                    max_tokens=self.MAX_TOKENS,
                    system=system_prompt,
                    messages=messages,
                    tools=tools,
                )
                return LLMResponse(
                    stop_reason=response.stop_reason,
                    content=response.content,
                    role=response.role,
                )

            except RateLimitError as e:
                # 429 限流：等待后重试
                last_error = e
                wait_time = self._parse_retry_after(e) or (self.RETRY_BASE_DELAY * (2 ** attempt))
                print(f"[警告] 触发频率限制，等待 {wait_time:.1f}s 后重试 ({attempt + 1}/{self.MAX_RETRIES})")
                time.sleep(wait_time)

            except APITimeoutError as e:
                # 超时：指数退避
                last_error = e
                wait_time = self.RETRY_BASE_DELAY * (2 ** attempt)
                print(f"[警告] API 超时，等待 {wait_time:.1f}s 后重试 ({attempt + 1}/{self.MAX_RETRIES})")
                time.sleep(wait_time)

            except APIError as e:
                # 其他 API 错误：4xx 不重试，5xx 重试
                last_error = e
                if hasattr(e, 'status_code') and 400 <= e.status_code < 500:
                    raise  # 客户端错误（401/400 等），重试无意义
                wait_time = self.RETRY_BASE_DELAY * (2 ** attempt)
                status = e.status_code if hasattr(e, 'status_code') else 'unknown'
                print(f"[警告] API 错误 (HTTP {status})，等待 {wait_time:.1f}s 后重试 ({attempt + 1}/{self.MAX_RETRIES})")
                time.sleep(wait_time)

        # 重试耗尽
        raise RuntimeError(f"LLM 调用失败，已重试 {self.MAX_RETRIES} 次: {last_error}")

    def _parse_retry_after(self, error: RateLimitError) -> float | None:
        """从 RateLimitError 中解析建议的等待秒数"""
        try:
            error_str = str(error)
            if "retry after" in error_str.lower():
                import re
                match = re.search(r'(\d+)', error_str)
                if match:
                    return float(match.group(1))
        except Exception:
            pass
        return None


class LLMResponse:
    """
    LLM 响应包装类。
    封装 Anthropic 的 stop_reason 判断和 content 提取。
    """

    def __init__(self, stop_reason: str, content: list, role: str):
        self.stop_reason = stop_reason
        # Anthropic stop_reason 的 5 种可能值：
        #   "end_turn"     — 模型正常完成回答
        #   "tool_use"     — 模型请求调用工具
        #   "max_tokens"   — 达到 max_tokens 限制
        #   "stop_sequence" — 触发了自定义停止序列
        #   "refusal"      — 模型因安全策略拒绝回答
        self.content = content
        self.role = role

    def is_tool_use(self) -> bool:
        """模型是否请求调用工具"""
        return self.stop_reason == "tool_use"

    def is_final_answer(self) -> bool:
        """模型是否给出了最终文本答案"""
        return self.stop_reason == "end_turn"

    def get_text(self) -> str:
        """提取响应中的纯文本内容"""
        texts = []
        for block in self.content:
            if hasattr(block, 'type') and block.type == "text":
                texts.append(block.text)
        return "\n".join(texts)

    def get_tool_uses(self) -> list[dict]:
        """提取所有 tool_use 块，返回 [{name, id, input}, ...]"""
        tool_uses = []
        for block in self.content:
            if hasattr(block, 'type') and block.type == "tool_use":
                tool_uses.append({
                    "name": block.name,
                    "id": block.id,
                    "input": dict(block.input) if block.input else {},
                })
        return tool_uses
```

三种错误分类处理，参考 [ValueStreamAI](https://valuestreamai.com/blog/ai-error-handling-patterns-2026) 的生产实践：429 限流用 Retry-After 头 + 指数退避；超时用指数退避；4xx 客户端错误直接失败不重试。

`LLMResponse` 封装了 `stop_reason` 解析——这是 Anthropic API 与 OpenAI 最大的不同点，Anthropic 用 `stop_reason`（而非 `finish_reason`）来表达模型意图。工具参数在 `block.input` 中，是 Anthropic 返回的原始 dict，无需 JSON 解析。

---

### Step 3 & 4：工具定义和执行器

Step 3（定义工具）和 Step 4（执行工具）紧密关联，放在一起讲。先让模型知道能调用什么，再用注册表模式映射到实际函数。

```python
import re
import math
import requests
from typing import Any, Callable

# ============================================================
# Step 3: 工具函数实现
# ============================================================

def calculator_tool(expression: str) -> str:
    """
    安全的数学计算器。
    只允许安全字符和 math 白名单函数，拒绝任意代码执行。
    """
    # 白名单：只允许这些 math 函数
    safe_funcs = {'sqrt', 'abs', 'pow', 'sin', 'cos', 'tan',
                  'log', 'log10', 'ceil', 'floor', 'pi', 'e', 'round'}

    # 检查表达式中的单词（潜在函数名），确认全部在白名单中
    words = set(re.findall(r'[a-zA-Z_]+', expression))
    if not words.issubset(safe_funcs):
        dangerous = words - safe_funcs
        return f"[错误] 表达式包含不允许的函数或变量: {', '.join(dangerous)}"

    # 去掉函数名后，检查剩余字符是否都是安全字符
    cleaned = re.sub(r'[a-zA-Z_]+', '', expression)
    cleaned = re.sub(r'[\d\s\+\-\*\/\(\)\.\,\%\^]+', '', cleaned)
    if cleaned.strip():
        return f"[错误] 表达式包含非法字符"

    try:
        # 构建安全的求值环境：不暴露 builtins，只暴露 math 白名单函数
        safe_globals = {"__builtins__": {}}
        for func_name in safe_funcs:
            if hasattr(math, func_name):
                safe_globals[func_name] = getattr(math, func_name)
        safe_globals['pi'] = math.pi
        safe_globals['e'] = math.e

        result = eval(expression, safe_globals)
        return f"计算结果: {result}"
    except SyntaxError as e:
        return f"[错误] 表达式语法错误: {e}"
    except ZeroDivisionError:
        return "[错误] 除零错误"
    except Exception as e:
        return f"[错误] 计算失败: {e}"


def web_fetch_tool(url: str) -> str:
    """
    获取网页内容并提取文本摘要。
    使用 requests 获取 HTML，用正则提取纯文本。
    """
    # 自动补全协议头
    if not url.startswith(('http://', 'https://')):
        url = 'https://' + url

    headers = {
        "User-Agent": "Mozilla/5.0 (compatible; AgentTutorial/1.0)"
    }

    try:
        response = requests.get(url, headers=headers, timeout=15)
        response.raise_for_status()
    except requests.exceptions.Timeout:
        return f"[错误] 请求超时：{url} 在 15 秒内未响应"
    except requests.exceptions.ConnectionError:
        return f"[错误] 连接失败：无法连接到 {url}，请检查 URL 是否正确"
    except requests.exceptions.HTTPError as e:
        return f"[错误] HTTP 错误：{e}"
    except Exception as e:
        return f"[错误] 获取失败: {e}"

    html = response.text

    # 简易 HTML 文本提取：去掉 script/style 标签，再去掉所有 HTML 标签
    # 生产环境建议使用 BeautifulSoup 或 html2text
    text = re.sub(r'<script[^>]*>.*?</script>', '', html, flags=re.DOTALL | re.IGNORECASE)
    text = re.sub(r'<style[^>]*>.*?</style>', '', text, flags=re.DOTALL | re.IGNORECASE)
    text = re.sub(r'<[^>]+>', ' ', text)
    text = re.sub(r'\s+', ' ', text).strip()

    # 截断过长内容，避免超出 LLM 上下文窗口
    if len(text) > 3000:
        text = text[:3000] + f"\n\n... [内容已截断，原文共 {len(text)} 字符]"

    return text


# ============================================================
# Step 4: 工具注册表和执行器
# ============================================================

class ToolRegistry:
    """
    工具注册表。
    维护工具名 → 函数 的映射，以及对应的 Anthropic tool schema。
    添加新工具只需一行 register() 调用。
    """

    def __init__(self):
        self._tools: dict[str, Callable] = {}   # 工具名 → 函数
        self._schemas: list[dict] = []           # Anthropic tool schema 列表

    def register(self, name: str, description: str,
                 func: Callable, parameters: dict):
        """
        注册一个工具。

        Args:
            name: 工具名称（模型用此名称调用）
            description: 工具描述（告诉模型做什么用）
            func: 实际执行的 Python 函数
            parameters: 参数的 JSON Schema 定义
        """
        self._tools[name] = func
        self._schemas.append({
            "name": name,
            "description": description,
            "input_schema": {
                "type": "object",
                "properties": parameters.get("properties", {}),
                "required": parameters.get("required", []),
            },
        })

    def get_schemas(self) -> list[dict]:
        """获取所有工具的 Anthropic tool schema 列表，传给 API 的 tools 参数"""
        return self._schemas

    def execute(self, name: str, arguments: dict) -> str:
        """
        安全执行指定工具。

        执行失败时不抛异常，而是返回 "[错误] xxx" 格式的字符串，
        让模型能看到错误信息并尝试修正（如改参数名、换 URL 等）。
        """
        if name not in self._tools:
            available = ', '.join(self._tools.keys())
            return f"[错误] 未知工具 '{name}'。可用工具: {available}"

        func = self._tools[name]

        try:
            result = func(**arguments)
            return str(result)
        except TypeError as e:
            # 参数不匹配：告诉模型参数传错了
            return f"[错误] 工具参数错误: {e}。请检查参数名和类型是否正确"
        except Exception as e:
            return f"[错误] 工具执行异常: {e}"


def create_tool_registry() -> ToolRegistry:
    """
    创建并初始化工具注册表。
    添加新工具只需在这里增加一行 registry.register()。
    """
    registry = ToolRegistry()

    # 注册计算器
    registry.register(
        name="calculator",
        description="执行数学计算。支持基本运算（加减乘除幂取余）和 math 函数"
                    "（sqrt, sin, cos, log 等）。示例: '2 + 3 * 4', 'sqrt(16) + pow(2, 10)'",
        func=calculator_tool,
        parameters={
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，如 '2 + 3 * 4' 或 'sqrt(144) + 10'",
                }
            },
            "required": ["expression"],
        },
    )

    # 注册网页抓取
    registry.register(
        name="web_fetch",
        description="获取指定 URL 的网页文本内容。返回纯文本摘要，可用于阅读文章、"
                    "查看网页数据等。",
        func=web_fetch_tool,
        parameters={
            "properties": {
                "url": {
                    "type": "string",
                    "description": "要获取的网页完整 URL，如 'https://news.ycombinator.com'",
                }
            },
            "required": ["url"],
        },
    )

    return registry
```

几个关键设计决策：

计算器安全沙箱参考了 [OneAppleFall](https://oneapplefall.com/build-ai-agents-from-scratch-python/) 的实践：`eval()` 必须配合 `{"__builtins__": {}}` 清空内置函数，再用白名单暴露安全函数。LLM 可能会生成含 `__import__` 或 `open()` 的表达式，不加沙箱就是远程代码执行。

工具出错时返回 `[错误] xxx` 而不是抛异常。模型看到错误后能自动修正，比如参数名拼错时下一轮会自动纠正。

网页内容 3000 字符截断防止工具结果撑爆 LLM 的上下文窗口。这是生产环境中的常见实践。

---

### Step 5：ReAct 主循环

这是 Agent 的灵魂。整个循环只有四个动作：发消息给 LLM → 检查 stop_reason → 执行工具 → 回传结果。

```python
# ============================================================
# Step 5: Agent 主循环（ReAct 模式）
# ============================================================

class Agent:
    """
    从零手写的 ReAct Agent。

    核心流程：
    Observe（构建消息）→ Think（调用 LLM）→ Act（执行工具）→ Feedback（回传结果）→ 循环
    """

    MAX_STEPS = 10  # 最大步数，防止死循环和费用失控

    def __init__(self, llm_client: LLMClient = None,
                 tool_registry: ToolRegistry = None):
        self.llm = llm_client or LLMClient()
        self.tools = tool_registry or create_tool_registry()
        self.system_prompt = build_system_prompt()
        self.messages: list[dict] = []  # 完整对话历史（Anthropic 格式）
        self.step_count = 0

    def run(self, user_query: str, verbose: bool = True) -> str:
        """
        运行 Agent，处理用户查询直到获得最终答案。

        Args:
            user_query: 用户的自然语言问题
            verbose: 是否打印中间步骤详情

        Returns:
            Agent 的最终回答文本
        """
        # 初始化消息历史（Anthropic API 格式）
        self.messages = [
            {"role": "user", "content": user_query}
        ]
        self.step_count = 0

        if verbose:
            print(f"{'='*60}")
            print(f"用户提问: {user_query}")
            print(f"{'='*60}\n")

        while self.step_count < self.MAX_STEPS:
            self.step_count += 1

            if verbose:
                print(f"--- 第 {self.step_count} 步 ---")

            # ============================================
            # 1. Observe: 将当前消息历史发给 LLM
            # ============================================
            try:
                response = self.llm.call(
                    system_prompt=self.system_prompt,
                    messages=self.messages,
                    tools=self.tools.get_schemas(),
                )
            except Exception as e:
                return f"Agent 执行失败（第 {self.step_count} 步）: {e}"

            # ============================================
            # 2. Think + Decide: 检查 LLM 的响应类型
            # ============================================
            if response.is_final_answer():
                # 模型给出了最终答案 → 任务完成
                final_text = response.get_text()
                if verbose:
                    preview = final_text[:200] + "..." if len(final_text) > 200 else final_text
                    print(f"[最终答案] {preview}")
                return final_text

            elif response.is_tool_use():
                # 模型请求调用工具 → 进入 Acting 阶段
                tool_uses = response.get_tool_uses()
                if not tool_uses:
                    if verbose:
                        print("[警告] stop_reason=tool_use 但没有 tool_use 块，跳过")
                    continue

                # ---------- 构建 assistant 消息（Anthropic 格式）----------
                # Anthropic 要求 assistant 消息的 content 是整个响应内容块列表
                assistant_content = []
                for block in response.content:
                    if hasattr(block, 'type'):
                        if block.type == "text":
                            assistant_content.append({
                                "type": "text",
                                "text": block.text,
                            })
                        elif block.type == "tool_use":
                            assistant_content.append({
                                "type": "tool_use",
                                "id": block.id,
                                "name": block.name,
                                "input": dict(block.input) if block.input else {},
                            })

                self.messages.append({
                    "role": response.role,
                    "content": assistant_content,
                })

                # ---------- 3. Act: 执行每个工具调用 ----------
                tool_results = []
                for tu in tool_uses:
                    if verbose:
                        args_str = json.dumps(tu["input"], ensure_ascii=False)
                        print(f"[调用工具] {tu['name']}({args_str})")

                    result = self.tools.execute(tu["name"], tu["input"])

                    if verbose:
                        preview = result[:150] + "..." if len(result) > 150 else result
                        print(f"[工具结果] {preview}")

                    # ---------- 4. Feedback: 构造 tool_result 块 ----------
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": tu["id"],
                        "content": result,
                    })

                # Anthropic 要求 tool_result 放在 user 消息的 content 数组中
                self.messages.append({
                    "role": "user",
                    "content": tool_results,
                })

                if verbose:
                    print()

            else:
                # 其他 stop_reason（max_tokens、refusal 等）
                if verbose:
                    print(f"[异常] stop_reason = {response.stop_reason}")
                text = response.get_text()
                if text:
                    self.messages.append({
                        "role": "assistant",
                        "content": text,
                    })
                    continue
                else:
                    return f"Agent 异常终止: stop_reason={response.stop_reason}"

        # 达到最大步数限制 → 安全终止
        return f"Agent 在 {self.MAX_STEPS} 步后仍未完成。请尝试简化问题或增加 MAX_STEPS。"


# ============================================================
# 主程序入口
# ============================================================

def main():
    """
    运行示例：Agent 解决"查询 Hacker News 热门文章并统计标题平均长度"
    """
    agent = Agent()

    query = (
        "帮我查一下 Hacker News 今天最热的文章前 10 条，"
        "然后统计这些文章标题的平均长度是多少个字符"
    )

    result = agent.run(query, verbose=True)

    print(f"\n{'='*60}")
    print("最终回答:")
    print(f"{'='*60}")
    print(result)


if __name__ == "__main__":
    main()
```

循环流程图解：

```
用户输入
   │
   ▼
┌──────────────────────────────────────────┐
│  while step < MAX_STEPS:                 │
│                                          │
│  ① llm.call(system, messages, tools)     │  ← Observe（发消息）
│     │                                    │
│     ▼                                    │
│  ② 检查 stop_reason                      │  ← Think（判断意图）
│     │                                    │
│     ├── "end_turn" ──▶ 返回文本答案       │  ← 终止：任务完成
│     │                                    │
│     ├── "refusal"   ──▶ 异常终止          │  ← 终止：安全拒绝
│     │                                    │
│     └── "tool_use"  ──▶ 继续执行          │
│            │                             │
│            ▼                             │
│  ③ registry.execute(name, args)          │  ← Act（执行工具）
│            │                             │
│            ▼                             │
│  ④ messages.append(tool_result)          │  ← Feedback（回传结果）
│            │                             │
│            └──────▶ 回到 ①               │
└──────────────────────────────────────────┘
```

三个终止条件：
1. `stop_reason == "end_turn"` — 模型任务完成，返回文本答案
2. `step_count >= MAX_STEPS` — 步数安全阀。[Building Agentic AI](https://buildingagenticai.com/blog/build-ai-agent-from-scratch-python/) 强调：没有步数限制的 Agent 会在困惑时无限循环，持续烧钱
3. 其他异常 `stop_reason` — `max_tokens`（token 超限）、`refusal`（安全拒绝）等

Anthropic 的消息结构与 OpenAI 完全不同，这是最容易踩的坑：

| 场景 | OpenAI 格式 | Anthropic 格式 |
|------|------------|---------------|
| 普通文本 | `{"role": "assistant", "content": "hello"}` | `{"role": "assistant", "content": [{"type": "text", "text": "hello"}]}` |
| 工具调用 | `tool_calls` 字段 | `content` 数组中的 `{"type": "tool_use", ...}` 块 |
| 工具结果 | `{"role": "tool", "tool_call_id": "xxx", "content": "result"}` | `{"role": "user", "content": [{"type": "tool_result", "tool_use_id": "xxx", "content": "result"}]}` |

核心差异：Anthropic 的 content 始终是一个内容块列表（不是字符串），且 tool_result 放在 user 消息中（不是独立的 tool 角色）。

---

## 3. 运行示例

把以上所有代码放到一个文件 `agent.py`，设置 API Key 后运行：

```bash
python agent.py
```

预期输出：

```
============================================================
用户提问: 帮我查一下 Hacker News 今天最热的文章前 10 条，然后统计这些文章标题的平均长度是多少个字符
============================================================

--- 第 1 步 ---
[调用工具] web_fetch({"url": "https://news.ycombinator.com"})
[工具结果] Hacker News new | past | comments | ask | show | jobs | submit | login
1. ▲ Building AI Agents from Scratch (buildingagenticai.com) 234 points by...
2. ▲ Show HN: I built a CLI tool for... 189 points by...
...

--- 第 2 步 ---
[调用工具] calculator({"expression": "(34 + 48 + 52 + 41 + 63 + 37 + 45 + 55 + 39 + 47) / 10"})
[工具结果] 计算结果: 46.1

--- 第 3 步 ---
[最终答案] Hacker News 首页前 10 条热门文章的标题平均长度约为 46.1 个字符...

============================================================
最终回答:
============================================================
（Agent 的完整回答，包含每篇文章标题和长度数据）
```

---

## 4. 完整错误处理体系

Agent 在每个层面都实现了错误处理。以下模式来自 [Geodocs.dev](https://geodocs.dev/ai-agents/agent-error-recovery-patterns-spec) 和 [ValueStreamAI](https://valuestreamai.com/blog/ai-error-handling-patterns-2026) 的生产环境最佳实践：

| 层级 | 错误类型 | 处理策略 | 原因 |
|------|----------|----------|------|
| LLM 调用 | 429 频率限制 | 解析 Retry-After 头 + 指数退避，最多 3 次 | 尊重服务端的限流窗口，避免雪崩 |
| LLM 调用 | 超时 | 指数退避重试，基础 1s，每次翻倍 | 网络波动通常短暂且自愈 |
| LLM 调用 | 4xx 客户端错误 | 直接失败，不重试 | 401/400 等错误重试不会变好 |
| LLM 调用 | 5xx 服务端错误 | 指数退避重试 | 服务端可能临时过载 |
| 工具执行 | 参数类型不匹配 | 捕获 TypeError，返回 `[错误]` 给模型 | 模型看到错误会主动修正参数 |
| 工具执行 | 业务异常 | 捕获所有 Exception，返回错误描述 | 不让单个工具错误拖垮整个 Agent |
| Web Fetch | 连接超时/失败 | 返回具体错误信息（URL、状态码） | 模型可能尝试备选 URL 或搜索引擎 |
| Agent 级别 | 超过 MAX_STEPS | 强制终止，返回友好提示 | 防止无限循环烧钱 |
| Agent 级别 | 无 tool_use 块 | 跳过当前轮，继续循环 | 模型有时会在 text 中思考后再调工具 |

三个核心原则：

1. 区分可恢复和不可恢复——频率限制和超时可重试；认证失败和请求错误不可重试。盲目重试不可恢复的错误只会浪费资源和时间。

2. 错误信息回传给模型——工具执行失败时不吞掉异常，也不终止 Agent，而是把错误信息作为 tool_result 返回。模型看到 `[错误] 参数错误: missing required argument 'url'` 后会自动修正。[Ninad Pathak](https://ninadpathak.com/blog/production-ai-agent-errors/) 指出：把原始错误直接丢给 LLM 然后指望它自己解决，这不叫错误处理。

3. 必须设置最大步数——没有步数限制的 Agent 会在困惑时无限循环，不断消耗 API 费用。`MAX_STEPS = 10` 对大多数场景足够。建议再加一个时间上限（如 `max_seconds=60`）做双重保护。

---

## 5. 核心代码量统计

不含注释和空行的实际代码行数：

| 模块 | 行数 | 占比 |
|------|------|------|
| System Prompt | 25 | 10% |
| LLMClient + LLMResponse | 70 | 27% |
| 工具函数（calculator + web_fetch） | 55 | 22% |
| ToolRegistry | 40 | 16% |
| Agent 主循环 | 65 | 25% |
| 总核心代码 | ~255 | 100% |

255 行纯 Python，实现了完整可运行的 ReAct Agent。如果去掉错误处理和注释，核心逻辑不到 150 行。

---

## 6. 扩展方向

当你需要更强的 Agent 时，可以从以下几个方向扩展：

1. 添加更多工具——在 `create_tool_registry()` 中加一行 `registry.register()`，比如天气查询、数据库查询、文件读写
2. 支持并行工具调用——当多个工具彼此独立时（如同时查天气和新闻），不必串行等待
3. 添加流式输出——使用 Anthropic 的 streaming API，让用户实时看到 Agent 的思考过程
4. 持久化对话记忆——把 `self.messages` 存入文件或数据库，让 Agent 跨会话记住上下文
5. 添加追踪日志——记录每步的 token 消耗、耗时、工具调用详情，方便后续分析优化

---

## 7. 今日小结

你从零实现了一个完整可运行的 ReAct Agent。关键收获：

- Agent = System Prompt + LLM Client + Tool Registry + Message History + 主循环
- ReAct 主循环 = Observe → Think → Act → Feedback，四个动作循环直到终止
- Anthropic SDK 的核心差异：content 始终是内容块列表，tool_result 在 user 消息中
- 工具设计安全第一：计算器要沙箱 eval，网络请求要处理超时
- 错误处理分层防御：每层有对应的重试策略，错误信息要回传给模型
- MAX_STEPS 是必备安全阀：没有上限的 Agent 会无限烧钱

手写一遍之后，后续学任何框架你都能看到底层在做什么。这种感觉是直接学框架得不到的。

---

> 下一篇 Day 06：Tool Calling 深度剖析——从 JSON Schema 设计、参数校验到错误恢复策略，设计 LLM 不会用错的工具接口。

---

### 参考资料

1. [Build an AI Agent From Scratch in Python, No Framework](https://buildingagenticai.com/blog/build-ai-agent-from-scratch-python/) — Building Agentic AI, 2026.05
2. [pguso/agents-from-scratch](https://github.com/pguso/agents-from-scratch) — 12 Lessons: Build AI agents from first principles
3. [Implement the ReAct Pattern in 50 Lines of Python](https://growthengineer.ai/blog/react-pattern-implementation-python) — Growth Engineer, 2026.05
4. [Build AI Agents From Scratch With Python: A Working Tutorial](https://oneapplefall.com/build-ai-agents-from-scratch-python/) — OneAppleFall, 2026.06
5. [AI Error Handling Patterns 2026: Circuit Breakers, Retries & Fallbacks](https://valuestreamai.com/blog/ai-error-handling-patterns-2026) — ValueStreamAI, 2026.05
6. [Agent Error Recovery Patterns Specification](https://geodocs.dev/ai-agents/agent-error-recovery-patterns-spec) — Geodocs.dev, 2026.05
7. [Anthropic Python SDK — Official Tool Use Example](https://github.com/anthropics/anthropic-sdk-python/blob/main/examples/tools.py)
8. [What Nobody Tells You About Error Handling in Production AI Agents](https://ninadpathak.com/blog/production-ai-agent-errors/) — Ninad Pathak, 2026.04
9. [When Agents Fail: Retry Logic, Circuit Breakers, and Dead Letter Queues](https://supergood.solutions/blog/systems-sunday-agent-failure-recovery-2026/) — Supergood Solutions, 2026.03
