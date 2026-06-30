# Agent Canvas — 三层抽象：Semantic Layer → Data Layer → Visual Grammar Layer

> 用户需求和视觉渲染之间存在两层转换。本文档定义完整的三层架构，理清每层的抽象对象和层间映射规则。

---

## 一、问题：缺了什么？

当前的设计从"意图解析"直接跳到 Mark（视觉标记），但这两者之间有一个语义鸿沟：

```
用户需求: "对比三个方案的性能、成本、运维"
                    ↓  (这一步跳了太多)
视觉层:   Mark(方案A性能=8, shape=bar, size=8) + Mark(方案B性能=7, ...)
```

Agent 在这个跳跃中需要同时完成三件事：理解"对比方案"是什么意思（语义）、把方案和维度拆成可量化的数据结构（数据）、决定用什么视觉形态展示（视觉）。三件事混在一起，认知负荷过大。

**缺的是两层中间抽象**：

- **语义层**：用户想表达的"意思"是什么——对比三个方案，维度是性能/成本/运维，每个方案在每个维度有一个评估值。
- **数据层**：这些"意思"落地为结构化的数据项——三个实体（方案），三个属性（性能/成本/运维），每个实体×属性交叉点有一个值。

语义层回答"要表达什么含义"，数据层回答"具体有哪些数据"，视觉层回答"用什么视觉形态呈现"。三层各司其职，层间通过映射规则转换。

---

## 二、三层架构总览

```
用户需求（自然语言）
    ↓
┌──────────────────────────────────────────────────────┐
│  语义层（Semantic Layer）                              │
│                                                      │
│  语义元素：用户意图中识别出的"意义单元"                  │
│  — 不含具体数据值，不含视觉属性                         │
│  — 描述"要表达什么含义"                                 │
│                                                      │
│  语义关系：语义元素之间的关系                            │
│  — 比较、归属、流转、序列、因果...                       │
└──────────────────────┬───────────────────────────────┘
                       ↓  语义→数据 映射
┌──────────────────────────────────────────────────────┐
│  数据层（Data Layer）                                  │
│                                                      │
│  数据项：从语义元素实例化的具体数据                      │
│  — 有实体ID、属性名、具体值、数据类型                    │
│  — 不含任何视觉属性                                    │
│                                                      │
│  数据关系：数据项之间的关系（来自语义关系的实例化）       │
└──────────────────────┬───────────────────────────────┘
                       ↓  数据→视觉 映射 (ChannelMapping)
┌──────────────────────────────────────────────────────┐
│  视觉层（Visual Layer）— 即 GRAMMAR.md 中的四种原语     │
│                                                      │
│  Mark：有位置和视觉通道的标记                           │
│  Relation：标记间的连接（Connection/Flow/Containment/  │
│           Sequence/Alignment）                        │
│  Boundary：标记的视觉圈定                              │
│  ChannelMapping：数据字段→视觉通道的映射                │
└──────────────────────────────────────────────────────┘
```

### 2.1 三层的职责边界

| 维度 | 语义层 | 数据层 | 视觉层 |
|------|--------|--------|--------|
| 回答什么 | 要表达什么含义 | 具体有哪些数据 | 怎么呈现 |
| 包含值吗 | 不含 | 含具体值 | 含视觉属性 |
| 关心布局吗 | 不关心 | 不关心 | 关心 |
| 关心颜色吗 | 不关心 | 不关心 | 关心 |
| 举例 | "对比三个方案的三个维度" | 方案A性能=8, 方案A成本=6... | Mark(shape=bar, size=8) |
| Agent 的任务 | 理解意图 | 提取数据 | 选择映射 |

---

## 三、语义层设计

### 3.1 语义元素

语义元素是用户意图中的"意义单元"。它描述要表达的含义，不含具体数据值。

