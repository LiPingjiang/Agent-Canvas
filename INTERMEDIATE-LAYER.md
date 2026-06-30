# Agent Canvas — 中间层设计：从需求到可视化的语义桥梁

> 用户的需求是混沌的、自然语言的、意图驱动的；最终的 Canvas Schema 是结构化的、组件化的、渲染就绪的。这两者之间存在巨大的语义鸿沟。本文档定义这个中间层——如何把用户需求抽离拆解为语义元素，按规则组织，再映射到可视化表达。

---

## 一、问题：为什么需要中间层？

### 1.1 当前的跳跃

DESIGN.md 中的流程是：

```
用户需求 → Agent 生成 Canvas Schema (JSON) → 渲染
```

这里 Agent 直接从"理解需求"跳到"写 Schema"，中间没有一个显式的中间表示。这带来几个问题：

**问题一：Agent 的认知负荷过大**。Agent 需要同时完成"理解要展示什么""决定怎么组织""选择什么组件""设置什么属性"四件事，没有分步推理的空间。

**问题二：难以复用和组合**。同样的信息元素（如"一组带趋势的指标"）在不同场景下应该用不同组件展示（仪表盘用 StatCard+Sparkline，报告用表格+折线图），但没有中间层就无法"一次拆解、多种呈现"。

**问题三：难以做视觉反馈**。视觉反馈闭环如果只检查"Schema 渲染结果对不对"，就太晚了——可能是需求理解阶段就错了。中间层可以在更早的阶段捕获理解偏差。

### 1.2 中间层的位置

```
用户需求（自然语言）
    ↓
┌──────────────────────────────────────────┐
│            中 间 层 (本文档)               │
│                                          │
│  Step 1: 意图解析 — 理解"要展示什么"       │
│  Step 2: 元素拆解 — 提取信息原子           │
│  Step 3: 结构组织 — 按规则组织元素          │
│  Step 4: 视觉映射 — 映射到可视化原语        │
│                                          │
└──────────────────────────────────────────┘
    ↓
Canvas Schema (JSON) → 渲染引擎 → 视觉输出
```

中间层的产物不是最终的 Canvas Schema，而是一个**语义中间表示（Semantic Intermediate Representation, SIR）**。SIR 是"我想表达什么"的结构化描述，与"用什么组件表达"解耦。同一个 SIR 可以映射到不同的 Canvas Schema（如仪表盘版、报告版、移动端版）。

---

## 二、Step 1: 意图解析

### 2.1 解析什么？

从用户的自然语言请求中提取六个维度的意图信息：

**维度一：信息目标（What）**
用户想展示什么信息？是数据指标、关系结构、流程过程、还是叙事论证？

示例：
- "展示Q2销售数据" → 信息目标：数据指标 + 时间趋势
- "画出微服务依赖关系" → 信息目标：关系结构
- "说明这次故障的复盘" → 信息目标：叙事论证 + 时间序列

**维度二：受众（Who）**
给谁看？高管看概览、工程师看细节、客户看结果、公众看透明度。

示例：
- "给CEO看一下" → 受众：高管，需要极简、高密、关键数字
- "发给团队同步" → 受众：执行层，需要完整、可操作

**维度三：场景（Where）**
在什么场景下看？仪表盘大屏、手机推送、邮件附件、文档内嵌。

示例：
- "投到会议大屏上" → 场景：大屏，需要高对比、大字体、少交互
- "发到群里" → 场景：移动端，需要响应式、轻量

**维度四：目的（Why）**
为什么要展示？是告知、说服、探索、决策？

示例：
- "说服老板批准预算" → 目的：说服，需要对比、趋势、预测
- "帮我看看数据有什么异常" → 目的：探索，需要交互、多维、可筛选

**维度五：数据形态（DataShape）**
用户提供的或可获取的数据是什么形态？单个数字、时间序列、分类数据、关系数据、文本。

示例：
- "营收1234万，环比+12%" → 数据形态：标量 + 变化率
- "过去12个月每月的DAU" → 数据形态：时间序列

**维度六：约束（Constraints）**
有什么约束？篇幅限制、重点强调、品牌要求、数据敏感度。

示例：
- "一页纸搞定" → 约束：空间有限，需要压缩
- "重点突出Q2的突破" → 约束：需要视觉强调

### 2.2 意图解析的输出

