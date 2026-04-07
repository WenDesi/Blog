# Incubator - 博客构思孵化器

---

## 1. 一个 Prompt 的诞生

解析 Codex 是如何把用户输入的 prompt 经过层层拼接，最终组装成发送给 LLM 的完整调用的。

**状态：** 💡 构思中

---

## 2. 什么是 Sub Agent

讲解 Sub Agent 的概念：主 Agent 在执行复杂任务时，如何拆分出子任务并委派给独立的 Sub Agent 去完成，以及它们之间的通信和协作机制。

**状态：** 💡 构思中

---

## 3. Sandbox 原理

聚焦 Windows 平台，讲解 Codex 如何利用 Windows 沙盒机制（如 Windows Sandbox、Job Objects、AppContainer 等）实现代码的隔离执行与权限限制。

**状态：** 💡 构思中

## 4. Guardian 是什么
 Guardian 是一个独立的子 Agent（子 Codex 会话）
 Guardian 是安全审计员，独立坐在旁边，只能看、不能改，用自己的标准判断这个操作安不安全。

审计员不能用员工的权限卡，也不看员工自己写的"我可以做 XXX"的清单——否则审计就没意义了。

需要看他是怎么样保护的。