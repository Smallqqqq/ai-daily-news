# 从零学会Agent开发（四）：ReAct 模式——Agent 的核心思维循环

> **系列定位**：《Agent 开发 30 天》第 4 天，面向有编程基础但 Agent 零经验的开发者。
> **前情提要**：Day 02 建立了 Prompt 的系统认知，Day 03 打通了 Function Calling 的完整链路。今天把两者整合，理解 Agent 最核心的思维模式——ReAct。

---

![[illustration-1.png]]

## 1. 从一个侦探故事说起

想象你是一个侦探，接手了一桩案件：

1. **思考**：死者身上的伤口很特殊，我需要查一下凶器来源。
2. **行动**：你走进档案室，翻出凶器登记记录。
3. **观察**：记录显示这把刀只在某个小镇的铁匠铺出售过。
4. **再思考**：凶手一定跟那个小镇有关系，我还需要确认死者的社会关系。
5. **再行动**：你调取死者的通讯录和银行流水。
6. **观察**：通讯录里有一个号码，归属地正是那个小镇。
7. **最终思考**：证据链完整，凶手就是这个人。

人类解决复杂问题的自然方式是：先想，再做，看了结果再想，直到问题解决。

现在把这个过程套到 AI 身上。一个普通 LLM 被问"今天北京天气多少度"，它只能回答训练数据截止日期之前的知识。它没有眼睛去看外面，没有手去点开网页。但一个 ReAct Agent 可以：

- **思考**：我不知道今天的天气，需要查询。
- **行动**：调用 `search("2026年6月23日 北京天气")`
- **观察**：得到搜索结果——32°C，晴天。
- **思考**：我已经有了答案。
- **最终答案**：今天北京 32°C，晴天。

ReAct 模式让 AI 像人类侦探一样，通过推理（Reasoning）和行动（Acting）的交替循环来解决问题。

---

## 2. 论文溯源：ReAct 从哪里来

2022 年 10 月，普林斯顿大学和 Google Research 联合发布了论文《**ReAct: Synergizing Reasoning and Acting in Language Models**》（2023 年发表于 ICLR）。论文提出了一个根本性的主张：语言模型应该边想边做，让推理和行动交替进行，而非先想完再做。

论文的核心实验数据：

| 基准测试 | 纯 CoT（只推理） | 纯 Act-only（只行动） | **ReAct** | 关键差异 |
|----------|:---------------:|:-------------------:|:---------:|----------|
| HotPotQA 幻觉率 | 56% | — | **6%** | 幻觉率降低 50 个百分点 |
| ALFWorld 成功率 | 基线 | 基线 | **+34%（绝对提升）** | 远超模仿学习和 RL 基线 |
| WebShop 成功率 | 基线 | 基线 | **+10%（绝对提升）** | 商业化决策任务同样有效 |
| FEVER 事实验证 | 低于 ReAct | — | 最优 | 推理+检索交叉验证 |

CoT 的每一步推理都建立在上一步模型自己生成的内容之上——一旦某一步想错了，错误就像滚雪球一样向下传播。ReAct 在每步推理之间插入一个来自真实世界的 Observation，工具返回的是真实数据，不是模型脑补的。这就打断了错误累积链，把模型重新"锚定"在事实上。

ReAct 论文首次证明：把推理过程和行动能力放进同一个循环里，会产生 1+1>2 的效果。它因此成为现代 AI Agent 框架的事实标准基础——LangChain 的 AgentExecutor、LlamaIndex 的 ReActAgent、OpenAI Assistants API、Anthropic Claude Agent SDK，底层都是 ReAct 或其变体。

---

## 3. ReAct 循环的形式化定义

ReAct 的核心结构是一个三步循环，每一步产生一种状态输出：

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Thought    │ ──> │   Action     │ ──> │ Observation  │
│   （推理）    │ <── │   （行动）    │     │   （观察）    │
└──────────────┘     └──────────────┘     └──────────────┘
       ^                                         │
       └─────────────────────────────────────────┘
                      循环直到完成