```typescript
interface Intent {
  what: InformationGoal[];    // 信息目标
  who: AudienceType;           // 受众
  where: ScenarioType;         // 场景
  why: PurposeType;            // 目的
  dataShape: DataShape[];      // 数据形态
  constraints: Constraint[];   // 约束
}

// 信息目标类型
type InformationGoal =
  | "metrics"       // 指标展示
  | "trend"         // 趋势追踪
  | "comparison"    // 对比分析
  | "composition"   // 构成分析
  | "distribution"  // 分布分析
  | "relationship"  // 关系展示
  | "flow"          // 流程展示
  | "hierarchy"     // 层级结构
  | "narrative"     // 叙事论证
  | "decision"      // 决策辅助
  | "navigation"    // 导航指引
  | "simulation";   // 模拟预测

// 受众类型
type AudienceType =
  | "executive"     // 高管
  | "manager"       // 中层管理
  | "engineer"      // 技术人员
  | "analyst"       // 分析师
  | "customer"      // 外部客户
  | "public";       // 公众

// 场景类型
type ScenarioType =
  | "dashboard"     // 仪表盘
  | "report"        // 报告文档
  | "presentation"  // 演示
  | "mobile"        // 移动端
  | "wall-display"  // 大屏
  | "embed";        // 内嵌

// 目的类型
type PurposeType =
  | "inform"        // 告知
  | "persuade"      // 说服
  | "explore"       // 探索
  | "decide"        // 决策
  | "monitor"       // 监控
  | "explain";      // 解释

// 数据形态
type DataShape =
  | "scalar"        // 标量（单个数字）
  | "change"        // 变化量/率
  | "timeseries"    // 时间序列
  | "categorical"   // 分类数据
  | "relational"    // 关系数据
  | "hierarchical"  // 层级数据
  | "spatial"       // 空间/地理数据
  | "textual"       // 文本
  | "tabular";      // 表格

// 约束
interface Constraint {
  type: "space" | "emphasis" | "brand" | "sensitivity" | "interaction";
  value: string;
}
```

---

## 三、Step 2: 元素拆解

### 3.1 什么是信息元素？

信息元素是需求拆解后的**最小语义单元**——每个元素承载一个独立的信息含义。它是从用户需求中"抽离"出来的。

关键特征：
- **原子性**：每个元素表达一个独立的信息含义，不可再分
- **语义性**：元素描述的是"要表达什么"，不是"怎么表达"
- **可组合**：元素可以组合成更复杂的信息结构

### 3.2 十种信息元素类型

从 301 个 case 中归纳，所有可视化需求可以拆解为以下十种信息元素：

**E1. 数据点（DataPoint）**
一个带标签和值的单一数据项，可附带变化趋势。

```
{
  "type": "DataPoint",
  "label": "总营收",
  "value": "¥1,234,567",
  "change": { "direction": "up", "amount": "12.5%" },
  "period": "2025 Q2"
}
```

> 对应 case：B01 KPI卡片、B06 目标完成度、HR06 人均成本

**E2. 数据序列（DataSeries）**
按某个维度（通常是时间）排列的一组数据点。

```
{
  "type": "DataSeries",
  "label": "月度营收",
  "xAxis": "月份",
  "yAxis": "金额(万元)",
  "points": [
    { "x": "4月", "y": 380 },
    { "x": "5月", "y": 420 },
    { "x": "6月", "y": 435 }
  ],
  "annotations": [
    { "at": "5月", "label": "大促活动" }
  ]
}
```

> 对应 case：B05 同环比趋势、F03 股票走势、PL01 健身追踪

**E3. 对比项（Comparison）**
两个或多个对象在多个维度上的差异。

```
{
  "type": "Comparison",
  "subjects": ["方案A", "方案B", "方案C"],
  "dimensions": [
    { "name": "性能", "values": [8, 7, 9] },
    { "name": "成本", "values": [6, 8, 5] },
    { "name": "运维", "values": [7, 9, 6] }
  ],
  "recommendation": "方案A（综合得分最高）"
}
```

> 对应 case：T10 技术选型、PD03 功能对比、L05 购物比较、DM01 权衡矩阵

**E4. 构成项（Composition）**
部分与整体的构成关系，各部分占比之和为100%。

