# Agent Canvas — 视觉语法：从组件分类到信息本质

> 重新审视我们的元素抽象。当前十种信息元素太偏向前端组件分类，不够触及信息表达的本质。本文档定义更底层、更本源的视觉语法体系，并对照 301 个 case 做覆盖度评估。

---

## 一、当前十种元素的本质问题

### 1.1 问题诊断

当前的十种信息元素（DataPoint / DataSeries / Comparison / Composition / Relation / Process / Hierarchy / Distribution / Narrative / Decision）存在三个根本性问题：

**问题一：分类标准不统一**

有的按数据结构分（DataSeries 是序列数据，Distribution 是散布数据），有的按信息意图分（Comparison 是对比意图，Narrative 是叙事意图），有的按关系结构分（Relation 是关联，Hierarchy 是层级）。三个分类维度混在一起，导致元素之间有大量重叠和模糊地带。

**问题二：与渲染组件过度绑定**

DataPoint → StatCard，DataSeries → Chart(line)，Comparison → ComparisonTable——每个元素都预设了一种渲染形态。但同一个"月营收 380 万"的信息，在仪表盘里可能是一个数字卡片，在趋势图里是一个柱子的高度，在表格里是一个单元格的颜色深浅。**元素的类型不应该由它的渲染形态决定**。

**问题三：无法表达"同一数据、多种呈现"**

你举的例子：Distribution 可能就是一组点，DataSeries 也可能是一组点。它们在视觉上可能是完全相同的散点图，区别仅在于"点之间有没有顺序关系"。但当前框架把它们定义为两种不同的元素，导致 Agent 在拆解时就要做渲染决策，而这应该是后面才做的事。

### 1.2 例子：为什么 Distribution 和 DataSeries 重叠

```
DataSeries 定义：按某个维度排列的一组数据点
Distribution 定义：数据在一个或多个维度上的散布

问题：一组按时间排列的 (x, y) 数据点
  - 如果 x 是时间 → DataSeries（时间序列）
  - 如果 x 是另一个连续变量 → Distribution（散点分布）
  - 如果 x 是类别 → 既不是 DataSeries 也不是 Distribution？

本质上，这就是「一组有位置的标记」。标记之间是否有顺序、
是否用线连接、是否对齐到某个轴——这些都是「关系」和「映射」的属性，
不应该在元素定义阶段就固化。
```

---

## 二、重构：四种视觉语法原语

从信息表达的本质出发，所有可视化都可以分解为四种原语的组合：

### 原语一：Mark（标记）— 视觉表达的最小单元

一个 Mark 是视觉空间中的一个原子单位。它有**位置**和若干**可映射的视觉通道**。

```
Mark = {
  position: (x, y)              // 在坐标系中的位置
  channels: {                   // 可映射的视觉通道
    size?: number               // 大小（柱长/圆半径/面积）
    color?: string              // 颜色（色相/饱和度/明度）
    shape?: "circle" | "bar" | "square" | "triangle" | "text" | ...
    orientation?: number        // 方向角度
    texture?: string            // 纹理/图案
    opacity?: number            // 透明度
  }
  label?: string                // 文字标签
  value?: any                   // 原始数据值
  metadata?: Record<string, any>  // 元数据（实体ID等，供关联层引用）
}
```

**关键洞察：Mark 不预设渲染形态**。

同一个数据点"月营收 380 万"，它的 Mark 是：
```
{
  value: 3800000,
  label: "4月营收",
  metadata: { period: "2025-04" }
}
```

渲染时，根据意图和上下文：
- 需要精确读数 → shape: "text"，显示 "¥380万"
- 需要比较大小 → shape: "bar"，size 映射 value
- 需要看趋势 → shape: "circle"，position.x = 时间，与其他 Mark 用线连接
- 需要在矩阵中高亮 → shape: "square"，color 深浅映射 value

**Mark 的形态由 Channel Mapping（通道映射）决定，不由 Mark 自身的类型决定**。

### 原语二：Relation（关系）— 标记之间的连接

关系描述 Mark 和 Mark 之间怎么连接。关系有五种基本类型：

```
Relation = {
  type: RelationType
  from: MarkRef          // 起始标记
  to: MarkRef            // 目标标记（或多个）
  label?: string         // 关系标签
  weight?: number        // 权重/强度（映射到线宽）
}
```

