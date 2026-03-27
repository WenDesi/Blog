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

> 完整的request 路径： [1_request.json](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/1_request.json)

从这里就能看到 OpenAI 的代码风格与 OpenClaw 差别极大。

就以下面的参数为例，同样是要求大模型保持简洁回复，OpenAI 倾向于结构化的数据格式，而 OpenClaw 则喜欢放入到文本一把梭哈。这也可能是因为 OpenAI 是模型提供商，他对模型有更强大的控制能力，而 OpenClaw 作为模型应用只能选择相信大模型这一条路。

```
"text": {  
  "verbosity": "low"  
}
```

下面我们深入看一下 instructions, input 还有 tools 是如何拼接到 Prompt 当中的。

## **Instructions**

Instructions（AKA system prompt）对 CodeX agent 做了一些约束，例如工具偏好（强制使用 apply_patch）、执行策略（端到端交付）等。

> Instruction example: [request_instructions.md](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/request_instructions.md)

你可以通过配置文件使用自己的 Instructions，也可以根据模型使用默认的配置。

* OpenAI 模型专属 Instructions：[models.json](https://github.com/openai/codex/blob/main/codex-rs/core/models.json)
* 兜底的 Instructions：[default.md](https://github.com/openai/codex/blob/main/codex-rs/protocol/src/prompts/base\_instructions/default.md)

> 吐槽一下：Prompt 的位置和文件类型感觉毫无规律啊\!\!

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

> 吐槽：网页版的 ChatGPT 无比啰嗦，但是 CodeX 却回复得很简洁。没准是 OpenAI 团队自己每天用 CodeX，也嫌它烦，所以才这么设计。

## **Input**

Input 对应的就是之前 chat 模式下的 messages。我们以第一轮对话为例分析一下内容，你会发现用户的原始 prompt 并不是直接发过去的，CodeX 在前面预先注入了一个 developer 消息和一个上下文 user 消息。

```
{
  "input": [
    {
      "type": "message",
      "role": "developer",
      "content": [...]
    },
    {
      "type": "message",
      "role": "user",
      "content": [...]
    },
    {
      "type": "message",
      "role": "user",
      "content": [
        {
          "type": "input_text",
          "text": "Check how the prompt sent to LLM is spliced in the codex."
        }
      ]
    }
  ]
}
```

简单来说，CodeX 将执行过程中可变化的内容都放置在了 Input 模块中，而完全不变的内容（比如最高纲领）则放置在 Instruction 模块中。在压缩上下文时不会对 Instruction 有任何的修改。

### Developer

Developer message 里一共套了三个核心子模块：
 - permission instructions
 - collaboration mode
 - skills instructions

```
{
    "type": "message",
    "role": "developer",
    "content": [
        {
            "type": "input_text",
            "text": "<permissions instructions>\nFilesystem sandboxing defines which fil..."
        },
        {
            "type": "input_text",
            "text": "<collaboration_mode># Collaboration Mode: Default\r\n\r\nYou are no..."
        },
        {
            "type": "input_text",
            "text": "<skills_instructions>\n## Skills\nA skill is a set of local instruc..."
        }
    ]
}
```

> 再次吐槽一下，permissions instructions 中间为啥是空格，collaboration_mode 中间为啥是下划线？

#### permissions instructions
Permissions instructions 用来控制权限，其由三个模块组成
```
<permissions instructions>
  sandbox_mode
  approval_policy
  writable_roots          
</permissions instructions>
```

> permissions instructions example: [request_developer_permissions_instructions.md](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/request_developer_permissions_instructions.md)

Sandbox 是用来告诉LLM目前workspace的执行边界，避免其做一些无畏的尝试，例如用户设置了read only模式，LLM就不要去生成写指令了，反正也执行不了。它包括以下三种模式：
| 模式 | Notes |
| --- | --- | 
| [workspace write](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/sandbox_mode/workspace_write.md?plain=1) | 可读全部，可写 cwd + writable_roots|
| [read only](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/sandbox_mode/read_only.md) | 只读 |
| [danger full access](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/sandbox_mode/danger_full_access.md) | 无限制 |

Approval Policy 则是用来约束用户的审批流程
| 模式 | Notes |
| --- | --- | 
| [on request](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/approval_policy/on_request.md) | 按需请求审批 |
| [unless trusted](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/approval_policy/unless_trusted.md) | 除了安全白名单命令，其余都需审批 |
| [on failure](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/approval_policy/on_failure.md) | 先在沙箱里跑，失败了再升级 |
| [never](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/protocol/src/prompts/permissions/approval_policy/never.md) |  永不通过 |


Writable roots 用来告诉LLM哪些路径可以写入，包括但不限于：
1. 当前目录
2. Memory目录
3. config配置的目录
4. etc

#### collaboration mode
collaboration mode 是用来指导LLM的编程模式，目前有两种模式 Default 和 Plan，但是Repo中包含了两种新的模式，目前还没有启用。

> collaboration mode example: [request_developer_collaboration_mode.md](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/request_developer_collaboration_mode.md)

| 模式 | 用途 |
| --- | --- | 
| [Default](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/default.md) | 正常工作模式，直接执行用户的请求，自己做一些合理假设，不主动提问 |
| [Plan](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/plan.md) | 纯规划模式，只读文件，分三个阶段 （探索 -> 确认意图 -> 出方案）|
| [Pair Programming](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/pair_programming.md) | 结对编程，codex会频繁和用户进行交互，当遇到多条路径的时候出选项让用户选择，和Default模式相比，属于小碎步模式。|
| [Execute](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/execute.md) | 完全独立执行，自己构建Plan，并且根据Plan来执行，绝对不向用户提问，属于另一个极端，感觉比Default更激进一些 |

> 好难忍住不吐槽啊！！instruction prompt的路径是 xxx/prompts/yyy, 而这个又变成了 xxx/template/yyy. 能不能有一个统一的规范啊！！！

#### skills
下一个message part是skills，他是通过扫描 repo到CWD之间每层的skills folder、$HOME/.agents/skills/等路径下的SKILL.md文件，读取skill name和description，并将对应路径加入到prompt当中。

> skills example: [request_developer_skills.md](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/request_developer_skills.md)

在下面这段代码中进行拼接：https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core-skills/src/render.rs

> 吐槽加疑问：为啥在代码中自己拼接，而不是用例如liquid等package呢？

> 我们会在后续blog中详细介绍skill的工作原理

### User

第一段 user message中一共有以下两个模块：
 - AGENTS.md
 - environment_contrext

```
{
    "type": "message",
    "role": "user",
    "content": [
        {
            "type": "input_text",
            "text": "# AGENTS.md instructions..."
        },
        {
            "type": "input_text",
            "text": "<environment_context>\n  <cwd>..."
        }
    ]
}
```

#### AGENTS.md

AGENTS.md就是用户为当前repo设置的一些规则或者说是约束吧，
CodeX会从CWD开始向上遍历，找到repo根目录为止，将每一层的 AGENTS.md都读取出来。并按照 根目录到CWD的顺序拼接起来。
作为 role: "user" 的 contextual user message 片段放入 input。

> AGENTS.MD example: [request_user_agent.md](https://github.com/WenDesi/Blog/blob/main/_brain_dump/codex/message/request_user_agent.md)

有一点需要注意，AGENTS.md不能太大，所有AGENT.md的大小需要小于project_doc_max_bytes (由config配置，默认是[32KB](https://github.com/openai/codex/blob/d838c23867f6fe2e3e43a0c82d25c17cb3730b1d/codex-rs/core/src/config/mod.rs#L139))，超过的部分会直接截断。

#### environment_context
environment context是最后一块codex自己添加的message，它包含以下信息：

| name | notes |
| --- | --- |
| cwd | CodeX的工作目录 |
| shell | 默认的shell，例如windows是powershell |
| current_date | 当天日期 |
| timezone | 市区 |

## Tools
Tools是ReAct loop能够转起来的关键组件，不过这一步的prompt拼接通常没啥好说的，各家agents都做得大同小异，核心差异点在如何避免tools数量过多和如何选择到正确的tools，这个问题我们会在之后的blog中讨论。

下面说一下CodeX的tools的分类：
1. Built-in tools：系统内置的“一等公民”，像 shell、apply_patch 这种核心组件，拥有专属的 handler。
2. MCP tools：外部 MCP server 提供的工具。
3. Connector tools：源自内部特殊 MCP server ("codex_apps") 的工具集。
4. Discoverable tools：未启用但当前平台可用的插件。它们不会被直接注入定义，而是通过 tool_suggest 诱导模型主动向用户推荐。
5. Dynamic tool：由外部调用方在启动 Session 时临时注入，Codex仅负责转发调用请求。

关键设计：动态分发

针对 Connector tools，CodeX 有个很有意思的设定：

- 数量 < 100：直接全量丢进 Prompt 定义。
- 数量 > 100：不再直接暴露工具细节，而是只给模型一个 tool_search 元工具。
tool_search 原理：当工具泛滥时，模型必须先调用 tool_search。CodeX 会利用 name、description 和 input_schema 等字段做 BM25 检索，按需把命中的工具定义喂给模型。这种“按需索取”的策略极大地节省了 Context 空间。

# **Summary**

说到底，CodeX 的 Prompt 拼接本质上是在给大模型构建一套**受控的指令运行时环境**。

其精髓就在于**用极致的约束去换取执行的确定性**：Instructions 设定了底层的物理常数，Developer 消息注入了包含权限与协作模式的内核逻辑，而 Tools 则是向上层暴露的系统调用（Syscalls）。

静态的“运行时环境”搭建完了，下一步就是让它跑起来。下一篇，我们拆解这个内核是如何驱动 ReAct 循环，驱动提示词在动态博弈中转化成真实的代码变更的。