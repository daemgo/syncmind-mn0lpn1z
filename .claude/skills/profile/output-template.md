# 输出规则

## Markdown 展示模板

```markdown
# 【数字化诊断档案】：{companyName}

> {summary}

## 0. 企业画像
[profile 模块数据：行业、规模、主营业务、产品、商业模式、标签]

## 1. 工商信息解读
[registration 模块：不是罗列字段，而是解读文本 + 关键发现]

## 2. 组织架构与决策链
[organization 模块：组织类型、决策模式、关键角色]

## 3. 政策敏感度
[policyAnalysis 模块：命中的政策信号 + 每条的销售含义]

## 4. 时机判断
[timing 模块：所处阶段、判断依据、建议切入策略]

## 5. 信用与风险
[creditRisk 模块：风险等级、具体风险项 + 销售含义]

## 6. 数字化成熟度
[techStack 模块：当前系统、技术方向、数字化缺口]

## 7. 痛点与机会
[painPoints + opportunities 模块，标注哪些是假设]

## 8. 关键数据速查
[keyMetrics 模块：表格形式]

## 9. 待验证信息
[informationGaps 模块：明确列出信息缺口和建议验证方式]

## 10. 建议访谈提纲
[interviewOutline 模块：仅 Path B/B+ 时输出]

## 11. 信息来源
[metadata.sources：搜索中引用的所有 URL]
```

> 档案生成完毕。运行 `/sales-guide` 获取销售进攻指南。

---

## JSON 写入规则

写入路径：`docs/customer/profile.json`

类型定义：`src/types/customer.ts` 中的 `CustomerProfile` 接口

### 字段映射

| 分析模块 | → JSON 字段 | 说明 |
|----------|------------|------|
| profile.companyName | companyName | |
| profile.shortName | shortName | |
| profile.industry | industry | |
| profile.subIndustry | subIndustry | |
| profile.scale | scale | |
| profile.mainBusiness | mainBusiness | |
| profile.products | products[] | 映射为 Product 接口 |
| profile.targetCustomers | targetCustomers[] | |
| profile.businessModel | businessModel | |
| profile.rating | rating | 根据综合评估给出 A/B/C/D |
| profile.tags | tags[] | |
| profile.summary | metadata.notes | CustomerProfile 无顶层 summary 字段 |
| organization.type | type | 企业类型 |
| organization.decisionPattern | decision.criteria[] | 转为决策标准列表 |
| organization.estimatedDecisionCycle | decision.timeline | |
| organization.keyRoles | decision.makers[] + decision.influencers[] | |
| policyAnalysis.signals（hit=true） | tags[]（追加）+ opportunities[] | 命中的政策写入标签和机会 |
| timing.phase | strategy.phase | |
| timing.* | strategy.growthStage | 综合时机判断 |
| creditRisk.riskSignals | strategy.riskSignals[] | |
| techStack.* | techStack.* | 直接映射 |
| strategy.* | strategy.* | 直接映射 |
| painPoints[] | painPoints[] | 映射为 PainPoint 接口 |
| opportunities[] | opportunities[] | 映射为 Opportunity 接口 |

### 不写入的字段（其他 skill/系统负责）

| 字段 | 负责方 |
|------|--------|
| salesGuide | `/sales-guide` skill |
| requirements | `/requirements` skill |
| contacts | 平台 CRM / 用户手动录入 |
| tracking | 平台系统自动维护 |
| budget | 面访后由销售补充 |

### metadata 字段

| 字段 | 规则 |
|------|------|
| createdAt | 首次生成时写入当前时间（ISO 8601） |
| updatedAt | 每次更新时写入当前时间 |
| createdBy | 固定 `"AI"` |
| version | 见 SKILL.md 版本管理规则 |
| confidence | 信息丰富度分数（20-100） |
| sources | 所有搜索中引用的有效 URL 列表 |

### 写入策略

- **merge 写入**：保留 JSON 中已有的其他字段值，只更新 profile 负责的字段
- **首次写入**：如果文件不存在，创建完整的 CustomerProfile 结构，未填充字段使用类型默认值（空字符串/空数组/0）
- **更新写入**：读取现有文件，仅覆盖 profile 负责的字段，保留其他 skill 已写入的数据