```

![[个人/agent-dev-30days/day-04/illustration-2.png]]

### 3.1 Thought（推理 / 思考）

模型在这一步自言自语，解释它已知什么、不确定什么、下一步打算做什么。

- **输入**：当前对话历史（用户的原始问题 + 之前所有步骤的 Thought、Action、Observation）
- **输出**：一段自然语言推理文本
- **本质**：模型的"内心独白"，让外部系统和人类调试者看到模型的决策过程

### 3.2 Action（行动）

模型基于 Thought 选择一个外部工具并指定参数。

- **输入**：刚完成的 Thought
- **输出**：一个结构化的工具调用（工具名 + 参数）
- **本质**：将意图转化为具体操作

在真实系统中，Action 不是自由文本——它通过 Function Calling / Tool Use API 以结构化 JSON 发出，由 Agent 运行时（你的代码）来实际执行。

### 3.3 Observation（观察）

工具执行结果被注入到对话历史中。

- **输入**：工具调用的返回值（或错误信息）
- **输出**：（由 Agent 运行时生成）格式化后的工具结果
- **本质**：外部世界的真实反馈注入推理链，打断模型的自说自话

### 3.4 完整轨迹示例

以下是一个 ReAct Agent 回答"2024 年诺贝尔物理学奖得主出生在哪里，当地天气如何"的完整执行轨迹：

```
--- 第 1 步 ---
Thought: 我需要先知道 2024 年诺贝尔物理学奖得主是谁。
Action: web_search("2024 Nobel Prize in Physics winner")
Observation: 2024 年诺贝尔物理学奖授予 John J. Hopfield 和 Geoffrey E. Hinton。

--- 第 2 步 ---
Thought: 已知得主是 Hopfield 和 Hinton。先查 Hopfield 的出生地。
Action: web_search("John Hopfield birthplace")
Observation: John Hopfield 出生于美国伊利诺伊州芝加哥市。

--- 第 3 步 ---
Thought: Hopfield 出生在芝加哥。现在查芝加哥的天气就可以给出最终答案了。
Action: weather("Chicago")
Observation: 芝加哥当前温度 6.5°C，多云，湿度 94%，风速 2.6 m/s。

--- 第 4 步 ---
Thought: 所有信息已齐备——得主是 Hopfield，出生在芝加哥，天气也有了。
Final Answer: 2024 年诺贝尔物理学奖得主之一 John Hopfield 出生于美国芝加哥。
当前芝加哥天气为 6.5°C，多云，湿度 94%。
```

每一步的 Observation 真正改变了后续 Thought 的方向——拿到得主名字后去查出生地，拿到出生地后去查天气。如果只有 CoT 没有工具，模型只能猜得主是谁；如果只有 Action 没有 Thought，模型不知道查完得主后下一步该查什么。

---

## 4. 推理和行动缺一不可

| 维度 | 纯 Act-only | 纯 CoT | **ReAct** |
|------|:----------:|:------:|:---------:|
| **推理过程** | 无——直接调工具 | 有——逐步推理 | **有——每步都显式推理** |
| **外部交互** | 有 | 无——全靠模型知识 | **有——每步推理后可调工具** |
| **幻觉风险** | 中——搜不到就瞎猜 | 高——HotPotQA 幻觉率 56% | **低——HotPotQA 幻觉率仅 6%** |
| **适合场景** | 单次检索 | 数学/逻辑推理 | **多步复杂任务、需要查资料** |
| **延迟/成本** | 低（1 次调用） | 低（1 次调用） | 中（通常 3~10 次调用） |
| **可审计性** | 差 | 中——推理可见但无法验证 | **好——每步推理+每步验证** |
| **错误传播** | 工具失败即死 | 每一步基于上一步"脑补"，错误累积 | **Observation 打断错误链** |

纯 Act-only 的代表场景：用户问"北京天气"，模型直接调 `weather("北京")`，拿结果返回。但如果用户问"北京和上海哪个更适合周末出游"，纯 Act-only 不知道先查什么、查完怎么比——它只能盲目地一次调好几个工具。

纯 CoT 的问题更致命：推理看起来很漂亮，但内容可能是编的。论文数据显示 CoT 的幻觉率高达 56%——模型读到"根据维基百科，北京的面积为 16410 平方公里"就信了，但如果实际是 16411 呢？CoT 没办法验证。

ReAct 把两者融合：思考决定了下一步做什么，行动的结果反过来修正思考。推理让行动有了方向，行动让推理有了依据。

---

## 5. 完整 Python 实现：从零构建 ReAct Agent

以下是一个完全可运行的 ReAct Agent 实现。我们使用模拟工具来替代真实 API，不需要任何 API Key 就能跑起来、理解整个循环。

代码分五个模块设计：**Environment（环境层）→ Thinker（推理层）→ StopCondition（停止判断层）→ Step（数据层）→ ReActLoop（编排层）**。这种分层让每个模块可以独立测试和替换。

### 5.1 完整代码

```python
# -*- coding: utf-8 -*-
"""
ReAct Agent 完整 Python 实现（无需真实 LLM API）
=================================================

架构分层：
  Environment   → 工具注册与执行，模拟外部世界
  Thinker       → 推理与行动决策（生产环境中替换为 LLM 调用）
  StopCondition → 独立封装的停止条件判断器
  ReActLoop     → 编排 Think → Act → Observe 的主循环

每一层的职责边界清晰：
  - Environment 不关心 Agent 在想什么
  - Thinker 不关心工具怎么执行
  - StopCondition 不关心任务内容
  - ReActLoop 只做编排，不含业务逻辑
"""

