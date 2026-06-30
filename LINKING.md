# Agent Canvas — 跨元素关联层设计：Cross-Element Linking

> 当两个独立视图中的数据实体存在语义对应关系时，如何表达和视觉化这种"跨视图关联"？
> 本文是对 SIR（语义中间表示）和组合策略的补充设计，解决当前框架无法表达跨视图关系的缺口。

---

## 一、问题：当前框架的缺口

### 1.1 场景

用户说："这是两批用户的消费分布散点图，左边是对照组，右边是实验组。我想标出两个图里同一批高价值用户在两组中的位置变化。"

当前框架的处理：
- 拆解为两个 Distribution 元素 → 并列排列（Juxtaposition）
- 两个元素各自独立，互不感知
- 无法表达"图A的点P1 对应 图B的点P2"这种跨视图关系

### 1.2 这不是冷门需求

从 301 个 case 中可以找到大量需要跨视图关联的真实场景：

| Case | 视图 A | 视图 B | 跨视图关联 |
|------|--------|--------|-----------|
| NET_AI04 推荐漏斗 | 召回阶段物品集 | 精排阶段物品集 | 同一批物品在四个阶段的对应 |
| D03 用户行为路径 | 页面 A 行为散点 | 页面 B 行为散点 | 同一用户的连续行为 |
| NET_DATA03 数据血缘 | 上游表字段视图 | 下游表字段视图 | 字段级映射关系 |
| B07 用户分层 | 消费频次×客单价 | RFM 雷达图 | 同一批用户在两个模型中的位置 |
| T03 API 时序 | 服务 A 的 lifeline | 服务 B 的 lifeline | 服务间消息传递 |
| B09 渠道归因 | 渠道入口视图 | 转化出口视图 | 用户从入口到出口的路径 |
| H02 临床试验 | 实验组生存曲线 | 对照组生存曲线 | 同一指标在两组间的差异标注 |
| MED01 门诊流量 | 内科时段流量 | 外科时段流量 | 相同时段的横向对比连线 |

### 1.3 学术基础

这个问题在可视化领域有成熟的理论框架：

- **Coordinated Multiple Views (CMV)** — 多视图协调，一个视图的操作影响其他视图（Roberts, 2007）
- **Brushing and Linking** — 刷选联动，在一个视图中选中数据，其他视图中对应数据高亮（Becker & Cleveland, 1987）
- **Shared Data Reference** — 多视图共享同一数据源的不同投影，通过 entity ID 建立对应（Boukhelifa et al., 2003）
- **Visual Linking** — 跨视图视觉连线，用线/带/箭头连接不同视图中的对应元素（Collins & Carpendale, 2007）

---

## 二、解决方案：跨元素关联层

### 2.1 设计思路

在 SIR 的元素树之上增加一个 **Overlay 层**——它不属于任何一个信息元素，而是描述元素之间的跨视图关系。

```
SIR 结构:
  root (SIRNode 树)
  ├── E1: Distribution("对照组消费分布")
  ├── E2: Distribution("实验组消费分布")
  └── linkings: [  ← NEW: 跨元素关联层
      {
        from: E1.point["user_001"],
        to: E2.point["user_001"],
        type: "entity_match"
      }
    ]
```

关联层是一个"覆盖层"，它不改变底层元素的独立性和内部结构，只是在它们之间建立显式的语义连接。

### 2.2 关联描述

```typescript
// 跨元素关联
interface CrossElementLink {
  id: string;

  // 关联类型
  type: LinkType;

  // 关联端点（可以有多个，不限于两端）
  endpoints: LinkEndpoint[];

  // 关联语义
  label?: string;           // 关联标签（如"同一用户"、"字段映射"）
  strength?: "strong" | "medium" | "weak";  // 关联强度
  direction?: "none" | "directed" | "bidirectional";  // 方向性
}

// 关联类型
type LinkType =
  | "entity_match"     // 实体匹配：同一实体在不同视图中的对应
  | "entity_flow"      // 实体流转：实体从视图A"流"到视图B
  | "field_mapping"    // 字段映射：数据字段间的对应关系
  | "comparison_link"  // 对比连线：同类指标在不同视图中的对比
  | "correspondence"   // 通用对应：时间对应、位置对应等
  | "causal"           // 因果关联：A中的事件导致B中的结果
  | "aggregation";     // 聚合关系：A中的多个实体聚合为B中的一个

// 关联端点
interface LinkEndpoint {
  elementRef: string;          // 引用 SIR 中的元素 ID
  selector: string;            // 元素内的选择器（如 "point[user_001]"）
  anchor?: "center" | "edge" | "auto";  // 视觉锚点
}

// SIR 中的关联集合
interface SIR {
  root: SIRNode;
  intent: Intent;
  layout: LayoutIntent;
  linkings?: CrossElementLink[];   // ← NEW
}
```