**R1. Connection（连接）— "有关联"**

用线连接两个 Mark。无方向。

```
A ──── B    "A 和 B 有关系"
```

- 线型：实线（强关联）、虚线（弱关联）、点线（潜在关联）
- 线宽：可映射数据（如关联强度、流量大小）
- 示例：服务依赖图中的两个服务、社交网络中的两个用户

**R2. Flow（流转）— "流向"**

有方向的连接，用箭头表示。A 流向 B。

```
A ──→ B    "A 流向 B"
```

- 变体：粗箭头（大流量）、细箭头（小流量）、弯曲带（桑基流）
- 示例：数据管道中的任务流转、用户从一个页面到另一个页面、审批流程

**R3. Containment（包含）— "属于"**

一个 Mark 包含另一个 Mark。用空间包含（包裹）或箭头（归属）表达。

```
┌─────────────┐
│ A           │    "B 属于 A"
│  ┌───────┐  │
│  │  B    │  │
│  └───────┘  │
└─────────────┘

或：A ──▶ B  "B 归属于 A"（用空心箭头）
```

- 变体：嵌套（矩形包含矩形）、树形（父节点连子节点）、包围（圆/多边形包围）
- 示例：组织架构（部门包含团队）、BOM 树（产品包含零件）、文件目录

**R4. Sequence（序列）— "有序"**

多个 Mark 按某种维度有序排列。不画线，但隐含顺序。

```
A → B → C → D    "按时间/优先级/步骤排序"
```

- 变体：线性序列（时间线）、环形序列（周期）、层级序列（优先级递减）
- 关键：序列关系不画显式连线，而是通过 Mark 的 position 编码顺序
- 示例：时间轴上的事件、流程的步骤、排行榜的排名

**R5. Alignment（对齐）— "对应"**

多个 Mark 在某个维度上共享位置，暗示它们是对应的。

```
  A1  A2  A3
  │   │   │      "同一时间点的不同指标"
  B1  B2  B3
```

- 变体：横向对齐（同列）、纵向对齐（同行）、网格对齐（同行列）
- 示例：表格中同一行的数据属于同一条记录、多图中同一 x 位置对应同一时间

### 原语三：Boundary（边界）— 视觉圈定

边界用视觉框线圈定一组 Mark，表达"这些是一个组"。

```
Boundary = {
  shape: "rect" | "circle" | "ellipse" | "freeform" | "none"
  marks: MarkRef[]           // 被圈定的标记
  label?: string             // 组标签
  style?: "solid" | "dashed" | "dotted" | "filled" | "shadow"
}
```

**五种边界形态**：

```
矩形边界（结构化分组）        圆形边界（有机分组）
┌─────────────────┐          ╭───────────╮
│  •    •    •    │          │  •    •   │
│       •    •    │          │     •  •  │
│  •              │          │  •        │
└─────────────────┘          ╰───────────╯
"指标卡组"                    "异常用户群"

阴影/填充边界（语义区域）      无边界（邻近分组）
┌░░░░░░░░░░░░░░░┐              •  •  •
│░░ •    •    •░│
│░░      •    •░│             •  •
│░░ •          ░│
└░░░░░░░░░░░░░░░┘              •  •  •
"高亮区域"                    "靠在一起就是一组"
```

**边界的作用**：
- 分组：圈定同类 Mark（如一个卡片内的所有指标）
- 分区：划分页面的语义区域（如仪表盘的"指标区"和"图表区"）
- 强调：用颜色/阴影突出某个区域
- 层级：嵌套边界表达层级关系（大框里套小框）

### 原语四：ChannelMapping（通道映射）— 数据到视觉的映射

定义数据字段如何映射到 Mark 的视觉通道。这是连接"数据语义"和"视觉呈现"的桥梁。

```
ChannelMapping = {
  position: {
    x?: { field: string, scale: "linear" | "ordinal" | "time" | "log" }
    y?: { field: string, scale: "linear" | "ordinal" | "time" | "log" }
  }
  size?:   { field: string, scale: "linear" | "sqrt" }   // 柱长/圆半径/面积
  color?:  { field: string, scale: "categorical" | "sequential" | "diverging" }
  shape?:  { field: string, scale: "categorical" }
  opacity?: { field: string, scale: "linear" }
  text?:   { field: string }   // 显示为文字标签
}
```