from typing import Callable, Dict, List, Optional, Tuple
from dataclasses import dataclass, field
import json
import re


# ============================================================
# 第一层：Step —— 数据结构
# 每次循环的完整快照，同时保留"当时的想法"和"环境的反馈"
# ============================================================

@dataclass
class Step:
    """ReAct 循环中的单步记录"""
    step_num: int
    thought: str         # Agent 当时的推理过程
    action: str          # 调用的工具名
    action_input: str    # 工具参数
    observation: str     # 环境返回的结果


# ============================================================
# 第二层：Environment —— 工具注册与执行
# 生产环境中，这里的工具连接真实 API、数据库、文件系统
# ============================================================

class Environment:
    """
    外部环境，负责工具的注册和执行。
    
    Agent 只负责"决定调用哪个工具"，实际执行由 Environment 完成。
    这种分离设计意味着：
    1. 可以限制 Agent 的可用工具集（安全边界）
    2. 可以在执行前做参数校验和权限检查
    3. 可以记录所有工具调用的审计日志
    """

    def __init__(self):
        # 工具注册表：工具名 → (函数, 参数定义)
        self._tools: Dict[str, Tuple[Callable, Dict]] = {}

    def register(self, name: str, func: Callable, params: Dict):
        """
        注册一个工具到环境中。
        
        参数：
          name   : 工具名（对应 Agent Action 中的工具名）
          func   : 工具执行函数
          params : 参数定义字典，用于生成工具描述
        """
        self._tools[name] = (func, params)

    def execute(self, action_name: str, action_input: str) -> str:
        """
        执行一个 Action，返回 Observation。
        
        这是 Agent 获取真实世界反馈的唯一入口。
        """
        if action_name not in self._tools:
            return (
                f"[错误] 工具 '{action_name}' 不存在。"
                f"可用工具：{list(self._tools.keys())}"
            )

        func, _ = self._tools[action_name]
        try:
            result = func(action_input)
            return str(result)
        except Exception as e:
            return f"[错误] 工具 '{action_name}' 执行失败：{e}"

    def tools_description(self) -> str:
        """生成工具列表的描述文本，用于注入系统提示词"""
        lines = []
        for name, (_, params) in self._tools.items():
            lines.append(f"- {name}({json.dumps(params, ensure_ascii=False)})")
        return "\n".join(lines)


# ============================================================
# 第三层：Thinker —— 推理与决策引擎
# 当前版本用规则模拟推理，生产环境中替换为 LLM API 调用
# ============================================================

