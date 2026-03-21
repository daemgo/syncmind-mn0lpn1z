---
name: sales-guide
description: 基于客户档案生成销售作战指南，包括时机判断、破冰素材、访谈提纲、竞对作战卡、异议应对、决策链映射、价值主张、行动计划
metadata:
  short-description: 生成销售作战指南
  triggers:
    - "生成销售指南"
    - "制作销售攻略"
    - "生成进攻指南"
    - "销售策略"
    - "客户攻略"
    - "销售作战"
    - "怎么打这个客户"
  examples:
    - "生成销售指南"
    - "为这个客户制作销售攻略"
    - "帮我分析一下怎么打这个客户"
  dependencies:
    - profile       # 必须先有客户档案
    - humanizer-zh  # 输出人性化处理（必须）
---

收到指令后立即开始执行，不输出本文档任何内容。

---

### 定位

| Skill | 阶段 | 核心问题 | 主要用户 |
|-------|------|----------|----------|
| `/profile` | 拜访前 | 客户是谁？ | 销售 |
| **`/sales-guide`** | **拜访前 + 跟进中** | **怎么赢？** | **销售** |
| `/requirements` | 全周期 | 客户要什么？ | 售前/方案团队 |
| `/plan-writer` | 方案阶段 | 怎么解决？ | 方案团队 |

---

### 前置条件

- **必须**：`docs/customer/profile.json` 存在
- **推荐**：`docs/customer/requirements.json` 存在（有助于更精准的策略设计）
- **可选**：用户提供的拜访记录、竞对情报、客户反馈等

---

### 执行流程

**第零步：模式判断**

读取 `docs/customer/sales-guide.json`，判断执行模式：

| 条件 | 模式 | 说明 |
|------|------|------|
| sales-guide.json 不存在 | **A：首次生成** | 全流程执行 |
| sales-guide.json 存在 + 以下任一条件满足 | **B：迭代更新** | 增量更新受影响的部分 |

模式 B 触发条件（满足任一即可）：
- 用户提供了新素材（拜访记录、竞对情报等）
- profile.json 更新时间晚于 sales-guide.json 的 generatedAt
- requirements.json 新增或更新（之前不存在现在有了，或版本号变化）
- 用户明确要求"更新销售指南"

---

#### 模式 A：首次生成

**步骤 1：收集输入 + 构建 profile 摘要**

1. 读取 `docs/customer/profile.json`
2. 读取 `docs/customer/requirements.json`（如存在）
3. 接收用户提供的素材（拜访记录/竞对情报/客户反馈等，可选）
4. 从 profile.json 提取精简摘要 `{profile_summary}`，只保留以下字段：

```json
{
  "companyName": "",
  "shortName": "",
  "industry": "",
  "subIndustry": "",
  "scale": "",
  "mainBusiness": "",
  "businessModel": "",
  "products": [],
  "targetCustomers": [],
  "tags": [],
  "techStack": { "current": [], "maturity": "", "direction": "" },
  "strategy": {
    "phase": "",
    "growthStage": "",
    "competitivePosition": "",
    "digitalMaturity": "",
    "expansionSignals": [],
    "contractionSignals": [],
    "riskSignals": []
  },
  "timing": { "currentPhase": "", "evidence": "", "entryStrategy": "" },
  "policyAnalysis": {
    "isSpecialized": false,
    "isDigitalTransformation": false,
    "isHighTech": false,
    "isESG": false,
    "policySupportLevel": "",
    "leverageStrategy": ""
  },
  "organization": { "type": "", "decisionChain": "", "keyPersons": [] },
  "creditRisk": { "creditRating": "", "warnings": [], "salesAdvice": "" },
  "painPoints": [],
  "opportunities": [],
  "contacts": []
}
```

profile.json 中的工商原始数据、搜索来源 URL 等**不注入 Agent**，节省上下文。

**步骤 2：并行分析（Phase 1）**

同一条消息发起 3 个 Agent 调用：