### 2.3 六种关联类型详解

**类型一：entity_match（实体匹配）**
同一实体在不同视图中的出现。最常见的跨视图关联。

```
场景：两组散点图中的同一用户
{
  type: "entity_match",
  endpoints: [
    { elementRef: "E1", selector: "point[user_001]" },
    { elementRef: "E2", selector: "point[user_001]" }
  ],
  label: "同一用户"
}
```

视觉化：虚线连接两点，或同色高亮，或 hover 联动。

**类型二：entity_flow（实体流转）**
实体从视图 A 流到视图 B，有方向性。本质上是 Flow 意图的跨视图版本。

```
场景：用户从页面A的行为流到页面B
{
  type: "entity_flow",
  endpoints: [
    { elementRef: "E1", selector: "point[user_042]" },
    { elementRef: "E2", selector: "point[user_042]" }
  ],
  label: "用户042的页面跳转",
  direction: "directed"
}
```

视觉化：带箭头的连线或贝塞尔曲线，桑基带。

**类型三：field_mapping（字段映射）**
数据字段之间的对应关系。常见于数据血缘、ETL 管道。

```
场景：上游表和下游表的字段对应
{
  type: "field_mapping",
  endpoints: [
    { elementRef: "E_upstream", selector: "field[order_id]" },
    { elementRef: "E_downstream", selector: "field[oid]" }
  ],
  label: "order_id → oid (重命名)"
}
```

视觉化：连线 + 变换标注。

**类型四：comparison_link（对比连线）**
同类指标在不同视图中的对应，用于强调对比关系。

```
场景：实验组和对照组在同一时间点的指标对比
{
  type: "comparison_link",
  endpoints: [
    { elementRef: "E_control", selector: "point[t=7day]" },
    { elementRef: "E_experiment", selector: "point[t=7day]" }
  ],
  label: "7日留存差异: +5.2%"
}
```

视觉化：实线连接 + 差异标注。

**类型五：correspondence（通用对应）**
时间对应、空间对应等非实体级的对应关系。

```
场景：内科和外科在相同时段的流量对比
{
  type: "correspondence",
  endpoints: [
    { elementRef: "E_internal", selector: "bar[t=8:00-10:00]" },
    { elementRef: "E_surgery", selector: "bar[t=8:00-10:00]" }
  ],
  label: "早高峰时段对应"
}
```

视觉化：横向虚线或色带连接。

**类型六：causal（因果关联）**
一个视图中的事件导致另一个视图中的结果。

```
场景：代码提交（图A）触发了构建失败（图B）
{
  type: "causal",
  endpoints: [
    { elementRef: "E_commits", selector: "point[commit_a1b2c3]" },
    { elementRef: "E_failures", selector: "point[build_#4521]" }
  ],
  label: "此提交引入了空指针异常",
  direction: "directed"
}
```

视觉化：带箭头的因果链，颜色标注严重度。

### 2.4 关联的组织规则

当存在多条跨元素关联时，需要组织规则避免视觉混乱：

**规则一：聚合优先**
多个个体级的关联应尽量聚合为组级关联。

```
❌ 1000 个用户各画一条连线 → 视觉灾难
✅ 聚合为 "高价值用户群(50人) 的位置偏移" → 一条粗带或一个标注框
```

**规则二：分层表达**
关联按重要性分层，不同层级用不同视觉强度。

```
L1 关联（强）: 实线、粗线、高对比色 → 核心关系
L2 关联（中）: 虚线、中等粗细 → 支撑关系
L3 关联（弱）: 点线、细线、低对比色 → 辅助关系
或默认隐藏，hover/选中时显示
```

**规则三：密度控制**
单个页面上的可见跨视图关联不超过 15-20 条。超过时：
- 聚合为群体关联（桑基带、热力连线）
- 折叠为可展开的关联列表
- 只显示 L1 关联，其余按需展示

**规则四：避免交叉**
多条连线应尽量避免交叉。通过：
- 调整视图顺序（关联密集的视图相邻）
- 使用贝塞尔曲线而非直线
- 高亮当前关注关联，弱化其余

---

## 三、关联的视觉化方式

### 3.1 五种视觉化手段