```typescript
interface SemanticElement {
  id: string;
  type: SemanticType;
  label: string;                    // 人类可读的含义描述
  // 不含具体数据值，不含视觉属性

  // 语义角色：这个元素在整体表达中扮演什么角色
  role: "subject" | "attribute" | "measure" | "relation" | "context";

  // 语义约束：影响后续数据提取和视觉映射
  constraints?: {
    ordered?: boolean;              // 是否有内在顺序
    quantitative?: boolean;         // 是否可量化
    comparable?: boolean;           // 是否可比较
    hierarchical?: boolean;         // 是否有层级
    directional?: boolean;          // 是否有方向
  };
}

type SemanticType =
  | "entity"        // 实体：被讨论的对象（方案A、订单服务、华东区）
  | "attribute"     // 属性：实体的某个方面（性能、营收、CPU使用率）
  | "measure"       // 度量：可量化的属性值（8分、380万、92%）
  | "set"           // 集合：一组同类实体的聚合（高价值用户群、TOP10商品）
  | "event"         // 事件：某个时间点发生的事（大促、上线、故障）
  | "process"       // 过程：有序的步骤序列（审批流程、部署流水线）
  | "narrative"     // 叙事：一段论述（背景、分析、结论）
  | "constraint";   // 约束：限制条件（一页纸、重点突出Q2）
```

### 3.2 语义角色详解

**subject（主体）** — 被讨论的对象。是信息的"主角"。

```
"对比三个方案" → subject: 方案A, 方案B, 方案C
"展示各部门离职率" → subject: 研发部, 市场部, 销售部, ...
```

**attribute（属性）** — 主体的某个被关注的方面。

```
"对比三个方案的性能、成本、运维" → attribute: 性能, 成本, 运维
"展示各部门离职率" → attribute: 离职率
```

**measure（度量）** — 属性的可量化值。

```
"方案A性能8分" → measure: 8 (属于 方案A × 性能)
"研发部离职率16%" → measure: 16% (属于 研发部 × 离职率)
```

**relation（关系）** — 主体之间的语义连接。

```
"订单服务依赖用户服务" → relation: 依赖 (订单服务 → 用户服务)
"页面A到页面B的转化" → relation: 流转 (页面A → 页面B)
```

**context（上下文）** — 背景信息，影响表达但不直接展示。

```
"这是Q2的数据" → context: 时间范围=Q2
"给CEO看" → context: 受众=高管
```

### 3.3 语义关系

语义元素之间的关系，描述它们在含义上怎么连接：

```typescript
interface SemanticRelation {
  id: string;
  type: SemanticRelationType;
  from: string;           // SemanticElement ID
  to: string;             // SemanticElement ID
  label?: string;
}

type SemanticRelationType =
  | "compare"          // 比较：两个主体在某属性上的差异
  | "belong_to"        // 归属：一个主体属于另一个（部门属于公司）
  | "flow_to"          // 流转：一个主体流向另一个（数据流、审批流）
  | "cause"            // 因果：A导致B
  | "part_of"          // 构成：A是B的一部分（外卖是营收的一部分）
  | "sequence"         // 序列：A在B之前
  | "correlate"        // 相关：A和B有统计相关
  | "reference";       // 引用：叙事A引用了度量B
```

### 3.4 语义层示例

用户需求："对比三个技术方案的性能、成本和运维，给出推荐。"

```
语义元素:
  e1: { type:"entity", label:"方案A", role:"subject" }
  e2: { type:"entity", label:"方案B", role:"subject" }
  e3: { type:"entity", label:"方案C", role:"subject" }
  e4: { type:"attribute", label:"性能", role:"attribute", constraints:{quantitative:true, comparable:true} }
  e5: { type:"attribute", label:"成本", role:"attribute", constraints:{quantitative:true, comparable:true} }
  e6: { type:"attribute", label:"运维", role:"attribute", constraints:{quantitative:true, comparable:true} }
  e7: { type:"narrative", label:"推荐结论", role:"context" }

语义关系:
  r1: { type:"compare", from:"e1", to:"e2", label:"性能对比" }
  r2: { type:"compare", from:"e2", to:"e3", label:"性能对比" }
  r3: { type:"compare", from:"e1", to:"e3", label:"性能对比" }
  // ... 成本、运维的对比关系
  r7: { type:"reference", from:"e7", to:"e1", label:"推荐方案A" }
```

注意：语义层没有"8分""6分"这些具体值，也没有"柱状图""雷达图"这些视觉决策。它只描述"要对比三个方案的三个维度"这个**含义**。

---

## 四、数据层设计

### 4.1 数据项

数据项是语义元素的**实例化**——把抽象的含义落地为具体的数据。

