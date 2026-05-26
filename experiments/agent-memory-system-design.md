# Agent Memory System Design — 笔记

> **课程**：Long-term Memory for AI Agents：如何让 Agent 拥有持续上下文与长期一致性
> **日期**：2026-05-25（周一）Week 2 · Day 1
> **回放链接**：<https://x.com/i/broadcasts/1mGPaLvzdLmJN>
> **关联任务**：`cmpla20yg3b1amq01davjhcex` · `cmpla21ed3b1dmq01vf8ifdyj`

---

## 🧠 核心问题

**为什么 Agent 需要记忆系统？**

> 每一次对话都是全新的开始 — 这是 LLM 的天然限制。没有记忆，Agent 无法：
> - 记住用户是谁、他的偏好、他的项目
> - 跨 session 复用之前解决过的问题的经验
> - 从错误中学习，下次做得更好
> - 在长时间的工作流中保持上下文一致性

---

## 📐 记忆的三层架构

记忆系统通常分为三层，分层设计 —— **存储、检索、应用**：

```
┌─────────────────────────────────────────┐
│           应用层 (Application)             │
│  context injection / preference matching │
├─────────────────────────────────────────┤
│           检索层 (Retrieval)              │
│  semantic search / recency / relevance   │
├─────────────────────────────────────────┤
│           存储层 (Storage)                │
│  vector DB / KV store / file system      │
└─────────────────────────────────────────┘
```

---

## 🏗️ 记忆的三种类型

### 1. 情景记忆 (Episodic Memory)
| 属性 | 说明 |
|------|------|
| **存储内容** | 具体事件、对话历史、操作记录 |
| **典型实现** | 对话日志、session 回放、时间线 |
| **用途** | "上次我们讨论到哪了"、"那个错误是怎么修的" |
| **持久性** | 通常短期（几天到几周），可滚动淘汰 |
| **示例** | Hermes Agent 的 `session_search` 搜索历史对话 |

### 2. 语义记忆 (Semantic Memory)
| 属性 | 说明 |
|------|------|
| **存储内容** | 事实、概念、用户偏好、项目配置 |
| **典型实现** | key-value store、向量数据库、memory 工具 |
| **用途** | "用户喜欢用 DeepSeek"、"时区是 UTC+8"、"项目用 pytest" |
| **持久性** | 长期，除非主动删除 |
| **示例** | Hermes Agent 的 `memory` 工具 + `user`/`memory` 双区 |

### 3. 程序记忆 (Procedural Memory)
| 属性 | 说明 |
|------|------|
| **存储内容** | 工作流、技能、工具调用模式 |
| **典型实现** | skill 文件、脚本、模板 |
| **用途** | "如何部署合约"、"如何做代码审查" |
| **持久性** | 长期，可迭代更新 |
| **示例** | Hermes Agent 的 `skill_manage` + `skill_view` 技能系统 |

> 💡 **关键洞见**：好的 Agent 需要三种记忆配合工作。语义记忆回答"用户是谁"，情景记忆回答"我们刚干了什么"，程序记忆回答"这事该怎么做"。

---

## 🗄️ 记忆存储方案对比

| 方案 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **文件系统** (Markdown/JSON) | 个人 Agent、小规模 | 简单、可读、可版本控制 | 检索慢、不支持语义搜索 |
| **Key-Value Store** (Redis/LevelDB) | 偏好、配置类记忆 | 极快、简单 | 不支持相似性搜索 |
| **向量数据库** (Chroma/Pinecone/Qdrant) | 语义搜索、RAG | 相似性检索强、可扩展 | 需要 embedding 模型、运维成本 |
| **关系型数据库** (SQLite/PostgreSQL) | 结构化记忆、关联查询 | 强一致、事务支持 | 不适合非结构化数据 |
| **图数据库** (Neo4j) | 实体关系复杂的记忆 | 关系查询强 | 学习成本高 |

---

## 🔍 记忆检索策略

### 1. 最近优先 (Recency-based)
- 按时间排序，最近的使用最频繁
- 适合：对话上下文、操作历史
- **缺点**：可能错过久远但重要的信息

### 2. 语义匹配 (Semantic-based)
- 用 embedding 计算相似度，找到最相关的内容
- 适合：知识查询、问题匹配
- **缺点**：需要 embedding 模型 + 向量数据库