```
{
  "type": "Composition",
  "whole": "总营收",
  "parts": [
    { "label": "外卖", "value": 45, "unit": "%" },
    { "label": "到店", "value": 25, "unit": "%" },
    { "label": "酒旅", "value": 20, "unit": "%" },
    { "label": "其他", "value": 10, "unit": "%" }
  ],
  "drilldown": true
}
```

> 对应 case：B04 营收构成、FIN01 成本构成、RTL03 品类占比

**E5. 关系网络（Relation）**
实体之间的关联关系，包含节点和边。

```
{
  "type": "Relation",
  "nodes": [
    { "id": "order", "label": "订单服务", "category": "core" },
    { "id": "user", "label": "用户服务", "category": "support" },
    { "id": "inventory", "label": "库存服务", "category": "support" }
  ],
  "edges": [
    { "from": "order", "to": "user", "label": "依赖", "direction": "directed" },
    { "from": "order", "to": "inventory", "label": "依赖", "direction": "directed" }
  ]
}
```

> 对应 case：T02 服务依赖、T04 ER图、D10 知识图谱、NET_SEC01 攻击链

**E6. 流程过程（Process）**
有顺序的步骤或阶段，可能有分支和并行。

```
{
  "type": "Process",
  "steps": [
    { "id": "s1", "label": "代码提交", "actor": "开发者" },
    { "id": "s2", "label": "CI构建", "actor": "系统" },
    { "id": "s3", "label": "单元测试", "actor": "系统" },
    { "id": "s4", "label": "部署", "actor": "系统", "gate": "审批" }
  ],
  "flows": [
    { "from": "s1", "to": "s2" },
    { "from": "s2", "to": "s3" },
    { "from": "s3", "to": "s4", "condition": "测试通过" }
  ]
}
```

> 对应 case：T05 部署流水线、T08 状态机、B02 销售漏斗、T03 API时序

**E7. 层级结构（Hierarchy）**
树形嵌套的包含/从属关系。

```
{
  "type": "Hierarchy",
  "root": {
    "label": "技术部",
    "children": [
      {
        "label": "前端组",
        "children": [
          { "label": "Web前端" },
          { "label": "移动端" }
        ]
      },
      {
        "label": "后端组",
        "children": [
          { "label": "服务端" },
          { "label": "中间件" }
        ]
      }
    ]
  }
}
```

> 对应 case：T13 技术栈、HR05 继任计划、E01 知识结构图、M03 BOM树

**E8. 分布（Distribution）**
数据在一个或多个维度上的散布情况。

```
{
  "type": "Distribution",
  "dimensions": ["年龄", "消费金额"],
  "data": [
    { "age": 25, "spend": 3200 },
    { "age": 28, "spend": 5800 },
    { "age": 32, "spend": 4200 }
  ],
  "bins": "auto"
}
```

> 对应 case：D01 探索分析、B07 用户分层、D06 异常检测、S03 票房分布

**E9. 叙事块（Narrative）**
一段结构化的文字论述，可能嵌入数据引用。

```
{
  "type": "Narrative",
  "sections": [
    {
      "heading": "背景",
      "body": "Q2 受市场竞争加剧影响，获客成本上升 18%。",
      "dataRefs": ["获客成本趋势"]
    },
    {
      "heading": "分析",
      "body": "主要竞品在3月推出补贴策略，导致自然流量被分流。",
      "dataRefs": ["流量对比"]
    },
    {
      "heading": "结论",
      "body": "建议Q3 增加品牌投放，降低对效果广告的依赖。",
      "emphasis": "high"
    }
  ]
}
```

> 对应 case：B10 商业计划书、P08 周报、T12 故障复盘、NPO04 影响力报告

**E10. 决策项（Decision）**
需要在多个选项中做选择的场景，含评估维度和权重。

```
{
  "type": "Decision",
  "question": "选择哪个日志方案？",
  "options": ["ELK", "Loki", "Fluentd+S3"],
  "criteria": [
    { "name": "性能", "weight": 0.3, "scores": [8, 7, 6] },
    { "name": "成本", "weight": 0.25, "scores": [6, 8, 7] },
    { "name": "运维", "weight": 0.25, "scores": [7, 9, 8] },
    { "name": "生态", "weight": 0.2, "scores": [9, 6, 5] }
  ],
  "recommendation": "ELK（加权得分 7.55）"
}
```

> 对应 case：RD05 技术选型、H05 手术方案、DM01 offer选择、G02 合规检查