```typescript
interface DataItem {
  id: string;
  semanticRef: string;        // 来源语义元素 ID
  entityId: string;           // 实体标识（如 "plan_a"）
  entityLabel: string;        // 实体显示名（如 "方案A"）
  attribute: string;          // 属性名（如 "performance"）
  attributeLabel: string;     // 属性显示名（如 "性能"）
  value: any;                 // 具体值（8, "¥320", true, ...）
  dataType: "number" | "string" | "boolean" | "date" | "array";
  unit?: string;              // 单位（"分", "万", "%", "ms"）
  metadata?: Record<string, any>;  // 附加元数据
}
```

### 4.2 数据层示例

（接续上例，语义→数据 映射后）

```
数据项:
  d1: { semanticRef:"e1×e4", entityId:"plan_a", entityLabel:"方案A", attribute:"performance", value:8, dataType:"number", unit:"分" }
  d2: { semanticRef:"e2×e4", entityId:"plan_b", entityLabel:"方案B", attribute:"performance", value:7, dataType:"number", unit:"分" }
  d3: { semanticRef:"e3×e4", entityId:"plan_c", entityLabel:"方案C", attribute:"performance", value:9, dataType:"number", unit:"分" }
  d4: { semanticRef:"e1×e5", entityId:"plan_a", entityLabel:"方案A", attribute:"cost", value:6, dataType:"number", unit:"分" }
  d5: { semanticRef:"e2×e5", entityId:"plan_b", entityLabel:"方案B", attribute:"cost", value:8, dataType:"number", unit:"分" }
  d6: { semanticRef:"e3×e5", entityId:"plan_c", entityLabel:"方案C", attribute:"cost", value:5, dataType:"number", unit:"分" }
  d7: { semanticRef:"e1×e6", entityId:"plan_a", entityLabel:"方案A", attribute:"ops", value:7, dataType:"number", unit:"分" }
  d8: { semanticRef:"e2×e6", entityId:"plan_b", entityLabel:"方案B", attribute:"ops", value:9, dataType:"number", unit:"分" }
  d9: { semanticRef:"e3×e6", entityId:"plan_c", entityLabel:"方案C", attribute:"ops", value:6, dataType:"number", unit:"分" }

数据关系:
  dr1: { type:"compare", from:"d1", to:"d2", label:"性能差异" }
  // ... 同语义关系，但绑定到了具体数据项
```

### 4.3 语义→数据 映射规则

**规则一：实体×属性交叉展开**

语义层的 N 个 subject × M 个 attribute，在数据层展开为 N×M 个数据项。每个数据项是一个 (entity, attribute, value) 三元组。

```
3个方案 × 3个维度 = 9个数据项
每个数据项 = (方案X, 维度Y, 具体值)
```

**规则二：语义关系实例化**

语义层的每个关系在数据层实例化——绑定到具体的数据项。

```
语义关系: compare(e1, e2)  "方案A和方案B的对比"
  → 数据关系: compare(d1, d2)  "方案A性能8 vs 方案B性能7"
  → 数据关系: compare(d4, d5)  "方案A成本6 vs 方案B成本8"
  → 数据关系: compare(d7, d8)  "方案A运维7 vs 方案B运维9"
```

**规则三：值从用户输入或数据源获取**

数据项的 value 不在语义层产生——它来自用户的自然语言描述、结构化数据源、或 Agent 的推理结果。

```
用户说："方案A性能8分" → 直接提取 value=8
用户说："方案A性能很好" → Agent 推理 value=8（或要求用户补充）
数据源查询 → SELECT performance_score FROM plans WHERE name='A' → value=8
```

---

## 五、数据→视觉 映射

### 5.1 映射流程

数据层的 DataItem 集合通过 ChannelMapping 映射到视觉层的 Mark 集合：

```
DataItem[] → ChannelMapping → Mark[]

  选择哪些 DataItem 成为一组 Mark
  → 决定每个 Mark 的 position/size/color/shape 映射到哪个字段
  → 决定 Mark 之间的 Relation
  → 决定 Mark 的 Boundary
```

### 5.2 映射规则

**规则一：DataItem → Mark 一一对应（默认）**

每个 DataItem 默认映射为一个 Mark。Mark 的 value/label/metadata 来自 DataItem。

```
d1: (方案A, 性能, 8) → m1: { value:8, label:"方案A", metadata:{attribute:"performance"} }
```

**规则二：ChannelMapping 决定视觉形态**

同一个 9 个 DataItem（3方案×3维度），不同映射产生不同图表：

