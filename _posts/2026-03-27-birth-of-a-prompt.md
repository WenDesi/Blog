---
layout: post
title: "一条 Prompt 的诞生"
date: 2026-03-27
categories: [tech]
tags: [codex, ai]
---

# **Overview**

这段时间刷了不少 Agents 项目，质量虽说参差不齐，但剥开外壳看底子，核心逻辑基本都没逃出 **ReAct** 那一套。

某种程度上，单 Agent 的架构模式已经步入了“稳态”。所以，我打算以 **CodeX** 为标杆开个新坑，把这些 Agents 的架构彻底深扒一遍。

开篇之作，我想聊聊 Prompt 的拼接逻辑。两年前的我很难想象，一个仅仅靠“预测下一个词”起家的模型，竟然能发展到几乎代替人类编程的程度。而这一切的起点，其实都藏在第一行发给模型的指令里。

影史上有一部《一个国家的诞生》，它标志着电影叙事艺术的开端。我也把这个系列叫作 **《一条 Prompt 的诞生》**，想以此致敬迈向 LLM 时代那最关键的第一声“招呼”。

Talk is cheap, let's peek at the payload. 既然是“诞生”，那我们就从最底层的 Request 数据结构开始溯源。

# **Request**

拆解 CodeX 的 Request body，你会发现它比普通的 Chat API 要臃肿得多。为了不漏掉细节，我把全量 Payload 字段都整理在了下面：