class Thinker:
    """
    推理引擎。
    
    输入 task + history + available_tools → 输出 thought + action + action_input。
    
    当前实现：模拟推理（规则匹配），仅用于教学演示。
    生产环境：将 _simulate() 替换为对 LLM（Claude/GPT）的 API 调用，
              接口签名完全不变。
    """

    def think(
        self,
        task: str,
        history: List[Step],
        available_tools: str,
        max_steps: int,
        current_step: int,
    ) -> Tuple[str, str, str]:
        """
        基于当前状态进行推理。
        
        返回：(thought, action, action_input)
        """
        # ---- 生产环境中，替换为 LLM API 调用 ----
        # prompt = f"""
        #   任务：{task}
        #   可用工具：{available_tools}
        #   历史步骤：{self._format_history(history)}
        #   当前第 {current_step}/{max_steps} 步。
        #   请输出 Thought，然后输出 Action。
        # """
        # response = llm.generate(prompt)
        # return self._parse(response)
        # ------------------------------------
        
        return self._simulate(task, history, current_step, max_steps)

    def _simulate(
        self, task: str, history: List[Step], current_step: int, max_steps: int
    ) -> Tuple[str, str, str]:
        """
        模拟推理逻辑——根据任务关键词匹配预设的推理路径。
        
        这只是一个简化演示。真实 LLM 的推理远比这复杂和灵活。
        但它展示了 ReAct 的核心思想：根据当前状态决定下一步。
        """
        task_lower = task.lower()

        # ---- 场景 1：天气查询 ----
        if "天气" in task_lower and current_step == 1:
            return (
                "用户想知道天气情况，我需要调用 search_weather 获取数据。",
                "search_weather",
                task,
            )
        
        if "天气" in task_lower and current_step >= 2:
            last_obs = history[-1].observation if history else ""
            conclusion = (
                f"根据查询结果，{last_obs}。这些信息已足够回答用户的问题。"
            )
            return (conclusion, "finish", last_obs)

        # ---- 场景 2：计算任务 ----
        if any(kw in task_lower for kw in ["计算", "算", "等于", "+", "-", "*", "/"]):
            if current_step == 1:
                return (
                    "这涉及数学计算。我应该调用 calculator 工具来确保精度。",
                    "calculator",
                    task,
                )
            else:
                last_obs = history[-1].observation if history else ""
                return (
                    f"计算结果已得出：{last_obs}。任务完成。",
                    "finish",
                    last_obs,
                )

        # ---- 场景 3：搜索任务 ----
        if any(kw in task_lower for kw in ["搜索", "搜索", "查找", "什么是", "是谁"]):
            if current_step == 1:
                return (
                    "用户想了解某个信息。我需要调用 web_search 获取相关资料。",
                    "web_search",
                    task,
                )
            else:
                last_obs = history[-1].observation if history else ""
                return (
                    f"搜索已返回结果。我已掌握足够信息来回答用户。",
                    "finish",
                    f"根据搜索到的信息，{last_obs}",
                )

        # ---- 默认：无法判断任务类型 ----
        return (
            f"当前第 {current_step} 步。我需要分析任务：'{task}'。",
            "finish",
            f"任务 '{task}' 已处理。",
        )


# ============================================================
# 第四层：StopCondition —— 停止条件判断器
# 这是生产环境中最容易被低估的部分
# ============================================================

class StopCondition:
    """
    判断 Agent 循环是否应该终止。
    
    三种核心停止条件（按优先级排列）：
      1. Agent 主动调用 finish   → 任务完成（最优）
      2. 不可恢复错误            → 工具不存在等无法自愈的错误
      3. 达到步数上限            → 兜底机制，防止无限循环
    
    生产环境中还需扩展：token 预算、时间预算、用户中断等。
    """

    def __init__(self, max_steps: int = 10):
        self.max_steps = max_steps

    def should_stop(
        self,
        action: str,
        observation: str,
        current_step: int,
    ) -> Tuple[bool, Optional[str]]:
        """
        判断是否应该终止循环。

        返回：
          (should_stop: bool, stop_reason: Optional[str])
        
        stop_reason 的明确化对生产运维至关重要——出问题时，
        你能立刻知道 Agent 是"自己觉得做完了"还是"被系统杀了"。
        """
        # 条件 1（最高优先级）：Agent 主动声明任务完成
        # finish 是特殊的 Action——它不需要环境执行
        if action == "finish":
            return True, "success"

        # 条件 2：不可恢复的错误
        # 工具不存在意味着 Agent 产生了"幻觉"工具名
        # 这是 ReAct Agent 最常见的失败模式之一
        if "[错误] 工具" in observation and "不存在" in observation:
            return True, "unrecoverable_error"

        # 条件 3（兜底）：步数耗尽
        # 防止无限循环——每个生产 Agent 都应有此限制
        if current_step >= self.max_steps:
            return True, f"max_steps_reached ({self.max_steps})"

        return False, None