**映射决定渲染形态**：

同一个 Mark 集合，不同的 ChannelMapping 产生完全不同的可视化：

```
数据：[{month:"4月", revenue:380}, {month:"5月", revenue:420}, {month:"6月", revenue:435}]

映射A（柱状图）:
  position.x ← month (ordinal)
  size ← revenue (linear)        // 柱高度
  shape = "bar"
  → 📊 三根柱子

映射B（折线图）:
  position.x ← month (ordinal)
  position.y ← revenue (linear)
  shape = "circle"
  + Relation: Sequence (按 month 排序) + Connection (相邻点连线)
  → 📈 三个点用线连起来

映射C（数字卡片）:
  text ← revenue
  shape = "text"
  → 📋 三个数字 380 / 420 / 435

映射D（热力图）:
  position.x ← month (ordinal)
  color ← revenue (sequential)   // 颜色深浅
  shape = "square"
  → 🟥 三个色块，深浅不同
```

**这就是你说的"同一个元素，根据是否需要展示高度来判断是柱形还是单点"**。Mark 不变，映射变，渲染结果就变。

---

## 三、四种原语如何组合出所有可视化

### 3.1 从原语到传统图表类型

所有传统图表类型都是 Mark + Relation + Boundary + ChannelMapping 的特定组合：

| 传统图表 | Mark 形态 | Relation | Boundary | ChannelMapping |
|---------|----------|----------|----------|---------------|
| 柱状图 | bar | — | 可选(分组框) | x=类别, size=值 |
| 折线图 | circle | Connection(相邻连线) + Sequence(时间) | — | x=时间, y=值 |
| 饼图 | bar(扇形) | Containment(都属于整体) | 圆形(整体边界) | angle=占比 |
| 散点图 | circle | — | — | x=变量1, y=变量2, size=可选, color=可选 |
| 气泡图 | circle | — | — | x=变量1, y=变量2, size=变量3, color=变量4 |
| 热力图 | square | Alignment(网格对齐) | 矩形(矩阵边界) | x=类别1, y=类别2, color=值 |
| 堆叠柱 | bar | Containment(部分属于整体) + Alignment(同列对齐) | 矩形(柱边界) | x=类别, y=值分段, color=子类别 |
| 桑基图 | bar(流带) | Flow(有向流) | — | x=阶段, 流宽=量 |
| 漏斗图 | bar(梯形) | Sequence(阶段顺序) + Flow(转化) | 梯形(漏斗边界) | y=阶段, size=量 |
| 雷达图 | circle + polygon | Alignment(同维度对齐) | 多边形(雷达边界) | angle=维度, radius=值 |
| 时间线 | circle/text | Sequence(时间) | 可选 | x=时间 |
| 流程图 | rect(节点) | Flow(有向箭头) | 可选(泳道) | — |
| 网络图 | circle(节点) | Connection(边) | 可选(社区圈) | 力导向布局 |
| 组织架构 | rect(节点) | Containment(归属) | — | 层级布局 |
| 思维导图 | text(节点) | Containment(分支) | — | 径向布局 |
| 看板 | rect(卡片) | Containment(列包含卡片) | 矩形(列边界) | y=状态列 |
| 甘特图 | bar(任务条) | Sequence(时间) + Containment(资源包含任务) | 矩形(行边界) | x=时间, y=任务, size=时长 |
| 表格 | text(单元格) | Alignment(行列对齐) | 矩形(表格边界) | 行=记录, 列=字段 |
| KPI卡片 | text(数字) | Containment(属于卡片) | 矩形(卡片边界) | text=值 |
| 地图 | circle/square(区域) | Containment(区域包含点) | freeform(地理边界) | position=经纬度 |
| 决策树 | rect(节点) | Flow(分支箭头) | — | 层级布局 |
| K线 | bar(蜡烛体) + line(影线) | Sequence(时间) | — | x=时间, y=价格区间 |

### 3.2 从原语到原十种信息元素

原十种信息元素都可以分解为四种原语的组合：

**DataPoint →** 单个 Mark + Boundary(卡片) + ChannelMapping(text 或 size)

