# Agent Canvas

> 让 Agent 用语义描述而非代码来生成专业级可视化页面。

## 问题：Agent 可视化的根本困境

当 Agent 需要生成可视化时，现有方案都存在根本性限制：

| 方案 | 可靠性 | 审美 | 灵活性 | 核心问题 |
|------|--------|------|--------|---------|
| Markdown 渲染 | 极高 | 低 | 极低 | 表达力不足，只能简单排版 |
| HTML/CSS/JS 生成（Claude Artifacts） | 中 | 不稳定 | 极高 | 代码可能出错，审美取决于代码质量 |
| Python 绘图（matplotlib） | 中 | 低-中 | 中 | 需要运行时，审美不可控 |
| DSL 配置（ECharts/Vega-Lite） | 高 | 中 | 低 | 受限于特定图表类型 |

根本原因不是技术不够，而是**模态错配**：LLM 在文本序列上运作，而可视化是空间性的。具体表现为四个层面的问题——空间推理能力缺失（[LaySPA](https://arxiv.org/abs/2509.16891)）、缺乏视觉反馈闭环（[MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/) 证明反馈可提升 12-13 分）、概率生成与视觉精确性的冲突、布局约束求解能力不足（[VisEval](https://arxiv.org/abs/2407.00981) 数据：复杂场景成功率仅 20-40%）。

## 解决方案：四层处理管线 × 三层概念抽象

Agent Canvas 的核心思路是**分离关注点**——把"理解要展示什么"和"怎么渲染"拆开，让 Agent 专注于语义决策，让确定性引擎负责视觉执行。

架构包含两个视角：**处理管线**（四层流程，描述 Agent 的执行步骤）和**概念抽象**（三层模型，描述从需求到视觉的转换本质）。

```
用户需求（自然语言）
    ↓
┌──────────────────────────────────────────────────────────┐
│  第一层：意图解析                                          │
│  从自然语言中抽离六维意图：信息目标/受众/场景/目的/数据形态/约束  │
└────────────────────────┬─────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│  第二层：语义中间表示（SIR）                                │
│  元素拆解 → 十种信息元素                                    │
│  结构组织 → 六条组织规则 → 元素树 + 主辅层级                 │
│  组合策略 → 组合决策树 → 布局模式 + 空间分配                 │
└────────────────────────┬─────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│  第三层：视觉映射                                          │
│  SIR → Canvas Schema（JSON）                              │
│  信息元素 → 组件映射 / 组织结构 → 布局映射 / 主辅 → 视觉权重  │
│  同一 SIR 可映射多场景 Schema（报告版/大屏版/移动版）         │
└────────────────────────┬─────────────────────────────────┘
                         ↓
┌──────────────────────────────────────────────────────────┐
│  第四层：确定性渲染 + 视觉反馈闭环                          │
│  Canvas Schema → React 渲染引擎 → 视觉输出                 │
│  截图 → 多模态 LLM 审查 → 修正建议 → 迭代                   │
│  两层反馈：SIR 级（理解对不对）+ Schema 级（渲染对不对）      │
└──────────────────────────────────────────────────────────┘
```

### 概念抽象：三层架构

在四层处理管线的背后，是一个三层概念抽象——这是从 301 个 case 和学术模型对比中提炼出的核心设计：

```
用户需求（自然语言）
    ↓
┌─────────────────────────────────────────────┐
│  Semantic Layer（语义层）                     │
│  8 种语义元素：entity / attribute / measure  │
│  / set / event / process / narrative / constraint │
│  5 种角色 + 8 种关系                          │
│  → 回答"要表达什么含义"                       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  Data Layer（数据层）                         │
│  DataItem 三元组：entity × attribute → value │
│  语义元素实例化为具体数据项                     │
│  → 回答"具体数据是什么"                       │
└──────────────────┬──────────────────────────┘
                   ↓
┌─────────────────────────────────────────────┐
│  Visual Grammar Layer（视觉语法层）             │
│  4 种视觉原语：Mark / Relation / Boundary /   │
│  ChannelMapping                               │
│  → 回答"用什么视觉形式表达"                    │
└─────────────────────────────────────────────┘
```

三层架构的学术基础：与 [Ed Chi Data State Model (2000)](https://ics.uci.edu/~kobsa/courses/ICS280/InfoViz2000/ed-chi.pdf) 的 Analytical Abstraction → Value → Visualization Abstraction 对应，但将学术界隐含的语义层显式化——这对人类不是必须的，但对 Agent 是必要的。详见 [THREE-LAYERS.md](./THREE-LAYERS.md)。

**视觉语法层的 4 种原语**取代了传统"图表类型"的概念——任何图表都是 Mark + Relation + Boundary + ChannelMapping 的组合。覆盖 301 个 case 中 99% 的场景。详见 [GRAMMAR.md](./GRAMMAR.md)。

### 第一层：意图解析（处理管线）

从用户的自然语言请求中抽离六个维度的结构化意图，把混沌的需求变成可推理的结构：

- **信息目标**（What）— 展示指标、趋势、对比、构成、关系、流程、叙事、决策...
- **受众**（Who）— 高管要极简高密、工程师要完整可操作、客户要看结果
- **场景**（Where）— 仪表盘、报告、大屏、移动端、内嵌
- **目的**（Why）— 告知、说服、探索、决策、监控、解释
- **数据形态**（DataShape）— 标量、时间序列、分类、关系、层级、空间、文本
- **约束**（Constraints）— 篇幅、强调重点、品牌要求、数据敏感度

### 第二层：语义中间表示（SIR）

这是连接"需求"和"渲染"的桥梁，对应三层概念抽象中的 Semantic Layer 和 Data Layer。分为三步：

**元素拆解** — 从需求中提取最小语义单元，归纳为十种信息元素：

> **演进说明**：这十种信息元素是早期版本的设计。在深入分析中发现它们偏向前端组件分类，不够本质。最终提炼为 8 种纯语义元素（entity / attribute / measure / set / event / process / narrative / constraint）+ 5 种语义关系，详见 [THREE-LAYERS.md](./THREE-LAYERS.md)。视觉表达层面则进一步抽象为 4 种视觉语法原语（Mark / Relation / Boundary / ChannelMapping），详见 [GRAMMAR.md](./GRAMMAR.md)。

| 元素 | 含义 | 典型来源 |
|------|------|---------|
| DataPoint | 带标签和值的单一数据项 | KPI 卡片、统计数字 |
| DataSeries | 按维度排列的一组数据点 | 趋势图、时间线 |
| Comparison | 多对象在多维度上的差异 | 竞品对比、方案选择 |
| Composition | 部分与整体的构成关系 | 预算分解、营收构成 |
| Relation | 实体之间的关联关系 | 服务依赖、知识图谱 |
| Process | 有顺序的步骤或阶段 | 部署流水线、审批流程 |
| Hierarchy | 树形嵌套的从属关系 | 组织架构、技术栈 |
| Distribution | 数据在维度上的散布 | 用户分布、异常检测 |
| Narrative | 结构化的文字论述 | 报告、复盘、分析 |
| Decision | 多选项的评估和选择 | 技术选型、风险评估 |

**结构组织** — 六条规则把散落的元素搭成结构：

语义亲和（相关元素靠近）→ 信息层级（L1 焦点 / L2 支撑 / L3 补充）→ 阅读流（总→分、因→果、并列、时间、下钻五种模式）→ 分组容器 → 空间平衡 → 约束适配。

**组合策略** — 当多种信息意图需要共存于一个页面时，通过组合决策树选择策略：

- **单图内多编码组合** — 双轴叠加（趋势+对比）、大小+颜色（分布+构成）、标注+趋势（追踪+叙事），最多 4 种意图
- **多图布局组合** — 六种模式：并列、堆叠、主辅环绕、网格平铺、左右分栏、小多图
- **多区域语义组合** — 指标区→趋势区→构成区→对比区→明细区→叙事区→导航区→决策区的编排规则

主辅关系是组合的核心：L1 占 50-60% 空间获得最大视觉权重，L2 占 30-40%，L3 占 5-10% 或折叠。通过 span 跨列、字体大小、色彩饱和度、卡片阴影、图表尺寸五个维度差异化。

### 第三层：视觉映射

SIR 到 Canvas Schema 的机械映射，Agent 在此步骤的认知负荷很低：

**元素 → 组件映射**：

| 信息元素 | 首选组件 | 备选组件 |
|---------|---------|---------|
| DataPoint | StatCard | Gauge / Badge / KPIBlock |
| DataSeries | Chart(line) | Chart(bar/area) |
| Comparison | ComparisonTable | RadarChart / GroupedBarChart |
| Composition | Chart(donut) | TreeMap / StackedBar |
| Relation | NetworkGraph | RelationGraph / ERDiagram |
| Process | FlowChart | SequenceDiagram / Sankey / Funnel |
| Hierarchy | TreeView | OrgChart / MindMap / TreeMap |
| Distribution | Chart(scatter) | Heatmap / Histogram |
| Narrative | Report | Card / Accordion |
| Decision | DecisionMatrix | DecisionTree / ProConTable |

**组织 → 布局映射**：dashboard→Grid、report→Stack、comparison→Grid(N列)、timeline→Stack...

**同一 SIR 可映射多种场景**：例如同一个"故障复盘"SIR，报告版用 Stack+Timeline+Chart，大屏版用 Grid+简化Timeline+Callout，移动版用单列+Accordion 折叠。

### 第四层：确定性渲染 + 视觉反馈闭环

**渲染引擎**基于 React + Tailwind CSS，将 Canvas Schema 转化为实际视觉输出。关键设计：

- 使用 CSS Grid/Flexbox 而非绝对定位，减少空间推理需求
- 组件内部 CSS-in-JS 确保样式隔离和主题感知
- 布局引擎支持自动换行、自动间距、自动对齐
- 语义化属性（如 `emphasis: "high"`）而非样式化属性（如 `color: "red"`）

**视觉反馈闭环**是区别于 Claude Artifacts、Vercel v0 等方案的关键能力：

```
Schema → 渲染 → 截图(Playwright) → 多模态 LLM 审查 → 修正建议 → 迭代
```

两层反馈分工：SIR 级审查捕获"理解错误"（是否遗漏关键信息、层级是否合理），Schema 级审查捕获"渲染错误"（是否重叠溢出、视觉层级是否清晰）。

## 认知框架：11 种可视化原子模式

从 301 个 case 中归纳出所有可视化都是以下模式的组合：

**描述性模式**（描述事物状态）：

| 模式 | 核心问题 | 频次 |
|------|---------|------|
| Compare（比较） | A 和 B 有什么不同？ | 55% |
| Track（追踪） | 随时间怎么变化？ | 51% |
| Narrate（叙事） | 故事脉络是什么？ | 41% |
| Arrange（组织） | 东西怎么组织的？ | 33% |
| Distribute（分布） | 数据怎么分布的？ | 32% |
| Compose（构成） | 整体由哪些部分构成？ | 28% |
| Flow（流转） | 东西从哪来到哪去？ | 20% |
| Connect（关联） | 谁和谁有关系？ | 14% |

**操作性模式**（支持人类行动）：

| 模式 | 核心问题 | 频次 |
|------|---------|------|
| Decide（决策） | 应该选哪个？ | 7% |
| Navigate（导航） | 怎么从这里到那里？ | 7% |
| Simulate（模拟） | 如果...会怎样？ | 4% |

## 与现有方案的对比

| 维度 | Claude Artifacts | Vercel v0 | A2UI | Agent Canvas |
|------|-----------------|-----------|------|-------------|
| Agent 产物 | HTML/CSS/JS 代码 | React 组件代码 | 声明式 JSON | Canvas Schema (JSON) |
| 生成复杂度 | 高（完整前端代码） | 高（React+TSX） | 低 | 低（描述结构和数据） |
| 可靠性 | 中（代码可能出错） | 中（需 AutoFix） | 高 | 高（Schema 验证+确定性渲染） |
| 审美质量 | 不稳定 | 较好（shadcn/ui） | 中 | 稳定（专业组件预设） |
| 视觉反馈 | 无 | AutoFix（代码级） | 无 | 有（截图+多模态审查） |
| 中间表示 | 无 | 无 | 无 | 有（SIR 语义中间表示） |
| 多场景适配 | 需重新生成 | 需重新生成 | 需重新生成 | 同一 SIR 映射多 Schema |
| 安全性 | 需沙盒 | 需沙盒 | 天然安全 | 天然安全（JSON 不可执行） |

核心差异：Agent Canvas 有中间层（SIR），把"理解需求"和"生成代码"解耦；有视觉反馈闭环，能自我纠错；同一 SIR 可适配多场景，不需重新推理。

## Case 验证集

累计 **301 个 case**，作为组件库解空间映射和效果验证基准：

- **19/20** 个国民经济行业门类（GB/T 4754-2017）
- **12** 个通用企业部门 + **15** 个互联网部门 + **20** 个行业特有部门
- **15+** 类岗位（从 CEO 到算法工程师、SRE、增长黑客）
- **30+** 种可视化形态（从仪表盘/报告到混淆矩阵/K线图/技术雷达）
- **10** 种企业类型（大企业/创业公司/公共机构/非营利组织/医院/学校等）
- 约 **60%** 的 case 涉及多意图组合，验证了组合规则体系的必要性

## 技术栈

| 层 | 技术 | 选型理由 |
|----|------|---------|
| 渲染引擎 | React 18+ / TypeScript | 组件化模型天然适配 Schema→组件树映射 |
| 组件样式 | Tailwind CSS / CSS Variables | 原子化样式 + 主题系统 |
| 图表渲染 | Recharts / ECharts | 封装为 Canvas 组件 |
| Schema 验证 | Zod | 运行时类型安全 + 自动类型推导 |
| 截图服务 | Playwright (headless) | 无头浏览器截图，支持精确等待 |
| 视觉审查 | 多模态 LLM (GPT-4o / Claude) | 布局/审美/内容多维审查 |
| 构建工具 | Vite | 快速构建和热更新 |

## 实施路线图

| 阶段 | 目标 | 关键交付 |
|------|------|---------|
| Phase 1 | 核心框架 + 基础组件 | Canvas DSL 规范 / 渲染引擎 / 8 个核心组件 / Light+Dark 主题 |
| Phase 2 | 组件完善 + 模板系统 | 40+ 组件全覆盖 / 图表/图/布局 / 预设模板 / 响应式 |
| Phase 3 | 视觉反馈闭环 | 截图服务 / 多模态审查 / 自动修正 / 迭代控制 |
| Phase 4 | Agent 集成 + 优化 | 提示词模板 / Schema 约束 / 中间层 SIR / 性能优化 |
| Phase 5 | 高级特性 | 动画 / 交互 / 实时数据 / 多页面 / 导出(PDF/图片/HTML) |

## 项目文档

### 设计文档

| 文档 | 说明 |
|------|------|
| [PAPER.md](./PAPER.md) | 学术论文 — 完整研究论文，涵盖问题分析、相关工作、三层架构设计、视觉语法原语、301 case 验证、组合策略、跨元素关联、利弊分析、Harness 工程方法论（38 篇引用） |
| [DESIGN.md](./DESIGN.md) | 架构设计 — 问题分析、Canvas DSL 规范、组件库设计、Agent 生态集成（第十章）、Harness 工程方法论（第十一章）、对比分析、参考文献 |
| [THREE-LAYERS.md](./THREE-LAYERS.md) | 三层架构 — Semantic Layer → Data Layer → Visual Grammar Layer 的完整定义、层间映射规则、学术验证、利弊分析 |
| [GRAMMAR.md](./GRAMMAR.md) | 视觉语法 — 4 种视觉原语（Mark / Relation / Boundary / ChannelMapping），覆盖 99% case，与传统图表的对应关系 |
| [INTERMEDIATE-LAYER.md](./INTERMEDIATE-LAYER.md) | 中间层设计 — 意图解析、十种信息元素（早期版本）、组织规则、SIR 语义中间表示、视觉映射 |
| [COMPOSITION.md](./COMPOSITION.md) | 组合与堆叠 — 三层组合体系、七种布局模式(含关联并列)、主辅关系、组合决策树、空间分配算法 |
| [LINKING.md](./LINKING.md) | 跨元素关联层 — 七种关联类型、关联组织规则、五种视觉化手段、Linked Views 模式、位置注册表机制 |

### Case 收集与覆盖度验证

| 文档 | 说明 |
|------|------|
| [CASES.md](./CASES.md) | 认知框架 + 77 个基础 case — 八种原子模式、十领域场景 |
| [COVERAGE-ANALYSIS.md](./COVERAGE-ANALYSIS.md) | 覆盖度分析 — 基于 ISCO-08/GB-T 4754/Shneiderman/Munzner 多维度检验 + 48 个补充 case |
| [CASES-EXPANSION.md](./CASES-EXPANSION.md) | 行业×部门×岗位深挖 — 95 个 case，覆盖医疗/金融/制造/农业/法律等 |
| [CASES-INTERNET.md](./CASES-INTERNET.md) | 互联网行业专项 — 81 个 case，含算法AI/SRE/增长/商业化等 |

## License

[Apache License 2.0](./LICENSE)