### 3.3 元素拆解的规则

从用户需求中拆解信息元素时，遵循以下规则：

**规则一：完整性**。需求中的每个独立信息含义都应该被拆解为一个元素。不能遗漏，也不能合并两个不同含义的信息。

**规则二：颗粒度**。元素的颗粒度是"一个独立的含义单元"。"Q2营收1234万，环比增长12%"是一个 DataPoint（带 change），不是两个元素。

**规则三：保留语义，丢弃样式**。拆解时只记录"要表达什么"，不记录"用什么图表"。比如"月度营收趋势"是 DataSeries，不管最终用折线图还是柱状图。

**规则四：标注关系**。元素之间可能有关联（如一个 Narrative 引用一个 DataSeries），这种关联需要在拆解时保留。

---

## 四、Step 3: 结构组织

### 4.1 为什么要组织？

拆解后的信息元素是"散落的积木"。组织就是把积木按规则搭成结构——决定哪些元素放在一起、谁先谁后、谁是主角谁是配角。

### 4.2 六条组织规则

**规则一：语义亲和（Proximity）**
语义相关的元素应该在空间上邻近。

```
示例：
  DataPoint("总营收") + DataPoint("订单量") + DataPoint("客单价")
  → 这三个都是"核心经营指标"，应该组织在一起形成"指标卡组"
```

**规则二：信息层级（Hierarchy）**
信息有主次之分，重要信息应该获得更多视觉权重。

```
层级分为三级：
  L1 (焦点) — 最重要、最先看到的信息（如核心KPI、关键结论）
  L2 (支撑) — 支撑焦点的详细信息（如趋势图、明细表）
  L3 (补充) — 辅助理解的背景信息（如注释、数据来源）
```

**规则三：阅读流（Flow）**
信息应该按照自然的阅读流组织。常见阅读流模式：

```
模式A: 总→分（高管视角）
  概览指标 → 趋势图表 → 明细表格

模式B: 因→果（分析视角）
  背景叙述 → 数据证据 → 分析结论

模式C: 并列对比（决策视角）
  选项A详情 | 选项B详情 | 选项C详情 → 对比矩阵 → 推荐

模式D: 时间流（叙事视角）
  事件1 → 事件2 → 事件3 → 总结

模式E: 层级下钻（探索视角）
  全局概览 → 区域筛选 → 单点详情
```

**规则四：分组容器（Grouping）**
多个元素如果属于同一语义组，应该用容器包裹。

```
常见分组：
  "指标卡组" — 多个相关的 DataPoint
  "图表组" — 多个相关的 DataSeries/Composition
  "对比组" — 多个 Comparison 的不同维度
  "叙事段" — 一个 Narrative 的多个 section
```

**规则五：空间平衡（Balance）**
组织后的结构在视觉上应该平衡——避免一侧过密一侧过空。

```
平衡规则：
  - 偶数个同级元素 → 对称排列
  - 奇数个同级元素 → 焦点居中 + 两侧辅助
  - 不同尺寸元素 → 大的在上/左，小的在下/右
```

**规则六：约束适配（Constraint-Aware）**
组织时必须考虑意图解析阶段提取的约束。

```
约束适配：
  - "一页纸" → 压缩层级，L3 隐藏，L1/L2 合并
  - "大屏展示" → 减少元素数量，放大 L1，去掉细节
  - "移动端" → 纵向流，单列，L1 置顶
  - "重点突出Q2突破" → 该元素 emphasis=high，占据更大空间
```

### 4.3 组织后的产物：语义中间表示（SIR）

```typescript
interface SIR {
  // 元数据（来自意图解析）
  intent: Intent;

  // 组织后的元素树
  root: SIRNode;

  // 全局布局意图
  layout: {
    pattern: "dashboard" | "report" | "comparison" | "timeline" | "flow" | "grid";
    readingFlow: "top-down" | "left-right" | "z-pattern" | "f-pattern";
    density: "sparse" | "normal" | "dense";
  };
}

interface SIRNode {
  // 元素引用或内联元素
  element?: InformationElement;    // 叶子节点：直接是一个信息元素
  children?: SIRNode[];            // 容器节点：包含子节点

  // 组织信息
  role: "focus" | "support" | "supplement";  // 信息层级 L1/L2/L3
  group?: string;                  // 所属分组名
  emphasis?: "high" | "medium" | "low";      // 视觉强调度
  order?: number;                  // 阅读顺序
}
```