**DataSeries →** 多个 Mark + Relation(Sequence + Connection) + ChannelMapping(position.x=时间, position.y=值)

**Comparison →** 多个 Mark + Relation(Alignment) + ChannelMapping(position=对象×维度, color=对象) + Boundary(对比框)

**Composition →** 多个 Mark + Relation(Containment) + Boundary(圆形/整体) + ChannelMapping(size 或 angle=占比)

**Relation →** 多个 Mark + Relation(Connection) + ChannelMapping(position=力导向/布局)

**Process →** 多个 Mark + Relation(Flow) + ChannelMapping(position=流程布局) + Boundary(可选泳道)

**Hierarchy →** 多个 Mark + Relation(Containment) + ChannelMapping(position=树布局)

**Distribution →** 多个 Mark + ChannelMapping(position.x, position.y) （无显式 Relation，或可选 Containment 表示聚类）

**Narrative →** text Mark + Relation(Sequence) + Boundary(段落/卡片)

**Decision →** 多个 Mark + Relation(Alignment + Containment) + ChannelMapping(position=选项×维度) + Boundary(矩阵框)

**关键发现**：分解后可以看出，DataSeries 和 Distribution 的唯一区别是——DataSeries 有 Sequence + Connection 关系，Distribution 没有。它们用到的 Mark 是完全一样的。**关系不同，不是元素不同**。

---

## 四、新的 SIR 结构

基于四种原语，SIR 从"元素树"重构为"标记图"：

```typescript
interface SIR {
  intent: Intent;

  // 标记集合（扁平，不预设层级）
  marks: Mark[];

  // 关系集合（标记之间的连接）
  relations: Relation[];

  // 边界集合（标记的视觉圈定）
  boundaries: Boundary[];

  // 通道映射（数据 → 视觉通道）
  mappings: ChannelMapping[];

  // 布局意图（指导渲染引擎如何排列）
  layout: LayoutIntent;

  // 跨元素关联（保留原有设计）
  linkings?: CrossElementLink[];
}

interface Mark {
  id: string;
  value?: any;                    // 原始数据值
  label?: string;                 // 文字标签
  metadata?: Record<string, any>; // 元数据（实体ID等）
  // 注意：Mark 不预设 shape/size/color
  // 这些由 ChannelMapping 决定
}

interface Relation {
  id: string;
  type: "connection" | "flow" | "containment" | "sequence" | "alignment";
  from: string[];   // Mark ID 列表
  to: string[];     // Mark ID 列表
  label?: string;
  weight?: number;
  direction?: "none" | "directed" | "bidirectional";
}

interface Boundary {
  id: string;
  shape: "rect" | "circle" | "ellipse" | "freeform" | "none";
  marks: string[];   // 被圈定的 Mark ID
  label?: string;
  style?: "solid" | "dashed" | "dotted" | "filled" | "shadow" | "none";
  nestLevel?: number;  // 嵌套层级（用于嵌套边界）
}

interface ChannelMapping {
  markSet: string[];   // 应用此映射的 Mark ID 集合
  position?: {
    x?: { field: string; scale: "linear" | "ordinal" | "time" | "log" };
    y?: { field: string; scale: "linear" | "ordinal" | "time" | "log" };
  };
  size?: { field: string; scale: "linear" | "sqrt" };
  color?: { field: string; scale: "categorical" | "sequential" | "diverging" };
  shape?: { field: string; scale: "categorical" };
  opacity?: { field: string; scale: "linear" };
  text?: { field: string };
}
```

### 4.1 示例：月度营收趋势

```
传统方式（原 SIR）:
  DataSeries("月度营收", [{month:"4月",revenue:380}, ...])

新方式（标记图）:
  marks: [
    { id:"m1", value:3800000, label:"4月", metadata:{month:"4月"} },
    { id:"m2", value:4200000, label:"5月", metadata:{month:"5月"} },
    { id:"m3", value:4350000, label:"6月", metadata:{month:"6月"} }
  ]
  relations: [
    { type:"sequence", from:["m1"], to:["m2","m3"], direction:"directed" },
    { type:"connection", from:["m1"], to:["m2"] },
    { type:"connection", from:["m2"], to:["m3"] }
  ]
  boundaries: []
  mappings: [{
    markSet: ["m1","m2","m3"],
    position: { x: {field:"month", scale:"ordinal"}, y: {field:"value", scale:"linear"} },
    shape: { field: null, scale: "categorical" },  // 固定为 circle
  }]

  → 渲染为折线图（有序的点 + 连线）

  如果改为 mappings.shape = "bar"，去掉 connection 关系:
  → 渲染为柱状图（柱子的高度 = value）

  如果改为 mappings.text = "value"，加 boundary:
  → 渲染为数字卡片组

  Mark 不变，关系和映射变，渲染结果完全不同。
```

