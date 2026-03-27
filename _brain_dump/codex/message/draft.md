
# Overview
这段时间刷了不少 Agents 项目，质量虽说参差不齐，但剥开外壳看底子，核心逻辑基本都没逃出 ReAct 那一套。

某种程度上，单 Agent 的架构模式已经步入了“稳态”。所以，我打算以 CodeX 为标杆开个新坑，把这些 Agents 的架构彻底深扒一遍。

开篇之作，我想聊聊 Prompt 的拼接逻辑。两年前的我很难想象，一个仅仅靠“预测下一个词”起家的模型，竟然能发展到几乎代替人类编程的程度。而这一切的起点，都藏在第一行发给模型的指令里。

影史上有一部《一个国家的诞生》，它标志着电影叙事艺术的开端。我也把这个系列叫作**一条 Prompt 的诞生**，想以此致敬迈向 LLM 时代那最关键的第一声“招呼”。

Talk is cheap, let's peek at the payload. 既然是‘诞生’，那我们就从最底层的 Request 数据结构开始溯源。

# Request
拆解 CodeX 的 Request body，你会发现它比普通的 Chat API 要臃肿得多。为了不漏掉细节，我把全量 Payload 字段都整理在了下面：


| Column | Sample |Notes |
| --- | --- | --- |
| model | gpt-5.4 | 用户选择的模型 |
| instructions | "You are Codex..." | OpenAI Responses API的结构，约等于system prompt |
| input | [{"type": "message",... | OpenAI Responses API的结构，等同于messages |
| tools | [{"type": "function"... | 提供的工具|
| tool_choice | auto | 控制模型是否/如何调用工具, auto就是让模型自己决定是否调用工具 |
| parallel_tool_calls | true | 是否允许并行调用多个工具 |
| reasoning | {"effort": "medium"} | 控制推理的强度 |
| store | false | 控制是否在OpenAI存储response |
| stream | true | 决定打包返回还是流式返回 |
| include | ["reasoning.encrypted_content"] | 又是OpenAI Responses API独有的结构，用来控制返回哪些额外内容|
| prompt_cache_key | "xxxx" | 用来控制缓存的 |
| text | {"verbosity":"low"} | 控制返回文本输出格式 | 

从这里就能看到 OpenAI的代码风格与OpenClaw差别极大。

就以下面的参数为例，同样是要求大模型保持间接回复，OpenAI倾向于结构化的数据格式，而OpenClaw则喜欢放入到文本一把梭哈。
这也可能是因为OpenAI是模型提供商，他对模型有更强大的控制能力，而OpenClaw作为模型应用只能选择相信大模型这一条路。
```
"text": {
  "verbosity": "low"
}
```

下面我们深入看一下instructions, input还有tools是如何拼接到Prompt当中的。

## Instructions
Instructions, AKA system prompt, 对CodeX agent做了一些约束，例如工具偏好（强制使用apply_patch），执行策略等（端到端交付）等。
可以通过配置文件使用自己的Instructions，也可以根据模型使用默认的Instructions。

- OpenAI 自己的模型的Instructions保存在：https://github.com/openai/codex/blob/main/codex-rs/core/models.json

- 兜底的Instructions则在：https://github.com/openai/codex/blob/main/codex-rs/protocol/src/prompts/base_instructions/default.md

> 吐槽一下Prompt的位置和文件类型，感觉毫无规律啊!! 

下面是各模型和default instructions的差别，有一些比较有意思的发现：

- gpt-5.4之前，codex模型和gpt通用模型的instructions差别很大，并且codex的instructions要短得多，怀疑是codex训练的时候就已经把instructions的内容训练进去了。
- gpt-5.4是一个全新的模型，包含了codex模型的能力，这一点也可以从models.json的设置中看出来，例如gpt等通用模型truncation_policy都设置是byte，但是codex模型设置的都是token，gpt-5.4设置的是token.


| Model | Chars | Lines | Similarity to Base |
|-------|-------|-------|--------------------|
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
| gpt-5.4 | 14100 | 121 | 2.8% |


Instructions 可以配置性格, 可以配置以下两种：
 - friendly，温暖、鼓励、用"we/let's"、团队协作风格      
 - pragmatic， 直接、务实、简洁、不废话（默认值）

但是CodeX没有提供配置项让用户自己去设置性格，这点不知道是为什么？

> 网页版的chatgpt无比啰嗦，但是codex却回复的很简洁，没准是openAI团队自己每天用codex，也嫌它烦，所以这么设计。

## Input
Input就是之前chat 模式的messages，我们以第一轮对话为例，分析一下里面的内容：
可以看到，用户的prompt前被插入了一个developer message和一个user message。

```
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
```

简单来说，CodeX 将执行过程中可变化的内容都放置在了 Input 模块中，对于完全不变的内容则放置在 Instruction 模块中。在压缩上下文时不会对 Instruction 有任何的修改。

### Developer

Developer message中一共有以下三个模块：
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

| 模式 | 用途 |
| --- | --- | 
| [Default](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/default.md) | 正常工作模式，直接执行用户的请求，自己做一些合理假设，不主动提问 |
| [Plan](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/plan.md) | 纯规划模式，只读文件，分三个阶段 （探索 -> 确认意图 -> 出方案）|
| [Pair Programming](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/pair_programming.md) | 结对编程，codex会频繁和用户进行交互，当遇到多条路径的时候出选项让用户选择，和Default模式相比，属于小碎步模式。|
| [Execute](https://github.com/openai/codex/blob/7ef3cfe63e435ee03812cfb818c4ba8a063a6833/codex-rs/core/templates/collaboration_mode/execute.md) | 完全独立执行，自己构建Plan，并且根据Plan来执行，绝对不向用户提问，属于另一个极端，感觉比Default更激进一些 |

> 好难忍住不吐槽啊！！instruction prompt的路径是 xxx/prompts/yyy, 而这个又变成了 xxx/template/yyy. 能不能有一个统一的规范啊！！！

#### skills
下一个message part是skills，他是通过扫描 repo到CWD之间每层的skills folder、$HOME/.agents/skills/等路径下的SKILL.md文件，读取skill name和description，并将对应路径加入到prompt当中。

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

1. Built-in tools
CodeX中有很多内置的工具，他们都有各自专属的handler，例如shell, apply_patch，update_plan等，这些工具都是codex中的一等公民。

2. MCP tools
是用户配置的外部MCP server提供的工具。

3. Connector tools
本质上也是一种MCP tools，codeX在内部配置其来源自一个特殊的MCP server "codex_apps"。

Connector tools有一个比较有意思的设定，如果tools的数量少于100个，则直接加入到prmopt当中，如果多余100个则不直接暴露，而是通过增加一个tool_search的工具，让LLM先去调用这个工具进行搜索，然后再按需调用。

> tool_search，用connector tool的name,title, description还有input_schema等字段做BM25进行检索。

4. Discoverable tools

Discoverable tools 是当前 session 中还没有启用、但平台上可用的 connector 和 plugin。它们不直接暴露为可调用工具，而是通过一个  tool_suggest 元工具让模型"发现"并推荐给用户。

5. Dynamic tool 
Dynamic tool是外部调用方在启动 Codex session 时注入的自定义工具。Codex
  自己不知道怎么执行它们，只是把工具定义注册给模型，当模型调用时再把请求转发回调用方处理。

# Summary
以上就是 CodeX 是如何拼接 request的，