```
映射A — 雷达图:
  position: angle ← attribute, radius ← value
  color ← entityLabel
  shape = "circle" + 连线
  Boundary: 多边形（雷达轮廓）
  → 3条雷达线，每条3个点

映射B — 分组柱状图:
  position: x ← entityLabel(分组), x偏移 ← attributeLabel(组内)
  size: height ← value
  color ← attributeLabel
  shape = "bar"
  Boundary: 矩形（图表框）
  → 3组×3柱 = 9根柱子

映射C — 热力矩阵:
  position: x ← entityLabel, y ← attributeLabel
  color ← value (sequential scale)
  shape = "square"
  Boundary: 矩形（矩阵框）
  → 3×3 = 9个色块

映射D — 表格:
  text ← value
  position: x ← attributeLabel(列), y ← entityLabel(行)
  shape = "text"
  Relation: Alignment (行列对齐)
  Boundary: 矩形（表格框）
  → 3行×3列的表格
```

**同一个数据集，4种完全不同的可视化——区别只在 ChannelMapping**。

### 5.3 语义约束如何影响映射

语义层的 constraints 引导（但不强制）映射决策：

| 语义约束 | 影响的映射决策 | 示例 |
|---------|-------------|------|
| ordered=true | position.x 用 ordinal/time scale | 时间序列→折线图 |
| quantitative=false | 不能映射到 size/position.y | 分类数据→不用柱状图高度 |
| comparable=true | 适合映射到 size 做比较 | 对比→柱状图 |
| hierarchical=true | 适合 Containment Relation + 嵌套 Boundary | 组织架构→树形图 |
| directional=true | 适合 Flow Relation 带箭头 | 流程→流程图 |

### 5.4 意图如何影响映射

意图解析的六个维度也影响映射：

| 意图维度 | 映射影响 | 示例 |
|---------|---------|------|
| what=metrics | shape=text, 显示数值 | KPI卡片 |
| what=trend | position.x=time, Connection连线 | 折线图 |
| what=comparison | position 分组, color 区分 | 分组柱/雷达 |
| what=composition | Containment + Boundary(圆形) | 饼图 |
| who=executive | 大 size, 少 Mark, 高对比 | 极简仪表盘 |
| where=mobile | 少列, 纵向 Stack | 单列布局 |
| why=persuade | emphasis 高亮, 对比强化 | 差异标注 |
| constraints=一页纸 | 减少Mark数量, 压缩Boundary | 紧凑布局 |

---

## 六、完整端到端示例

### 6.1 示例：Q2 经营仪表盘

用户需求："帮我做一个Q2经营仪表盘，给CEO看，投到大屏上。包含营收、订单量、客单价三个核心指标，附带月度趋势和各业务线构成。"

**语义层：**

```
语义元素:
  // 主体
  e1: { type:"entity", label:"总营收", role:"subject", constraints:{quantitative:true} }
  e2: { type:"entity", label:"订单量", role:"subject", constraints:{quantitative:true} }
  e3: { type:"entity", label:"客单价", role:"subject", constraints:{quantitative:true} }

  // 属性
  e4: { type:"attribute", label:"Q2值", role:"attribute", constraints:{quantitative:true, comparable:true} }
  e5: { type:"attribute", label:"月度趋势", role:"attribute", constraints:{ordered:true} }
  e6: { type:"attribute", label:"业务线构成", role:"attribute", constraints:{hierarchical:true} }

  // 集合（构成部分）
  e7: { type:"set", label:"外卖", role:"subject" }
  e8: { type:"set", label:"到店", role:"subject" }
  e9: { type:"set", label:"酒旅", role:"subject" }

  // 上下文
  e10: { type:"constraint", label:"大屏展示", role:"context" }
  e11: { type:"constraint", label:"CEO受众", role:"context" }

语义关系:
  r1: { type:"part_of", from:"e7", to:"e1", label:"外卖是营收的一部分" }
  r2: { type:"part_of", from:"e8", to:"e1", label:"到店是营收的一部分" }
  r3: { type:"part_of", from:"e9", to:"e1", label:"酒旅是营收的一部分" }
  r4: { type:"sequence", from:"e5", to:"e5", label:"4月→5月→6月" }
```

**数据层（语义→数据映射后）：**