### 4.2 示例：散点图中的用户分布 + 高价值用户圈定

```
marks: [
  { id:"u1", metadata:{spend:3200, freq:12, tier:"high"} },
  { id:"u2", metadata:{spend:5800, freq:8, tier:"high"} },
  { id:"u3", metadata:{spend:450, freq:2, tier:"low"} },
  ...
]
relations: []  // 散点图标记之间无显式关系
boundaries: [
  { shape:"circle", marks:["u1","u2",...], label:"高价值用户群", style:"dashed" }
]
mappings: [{
  markSet: ["u1","u2",...],
  position: { x:{field:"freq",scale:"linear"}, y:{field:"spend",scale:"linear"} },
  color: { field:"tier", scale:"categorical" },
  size: { field:"spend", scale:"sqrt" }
}]

→ 散点图，点的大小=消费额，颜色=层级，虚线圈出高价值用户
```

---

## 五、301 个 Case 的覆盖度评估

### 5.1 评估方法

将 301 个 case 逐个检查：能否用 Mark + Relation + Boundary + ChannelMapping 的组合来表达？

### 5.2 覆盖度总表

| 能力维度 | 可覆盖 case 数 | 占比 | 说明 |
|---------|--------------|------|------|
| 标准 2D 图表 | 162 | 54% | 柱/线/饼/散点/面积/雷达/热力/堆叠/气泡 |
| 结构关系图 | 48 | 16% | 流程图/网络图/ER/组织架构/思维导图/看板 |
| 表格/矩阵 | 32 | 11% | 对比表/评分矩阵/清单 |
| 文本/报告 | 28 | 9% | 报告/文档/卡片/Callout |
| 仪表盘/组合 | 22 | 7% | 多组件组合仪表盘（需 Boundary + 多组 Mark） |
| 地图/地理 | 5 | 2% | 需要 freeform Boundary + 地理坐标 |
| K线/蜡烛图 | 2 | 1% | 需要 composite Mark（一个 Mark 含 OHLC 四个值） |
| 3D/立体 | 2 | 1% | ❌ 无法覆盖 |
| **总计** | **301** | **99%** | |

### 5.3 可以覆盖的（299/301 = 99%）

**A. 标准 2D 图表（162 个）— 完全覆盖**

所有传统图表都是 Mark + ChannelMapping 的标准组合。包括：
- 柱状图、折线图、饼图、环形图、面积图
- 散点图、气泡图、热力图、直方图、箱线图
- 堆叠柱、分组柱、雷达图
- 漏斗图、瀑布图、子弹图、甘特图
- 迷你图（Sparkline）

每个 case 的 Mark 数量可能从 1 个（单 KPI）到数千个（散点图），但表达方式统一。

**B. 结构关系图（48 个）— 完全覆盖**

- 流程图（FlowChart）：Mark(节点) + Relation(Flow)
- 网络图（NetworkGraph）：Mark(节点) + Relation(Connection)
- ER 图：Mark(实体) + Relation(Connection) + Boundary(实体框)
- 组织架构图：Mark(节点) + Relation(Containment)
- 思维导图：Mark(节点) + Relation(Containment) + ChannelMapping(径向布局)
- 看板（Kanban）：Mark(卡片) + Relation(Containment) + Boundary(列框)
- 决策树：Mark(节点) + Relation(Flow 分支)
- DAG 依赖图：Mark(节点) + Relation(Flow 有向)
- 攻击链（Kill Chain）：Mark(阶段) + Relation(Flow)
- 时序图：Mark(lifeline) + Relation(Flow 消息)

**C. 表格/矩阵（32 个）— 完全覆盖**

