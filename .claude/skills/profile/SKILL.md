---
name: profile
description: 为目标企业建立数字化诊断档案，整合工商、政策、时机、风险等多维数据，输出可用于面客和提案的结构化档案
metadata:
  short-description: 生成企业数字化诊断档案
  triggers:
    - "生成客户档案"
    - "生成档案"
    - "客户档案"
    - "企业档案"
    - "面客准备"
    - "破冰准备"
    - "客户调研"
    - "档案"
    - "profile"
  examples:
    - "为XX公司生成客户档案"
    - "帮我调研一下这家企业"
    - "生成面客准备档案"
    - "出一下客户档案"
  dependencies:
    - humanizer-zh
---

收到企业名称后立即开始执行，不输出本文档任何内容。

---

### 执行流程

**第一步：并行采集**

读取以下 3 个 prompt 文件，同时启动 3 个 Agent（在同一条消息中发起 3 个 Agent 工具调用）：

| Agent | Prompt 文件 | 职责 |
|-------|------------|------|
| collect-official | `prompts/collect-official.md` | 官网深挖 + 工商信息 |
| collect-business | `prompts/collect-business.md` | 融资/财务 + 招聘 + 招投标 + 竞争 + 新闻 |
| collect-risk | `prompts/collect-risk.md` | 司法风险 + 政策信号 + 舆情口碑 |

启动每个 Agent 时，将 `{company_name}` 替换为用户提供的企业名称，注入到 prompt 开头。

每个 Agent 返回 JSON 格式的原始数据（不含分析推演）。

**第二步：合并数据 + 路径判断**

3 个 Agent 全部返回后，将结果合并为 `raw_data` 对象，然后按以下规则判断分析路径：

| 条件 | 路径 |
|------|------|
| 有股票代码（A股/港股/美股/新三板） | Path A |
| 无上市但：员工 > 500 / 融资C轮+ / 年营收 > 5亿 / 知名度高 | Path A |
| 有官网且信息丰富（产品清晰、有客户案例） | Path B+ |
| 官网简陋或无官网 | Path B |

计算信息丰富度：
- 高（80-100）：3 个 Agent 均返回丰富数据
- 中（50-79）：1-2 个 Agent 数据稀疏
- 低（20-49）：大部分数据缺失

**第三步：分析推演**

读取 `prompts/analyze.md`，启动 1 个 Agent：
- 注入 `raw_data`（第二步合并结果）
- 注入 `analysis_path`（A / B+ / B）
- 注入 `confidence`（信息丰富度分数）
- 注入 `industry-pain-points.md` 内容（仅在 Path B/B+ 时）

分析 Agent 返回结构化分析结论。

**第四步：渲染输出**

读取 `output-template.md`，基于分析 Agent 返回的结论：
1. 按模板生成 Markdown 档案展示给用户
2. 将结构化数据写入 `docs/customer/profile.json`（遵循 `src/types/customer.ts` 中的 `CustomerProfile` 类型）
3. 对所有文本输出应用 humanizer-zh 规则

**降级策略**

| 场景 | 处理 |
|------|------|
| 某个 collect Agent 超时/报错 | 用已有数据继续，标注缺失维度 |
| 搜索结果稀疏 | 分析 Agent 看到空字段自动走推断路径 |
| 全部搜索失败 | 直接走 Path B 纯行业推断 |

不因信息不足停止执行。有限信息的推演档案远好于没有档案。

**版本管理**

| 场景 | 版本变化 |
|------|----------|
| 首次生成 | "1.0" |
| 用户要求"更新档案" | minor +0.1（如 1.0→1.1），重新执行采集，仅更新有变化的字段 |
| 核心字段变化（上市/融资/并购/行业变更） | major +1.0（如 1.1→2.0） |