### 3. 重要性排序 (Importance-based)
- 给每条记忆打重要性分数，高优先级的被保留
- 适合：长期记忆的缓存淘汰策略
- **实现**：可用 LLM 自评"这条记忆多重要？"

### 4. 混合策略 (Hybrid)
```
score = w₁ × recency + w₂ × relevance + w₃ × importance
```
- 实际系统中通常组合多种策略
- 权重可根据场景动态调整

---

## 🧩 Hermes Agent 记忆系统实例分析

Hermes Agent 实现了完整的记忆三层架构，可作为参考设计：

### 记忆层 (Semantic Memory)
```python
# 工具：memory
# 双区存储：
memory(action="add", target="user", content="User prefers DeepSeek")
memory(action="add", target="memory", content="Timezone set to Asia/Shanghai")
```
- **`user` 区**：用户个人信息、偏好
- **`memory` 区**：Agent 自身笔记、环境事实、项目约定
- 每次对话自动注入，Agent 无需重新了解用户

### 技能层 (Procedural Memory)
```python
# 工具：skill_manage / skill_view / skills_list
skill_manage(action="create", name="deploy-contract", content="...")
```
- 记录可复用的工作流
- 支持增量补丁（patch），不从头重写
- 自动匹配加载（任务匹配时技能自动注入）

### 会话搜索 (Episodic Memory)
```python
# 工具：session_search
session_search(query="testnet transaction Sepolia")
```
- 搜索历史对话，类似"Agent 的短期记忆回放"
- 支持跨 session 查询

### 记忆优先级原则
Hermes 的 memory 工具明确规定了：
- ✅ **保存**：用户偏好、环境事实、项目约定 —— 长期有用
- ❌ **不保存**：任务进度、PR 编号、commit SHA、临时状态 —— 会过期

> 💡 这是很实用的设计原则：**记忆只存不会过期的事实**。临时状态靠 session_search 从历史里捞。

---

## 🔗 与 Web3 Agent 的结合

### 场景 1：链上身份关联记忆
```
Agent 记住用户的链上身份（ENS / 地址 / Safe 地址）
→ 下次对话自动识别用户身份
→ 关联对应的合约、交易、PoW 记录
```

### 场景 2： Session Key 权限记忆
```
Agent 记住已授权的 Session Key 策略
→ 不需要每次重新授权
→ 但记忆不存私钥！（永远不可跨 session 存储私钥）
```

### 场景 3：交易上下文记忆
```
Agent 记住之前的交易上下文
→ 如 "上次部署到 Sepolia 的合约地址是 0x..."
→ 下次操作可以直接交互，不需要重新查询
```

---

## ⚠️ 安全边界与注意事项

| 规则 | 说明 |
|------|------|
| **绝对不存私钥/助记词** | 记忆系统不能跨 session 持久化敏感凭证 |
| **记忆可被用户查看和删除** | 用户应有完全控制权 |
| **记忆不能无限增长** | 需要大小限制、淘汰策略 |
| **跨 session 记忆有隐私风险** | 如果多个用户共享一个 Agent，需隔离记忆空间 |
| **程序记忆需要人工确认** | 自动保存的技能可能包含错误，需要 review 机制 |

---

## 📝 我的理解 & 关键 takeaways

1. **记忆是 Agent 与普通 LLM 调用的核心区别** — 没有记忆，每一次对话都是"初次见面"
2. **分层设计比单体设计更健壮** — 情景/语义/程序三层互相补充，各自解决不同问题
3. **存储不是关键，检索才是** — 不能高效找到的记忆等于不存在
4. **写 Memory 比读 Memory 更难** — 决定"什么值得记"比"怎么取用"更需要设计
5. **Web3 Agent 对记忆的要求更高** — 链上操作有不可逆性，记忆的准确性直接影响资产安全

---

## 🔗 参考资源

- **回放地址**：<https://x.com/i/broadcasts/1mGPaLvzdLmJN>
- **AI × Web3 School Handbook**：<https://aiweb3.school/zh/handbook/>
- **Hermes Agent 文档**：<https://hermes-agent.nousresearch.com/docs>
- **关联课程任务**：Week 2 方向探索与问题地图

---

*笔记创建于 2026-05-26 · 后续可补充实践案例*