- 表格：Mark(单元格) + Relation(Alignment 行列对齐) + Boundary(表格框)
- 对比矩阵：同上 + ChannelMapping(color=高亮差异)
- 评分矩阵：同上 + ChannelMapping(color=分数)
- 检查清单：Mark(项) + Relation(Sequence) + Boundary(列表框)

**D. 文本/报告（28 个）— 完全覆盖**

- 报告/文档：Mark(text) + Relation(Sequence) + Boundary(段落/章节)
- KPI 卡片：Mark(text/number) + Boundary(卡片框)
- Callout：Mark(text) + Boundary(提示框)
- Accordion：Mark(text) + Relation(Containment) + Boundary(折叠框)

**E. 仪表盘/组合（22 个）— 完全覆盖**

- Dashboard：多组 Mark + 多个 Boundary(分区) + Relation(Alignment 对齐)
- 大屏：同上 + ChannelMapping(大字体/高对比)
- Linked Views：多组 Mark + CrossElementLink

**F. 地图/地理（5 个）— 可覆盖但需扩展**

需要 freeform Boundary（地理边界）和地理坐标系（经纬度 → 屏幕坐标的投影）。Mark 的 position 映射到经纬度而非笛卡尔坐标。这是 ChannelMapping 的 scale 扩展，不是原语的扩展。

- 地图热力：Mark(square) + ChannelMapping(position=geo, color=值)
- 地图标注：Mark(circle) + ChannelMapping(position=geo)
- 路线图：Mark(点) + Relation(Flow) + ChannelMapping(position=geo)

**G. K线/蜡烛图（2 个）— 可覆盖但需 composite Mark**

K线一个 Mark 需要承载 OHLC 四个值，需要 composite Mark（一个 Mark 由多个子视觉元素组成）。这是 Mark 的扩展，不是原语的扩展。

### 5.4 不能覆盖的（2/301 = 1%）

**3D 可视化（2 个）**

- SC03 气候变化 3D 模拟 — 需要三维 Mark 和三维坐标系
- 建筑师方案展示 3D 渲染 — 需要 WebGL/Three.js 级别渲染

这两个 case 需要 3D 渲染管线（WebGL），超出当前 2D 视觉语法的范围。建议作为后续扩展方向，需要增加 3D Mark（有 z 坐标）和 3D Boundary。

### 5.5 需要扩展但可覆盖的边界情况

| 场景 | 需要的扩展 | 难度 |
|------|-----------|------|
| 地理地图 | ChannelMapping 增加 geo 坐标系 | 中 |
| K线图 | Mark 支持 composite（子元素组合） | 低 |
| 桑基图 | Relation(Flow) 支持 band 形态 | 低 |
| 实时流数据 | Mark 生命周期管理（创建/更新/过期） | 中 |
| 混淆矩阵 | 表格 + ChannelMapping(color=值) 的组合 | 低（已支持） |
| 技术雷达 | Boundary(圆环) + ChannelMapping(angle=分类, radius=状态) | 低 |
| 生存曲线 | DataSeries 变体 + 置信区间 Boundary | 低 |
| 树形图(Treemap) | Containment + ChannelMapping(size=面积) 的嵌套 | 中 |

---

## 六、新框架对 Agent 工作流的影响

### 6.1 工作流变化

```
原工作流:
  意图解析 → 元素拆解(选 DataPoint/DataSeries/...) → 组织 → 视觉映射(元素→组件)

新工作流:
  意图解析 → 标记提取(识别 Mark) → 关系识别(识别 Relation) → 
  映射决策(选 ChannelMapping) → 布局组织(Boundary + 组合策略) → 渲染
```

**关键变化**：Agent 不再在拆解阶段就决定"这是 DataSeries 还是 Distribution"——它只提取 Mark 和它们之间的关系。渲染形态（柱状图还是散点图）由 ChannelMapping 决定，这是一个更下游的、可以基于意图和约束自动推导的决策。

### 6.2 Agent 推理示例

用户说："展示过去 3 个月每月的营收和订单量趋势。"

**Step 1: 标记提取**
```
6 个 Mark:
  m1: {value:3800000, metadata:{month:"4月", metric:"revenue"}}
  m2: {value:4200000, metadata:{month:"5月", metric:"revenue"}}
  m3: {value:4350000, metadata:{month:"6月", metric:"revenue"}}
  m4: {value:12000, metadata:{month:"4月", metric:"orders"}}
  m5: {value:13500, metadata:{month:"5月", metric:"orders"}}
  m6: {value:14200, metadata:{month:"6月", metric:"orders"}}
```

