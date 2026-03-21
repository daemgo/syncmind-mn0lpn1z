# 输出规则

## Markdown 展示模板

### 模式 B（迭代更新）前缀

仅在迭代模式下，在正文前插入变更摘要：

```markdown
---
### 本次更新摘要

**版本**：{oldVersion} → {newVersion}
**输入**：{新素材简述}

**变更**：
- {changeSummary 各条}
---
```

### 正文模板

```markdown
# 【销售作战指南】：{customerName}

> v{version} · {urgency}紧急度 · {timingStage} · {generatedAt}

## 1. 时机判断

**阶段**：{timingStage}
**紧急度**：{urgency}

**判断依据**：
- {analysisBasis 各条}

**切入策略**：{entryStrategy}

**时机窗口**：
- 利好因素：{openFactors}
- 风险因素：{closeFactors}
- 最佳窗口：{optimalWindow}

## 2. 切入素材与禁区

### 可用素材

[按类型分组展示 primaryHooks，每条含素材内容 + 可用场景]

| 类型 | 素材 | 可用场景 |
|------|------|---------|
| {type} | {content} | {rationale} |

### 各阶段关注点

**首次接触**：{firstContact hooks}
**跟进沟通**：{followUp hooks}
**产品演示**：{demo hooks}
**商务谈判**：{negotiation hooks}
**签约收尾**：{closing hooks}

### ⚠️ 禁区话题

- {avoidTopics 各条，标注原因}

## 3. 价值主张

> {coreValue — 一句话核心价值}

**利益点**：
1. {benefitPoint — 绑定具体痛点}
2. ...

**证明点**：
- {proofPoints — 案例/数据/资质}

**风险缓解**：
- {riskMitigation 各条}

**ROI**：{roi 说明}

## 4. 访谈提纲

### 首次筛选（快速判断客户质量）

| # | 问题 | 目的 | 追问 |
|---|------|------|------|
| {id} | {question} | {purpose} | {followUp} |

### 深度访谈（挖掘真实需求）

[同上格式，展示 deepDive 问题]

### 收尾问题（推动下一步）

[同上格式，展示 closingQuestions]

### 定制问题（基于客户特征）

[同上格式，展示 customQuestions]

## 5. 竞对作战卡

[对每个竞对，输出一个卡片]

### {competitor}

**定位**：{position}

| 维度 | 内容 |
|------|------|
| 竞对优势 | {strengths} |
| 竞对弱点 | {weaknesses} |
| 我方优势 | {ourAdvantage} |
| 差异化策略 | {differentiation} |

**防御策略**：{defenseStrategy}

---

[重复每个竞对]

## 6. 异议应对

### 常见异议预判

| 异议 | 客户真正担心的 | 应对策略 | 转向 |
|------|---------------|---------|------|
| {objection} | {underlyingConcern} | {response} | {pivot} |

### 预防策略

- {objectionPrevention 各条}

### 处理框架

{handlingFramework}

## 7. 决策链条

### 关键角色

**决策者**：
[对每个 decisionMaker 展示：角色、姓名（如有）、关注点、接触策略]

**影响者**：
[同上]

**潜在阻碍者**：
[同上，标注风险和化解策略]

### 决策流程

{decisionProcess}

**审批路径**：{approvalPath，用 → 连接}

**预估周期**：{timeline}

### 推荐接触序列

[基于行动计划中的接触类行动项，按时间顺序提取展示]

1. {target} — {purpose}（{timing}）
2. ...

## 8. 行动计划

| 优先级 | 行动 | 截止时间 | 执行人 | 状态 |
|--------|------|---------|--------|------|
| {priority} | {action} | {deadline} | {owner} | {status} |
```

> 作战指南生成完毕。运行 `/requirements` 整理客户需求。

---

## JSON 写入规则

写入路径：`docs/customer/sales-guide.json`

类型定义：`src/types/customer.ts` 中的 `SalesGuide` 接口

### 字段映射

synthesize-strategy Agent 的输出直接对应 SalesGuide 接口的字段，无需额外映射。

关键注意事项：

| 字段 | 规则 |
|------|------|
| customerId | 从 profile.json 的 id 字段获取 |
| customerName | 从 profile.json 的 companyName 字段获取 |
| generatedAt | 写入当前时间（ISO 8601） |
| version | 见 SKILL.md 版本管理规则 |
| elevatorPitch | 填空字符串 "" |

### 不写入的字段（其他 skill/系统负责）

SalesGuide 写入的是独立的 `docs/customer/sales-guide.json` 文件。

profile.json 中的 `salesGuide` 字段由平台系统同步，本 skill 不直接修改 profile.json。

### 写入策略

- **首次写入**：创建完整的 SalesGuide JSON
- **迭代写入**：读取现有文件，只覆盖有变化的字段，保留无变化的部分
- **始终更新**：generatedAt、version 字段
- **UTF-8 编码**，JSON 缩进 2 个空格

### humanizer-zh 处理清单

以下字段在写入前必须经过 humanizer-zh 规则处理：

| 字段路径 | 处理重点 |
|----------|---------|
| timing.entryStrategy | 不要泛泛，要具体 |
| iceBreaker.primaryHooks[].content | 事实陈述，不要 AI 腔 |
| iceBreaker.avoidTopics[] | 直接说明，不要绕 |
| interviewGuide.*[].question | 口语化，像同事在聊天 |
| interviewGuide.*[].followUp | 自然追问，不要审问感 |
| competitorAnalysis[].defenseStrategy | 像有经验的销售在教新人 |
| objectionHandling.commonObjections[].response | 自然回应，不要辩论稿 |
| valueProposition.coreValue | 一句话，不要缩写版宣传册 |
| valueProposition.roi | 讲逻辑，不要编数字 |
| nextActions[].action | 具体到可执行，不要"加强沟通" |