### 4.4 组织示例

以"展示Q2经营数据，给CEO看，投到大屏上"为例：

```
意图解析:
  what: [metrics, trend, composition]
  who: executive
  where: wall-display
  why: inform
  dataShape: [scalar, change, timeseries, categorical]
  constraints: [{ type: "space", value: "大屏，远距离阅读" }]

元素拆解:
  E1: DataPoint("总营收", "¥1,234,567", change: +12.5%)
  E2: DataPoint("订单量", "89,012", change: +8.3%)
  E3: DataPoint("客单价", "¥139", change: +3.8%)
  E4: DataSeries("月度营收趋势", [4月:380万, 5月:420万, 6月:435万])
  E5: Composition("营收构成", [外卖:45%, 到店:25%, 酒旅:20%, 其他:10%])

结构组织 (SIR):
  root (dashboard, top-down, sparse)
  ├── group: "核心指标" (L1 focus)
  │   ├── E1 (emphasis: high)
  │   ├── E2 (emphasis: medium)
  │   └── E3 (emphasis: medium)
  ├── group: "趋势" (L2 support)
  │   └── E4
  └── group: "构成" (L2 support)
      └── E5
```

---

## 五、Step 4: 视觉映射

### 5.1 映射规则

SIR 中的信息元素和组织结构映射到 Canvas DSL 组件。映射分两层：元素→组件，组织→布局。

### 5.2 元素到组件的映射表

| 信息元素 | 首选组件 | 备选组件 | 选择依据 |
|---------|---------|---------|---------|
| DataPoint | StatCard | Badge / KPIBlock / Gauge | 有趋势→StatCard+Sparkline；有目标→Gauge；纯数字→Badge |
| DataSeries | Chart(line) | Chart(bar/area) | 时间序列→折线；离散对比→柱状；累积→面积 |
| Comparison | ComparisonTable | RadarChart / GroupedBarChart | 少维度→雷达图；多维度→对比表；纯数值→分组柱 |
| Composition | Chart(pie) | Chart(donut) / TreeMap / StackedBar | 少分类→饼图/环形；多分类→树形图；有时间维度→堆叠柱 |
| Relation | NetworkGraph | RelationGraph / ERDiagram | 通用关系→网络图；数据库关系→ER图 |
| Process | FlowChart | SequenceDiagram / Sankey / Funnel | 步骤流程→流程图；时序交互→序列图；转化漏斗→漏斗图 |
| Hierarchy | TreeView | OrgChart / MindMap / TreeMap | 组织结构→组织架构图；知识结构→思维导图；占比层级→树形图 |
| Distribution | Chart(scatter) | Heatmap / Histogram / BoxPlot | 二维分布→散点；密度→热力图；一维分布→直方图 |
| Narrative | Report(Heading+Text+Callout) | Card / Accordion | 完整报告→Report；片段→Card；可折叠→Accordion |
| Decision | DecisionMatrix | DecisionTree / ProConTable | 多维度评分→矩阵；分支逻辑→决策树；二元权衡→利弊表 |

### 5.3 组织到布局的映射表

| 组织模式 | 布局组件 | 布局属性 |
|---------|---------|---------|
| dashboard | Grid | columns: 2-3, gap: 16, responsive: true |
| report | Stack | direction: column, gap: 32 |
| comparison | Grid | columns: N(选项数), gap: 16 |
| timeline | Stack | direction: column, gap: 12 |
| flow | Stack | direction: column, gap: 16 |
| grid | Grid | columns: auto, gap: 12 |

### 5.4 信息层级到视觉权重的映射

| 信息层级 | 视觉权重 | 实现方式 |
|---------|---------|---------|
| L1 (focus) | 高 | 更大尺寸、强调色、卡片阴影、顶部位置 |
| L2 (support) | 中 | 标准尺寸、常规色、正常位置 |
| L3 (supplement) | 低 | 更小尺寸、弱化色、折叠/次要位置 |

### 5.5 约束到样式调整的映射

