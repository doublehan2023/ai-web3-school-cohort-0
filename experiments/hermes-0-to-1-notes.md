# 5.19 回放笔记 — AI Agent 入门：Hermes 从 0 到 1

> 日期：2026-05-20 补看回放
> 任务：Week 1｜线上活动｜Watch 5.19 Recording｜AI Agent Intro: Hermes from 0 to 1

## 📌 笔记 1：Hermes 最核心的设计理念 — Provider 无关

Hermes 不是只绑定一个模型，而是支持 20+ 个 provider（OpenRouter、Anthropic、OpenAI、DeepSeek、本地模型等）。这意味着：

- 同一个 Agent 流程可以在不同模型间切换测试
- Credential pool 自动轮换多个 API key
- 坏 key 会被标记 exhaustion，自动换下一个
- 这对于 Web3 Agent 场景特别重要：不同任务（读链 vs 写交易）可以用不同模型

## 📌 笔记 2：Skills 机制让 Agent 能自我进化

Skill 不是固定的 prompt 库 — Agent 可以在工作中自动保存步骤，把试错经验持久化为可复用的技能文件：

- Agent 解决复杂问题后，可以 save as skill
- 下次遇到同类问题，skill 自动加载，不需要从零开始
- skill 可以补丁更新（patch），随着使用越来越准确
- 这意味着 AI × Web3 School 里踩过的坑都可以沉淀为技能，下期学员直接受益

## 📌 笔记 3：Hermes 的 Multi-Agent 和 Cron 系统

Hermes 不只是单次对话工具，它有 4 个持久化系统支撑复杂工作流：

1. **Delegate Task** — 同步子 Agent，适合并行处理
2. **Cron** — 定时任务，比如每天早上 9 点自动拉 WCB 课程 + 生成打卡草稿
3. **Curator** — 自动管理技能生命周期，清理过期技能
4. **Kanban** — 多 Agent 协作看板，适合分工开发

这让我想到在 Web3 Agent 场景里，Cron 可以定时检查链上状态，Delegate Task 可以并行分析多个合约。