```
数据项:
  // 指标值
  d1: { entity:"revenue", attribute:"q2_value", value:1234567, unit:"元" }
  d2: { entity:"orders", attribute:"q2_value", value:89012, unit:"单" }
  d3: { entity:"aov", attribute:"q2_value", value:139, unit:"元" }

  // 月度趋势
  d4: { entity:"revenue", attribute:"month", value:3800000, metadata:{month:"4月"} }
  d5: { entity:"revenue", attribute:"month", value:4200000, metadata:{month:"5月"} }
  d6: { entity:"revenue", attribute:"month", value:4350000, metadata:{month:"6月"} }

  // 构成
  d7: { entity:"waimai", attribute:"revenue_share", value:45, unit:"%" }
  d8: { entity:"daodian", attribute:"revenue_share", value:25, unit:"%" }
  d9: { entity:"jiulv", attribute:"revenue_share", value:20, unit:"%" }
  d10: { entity:"other", attribute:"revenue_share", value:10, unit:"%" }
```

**视觉层（数据→视觉映射后）：**

```
// 指标区 — 3个 KPI 卡片
Mark组A (d1,d2,d3):
  mapping: text ← value, shape = "text"
  relation: Alignment (横向对齐)
  boundary: 3个矩形卡片框
  → "¥1,234,567" | "89,012" | "¥139"

// 趋势区 — 折线图
Mark组B (d4,d5,d6):
  mapping: position.x ← month(ordinal), position.y ← value(linear), shape = "circle"
  relation: Sequence (按month) + Connection (相邻连线)
  boundary: 矩形图表框
  → 三点折线

// 构成区 — 环形图
Mark组C (d7,d8,d9,d10):
  mapping: angle ← value, color ← entityLabel, shape = "bar"(扇形)
  relation: Containment (都属于整体) + Alignment (环绕对齐)
  boundary: 圆形（整体边界）
  → 四扇环形图

// 布局
Boundary 嵌套:
  外层: 矩形（整个仪表盘）
    ├── 矩形（指标区，顶部，3列）
    ├── 矩形（趋势区，左下，span 2）
    └── 矩形（构成区，右下，span 1）
```

### 6.2 同一语义的不同视觉映射

上面的语义层和数据层不变，如果用户改说"发到群里"（移动端），视觉映射变化：

```
// 指标区 — 纵向卡片（移动端单列）
Mark组A: boundary 改为纵向 Stack，1列

// 趋势区 — 简化为迷你图（Sparkline）
Mark组B: shape = "circle" 缩小，去掉坐标轴

// 构成区 — 改为水平堆叠条（比环形图更适合窄屏）
Mark组C: mapping 改为 position.x ← value(堆叠), shape = "bar"(水平)

→ 语义不变、数据不变，只有 ChannelMapping 变了 → 适配移动端
```

---

## 七、三层在 Agent 工作流中的位置

```
用户需求
  ↓
Step 1: 意图解析（INTERMEDIATE-LAYER.md 已有）
  → Intent: { what, who, where, why, dataShape, constraints }
  ↓
Step 2: 语义建模 ← NEW
  → SemanticElement[] + SemanticRelation[]
  → "要表达什么含义"
  ↓
Step 3: 数据实例化 ← NEW
  → DataItem[] + DataRelation[]
  → "具体有哪些数据"
  ↓
Step 4: 视觉映射（GRAMMAR.md 已有，增强）
  → ChannelMapping → Mark[] + Relation[] + Boundary[]
  → "怎么呈现"
  ↓
Step 5: 组合与布局（COMPOSITION.md 已有）
  → CompositionPlan → 空间分配 + 主辅层级
  ↓
Step 6: 渲染
  → Canvas Schema → React 渲染引擎
  ↓
Step 7: 视觉反馈（已有）
  → 截图 → 多模态审查 → 修正
```

### 7.1 两层反馈增强

有了三层架构，视觉反馈也可以分三层：

```
语义层审查: "有没有遗漏用户要表达的含义？"
  → 检查语义元素是否覆盖了意图中的所有 what

数据层审查: "数据对不对？全不全？"
  → 检查数据项是否完整、值是否合理

视觉层审查: "渲染出来好不好看？对不对？"
  → 检查布局、对齐、重叠、审美
```

---

## 八、三层架构 vs 原架构

| 维度 | 原架构 | 新三层架构 |
|------|--------|-----------|
| 层数 | 2层（意图→Mark） | 4层（意图→语义→数据→视觉） |
| 拆解阶段的产物 | 十种信息元素 | 语义元素（8种）→ 数据项 |
| 渲染决策时机 | 拆解阶段就绑定组件 | 到视觉映射阶段才决定 |
| 同一需求多场景 | 需重新拆解 | 语义/数据不变，只改映射 |
| Agent 认知负荷 | 高（混合三层） | 低（每层只做一件事） |
| 反馈层级 | 1层（渲染后） | 3层（语义/数据/视觉） |