| Agent | Prompt 文件 | 注入数据 | 职责 |
|-------|------------|---------|------|
| analyze-timing | `prompts/analyze-timing.md` | `{profile_summary}` | 时机阶段 + 切入策略 + 紧急程度 + 窗口分析 |
| build-battle-cards | `prompts/build-battle-cards.md` | `{profile_summary}`、`{knowledge_base_index}`（如有） | 竞对识别 + 作战卡片 + 防御策略 |
| map-decision-chain | `prompts/map-decision-chain.md` | `{profile_summary}`、`{requirements_summary}`（如有） | 决策链 + 接触策略 + 审批路径 |

`{requirements_summary}` 从 requirements.json 提取精简摘要：

```json
{
  "status": "",
  "currentVersion": "",
  "topNeeds": [],
  "constraints": { "budget": "", "timeline": "" },
  "pendingQuestions": []
}
```

`{knowledge_base_index}` 读取 `docs/knowledge-base/index.json`（如存在）。

**步骤 3：综合策略（Phase 2）**

读取 `prompts/synthesize-strategy.md`，启动 1 个 Agent：
- 注入 Phase 1 全部输出
- 注入 `{profile_summary}`
- 注入 `{requirements_summary}`（如有）
- 注入用户素材（如有）
- 输出：完整的 SalesGuide JSON

**步骤 4：渲染输出**

读取 `output-template.md`，基于 synthesize 返回的结论：
1. 按模板生成 Markdown 展示给用户
2. 将结构化数据写入 `docs/customer/sales-guide.json`（遵循 `src/types/customer.ts` 中的 `SalesGuide` 接口）
3. 对所有文本输出应用 humanizer-zh 规则

---

#### 模式 B：迭代更新

**步骤 1：收集输入**

1. 读取现有 `docs/customer/sales-guide.json`
2. 读取 `docs/customer/profile.json`
3. 读取 `docs/customer/requirements.json`（如存在）
4. 接收用户提供的新素材

**步骤 2：并行分析（Phase 1）**

同模式 A 发起 3 个 Agent，但每个 Agent 额外注入：
- `{existing_sales_guide}`：现有 sales-guide.json 中对应部分
- `{user_input}`：用户提供的新情报

每个 Agent 职责：
1. 识别自己负责领域中什么发生了变化
2. 仅对有变化的部分产出更新结果
3. 无变化的部分标注 "no_change"

**步骤 3：综合策略（Phase 2）**

synthesize Agent 额外接收 `{existing_sales_guide}`，职责：
- Phase 1 标注 "no_change" 的部分：保留现有指南原文
- Phase 1 有更新的部分：重新生成该部分
- 跨节一致性检查：时机变了，破冰素材是否还成立？决策链变了，访谈问题要不要更新？
- 生成 `changeSummary[]` 记录变更

**步骤 4：渲染输出**

1. 先输出变更摘要（本次更新了什么、为什么）
2. 输出完整作战指南
3. 更新 `docs/customer/sales-guide.json`

---

### 降级策略

| 场景 | 处理 |
|------|------|
| profile.json 缺失 | **停止执行**，提示用户先运行 `/profile` |
| requirements.json 缺失 | 继续执行，竞对分析和价值主张走行业推演，标注信息来源有限 |
| knowledge-base 为空 | 跳过知识库匹配，纯行业推演 |
| 某个 Agent 超时/报错 | 用已有结果继续，标注受影响部分"[信息不足，建议补充]" |
| profile 置信度低（confidence < 50） | 全部输出标注"[基于有限信息推演]"，增加保守度 |

不因信息不足停止执行。有限信息的作战指南远好于没有指南。

---

### 版本管理

| 场景 | 版本变化 |
|------|----------|
| 首次生成 | "1.0" |
| 小幅迭代（新增情报、微调策略） | +0.1（1.0→1.1） |
| 重大变化（新竞对入场/决策链重组/时机阶段转变） | +1.0（1.1→2.0） |

销售作战指南是活文档，没有"定版"概念。`generatedAt` 时间戳标识新鲜度。

---

### 与其他 Skill 的关系

| Skill | 关系 |
|-------|------|
| `/profile` | 上游输入：企业画像、时机、组织架构、技术栈、痛点、联系人 |
| `/requirements` | 可选输入：已确认需求、预算约束、待确认问题 |
| `/knowledge-base` | 可选输入：行业知识库，匹配竞对和典型需求模式 |
| `/plan-writer` | 下游消费：方案团队参考作战指南中的价值主张和竞对分析 |