# ============================================================
# 第五层：ReActLoop —— 主循环编排器
# 把 Thinker、Environment、StopCondition 串联起来
# ============================================================

class ReActLoop:
    """
    ReAct 主循环编排器。
    
    生命周期：
      1. 接收用户任务
      2. 循环执行 Think → Act → Observe
      3. 每轮调用 StopCondition 检查是否终止
      4. 满足条件后退出，返回完整执行轨迹
    
    这个类本身不包含任何推理或工具逻辑——它是纯粹的胶水层。
    """

    def __init__(self, environment: Environment, max_steps: int = 10):
        self.environment = environment
        self.thinker = Thinker()
        self.stop_condition = StopCondition(max_steps=max_steps)

    def run(self, task: str) -> List[Step]:
        """
        运行 ReAct 循环，返回完整历史步骤。
        """
        history: List[Step] = []
        tools_desc = self.environment.tools_description()
        max_steps = self.stop_condition.max_steps

        print("=" * 60)
        print(f"  任务：{task}")
        print(f"  可用工具：\n{tools_desc}")
        print(f"  最大步数：{max_steps}")
        print("=" * 60)

        for step_num in range(1, max_steps + 1):
            print(f"\n--- 第 {step_num} 步 ---")

            # Step 1：Think（推理）—— Agent 分析现状，决定做什么
            thought, action, action_input = self.thinker.think(
                task=task,
                history=history,
                available_tools=tools_desc,
                max_steps=max_steps,
                current_step=step_num,
            )
            print(f"  [Thought] {thought}")
            print(f"  [Action]  {action}(\"{action_input}\")")

            # Step 2：Act（行动）—— 环境执行工具调用
            observation = self.environment.execute(action, action_input)
            print(f"  [Observation] {observation}")

            # 记录完整快照
            step = Step(
                step_num=step_num,
                thought=thought,
                action=action,
                action_input=action_input,
                observation=observation,
            )
            history.append(step)

            # Step 3：检查停止条件
            should_stop, reason = self.stop_condition.should_stop(
                action=action,
                observation=observation,
                current_step=step_num,
            )

            if should_stop:
                print(f"\n  >>> 循环终止：{reason}")
                break

        print(f"\n共执行 {len(history)} 步。")
        return history


# ============================================================
# 演示：运行 ReAct Agent
# ============================================================