---

## 九、语义层八种元素 vs 原十种信息元素

| 原信息元素 | 语义层对应 | 变化说明 |
|-----------|-----------|---------|
| DataPoint | entity + attribute + measure | 拆分为"什么实体、什么属性、什么值" |
| DataSeries | entity + attribute(ordered) + measure[] | 序列性从 constraints.ordered 表达 |
| Comparison | entity[] + attribute[] + compare关系 | 对比是关系，不是元素类型 |
| Composition | entity + set[] + part_of关系 | 构成是关系，不是元素类型 |
| Relation | entity[] + relation(各种类型) | 关系统一为语义关系 |
| Process | event[] + flow_to关系 | 过程是事件的有序流转 |
| Hierarchy | entity[] + belong_to关系 | 层级是归属关系的特例 |
| Distribution | entity[] + attribute(continuous) | 分布不需要独立元素类型 |
| Narrative | narrative元素 | 保持不变 |
| Decision | entity[] + attribute[] + measure[] + narrative | 决策是多种元素的组合 |

**关键变化**：DataSeries、Distribution、Comparison、Composition、Hierarchy 不再是独立的元素类型——它们的区别在语义关系（compare/part_of/flow_to/belong_to）和约束（ordered/hierarchical），不在元素本身。元素只保留 8 种纯粹的语义类型。

---

## 十、学术验证：三层架构的合理性

### 10.1 与经典可视化分层模型的对应

三层划分并非凭空设计，与信息可视化领域的经典分层模型有明确对应关系：

| 我们的层 | Ed Chi Data State Model (2000) | Card 参考模型 (1999) | Vega-Lite (2017) | Grammar of Graphics (2005) |
|---------|-------------------------------|---------------------|-----------------|---------------------------|
| **Semantic Layer** | Analytical Abstraction | （隐含，未显式定义） | （缺失） | Variables + Algebra |
| **Data Layer** | Value | Data Table | Data + Transform | Data |
| **Visual Grammar Layer** | Visualization Abstraction → View | Visual Structures → Views | Mark + Encoding + Scale + Layout | Geometry + Aesthetics + Coordinates |

**关键发现**：学术界的经典模型都有一个"数据到视觉"之间的抽象层，但大多被压缩或忽略了。Ed Chi 的 Analytical Abstraction 对应我们的 Semantic Layer，但他在论文中指出这一层最容易被跳过、也最容易出现理解错误。Card 的模型从 Data Table 直接跳到 Visual Structures，中间的语义转换是隐含的。Vega-Lite 和 GoG 也都缺少显式的语义层。

**我们的贡献不在于发明了三层，而在于把学术界隐含的语义层显式化了**。这对人类可视化工具不是必须的（人类在脑中完成语义→数据的转换），但对 Agent 来说是必要的——Agent 没有人类的隐性推理能力，每一步都需要显式表示。

### 10.2 参考文献