| 约束 | 样式调整 |
|------|---------|
| 大屏展示 | 字体放大 1.5x，减少组件数量，高对比度 |
| 移动端 | 单列布局，简化图表，去掉次要信息 |
| 一页纸 | 压缩间距，合并同类项，L3 隐藏 |
| 重点强调 | emphasis=high 的元素跨列(span 2)，强调色 |
| 数据敏感 | 敏感数据用 Badge 遮罩，需点击展开 |

### 5.6 完整映射示例

SIR → Canvas Schema 的映射（接续 4.4 的示例）：

```
SIR:
  root (dashboard, top-down, sparse)
  ├── group: "核心指标" (L1 focus, emphasis: high)
  │   ├── E1: DataPoint("总营收", change: +12.5%)
  │   ├── E2: DataPoint("订单量", change: +8.3%)
  │   └── E3: DataPoint("客单价", change: +3.8%)
  ├── group: "趋势" (L2 support)
  │   └── E4: DataSeries("月度营收趋势")
  └── group: "构成" (L2 support)
      └── E5: Composition("营收构成")

→ Canvas Schema:
{
  "type": "Dashboard",
  "props": { "title": "Q2 经营数据概览" },
  "layout": { "type": "grid", "columns": 3, "gap": 16 },
  "children": [
    // 核心指标组 → 3个 StatCard（L1，emphasis: high）
    {
      "type": "StatCard",
      "props": {
        "label": "总营收", "value": "¥1,234,567",
        "trend": { "direction": "up", "percentage": 12.5 },
        "emphasis": "high"
      }
    },
    {
      "type": "StatCard",
      "props": {
        "label": "订单量", "value": "89,012",
        "trend": { "direction": "up", "percentage": 8.3 }
      }
    },
    {
      "type": "StatCard",
      "props": {
        "label": "客单价", "value": "¥139",
        "trend": { "direction": "up", "percentage": 3.8 }
      }
    },
    // 趋势组 → Chart(line)（L2，跨2列）
    {
      "type": "Chart",
      "props": {
        "chartType": "line",
        "title": "月度营收趋势",
        "data": [...],
        "xAxis": "月份", "yAxis": "金额(万元)"
      },
      "layout": { "span": 2 }
    },
    // 构成组 → Chart(donut)（L2，1列）
    {
      "type": "Chart",
      "props": {
        "chartType": "donut",
        "title": "营收构成",
        "data": [...]
      }
    }
  ]
}
```

---

## 六、中间层的完整流程

### 6.1 端到端示例

以"帮我整理一下这次故障复盘，发给团队"为例：

```
Step 1: 意图解析
  what: [narrative, flow, timeline]
  who: engineer (团队)
  where: report (文档内嵌)
  why: explain
  dataShape: [textual, timeseries]
  constraints: [{ type: "emphasis", value: "突出根因和改进措施" }]

Step 2: 元素拆解
  E1: Narrative("故障概述", "14:03-14:25 订单服务不可用，影响约 12 万用户")
  E2: Process("事件时间线", [告警→确认→定位→回滚→恢复])
  E3: DataSeries("影响范围", [14:03:0, 14:05:5000, 14:10:12000, 14:20:8000, 14:25:0])
  E4: Narrative("根因分析", "数据库连接池耗尽，原因是新上线代码未释放连接")
  E5: Narrative("改进措施", "1.代码审查增加连接释放检查 2.增加连接池监控告警 3.灰度发布", emphasis: high)

Step 3: 结构组织 (SIR)
  root (report, top-down, normal)
  ├── E1 (L1 focus) — 故障概述，最先看到
  ├── E2 (L2 support) — 时间线，支撑概述
  ├── E3 (L2 support) — 影响范围数据，嵌入时间线旁
  ├── E4 (L2 support) — 根因分析
  └── E5 (L1 focus, emphasis: high) — 改进措施，最后强调

Step 4: 视觉映射 → Canvas Schema
  Report(layout: stack)
  ├── Callout(type: error, title: "故障概述", content: "...")
  ├── Timeline(items: [...])
  ├── Chart(chartType: area, title: "影响范围", data: [...])
  ├── Callout(type: warn, title: "根因分析", content: "...")
  └── Callout(type: success, title: "改进措施", content: "...", emphasis: high)
```

### 6.2 同一 SIR 的多场景映射

中间层的一个核心价值：**同一个 SIR 可以根据不同的 intent.where 映射到不同的 Canvas Schema**。

以 SIR = 上面的故障复盘为例：