| 手段 | 描述 | 适用场景 | 交互 |
|------|------|---------|------|
| **连接线** | 跨视图画线/曲线连接对应点 | 少量精确关联(1-15条) | hover 高亮 |
| **连接带** | 宽带连接，宽度编码流量 | 群体级流转关联 | 点击下钻 |
| **颜色联动** | 同一实体在不同视图中用相同颜色 | 实体匹配、多视图探索 | brush 刷选 |
| **标注框** | 用框/圈标出跨视图的对应区域 | 区域级对应 | 点击展开 |
| **幽灵标记** | 在视图B中显示视图A选中元素的"影子" | 联动探索 | hover 触发 |

### 3.2 连接线类型

```
实线 ───────  : 强关联（entity_match, comparison_link）
虚线 - - - -  : 弱关联（correspondence）
箭头 ───────→ : 有向关联（entity_flow, causal）
宽带 ███████  : 群体关联（aggregation, entity_flow 批量）
贝塞尔曲线 ⌒  : 远距离关联，避免直线交叉
```

### 3.3 视觉化决策树

```
关联数量是多少？
├── 1-5 条 → 连接线（实线/虚线/箭头），直接画
├── 5-15 条 → 连接线 + 分层（L1实线, L2虚线），hover 联动
├── 15-50 条 → 聚合为连接带 或 颜色联动（按群体着色）
└── 50+ 条 → 颜色联动 + brush 刷选，默认不画线
```

---

## 四、组合模式扩展

### 4.1 新增组合模式：Linked Views（关联并列）

在 COMPOSITION.md 的六种布局模式基础上，新增第七种：

**模式G：关联并列（Linked Juxtaposition）**
```
┌──────────────┐     ┌──────────────┐
│              │     │              │
│   Chart A    │ ──→ │   Chart B    │
│              │     │              │
│  • user_001  │ ──→ │  • user_001  │
│  • user_042  │ ──→ │  • user_042  │
│              │     │              │
└──────────────┘     └──────────────┘
       ↑                    ↑
    独立视图              独立视图
         跨视图关联（连线/颜色联动）
```

- 适用：两个视图各自独立，但数据实体存在对应关系
- 主辅关系：视图间可平等，也可有主辅
- 与普通 Juxtaposition 的区别：存在显式的跨视图关联层
- 示例：对照组 vs 实验组散点图、上游 vs 下游数据血缘

### 4.2 组合决策树更新

在 COMPOSITION.md 的决策树中增加关联判断分支：

```
Q3: 多种意图之间的关系是？
  ├── 同级别 → Grid / Small Multiples
  ├── 有明确主辅 → Hub-and-Spoke / Split Panel
  ├── 有阅读顺序 → Vertical Stack
  ├── 有层级关系 → Nesting
  └── 有跨视图实体关联 → NEW: Linked Views  ← 新增分支
      → 检查关联数量:
        ├── ≤15 条 → 连接线 + Linked Views
        ├── 15-50 条 → 连接带 / 颜色联动
        └── >50 条 → 颜色联动 + Brushing
```

---

## 五、端到端示例

### 5.1 示例：A/B 实验两组用户行为对比

**用户需求**："对比实验组和对照组的用户消费分布，标出高价值用户在两组中的位置变化。"

**Step 1: 意图解析**
```
what: [distribution, comparison, relationship]
who: analyst
where: dashboard
why: explore
dataShape: [spatial, relational]  ← 注意：有 relational，提示跨视图关联
constraints: [{ type: "emphasis", value: "高价值用户" }]
```

**Step 2: 元素拆解**
```
E1: Distribution("对照组消费分布", points: [{user:"u001", x:3.2, y:5800, tag:"high"}, ...])
E2: Distribution("实验组消费分布", points: [{user:"u001", x:4.1, y:7200, tag:"high"}, ...])
```

**Step 3: 结构组织 + 关联识别**
```
SIR:
  root (linked-views, left-right, normal)
  ├── E1 (L1, 左) — 对照组散点图
  ├── E2 (L1, 右) — 实验组散点图
  └── linkings: [
      // 高价值用户的跨视图关联（聚合表达，不逐个连线）
      {
        type: "entity_match",
        label: "高价值用户群(12人) 位置变化",
        endpoints: [
          { elementRef: "E1", selector: "points[tag=high]" },
          { elementRef: "E2", selector: "points[tag=high]" }
        ],
        strength: "strong"
      }
    ]
```

**Step 4: 视觉映射**
```
Canvas Schema:
{
  "type": "LinkedViews",
  "layout": { "type": "grid", "columns": 2, "gap": 24 },
  "children": [
    {
      "type": "Chart",
      "props": {
        "chartType": "scatter",
        "title": "对照组消费分布",
        "data": [...],
        "highlight": { "filter": "tag=high", "style": "emphasis" }
      }
    },
    {
      "type": "Chart",
      "props": {
        "chartType": "scatter",
        "title": "实验组消费分布",
        "data": [...],
        "highlight": { "filter": "tag=high", "style": "emphasis" }
      }
    }
  ],
  "links": [
    {
      "type": "connection",
      "from": { "chart": 0, "selector": "tag=high" },
      "to": { "chart": 1, "selector": "tag=high" },
      "style": "curve",
      "color": "semantic.primary",
      "label": "高价值用户群(12人)",
      "interaction": "brush-link"
    }
  ]
}
```