- **Chi, E. H.** "A Taxonomy of Visualization Techniques Using the Data State Reference Model" (IEEE InfoVis 2000) — [Paper](https://ics.uci.edu/~kobsa/courses/ICS280/InfoViz2000/ed-chi.pdf)
- **Card, S. K., Mackinlay, J. D. & Shneiderman, B.** *Readings in Information Visualization: Using Vision to Think* (Morgan Kaufmann, 1999) — [Book](https://www.cs.umd.edu/~ben/Card-Mackinlay-Shneiderman-Readings%20in%20Information%20Visualization-1999-v2.pdf)
- **Wilkinson, L.** *The Grammar of Graphics* (Springer, 2005) — [Book](https://link.springer.com/book/10.1007/0-387-28695-0)
- **Satyanarayan, A. et al.** "Vega-Lite: A Grammar of Interactive Graphics" (IEEE InfoVis 2017) — [Paper](https://idl.cs.washington.edu/files/2017-VegaLite-InfoVis.pdf)

---

## 十一、利弊分析：灵活性 vs 复杂性

### 11.1 增加灵活性的方面

**同一语义/数据可适配多场景**。以"Q2 经营数据"为例，语义层和数据层只构建一次，投大屏用 Grid+大字体映射，发群里用 Stack+迷你图映射，生成报告用 Report+表格映射。原框架需要为每个场景重新拆解元素。301 个 case 中约 18% 涉及多场景适配，这些 case 直接受益。

**反馈可以分层捕获错误**。语义层审查"有没有遗漏用户要表达的含义"（比如用户说了"还要展示同比"但语义层没有同比的 entity），数据层审查"数据对不对"（比如值提取错误），视觉层审查"渲染好不好"。原框架只有视觉层审查，发现错误时已经不知道是理解错了还是渲染错了。

**渲染决策延迟到最后一步**。Agent 在拆解阶段只提取语义和数据，不做渲染决策。到视觉映射阶段才决定"柱状图还是散点图"。这降低了早期错误的风险——如果映射选错了，只需改映射，不用重新理解需求。

### 11.2 增加复杂性的方面

**Agent 步骤增多**。原框架 3 步（意图→元素→Schema），新框架 5 步（意图→语义→数据→视觉映射→Schema）。每一步都有引入错误的风险。对于简单场景（"显示一个数字 380 万"），三层架构是过度设计。

**Token 消耗增加**。Agent 需要输出语义层 JSON + 数据层 JSON + 视觉层 JSON，比直接输出 Canvas Schema 多约 2-3 倍 token。对于高频简单场景，这是显著的额外成本。

### 11.3 Case 验证：三个代表性场景

**Case 1 — 简单场景："显示总营收 1234 万"**

```
原框架:  DataPoint("总营收","1234万") → StatCard → 渲染
         1步，Agent 只需 1 个 JSON 对象

三层:    Semantic: entity("营收"), attribute("值"), measure(1234万)
         Data: {entity:"revenue", attribute:"value", value:12340000}
         Visual: Mark(text, value) + Boundary(卡片)
         3步，Agent 需要输出 3 个 JSON 对象

结论: 三层是过度设计。简单场景用原框架更好。
```

**Case 2 — 中等场景："Q2 仪表盘，含 3 个 KPI + 趋势图 + 构成饼图，给 CEO 看大屏"**

```
原框架:  拆解为 DataPoint×3 + DataSeries + Composition
         → 选组件 → 布局 → Schema
         如果改成"发到群里"，需要重新拆解和选组件

三层:    Semantic: entity×5 + attribute×3 + relation(part_of) + context(CEO,大屏)
         Data: 10个数据项 + 数据关系
         视觉映射A(大屏): Mark(text,大字) + Mark(circle+line,趋势) + Mark(bar,饼) + Grid布局
         视觉映射B(群里): 同样的语义+数据 → Mark(text,小字) + Mark(circle,迷你) + Stack布局
         语义和数据层不变，只改映射

结论: 三层明显优于原框架。多场景适配和反馈分层有价值。
```

**Case 3 — 复杂场景："对比推荐系统新旧策略效果，展示 A/B 实验的 CTR/收入/留存对比，标出显著差异"**

```
原框架:  Comparison(CTR) + Comparison(收入) + Comparison(留存) +
         叙事(结论) + 跨元素关联(同一实验)
         → 元素类型多，关系复杂，Agent 容易混淆

三层:    Semantic: entity(新策略,旧策略) × attribute(CTR,收入,留存)
              + relation(compare) × 3 + narrative(结论)
         Data: 6个数据项 + 显著性数据 + 差异值
         视觉映射: Mark(bar,对比) × 6 + Relation(alignment) +
                   Boundary(分组) + 标注(显著性)
         → 语义层清晰识别"2个实体×3个维度的对比关系"
         → 数据层正确展开为6个数据项
         → 视觉层可以选择表格/雷达/柱状，灵活

结论: 三层显著优于原框架。语义层防止了理解错误，数据层确保了数据完整性。
```

### 11.4 净效果判断

**净效果是提高的**。301 个 case 中约 70% 是中等以上复杂度，而最容易出现 Agent 可视化效果差的场景恰恰是复杂场景——简单场景 Agent 本来就做得不错。三层架构正是针对"做不好"的场景提供了改善。

**建议：分级使用**。对于简单场景（单指标、单图表、约 30% 的 case），提供"快速路径"——Agent 可以跳过语义层，直接从意图到数据到视觉。对于中等和复杂场景（多指标、多意图组合、多场景适配、约 70% 的 case），使用完整的三层管线。