**场景A：报告文档（report）**
```
→ Stack 布局，纵向排列，每个元素独立段落
→ Narrative → Heading + Text + Callout
→ Process → Timeline
→ DataSeries → Chart(area)
```

**场景B：大屏展示（wall-display）**
```
→ Grid 2列布局，压缩信息
→ E1+E4 合并为一个 Callout（概述+根因）
→ E2 → Timeline（简化，只保留关键节点）
→ E3 → 去掉（大屏不适合精细数据图）
→ E5 → 大号 Callout 置底，强调色
```

**场景C：移动推送（mobile）**
```
→ Stack 单列，极度精简
→ 只保留 E1（概述）和 E5（改进措施）
→ E1 → Card 简短摘要
→ E5 → Card 高亮
→ E2/E3/E4 折叠为 Accordion
```

---

## 七、中间层与现有架构的关系

### 7.1 在整体架构中的位置

```
┌─────────────────────────────────────────────────────────────────┐
│                      Agent Canvas System                       │
│                                                                 │
│  用户需求                                                       │
│    ↓                                                            │
│  ┌─────────────┐                                               │
│  │ 意图解析     │ ← Intent (what/who/where/why/data/constraints)│
│  └──────┬──────┘                                               │
│         ↓                                                      │
│  ┌─────────────┐                                               │
│  │ 元素拆解     │ ← 10 种信息元素 (DataPoint/Comparison/...)    │
│  └──────┬──────┘                                               │
│         ↓                                                      │
│  ┌─────────────┐                                               │
│  │ 结构组织     │ ← SIR (语义中间表示)                           │
│  └──────┬──────┘     ↑                                         │
│         ↓          ┌──────┐                                    │
│  ┌─────────────┐   │视觉  │                                    │
│  │ 视觉映射     │──→│反馈  │                                    │
│  └──────┬──────┘   │闭环  │                                    │
│         ↓          └──────┘                                    │
│  Canvas Schema (JSON)                                          │
│    ↓                                                            │
│  Renderer → 视觉输出                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 与 Canvas DSL 的关系

中间层的产物（SIR）和 Canvas DSL（Schema）是**两个不同的抽象层次**：

| 维度 | SIR（中间表示） | Canvas Schema（最终产物） |
|------|----------------|------------------------|
| 描述什么 | "要表达什么" | "怎么渲染" |
| 是否含组件 | 不含，只有信息元素 | 含，指定具体组件 |
| 是否含样式 | 不含，只有 emphasis 层级 | 含，含布局/颜色/尺寸 |
| 可变性 | 同一需求只有一个 SIR | 同一 SIR 可映射出多个 Schema |
| Agent 参与度 | 高（核心推理） | 低（机械映射） |

### 7.3 对视觉反馈闭环的增强

有了中间层，视觉反馈闭环可以在**两个层次**工作：

**层次一：SIR 级审查**。在视觉映射之前，检查 SIR 的完整性：
- 是否遗漏了用户需求中的关键信息？
- 信息层级是否合理？
- 阅读流是否通顺？
- 约束是否被满足？

**层次二：Schema 级审查**（原有设计）。在渲染之后，检查视觉效果：
- 布局是否正确？
- 是否有重叠/溢出？
- 视觉层级是否清晰？

两层反馈的分工：SIR 级审查捕获"理解错误"，Schema 级审查捕获"渲染错误"。

---

## 八、对 Agent 提示词的影响

中间层直接影响 Agent 生成可视化时的工作方式。Agent 的提示词应该引导它分步执行：

```
你是一个可视化助手。当用户请求可视化时，按以下步骤工作：

Step 1 - 意图解析：
分析用户需求，提取信息目标、受众、场景、目的、数据形态、约束。

Step 2 - 元素拆解：
从需求中提取信息元素（DataPoint/DataSeries/Comparison/...），每个元素是独立的信息单元。

Step 3 - 结构组织：
按组织规则（语义亲和、信息层级、阅读流、分组、平衡、约束适配）将元素组织为 SIR。

Step 4 - 视觉映射：
根据映射规则，将 SIR 映射为 Canvas Schema（选择组件、设置布局、应用约束）。

每一步都可以输出中间结果供审查。
```

这种分步方式相比"直接生成 Schema"有几个优势：
- 每一步的推理更聚焦，出错率更低
- 中间结果可审查、可修正
- 同一 SIR 可复用到不同场景
