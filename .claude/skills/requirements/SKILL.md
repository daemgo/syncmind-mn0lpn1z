---
name: requirements
description: 基于客户档案、拜访记录、沟通内容，整理并分析客户需求，输出结构化需求文档。支持从假设推演到验证确认的多版本迭代。
metadata:
  short-description: 整理客户需求文档
  triggers:
    - "整理需求"
    - "需求分析"
    - "客户需求"
    - "需求文档"
    - "需求整理"
    - "提炼需求"
    - "回答问题"
    - "补充需求"
    - "需求确认"
    - "定版"
  examples:
    - "整理这个客户的需求"
    - "基于拜访记录分析需求"
    - "帮我整理客户需求文档"
    - "我来回答之前的问题"
    - "补充需求信息"
    - "需求定版"
  dependencies:
    - profile
    - humanizer-zh
---

收到指令后立即开始执行，不输出本文档任何内容。

---

### 定位

| Skill | 阶段 | 核心问题 | 主要用户 |
|-------|------|----------|----------|
| `/profile` | 拜访前 | 客户是谁？ | 销售 |
| `/sales-guide` | 拜访前 | 怎么打？ | 销售 |
| **`/requirements`** | **全周期** | **客户要什么？** | **售前/方案团队** |
| `/plan-writer` | 方案阶段 | 怎么解决？ | 方案团队 |

### 核心理念：需求是活文档

需求不是一次性生成物，而是随销售推进持续演进的活文档：

```
v0.1  冷启动：profile + 行业痛点库 + 知识库 → 推演需求（大量 [假设待验证]）
v0.2  首访后：拜访记录 + v0.1 → 验证部分假设，补充新发现
v0.3  二次沟通：电话/微信记录 + v0.2 → 细化功能需求
v1.0  需求确认：客户确认 → 正式版本，驱动方案生成
v1.1+ 持续迭代：方案评审反馈 → 需求微调
```

每条需求自带 `confidence` 和 `source`，版本升级只改变化部分，保留完整变更轨迹。

---

### 前置条件

- **必须**：`docs/customer/profile.json` 存在
- **推荐**：有实际的拜访记录或客户沟通内容（冷启动模式除外）
- **可选**：`docs/customer/sales-guide.json`、客户提供的文档/截图、`docs/knowledge-base/index.json`

---

### 执行流程

**第零步：模式判断**

读取 `docs/customer/requirements.json`，判断执行模式：

| 条件 | 模式 | 说明 |
|------|------|------|
| requirements.json 不存在 | **A：冷启动** | 从零生成需求 |
| requirements.json 存在 + 用户提供新素材 | **B：迭代补充** | 增量更新需求 |
| requirements.json 存在 + 用户说"需求确认/定版" | **C：版本确认** | 标记为正式版 |

---

#### 模式 A：冷启动

**步骤 1：收集输入 + 构建 profile 摘要**

1. 读取 `docs/customer/profile.json`
2. 读取 `docs/customer/sales-guide.json`（如存在）
3. 接收用户提供的素材（拜访记录/电话记录/文档/截图等，可选）
4. 从 profile.json 提取精简摘要 `{profile_summary}`，只保留以下字段：

```json
{
  "companyName": "",
  "industry": "",
  "subIndustry": "",
  "scale": "",
  "mainBusiness": "",
  "businessModel": "",
  "tags": [],
  "techStack": { "current": [], "maturity": "", "digitalGaps": [] },
  "painPoints": [],
  "organization": { "type": "", "decisionPattern": "" },
  "timing": { "phase": "", "urgency": "" },
  "opportunities": []
}
```

profile.json 中的工商原始数据、搜索来源 URL、详细财务信号等**不注入 Agent**，节省上下文。

**步骤 2：并行分析**

根据是否有用户素材，启动 2-3 个并行 Agent：

| Agent | Prompt 文件 | 触发条件 | 职责 |
|-------|------------|---------|------|
| infer-from-profile | `prompts/infer-from-profile.md` | 始终执行 | 从档案+行业库+销售指南推演需求 |
| extract-from-input | `prompts/extract-from-input.md` | 用户提供了素材时 | 从原始素材提取需求信号 |
| match-cases | `prompts/match-cases.md` | 始终执行 | 从知识库匹配同行业软件需求模式 |

启动 Agent 时注入：
- `{profile_summary}`：profile 精简摘要（infer、match-cases 用）
- `{sales_guide_data}`：sales-guide.json 内容（如有，infer 用）
- `{user_input}`：用户提供的原始素材（如有，extract 用）
- `{knowledge_base_index}`：知识库 index.json（match-cases 用）