if __name__ == "__main__":
    # 1. 创建环境，注册模拟工具
    env = Environment()

    def mock_search_weather(query: str) -> str:
        """模拟天气查询"""
        city_map = {
            "北京": "北京 6月23日 晴转多云，22-30°C，湿度45%，适合户外活动。",
            "上海": "上海 6月23日 小雨，24-28°C，湿度80%，建议携带雨具。",
            "芝加哥": "芝加哥 6月23日 多云，6.5°C，湿度94%，风速2.6 m/s。",
        }
        for city, info in city_map.items():
            if city in query:
                return info
        return f"未找到'{query}'的天气数据。"

    def mock_calculator(expression: str) -> str:
        """模拟计算器——仅允许数字和基本运算符"""
        if not re.match(r'^[\d+\-*/().\s]+$', expression):
            return f"表达式包含不支持的字符：{expression}"
        try:
            return f"{expression} = {eval(expression)}"
        except Exception as e:
            return f"计算错误：{e}"

    def mock_web_search(query: str) -> str:
        """模拟网页搜索——本地知识库匹配"""
        knowledge = {
            "2024诺贝尔物理学奖": "2024年诺贝尔物理学奖授予John J. Hopfield和Geoffrey E. Hinton，表彰他们在人工神经网络方面的基础性发现。",
            "hopfield出生地": "John Hopfield出生于美国伊利诺伊州芝加哥市。",
            "python": "Python是一种高级编程语言，由Guido van Rossum于1991年首次发布。",
            "iphone17": "iPhone 17 Pro Max起售价9999元（256GB），顶配14999元（1TB），2025年9月发布。",
        }
        q = query.lower()
        for k, v in knowledge.items():
            if k in q or any(w in q for w in k.split()):
                return v
        return f"未找到与'{query}'相关的信息，请尝试更具体的搜索词。"

    # 注册工具
    env.register("search_weather", mock_search_weather, {"query": "城市名或天气查询关键词"})
    env.register("calculator", mock_calculator, {"expression": "数学表达式，如 (123+456)*78"})
    env.register("web_search", mock_web_search, {"query": "搜索关键词"})

    # 2. 测试多个场景
    test_cases = [
        "查一下北京今天的天气",
        "计算 (123 + 456) * 78 的结果",
        "搜索2024诺贝尔物理学奖得主是谁",
    ]

    for i, task in enumerate(test_cases):
        print(f"\n\n{'█' * 60}")
        print(f"█  测试 {i+1}：{task}")
        print(f"{'█' * 60}\n")

        loop = ReActLoop(environment=env, max_steps=5)
        history = loop.run(task)

        # 打印完整执行轨迹（生产环境中可写入日志系统）
        print("\n" + "=" * 60)
        print("  完整执行轨迹：")
        for s in history:
            print(f"  步骤{s.step_num}: "
                  f"思考→{s.thought[:40]}... | "
                  f"行动→{s.action}({s.action_input[:30]}...) | "
                  f"观察→{s.observation[:40]}...")
```

### 5.2 设计要点讲解

| 模块 | 职责 | 关键设计决策 |
|------|------|-------------|
| **Environment** | 工具注册与执行 | 与 Thinker 完全解耦——Agent 只负责"想调什么"，Environment 负责"能不能调、怎么调" |
| **Thinker** | 推理与决策 | 接口统一：`think()` → `(thought, action, input)`。替换 LLM 只需改内部实现 |
| **StopCondition** | 终止判断 | 独立封装，三种条件按优先级执行：完成 > 不可恢复错误 > 步数上限 |
| **Step** | 数据记录 | 同时保留 thought 和 observation——为后续课程的反思/纠错功能提供信息基础 |
| **ReActLoop** | 编排调度 | 纯胶水层，不含业务逻辑——Think、Act、Observe、Check 四步循环 |

运行脚本后的典型输出：

```
============================================================
  任务：查一下北京今天的天气
  可用工具：
- search_weather({"query": "..."})
  最大步数：5
============================================================

--- 第 1 步 ---
  [Thought] 用户想知道天气情况，我需要调用 search_weather 获取数据。
  [Action]  search_weather("查一下北京今天的天气")
  [Observation] 北京 6月23日 晴转多云，22-30°C，湿度45%，适合户外活动。

--- 第 2 步 ---
  [Thought] 根据查询结果，北京 6月23日 晴转多云...这些信息已足够。
  [Action]  finish("...")
  [Observation] （finish 是终止动作，不需要环境执行）

  >>> 循环终止：success