| 字段 (Column) | 示例 (Sample) | 说明 (Notes) |
| :---- | :---- | :---- |
| **model** | gpt-5.4 | 用户选择的模型 |
| **instructions** | "You are Codex..." | OpenAI Responses API 的结构，约等于 system prompt |
| **input** | \[{"type": "message",... | OpenAI Responses API 的结构，等同于 messages |
| **tools** | \[{"type": "function"... | 提供的工具 |
| **tool\_choice** | auto | 控制模型是否/如何调用工具 |
| **parallel\_tool\_calls** | true | 是否允许并行调用多个工具 |
| **reasoning** | {"effort": "medium"} | 控制推理的强度 |
| **store** | false | 控制是否在 OpenAI 存储 response |
| **stream** | true | 决定打包返回还是流式返回 |
| **include** | \["reasoning.encrypted\_content"\] | 控制返回哪些额外内容 |
| **prompt\_cache\_key** | "xxxx" | 用来控制缓存的标识 |
| **text** | {"verbosity":"low"} | 控制返回文本的输出格式（冗余度控制） |

从这里就能看到 OpenAI 的代码风格与 OpenClaw 差别极大。

就以下面的参数为例，同样是要求大模型保持简洁回复，OpenAI 倾向于结构化的数据格式，而 OpenClaw 则喜欢放入到文本一把梭哈。这也可能是因为 OpenAI 是模型提供商，他对模型有更强大的控制能力，而 OpenClaw 作为模型应用只能选择相信大模型这一条路。

"text": {  
  "verbosity": "low"  
}

下面我们深入看一下 instructions, input 还有 tools 是如何拼接到 Prompt 当中的。

## **Instructions**

Instructions（AKA system prompt）对 CodeX agent 做了一些约束，例如工具偏好（强制使用 apply\_patch）、执行策略（端到端交付）等。

你可以通过配置文件使用自己的 Instructions，也可以根据模型使用默认的配置。

* OpenAI 模型专属 Instructions：https://github.com/openai/codex/blob/main/codex-rs/core/models.json  
* 兜底的 Instructions：https://github.com/openai/codex/blob/main/codex-rs/protocol/src/prompts/base\_instructions/default.md

吐槽一下：Prompt 的位置和文件类型感觉毫无规律啊\!\!

下面是各模型和 default instructions 的差别，这里有非常详尽的数据对比：

| Model | Chars | Lines | Similarity to Base |
| :---- | :---- | :---- | :---- |
| gpt-5 | 20771 | 275 | 100.0% |
| gpt-5-codex | 6621 | 68 | 6.8% |
| gpt-5-codex-mini | 6621 | 68 | 6.8% |
| gpt-oss-120b | 20771 | 275 | 100.0% |
| gpt-oss-20b | 20771 | 275 | 100.0% |
| gpt-5.1 | 24046 | 331 | 87.1% |
| gpt-5.1-codex | 6621 | 68 | 6.8% |
| gpt-5.1-codex-mini | 6621 | 68 | 6.8% |
| gpt-5.1-codex-max | 7563 | 80 | 5.6% |
| gpt-5.2 | 21544 | 298 | 83.6% |
| gpt-5.2-codex | 7563 | 80 | 5.6% |
| gpt-5.3-codex | 12341 | 114 | 4.4% |
| **gpt-5.4** | **14100** | **121** | **2.8%** |

从中可以发现一些有意思的点：

* **gpt-5.4 之前**，codex 模型和 gpt 通用模型的 instructions 差别很大，并且 codex 的 instructions 要短得多，怀疑是 codex 训练的时候就已经把 instructions 的内容训练进去了。  
* **gpt-5.4 是一个全新的模型**，包含了 codex 模型的能力，这一点也可以从 models.json 的设置中看出来，例如 gpt 等通用模型 truncation\_policy 都设置是 byte，但是 codex 模型设置的都是 token，gpt-5.4 设置的也是 token。

Instructions 预留了两种性格配置：

* **friendly**：温暖、鼓励、团队协作风格。  
* **pragmatic**：直接、务实、简洁（默认值）。

但奇怪的是，CodeX 并没有提供配置项让用户自己去设置性格，这点不知道是为什么？

吐槽：网页版的 ChatGPT 无比啰嗦，但是 CodeX 却回复得很简洁。没准是 OpenAI 团队自己每天用 CodeX，也嫌它烦，所以才这么设计。

## **Input**

Input 对应的就是之前 chat 模式下的 messages。我们以第一轮对话为例分析一下内容，你会发现用户的原始 prompt 并不是直接发过去的，CodeX 在前面**预先注入**了一个 developer 消息和一个上下文 user 消息。

"input": \[  
  { "type": "message", "role": "developer", "content": \[...\] },  
  { "type": "message", "role": "user", "content": \[...\] },  
  {  
    "type": "message",  
    "role": "user",  
    "content": \[  
      {  
        "type": "input\_text",  
        "text": "Check how the prompt sent to LLM is spliced in the codex."  
      }  
    \]  
  }  
\]

简单来说，CodeX 将执行过程中可变化的内容都放置在了 Input 模块中，而完全不变的内容（比如最高纲领）则放置在 Instruction 模块中。在压缩上下文时不会对 Instruction 有任何的修改。

### **Developer 消息**

Developer 消息里一共套了三个核心子模块：

* **permission instructions**：定义沙箱模式（Workspace Write/Read Only）、审批策略（On Request/Never）以及写入根路径。  
* **collaboration mode**：指导 LLM 怎么跟人打交道。目前只开放了 **Default** 和 **Plan** 两种模式，Pair Programming 和 Execute 模式尚在预留中。  
* **skills instructions**：扫描项目目录下的 SKILL.md 并注入。

吐槽一下：permissions instructions 中间是空格，而 collaboration\_mode 又是下划线，强迫症表示很难受。

### **User 消息**

第一段 user 消息主要塞了两块背景：

* **AGENTS.md**：用户给当前仓库设置的规则约束。注意：超过 32KB 会被截断。  
* **environment\_context**：环境快照，包括 cwd、默认 shell、当前日期以及时区（timezone）。

## **Tools**

Tools 是 ReAct loop 能转起来的燃料。CodeX 在分类和分发机制上玩得相当细致，其精髓在于**动态分发**：

* **Built-in / MCP / Connector / Discoverable / Dynamic**：五花八门的工具分类确保了扩展性。  
* **tool\_search 机制**：针对数量庞大的工具集（超过 100 个），不再全量暴露，而是只给模型一个 tool\_search 元工具。模型必须先通过 **BM25 检索** 找到需要的工具定义，这种“按需索取”的策略极大节省了 Context。

# **Summary**

说到底，CodeX 的 Prompt 拼接本质上是在给大模型构建一套**受控的指令运行时环境**。

其精髓就在于**用极致的约束去换取执行的确定性**：Instructions 设定了底层的物理常数，Developer 消息注入了包含权限与协作模式的内核逻辑，而 Tools 则是向上层暴露的系统调用（Syscalls）。

静态的“运行时环境”搭建完了，下一步就是让它跑起来。下一篇，我们拆解这个内核是如何驱动 ReAct 循环，驱动提示词在动态博弈中转化成真实的代码变更的。