**Step 2: 关系识别**
```
Sequence: m1→m2→m3（revenue 按月有序）
Sequence: m4→m5→m6（orders 按月有序）
Alignment: m1↔m4（同月），m2↔m5，m3↔m6
```

**Step 3: 映射决策**（基于意图：trend + comparison）
```
两组数据量纲不同（百万 vs 万），用双轴：
  Mapping A (revenue): position.x=month, position.y=value(左轴), shape=circle
  Mapping B (orders): position.x=month, position.y=value(右轴), shape=circle
  + Relation: Connection（相邻同指标点连线）

  → 渲染为双轴折线图

或者如果意图是"对比两个指标的趋势幅度"（归一化）：
  Mapping: position.x=month, position.y=value(normalized), color=metric
  → 渲染为分组折线图
```

**Step 4: 布局组织**
```
Boundary: 矩形框（标题"营收与订单量趋势"）
Layout: 单图，全宽
```

### 6.3 为什么这比原框架更好

1. **延迟渲染决策**：Agent 不在拆解阶段就决定图表类型，而是到映射阶段才决定。降低了早期错误的风险。

2. **消除重叠**：DataSeries 和 Distribution 不再是两种元素，而是"同一组 Mark + 不同 Relation"。

3. **支持即兴组合**：用户想"在散点图上加一条趋势线"——就是给一组 Mark(无 Relation) 加上一条 Relation(Sequence + Connection) 的回归线。原框架需要同时创建 Distribution 和 DataSeries 两个元素，新框架只操作 Relation。

4. **渲染灵活性**：同一个 Mark 集合可以通过改变 ChannelMapping 在柱状图、折线图、热力图之间切换，不需要重新拆解。

---

## 七、对现有文档的影响

### 7.1 INTERMEDIATE-LAYER.md 需要重构

- 十种信息元素 → 四种原语（Mark / Relation / Boundary / ChannelMapping）
- SIR 结构从"元素树"改为"标记图"
- 元素拆解步骤改为：标记提取 → 关系识别 → 映射决策
- 视觉映射步骤改为：ChannelMapping + 布局组织

### 7.2 COMPOSITION.md 需要更新

- 组合的对象从"信息元素"变为"Mark 集合"
- 主辅关系从"元素级别"下沉到"MarkSet 级别"
- 嵌套策略变为"Boundary 嵌套"

### 7.3 DESIGN.md 需要更新

- Canvas DSL 从"组件树"增加"标记图"层
- 渲染引擎需要支持 ChannelMapping → 视觉属性的自动应用
- 组件库从"预设组件"变为"Mark 渲染器 + Relation 渲染器 + Boundary 渲染器"

### 7.4 LINKING.md 需要更新

- 跨元素关联的"元素"变为"Mark"
- LinkEndpoint.selector 从"元素内选择器"变为"Mark ID"

---

## 八、与 Grammar of Graphics 的关系

本框架受 Leland Wilkinson 的 [Grammar of Graphics](https://link.springer.com/book/10.1007/0-387-28695-0) (2005) 启发，但有关键区别：

| 维度 | Grammar of Graphics | Agent Canvas 视觉语法 |
|------|--------------------|--------------------|
| 核心对象 | Mark + Aesthetic | Mark + Relation + Boundary + ChannelMapping |
| 关系表达 | 无（仅靠位置/对齐隐含） | 显式的 Relation 类型（Connection/Flow/Containment/Sequence/Alignment） |
| 边界表达 | 无 | 显式的 Boundary 原语 |
| 适用范围 | 图表（统计可视化） | 图表 + 关系图 + 流程图 + 报告 + 仪表盘 |
| Agent 友好性 | 低（需要理解统计概念） | 高（原语直观，映射可推导） |

关键区别是**我们增加了 Relation 和 Boundary 两种原语**。Grammar of Graphics 主要解决统计图表（柱/线/饼/散点）的问题，但无法表达流程图、组织架构、看板等"非统计"可视化。Relation（标记之间的显式连接关系）和 Boundary（视觉圈定）这两种原语填补了这个空白。