共执行 2 步。
```

---

## 6. Stop Condition 设计哲学

写 ReAct Agent 时，大多数人花 90% 的精力在 Prompt 和工具设计上。但停止条件的设计至少应占 30% 的关注度——它直接决定了你的 Agent 会不会陷入死循环、会不会在任务完成后继续做无用功、会不会在遇到不可恢复错误时优雅退出。

### 6.1 三种核心停止条件

| 优先级 | 停止条件 | 触发场景 | stop_reason | 说明 |
|--------|---------|---------|-------------|------|
| **1（最高）** | 任务完成 | Agent 输出 Final Answer / 调用 `finish` | `success` | 正常退出，是最理想的结果 |
| **2（中）** | 无法继续 | 工具不存在 / 连续执行失败 / 同一调用重复 3 次以上 | `unrecoverable_error` / `loop_detected` | Agent 遇到了自己无法解决的困境 |
| **3（兜底）** | 步数上限 | 循环次数达到预设上限 | `max_steps_reached` | 最后的保险丝，防止无限循环 |

优先级设计原则：能正常完成最好，完不成至少优雅退出，最差的情况是被系统强行打断。

### 6.2 生产环境的完整控制面板

以上三种是"最小可行方案"。生产环境中至少还需要：

| 扩展停止条件 | 触发阈值 | 用途 |
|-------------|---------|------|
| **Token 预算** | 累计输入 token 超过限制（如 100K） | 防止上下文膨胀导致 API 费用失控和推理质量下降 |
| **时间预算** | 总运行时间超过 N 秒 | 用户不会等 Agent 跑 5 分钟 |
| **工具调用次数上限** | 工具调用次数超过限制 | 有时循环步数没到上限但工具调用已经太多 |
| **重复调用检测** | 同一工具+相同/高度相似参数连续调用 3 次 | 检测死循环，注入提示"你已尝试过该方法，请换一种思路" |
| **外部取消信号** | 用户点击"停止"或上游服务超时 | 保存当前状态，允许后续续接 |

一个完善的 `stop_reason` 枚举值是调试和监控的第一抓手。每条终止日志都能让你立刻判断 Agent 是"自己觉得做完了"还是"被系统杀了"。

---

## 7. ReAct 的三大核心局限性（为后续课程铺垫）

ReAct 是 Agent 设计的基石模式，但它不是银弹。在生产环境中，三个问题反复出现。理解这些局限，才能理解为什么要学习后续课程中的 Plan-and-Execute、Reflexion、Memory 等模式。

### 7.1 循环死锁（Runaway Loops）

现象：Agent 反复调用同一个工具，用差不多的参数，得到差不多的结果，但就是不停。

```
Step 3: search("北京天气") → 32°C
Step 4: search("北京今天天气") → 32°C
Step 5: search("北京市天气温度") → 32°C
Step 6: search("北京 current weather") → 32°C
...（停不下来）
```

根因：ReAct 没有"我已经做过这个了，跳过"的内置机制。模型只根据最新上下文决定下一步，而上下文里没有显式的"已完成任务列表"标记。

在一个 200 任务的 ReAct 生产基准测试中，466 次重试中有 90.8% 是针对根本不存在的工具名——Agent 不是调错工具，是编造了不存在的工具名。这种"工具名幻觉"加上循环重试，构成了 ReAct Agent 最常见也最昂贵的失败模式。

缓解：在代码中记录最近 N 次调用的指纹（工具名 + 参数哈希），检测重复后注入提示"你已尝试过该方法，请换一种思路或直接给出当前已知的答案"。

后续课程：Day 08 的 Reflexion（自我反思）模式专门解决"Agent 是否意识到自己卡住了"的问题。

### 7.2 工具过度调用（Tool Overuse / Confirmation Loop）

现象：Agent 查到了正确答案，但不放心，又去验证一次，验证完了又验证验证结果。

```
Step 3: search("iPhone 17 Pro Max 价格") → 9999 元
Step 4: search("iPhone 17 Pro Max 多少钱") → 9999 元
Step 5: search("Apple iPhone 17 Pro Max 售价") → 9999 元
Step 6: 终于给出 Final Answer: 9999 元
```

Step 4 和 Step 5 是纯粹浪费——Agent 缺乏"证据已足够"的判断力。在成本敏感的 API 场景下，这种确认癖会直接体现在账单上。

缓解：在系统提示词中明确要求"如果已从可靠来源获得所需信息，不必重复验证"。同时在代码层设置工具调用次数上限。

后续课程：Day 10 的 Planning（规划）模式通过前置计划来减少盲目试错，从根本上降低不必要的工具调用。

### 7.3 上下文线性膨胀（Context Bloat）

现象：每步都向对话历史追加 Thought + Action + Observation。到第 12 步时，模型需要读完前面 11 步的全部历史才能决定第 12 步做什么。

研究发现，当 ReAct 轨迹超过约 4000 tokens 时，模型选择正确下一步的准确率急剧下降——这是"迷失在中间（Lost in the Middle）"现象在 Agent 内部的表现。

更糟的是，Observation 往往是噪声占比极高的数据。一次网页搜索可能返回 2000 字的搜索结果片段，其中 90% 跟当前任务无关。6 步 ReAct 循环典型消耗 8~14 倍于单次补全的输入 token，因为每一步都重放完整的 Thought/Action/Observation 历史。

缓解：
- 对 Observation 做截断/摘要后再注入历史（而非原样拼接）
- 超过一定步数后，对历史 Thought 做压缩（保留结论，去掉推理细节）
- 使用 ReAct 做规划阶段，执行阶段用更紧凑的格式

后续课程：Day 11 的记忆管理（上下文压缩、滑动窗口、向量化记忆）专门解决此问题。

---

## 8. ReAct 为什么是基石

后续所有 Agent 模式都构建在 ReAct 之上：

- **Reflection（反思）** = 在 ReAct 的 Thought 阶段增加一个"回顾历史 + 评价自己表现"的子步骤
- **Planning（规划）** = 在 ReAct 循环开始前，先跑一次"只 Think 不 Act"的计划生成阶段
- **Plan-and-Execute** = 把 ReAct 的 T→A→O 循环拆成两个独立阶段：规划时用 ReAct 充分推理，执行时用精简的 A→O
- **Multi-Agent（多智能体）** = 多个 ReAct 循环并行或串行运行，通过消息传递互相触发 Observation

搞懂 ReAct，后续学任何 Agent 模式都只需要理解"它在 ReAct 的基础上加了一层什么"。如果不理解 ReAct 就直接学框架，你只是在记 API 用法，不是在理解 Agent。

---

## 9. 今日小结

- ReAct = Reasoning + Acting，让推理和行动交替进行——先想再做，看了结果再想。
- ReAct 循环的标准结构是 Thought → Action → Observation 三段式，这是几乎所有现代 Agent 框架的底层协议。
- 与纯 CoT（幻觉率 56%）和纯 Act-only（盲目调用）相比，ReAct 将 HotPotQA 幻觉率降到 6%，同时保持了行动能力。
- 停止条件不是可选项——三种核心条件（任务完成 / 无法继续 / 步数上限）缺一不可，生产环境还需扩展 Token 预算、时间预算、重复检测。
- ReAct 有三大局限：循环死锁（90.8% 重试来自工具名幻觉）、工具过度调用（确认癖）、上下文线性膨胀（8~14 倍 token 消耗）。这些恰好为后续课程提供了演进方向。

明天（Day 05），我们将动手构建第一个真正调用 LLM 的 Agent——把今天写的模拟 Thinker 替换成真实的 Claude/GPT API，让它真正能查天气、做计算、搜索网页。

---

## 参考资料

1. Yao, S., Zhao, J., Yu, D., Du, N., Shafran, I., Narasimhan, K., & Cao, Y. (2022). *ReAct: Synergizing Reasoning and Acting in Language Models*. arXiv:2210.03629. 发表于 ICLR 2023.
2. AI/TLDR. (2026, June 12). *ReAct Agent Pattern: Reasoning + Acting in LLMs Explained*. https://ai-tldr.dev/learn/ai-agents/agent-fundamentals/react-agent-pattern/
3. Mullapudi, M. (2026, March 27). *ReAct Pattern — Reasoning and Acting in Language Models*. tutorialQ. https://tutorialq.com/ai/dl-applications/react-pattern
4. Outcomeschool. (2026, April 30). *ReAct Agent*. https://outcomeschool.com/blog/react-agent
5. Growth Engineer. (2026, May 4). *Implement the ReAct Pattern in 50 Lines of Python (Then 10 Lines with Claude Agent SDK)*. https://growthengineer.ai/blog/react-pattern-implementation-python
6. AgentPatterns.ai. (n.d.). *ReAct (Reason + Act): Interleaved Reasoning-Action Loops*. https://agentpatterns.ai/agent-design/react-pattern/
7. CallSphere. (2026, March 20). *Agent Reasoning and Planning: Chain-of-Thought, ReAct, and Tree-of-Thought Patterns*. https://callsphere.ai/blog/agent-reasoning-planning-chain-of-thought-react-tree-of-thought-2026
8. Building Agentic AI. (2026, May 24). *ReAct vs Plan-and-Execute: Which Agent Pattern, and How to Explain It in an Interview*. https://buildingagenticai.com/blog/react-vs-plan-and-execute/