**渲染效果**：
- 两个散点图左右并列
- 高价值用户点用强调色高亮
- 两组高价值用户点之间用贝塞尔曲线连接
- hover 一侧的点，另一侧对应点高亮
- 标注："平均消费从 ¥5800 → ¥7200 (+24%)"

### 5.2 示例：数据血缘字段映射

**用户需求**："展示 ods_order 表到 dwd_order_wide 表的字段映射关系。"

**元素拆解**：
```
E1: Hierarchy("ods_order 字段", [order_id, user_id, amount, create_time, status])
E2: Hierarchy("dwd_order_wide 字段", [oid, uid, order_amount, dt, order_status, user_level])
```

**关联识别**：
```
linkings: [
  { type: "field_mapping", from: "E1.field[order_id]", to: "E2.field[oid]", label: "重命名" },
  { type: "field_mapping", from: "E1.field[user_id]", to: "E2.field[uid]", label: "重命名" },
  { type: "field_mapping", from: "E1.field[amount]", to: "E2.field[order_amount]", label: "重命名" },
  { type: "field_mapping", from: "E1.field[create_time]", to: "E2.field[dt]", label: "格式转换" },
  { type: "field_mapping", from: "E1.field[status]", to: "E2.field[order_status]", label: "枚举映射" },
  { type: "aggregation", from: "E1.field[user_id]", to: "E2.field[user_level]", label: "JOIN dim_user 表聚合" }
]
```

**视觉映射**：Linked Views + 连接线（6条，≤15，直接画线）
- 左右两列字段列表
- 对应字段间用实线连接（映射）和虚线连接（聚合）
- 连线上标注变换类型

---

## 六、对现有文档的影响

### 6.1 INTERMEDIATE-LAYER.md 需要更新

1. SIR 数据结构增加 `linkings` 字段
2. 元素拆解规则增加"关联识别"步骤——检查不同元素之间是否存在实体级、字段级、时间级的对应关系
3. 意图解析的 DataShape 增加 `relational` 形态时，触发关联检查

### 6.2 COMPOSITION.md 需要更新

1. 新增第七种布局模式：Linked Views
2. 组合决策树增加"有跨视图关联"分支
3. 新增"关联的视觉化"章节（五种手段 + 决策树）

### 6.3 DESIGN.md 需要更新

1. Canvas DSL 增加顶层 `links` 字段
2. 渲染引擎需要支持跨组件连线渲染（SVG overlay 或 Canvas overlay）
3. 组件库增加 LinkedViews 容器组件

### 6.4 对渲染引擎的技术要求

跨视图关联的渲染需要特殊处理：

```
渲染流程更新:
  Schema → 组件树构建 → 独立渲染各组件
      ↓
  Links 处理:
    1. 解析 link 的 from/to 端点
    2. 计算端点在屏幕上的坐标（需要知道组件内部元素的位置）
    3. 在组件之上绘制 SVG/Canvas overlay 层
    4. 绑定交互（hover 联动、brush 刷选）
```

这要求渲染引擎维护一个 **位置注册表（Position Registry）**——每个组件注册其内部可锚定元素的位置，供 Links 层查询和连接。

---

## 七、参考文献

- **Roberts, J. C.** "State of the Art: Coordinated & Multiple Views in Exploratory Visualization" (2007) — [Paper](https://doi.org/10.1109/CMV.2007.19)
- **Becker, R. A. & Cleveland, W. S.** "Brushing Scatterplots" (1987) — [Paper](https://doi.org/10.1080/01621459.1987.10478448)
- **Boukhelifa, N. & Roberts, J. C.** "A Model for View-Point Navigation" (2003) — [Paper](https://doi.org/10.1109/INFVIS.2003.1249019)
- **Collins, C. & Carpendale, S.** "VisLink: Revealing Relationships Amongst Visualizations" (2007) — [Paper](https://doi.org/10.1109/TVCG.2007.70521)
- **Deng, D. et al.** "Revisiting the Design Patterns of Composite Visualizations" (IEEE VIS 2022) — [arXiv:2203.10476](https://arxiv.org/abs/2203.10476)
- **Toh, H. & Lim, K.** "Visual Linking: A Review of the State of the Art" (2022)