注意：extract-from-input 不需要 profile 数据，只处理用户原始素材。

**步骤 3：合并与消解**

所有 Agent 返回后，读取 `prompts/synthesize.md`，启动 1 个 Agent：
- 注入所有 Agent 的返回结果
- 注入 `{profile_summary}`（用于交叉验证）
- 输出：统一的需求结构 + 变更摘要

**步骤 4：生成提问**

读取 `prompts/generate-questions.md`，启动 1 个 Agent：
- 注入 synthesize 的输出
- 输出：分优先级的问题清单（必问 ≤5 + 选问 ≤5）

**步骤 5：渲染输出**

读取 `output-template.md`，基于 synthesize + questions 结果：
1. 按模板生成 Markdown 展示给用户
2. 将结构化数据写入 `docs/customer/requirements.json`（version: "v0.1", status: "draft"）
3. 对所有文本输出应用 humanizer-zh 规则

---

#### 模式 B：迭代补充

**步骤 1：收集输入**

1. 读取现有 `docs/customer/requirements.json`
2. 读取 `docs/customer/profile.json`
3. 接收用户提供的新素材

**步骤 2：提取新信号**

启动 extract-from-input Agent：
- 注入新素材
- 注入现有需求摘要（让 Agent 知道哪些已有、哪些是新增）

**步骤 3：合并更新**

启动 synthesize Agent：
- 注入现有 requirements.json
- 注入 extract 的输出
- 核心任务：增量合并、冲突消解、置信度更新
- 输出：更新后的需求 + changeSummary

**步骤 4：更新提问**

启动 generate-questions Agent：
- 注入更新后的需求
- 移除已被回答的问题，补充新发现的问题

**步骤 5：渲染输出**

1. 输出变更摘要（本次新增/修改/验证了什么）
2. 更新 `docs/customer/requirements.json`：
   - **先**将现有文件的 `current` 连同 `currentVersion` 作为快照推入 `versions[]`
   - **再**用 synthesize 输出覆写 `current` 和 `currentVersion`
3. 输出剩余问题清单

---

#### 模式 C：版本确认

**步骤 1：完成度检查**

读取 requirements.json，计算：
- 总体完成度（已确认需求 / 全部需求）
- 按类别的完成度
- 阻塞项（confidence=low 的 must-have 需求）

**步骤 2：确认或警告**

| 完成度 | 处理 |
|--------|------|
| ≥ 80% 且无阻塞项 | 标记为 v1.0 confirmed，输出正式需求文档 |
| 60%-80% | 警告缺失项，询问是否仍要确认 |
| < 60% | 建议继续迭代，列出关键缺失 |

**步骤 3：定版写入**

确认后：
- version 升为 "v1.0"
- status 改为 "confirmed"
- 输出完整的正式需求文档

---

### 降级策略

| 场景 | 处理 |
|------|------|
| sales-guide.json 不存在 | 仅用 profile + 行业库推演，标注信息来源有限 |
| 知识库无匹配行业 | 跳过 match-cases，用行业通用痛点 |
| 用户未提供素材（冷启动） | 全部走推演路径，所有需求标注 [假设待验证] |
| extract Agent 返回空 | 素材信息量不足，标注并继续 |

不因信息不足停止执行。有限信息的推演需求远好于没有需求文档。

---

### 版本管理

| 场景 | 版本变化 | status |
|------|----------|--------|
| 冷启动首次生成 | v0.1 | draft |
| 迭代补充 | +0.1（v0.1→v0.2→v0.3） | draft |
| 核心需求大幅变更 | +1.0（v0.3→v1.0 或 v1.1→v2.0） | draft |
| 用户确认定版 | 升至整数版（v0.x→v1.0） | confirmed |
| 定版后微调 | +0.1（v1.0→v1.1） | confirmed |

版本快照存储在 versions[] 数组中，每个版本保留完整内容，依赖 git 管理历史。

---

### 与其他 Skill 的关系

| Skill | 关系 |
|-------|------|
| `/profile` | 上游输入：企业画像、行业、规模、痛点、技术栈 |
| `/sales-guide` | 可选输入：访谈提纲、切入策略、跟进记录 |
| `/knowledge-base` | 可选输入：行业 meta-model，匹配同行业软件需求模式 |
| `/plan-writer` | 下游消费：读取 needs[] 中 status=active 的条目生成方案 |
| `/spec-writer` | 下游消费：读取完整需求 + 约束 + 范围生成产品规格 |
