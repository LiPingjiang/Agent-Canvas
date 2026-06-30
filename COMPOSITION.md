# Agent Canvas — 组合与堆叠设计：多意图可视化策略

> 当一个页面需要同时表达多种信息意图（如趋势+构成+对比）时，如何在一个视觉空间内有策略地组合它们？谁在上谁在下、谁在左谁在右、谁为主谁为辅？本文档定义组合与堆叠的规则体系。

---

## 一、调研基础

### 1.1 复合可视化设计模式（Composite Visualization Patterns）

Javed & Elmqvist (2012) 提出复合可视化设计空间，定义了五种组合模式。Deng et al. (IEEE VIS 2022, [Revisiting the Design Patterns of Composite Visualizations](https://arxiv.org/abs/2203.10476)) 对此进行了系统性修订和扩展，分析了 IEEE VIS 论文中 2400+ 个复合可视化实例。

五种组合模式：

**Juxtaposition（并列）** — 多个视图并排放置，各自独立但共享上下文。最常见，占复合可视化的约 60%。
- 优点：不互相干扰，理解简单
- 缺点：需要跨视图对比，认知负荷较高
- 示例：仪表盘中的多个独立卡片

**Superimposition（叠加）** — 多个视图共享同一坐标系，叠加显示。
- 优点：直接对比，节省空间
- 缺点：可能互相遮挡，需要透明度/颜色区分
- 示例：双轴折线图、多系列柱状图

**Overloading（重载）** — 在同一视图中用不同视觉编码通道承载多种信息。
- 优点：极高信息密度
- 缺点：理解门槛高，编码冲突风险
- 示例：气泡图（x=值, y=值, size=量, color=类别）

**Nesting（嵌套）** — 一个视图嵌入另一个视图内部。
- 优点：层级清晰，空间高效
- 缺点：嵌套层级过深导致理解困难
- 示例：卡片中嵌套图表、表格中嵌套迷你图

**Integration（融合）** — 多个视图融合为一个新的统一视图。
- 优点：最优的空间效率和信息整合
- 缺点：设计复杂度最高
- 示例：桑基图（流程+构成+量级）、子弹图（目标+实际+范围）

### 1.2 Medley 意图驱动组合（Intent-based Composition）

Tableau Research 的 [Medley](https://arxiv.org/abs/2208.03175)（IEEE VIS 2022, TVCG 2023）提出了基于分析意图的仪表盘组合推荐。核心思路：先识别用户的分析意图，再推荐匹配的视图组合。

Medley 定义了四种分析意图集合：
- **Trend Comparison（趋势对比）** — 趋势线 + 参考线 + 差异标注
- **Top-N / Ranking（排名）** — 排序条形图 + 明细表
- **Distribution / Outlier（分布/异常）** — 散点图 + 统计摘要
- **Composition / Part-to-Whole（构成）** — 饼图/堆叠图 + 下钻表

### 1.3 Stephen Few 仪表盘布局原则

[Stephen Few](https://www.analyticspress.com/idd.php) 在《Information Dashboard Design》中提出的布局原则：
- 最重要的信息放在左上角（西方阅读习惯的起点）
- 相关信息邻近放置
- 对比信息并排放置（而非上下）
- 使用分组和留白区分信息区
- 避免超过 5-7 个信息区（认知极限）

---

## 二、组合的三个层次

组合不是单一概念，而是三个递进层次：

```
层次一：单图内的多编码组合（Intra-Chart Composition）
  → 一个图表中用多个视觉通道承载多种信息

层次二：多图的布局组合（Inter-Chart Composition）
  → 多个独立图表在页面上的空间排列

层次三：多区域的语义组合（Inter-Region Composition）
  → 页面被分为多个语义区域，每区域承载一种意图
```

### 2.1 层次一：单图内的多编码组合

在单个图表内部，通过不同的视觉编码通道同时承载多种信息意图。对应学术框架中的 Overloading 和 Integration。

**组合维度**：一个图表可以同时承载 2-4 种信息意图。

| 组合方式 | 意图A | 意图B | 实现方法 | 示例 |
|---------|-------|-------|---------|------|
| 双轴叠加 | Track(趋势) | Compare(对比) | 左Y轴=指标A折线，右Y轴=指标B折线 | 营收趋势 vs 订单量趋势 |
| 大小+颜色 | Distribute(分布) | Compose(构成) | x/y=维度, size=量级, color=类别 | 用户分层气泡图 |
| 标注+趋势 | Track(趋势) | Narrate(叙事) | 折线图 + 关键事件标注点 | DAU趋势 + 大促标注 |
| 堆叠分面 | Compose(构成) | Track(趋势) | 堆叠面积图（时间×类别） | 月度各品类营收堆叠 |
| 参考+实际 | Compare(对比) | Track(趋势) | 实际折线 + 目标线 + 差异区 | 实际营收 vs 预算 |
| 迷你嵌入 | Track(趋势) | Compare(对比) | 表格行中嵌入 sparkline | 区域销售表+趋势迷你图 |

**多编码组合的约束规则**：

规则1 — **通道不冲突**。同一图表中，不同信息必须使用不同的视觉通道（位置/大小/颜色/形状/方向/纹理），不能两个信息都用颜色编码。

规则2 — **主从分明**。多编码中必须有一个是"主编码"（承载主意图），其余是"辅编码"。主编码占据最显著的视觉通道（通常是位置/大小）。

规则3 — **认知极限**。单图最多承载 4 种信息意图。超过 4 种应拆分为多图。

规则4 — **图例完整**。每增加一种编码通道，必须提供对应的图例说明。

### 2.2 层次二：多图的布局组合

多个独立图表在页面上的空间排列。对应学术框架中的 Juxtaposition。

**六种布局模式**：

**模式A：横向并列（Side-by-Side）**
```
┌──────────┬──────────┐
│  Chart A  │  Chart B  │
└──────────┴──────────┘
```
- 适用：对比两个同等重要的维度
- 主辅关系：A 和 B 地位平等
- 示例：营收趋势 | 利润趋势

**模式B：纵向堆叠（Vertical Stack）**
```
┌──────────────────────┐
│       Chart A         │
├──────────────────────┤
│       Chart B         │
└──────────────────────┘
```
- 适用：有先后阅读顺序的信息
- 主辅关系：A 在上（先看），B 在下（后看）
- 示例：概述卡片 → 详细图表

**模式C：主+辅环绕（Hub-and-Spoke）**
```
┌──────────────────────┐
│                      │
│     Main Chart       │
│    （大，占2/3）       │
│                      │
├─────────┬────────────┤
│  Sub 1   │   Sub 2    │
└─────────┴────────────┘
```
- 适用：一个核心视图 + 多个辅助视图
- 主辅关系：Main 为主，Sub1/2 为辅
- 示例：核心趋势图 + 两个分解小图

**模式D：网格平铺（Grid）**
```
┌────────┬────────┬────────┐
│ Card 1 │ Card 2 │ Card 3 │
├────────┼────────┼────────┤
│ Card 4 │ Card 5 │ Card 6 │
└────────┴────────┴────────┘
```
- 适用：多个同等重要的指标
- 主辅关系：全部平等（或通过 emphasis 属性区分）
- 示例：6 个 KPI 卡片

**模式E：左右分栏（Split Panel）**
```
┌──────────────┬───────┐
│              │       │
│  Primary     │ Side  │
│  Content     │ Panel │
│              │       │
└──────────────┴───────┘
```
- 适用：主体内容 + 辅助/过滤面板
- 主辅关系：Primary 为主（宽），Side 为辅（窄）
- 示例：主图表 + 筛选器面板

**模式F：小多图（Small Multiples）**
```
┌────┬────┬────┬────┬────┐
│ M1 │ M2 │ M3 │ M4 │ M5 │
├────┼────┼────┼────┼────┤
│ M6 │ M7 │ M8 │ M9 │ M10│
└────┴────┴────┴────┴────┘
```
- 适用：同一维度下多个分类的对比
- 主辅关系：全部平等，共享坐标轴
- 示例：10 个城市各自的 DAU 趋势

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
- 详细设计见 [LINKING.md](./LINKING.md)

### 2.3 层次三：多区域的语义组合

页面被分为多个语义区域，每个区域承载一种信息意图。这是最高层次的组合——不是图表之间的排列，而是"信息区"之间的语义编排。

**语义区域类型**：

| 区域类型 | 承载意图 | 典型内容 | 视觉特征 |
|---------|---------|---------|---------|
| 指标区 | Metrics | KPI 卡片组 | 顶部、大字、强调色 |
| 趋势区 | Track | 折线/面积图 | 中部、宽幅 |
| 构成区 | Compose | 饼图/树形图 | 中部、紧凑 |
| 对比区 | Compare | 柱状图/雷达图 | 中部、并列 |
| 明细区 | Detail | 表格 | 底部、全宽 |
| 叙事区 | Narrate | 文字+Callout | 穿插、中等宽度 |
| 导航区 | Navigate | 筛选器/Tabs | 侧边或顶部 |
| 决策区 | Decide | 决策矩阵 | 底部、全宽 |

**语义区域的编排规则**：

规则1 — **纵向优先**。主区域从上到下排列，符合自然阅读流。横向并排仅用于同级别信息。

规则2 — **指标先行**。如果存在指标区，放在最顶部（用户第一眼看到数字）。

规则3 — **趋势其次**。趋势图放在指标区下方，解释"指标为什么是这个值"。

规则4 — **明细殿后**。表格等高密度明细信息放在最底部（需要时才查看）。

规则5 — **叙事穿插**。叙事块可以穿插在各区域之间作为"过渡和解释"，但不应打断数据区的连续性。

规则6 — **导航包围**。筛选器/导航要么在顶部全局，要么在左侧固定，不与内容区混排。

---

## 三、主辅关系的设计规则

### 3.1 什么是主辅关系？

在组合中，不是所有信息都同等重要。"主"是用户最需要看到的，"辅"是帮助理解主的。主辅关系决定了空间分配、视觉权重、交互优先级。

### 3.2 主辅分配规则

**规则1：意图优先级 → 主辅分配**

从中间层的 Intent 解析中，信息目标的顺序决定优先级：

```
如果 intent.what = [metrics, trend, composition]
  → metrics 为主(L1), trend 为辅(L2), composition 为辅(L2)

如果 intent.what = [comparison, decision]
  → comparison 为主(L1), decision 为辅(L2) 或反过来

如果 intent.what = [narrative]
  → narrative 本身就是主，内部 section 分主辅
```

**规则2：空间分配比例**

主辅关系直接映射到空间分配：

```
L1 (焦点): 占 50-60% 的可用空间
L2 (支撑): 占 30-40% 的可用空间
L3 (补充): 占 5-10% 的可用空间（或隐藏/折叠）
```

具体到 Grid 布局：
```
3列 Grid:
  L1 → span 2 (占2列)
  L2 → span 1 (占1列)

2列 Grid:
  L1 → span 2 (全宽)
  L2 → span 1 (半宽)
```

**规则3：视觉权重差异化**

| 维度 | L1 主 | L2 辅 | L3 补 |
|------|-------|-------|-------|
| 字体大小 | 大 (24-48px) | 中 (14-18px) | 小 (12px) |
| 色彩饱和度 | 高（强调色） | 中（主色） | 低（灰色） |
| 卡片阴影 | 深 | 浅 | 无 |
| 边框/背景 | 填充色背景 | 白色+边框 | 透明 |
| 图表尺寸 | 大（高 300-400px） | 中（高 200-250px） | 小（高 100-150px） |
| 交互优先级 | 可点击/筛选 | 可展开/查看 | 仅展示 |

**规则4：阅读流引导主辅**

```
Z-Pattern（适合稀疏布局，L1 少）:
  L1(左上) → L1(右上)
       ↘
  L2(左下) → L2(右下)

F-Pattern（适合密集布局，多 L2）:
  L1(左上, 全宽)
  L2(左侧, 主体)  →  L2(右侧)
  L2(左侧, 主体)  →  L2(右侧)
  L3(底部, 全宽)

Layer-Cake（适合层次分明的报告）:
  L1(全宽)
  ─────────────
  L2(左) | L2(右)
  ─────────────
  L3(全宽)
```

---

## 四、组合的决策树

当 Agent 需要将多种信息意图组合到一个页面时，按以下决策树选择组合策略：

```
输入：SIR（含多个信息元素 + 组织结构 + 意图）

Q1: 有多少种不同的信息意图？
  ├── 1种 → 单图即可，选择对应组件
  │
  ├── 2种 → Q2
  │
  └── 3+种 → Q3

Q2: 两种意图是否可以共享坐标系？
  ├── 是 → Superimposition（叠加）
  │   → 如：双轴折线图（趋势+对比）
  │   → 如：堆叠面积图（趋势+构成）
  │
  └── 否 → Q2a
      Q2a: 两种意图是否有主辅关系？
        ├── 是 → Hub-and-Spoke（主+辅环绕）
        │   → 如：大趋势图 + 小饼图
        │
        └── 否 → Juxtaposition（并列）
            → 如：两个等大的柱状图

Q3: 多种意图之间的关系是？
  ├── 同级别（都重要）→ Q3a
  │   Q3a: 数量 ≤ 4?
  │     ├── 是 → Grid（网格平铺）
  │     └── 否 → Small Multiples（小多图）
  │
  ├── 有明确主辅 → Hub-and-Spoke 或 Split Panel
  │   → L1 占大空间，L2 围绕
  │
  ├── 有阅读顺序 → Vertical Stack（纵向堆叠）
  │   → 按阅读流从上到下
  │
  ├── 有层级关系 → Nesting（嵌套）
  │   → 如：卡片中嵌套图表+表格
  │
  └── 有跨视图实体关联 → Linked Views（关联并列）
      → 检查关联数量:
        ├── ≤15 条 → 连接线 + Linked Views
        ├── 15-50 条 → 连接带 / 颜色联动
        └── >50 条 → 颜色联动 + Brushing 刷选
```

### 4.1 决策树示例

**示例1：Q2 经营仪表盘**
```
意图：[metrics, trend, composition] (3种)
关系：metrics 是主，trend 和 composition 是辅
→ Q3 → 有明确主辅 → Hub-and-Spoke
  → 顶部：3个 KPI 卡片（L1，横排）
  → 下方左：营收趋势折线图（L2，占2/3宽）
  → 下方右：营收构成饼图（L2，占1/3宽）
```

**示例2：故障复盘**
```
意图：[narrative, timeline, distribution] (3种)
关系：有阅读顺序（概述→时间线→影响→根因→措施）
→ Q3 → 有阅读顺序 → Vertical Stack
  → 概述 Callout (L1)
  → 时间线 Timeline (L2)
  → 影响范围 Chart (L2)
  → 根因 Callout (L2)
  → 措施 Callout (L1, emphasis)
```

**示例3：方案对比决策**
```
意图：[comparison, decision] (2种)
关系：对比是主，决策是辅（基于对比做决策）
→ Q2 → 不共享坐标系 → Q2a → 有主辅 → Hub-and-Spoke
  → 主体：对比表格/雷达图 (L1，大)
  → 下方：决策结论卡片 (L2，全宽)
```

**示例4：10 城市销售对比**
```
意图：[comparison, trend] (2种)
关系：每个城市都需要看趋势，10个城市同级
→ Q3 → 同级别 → Q3a → 数量>4 → Small Multiples
  → 2×5 网格，每格一个城市的折线图
  → 共享 Y 轴刻度，便于对比
```

---

## 五、嵌套策略（Nesting）

嵌套是组合中特别重要的一种——在一个容器内嵌入多种元素，形成层次结构。

### 5.1 嵌套层级

```
Level 0: Page / Dashboard
  └── Level 1: Region / Section（语义区域）
       └── Level 2: Card / Panel（内容容器）
            └── Level 3: Chart / Table / Text（具体组件）
                 └── Level 4: Mini-element（迷你图/标注/徽章）
```

### 5.2 常见嵌套模式

**模式1：卡片嵌套图表**
```
Card
  ├── Heading: "营收趋势"
  ├── Chart(line): 趋势折线
  └── Badge: "+12.5%"
```
> 最常见的嵌套，Card 提供标题和上下文，Chart 提供数据可视化。

**模式2：表格嵌套迷你图（Sparkline）**
```
Table
  ├── Row: 华东 | ¥456万 | +15% | [sparkline ▁▃▅▇]
  ├── Row: 华北 | ¥321万 | +8%  | [sparkline ▁▂▄▆]
  └── Row: 华南 | ¥257万 | -2%  | [sparkline ▇▅▃▁]
```
> 在表格的每行嵌入一个迷你趋势图，同时呈现数值和趋势。

**模式3：图表嵌套标注**
```
Chart(line)
  ├── 折线: DAU 趋势
  ├── Annotation: "双11大促" (标在 11/11 位置)
  ├── ReferenceLine: 目标线 300万
  └── Band: 置信区间
```
> 在一个图表中叠加多种辅助信息。

**模式4：区域嵌套多组件**
```
Section("核心指标")
  ├── Grid(3列)
  │   ├── StatCard("总营收", "+12.5%")
  │   ├── StatCard("订单量", "+8.3%")
  │   └── StatCard("客单价", "+3.8%")
  └── Chart(bar) // 横向对比
```
> 一个语义区域内包含一个 Grid 和一个 Chart，形成更丰富的组合。

### 5.3 嵌套规则

规则1 — **最多 4 层**。超过 4 层嵌套会导致理解困难。

规则2 — **每层有明确的语义角色**。Level 1 是区域标题，Level 2 是内容容器，Level 3 是数据展示——不能无意义地嵌套。

规则3 — **内层尺寸适配外层**。嵌套的内部组件自动适配容器的剩余空间，不能溢出。

规则4 — **嵌套不等于隐藏**。嵌套不是为了折叠隐藏信息，而是为了组织关联信息。如果信息需要折叠，使用 Accordion/Tabs 而非深层嵌套。

---

## 六、空间分配算法

### 6.1 问题定义

给定 N 个信息元素，每个有优先级 L1/L2/L3，如何在可用空间内分配？

### 6.2 空间分配规则

**规则1：L1 优先分配**

L1 元素先获得空间，剩余空间分配给 L2，L3 最后分配（可能被折叠）。

**规则2：空间权重**

```
L1 权重 = 3
L2 权重 = 1.5
L3 权重 = 0.5

某元素分配空间 = (该元素权重 / 所有元素权重之和) × 总空间
```

**规则3：最小尺寸保障**

每个元素有最小尺寸，空间不足以满足所有元素的最小尺寸时：
1. 先缩减 L3 元素（折叠为 Accordion）
2. 再缩减 L2 元素（缩小图表高度）
3. L1 元素不可缩减

**规则4：跨列(span)分配**

在 Grid 布局中，根据优先级分配列跨度：

```
3列Grid:
  L1 元素 → span 2 或 span 3(全宽)
  L2 元素 → span 1
  L3 元素 → span 1 或隐藏

2列Grid:
  L1 元素 → span 2(全宽)
  L2 元素 → span 1
  L3 元素 → span 1 或隐藏
```

### 6.3 分配示例

**场景：3列Grid，4个元素（1个L1, 2个L2, 1个L3），总宽 1200px**

```
权重: L1=3, L2=1.5, L2=1.5, L3=0.5 → 总权重=6.5

L1 分配: 3/6.5 × 1200 = 554px → span 2 (约 800px)
L2 分配: 1.5/6.5 × 1200 = 277px → span 1 (约 385px)
L2 分配: 1.5/6.5 × 1200 = 277px → span 1 (约 385px)
L3 分配: 0.5/6.5 × 1200 = 92px → 不足最小宽度，折叠为 Tab

实际布局:
┌──────────────────────────┬──────────┐
│                          │          │
│    L1 (span 2)           │  L2 (1)  │
│                          │          │
├──────────┬───────────────┴──────────┤
│          │                          │
│  L2 (1)  │    L3 (折叠为 Tab/Accordion)│
│          │                          │
└──────────┴──────────────────────────┘
```

---

## 七、组合的完整工作流

将组合与堆叠规则嵌入中间层的工作流：

```
Step 1-2: 意图解析 + 元素拆解（已有）
  → 产出：N 个信息元素 + Intent

Step 3a: 结构组织（已有）
  → 产出：SIR 元素树 + 优先级标记(L1/L2/L3)

Step 3b: 组合策略选择（NEW）
  → 输入：SIR 元素树 + Intent
  → 决策：
    1. 判断意图数量 → 选择组合层次（单图/多图/多区域）
    2. 判断意图关系 → 选择组合模式（并列/叠加/嵌套/环绕/网格/小多图）
    3. 判断主辅关系 → 分配 L1/L2/L3
    4. 判断空间需求 → 分配 span/尺寸
  → 产出：组合策略（CompositionPlan）

Step 4: 视觉映射（已有，增强）
  → 输入：SIR + CompositionPlan
  → 映射：元素→组件 + 组合策略→布局 + 主辅→视觉权重
  → 产出：Canvas Schema
```

### 7.1 CompositionPlan 数据结构

```typescript
interface CompositionPlan {
  // 组合层次
  level: "intra-chart" | "inter-chart" | "inter-region";

  // 组合模式
  pattern: "juxtaposition" | "superimposition" | "overloading"
         | "nesting" | "integration" | "hub-spoke"
         | "grid" | "small-multiples" | "split-panel" | "stack";

  // 布局参数
  layout: {
    type: "grid" | "flex" | "stack" | "absolute";
    columns?: number;
    gap: number;
    readingFlow: "z-pattern" | "f-pattern" | "layer-cake" | "top-down";
  };

  // 主辅关系
  hierarchy: {
    primary: ElementRef[];    // L1 元素引用
    secondary: ElementRef[];  // L2 元素引用
    tertiary: ElementRef[];   // L3 元素引用
  };

  // 空间分配
  allocation: {
    [elementRef: string]: {
      span: number;          // 列跨度
      minHeight?: number;    // 最小高度
      maxHeight?: number;    // 最大高度
      weight: number;        // 空间权重
    };
  };

  // 嵌套关系
  nesting?: {
    [containerRef: string]: ElementRef[];  // 容器 → 内含元素
  };
}
```

---

## 八、从 301 个 case 验证组合模式

将我们的 case 集与组合模式交叉验证：

| 组合模式 | 典型 case | 验证 |
|---------|----------|------|
| Hub-and-Spoke | B01 仪表盘(KPI+趋势+饼图) | ✅ 主图+辅图环绕 |
| Grid | B03 区域业绩(多卡片) | ✅ 等权重网格 |
| Vertical Stack | T12 故障复盘(时间线+分析) | ✅ 阅读顺序堆叠 |
| Small Multiples | NET_GRO01 同期群留存 | ✅ 多城市趋势小图 |
| Split Panel | RTL02 商品陈列(图+列表) | ✅ 主内容+侧栏 |
| Nesting | SAL03 销售管线(卡片+图表) | ✅ 卡片嵌套图表 |
| Superimposition | B05 DAU+大促标注 | ✅ 折线+标注叠加 |
| Integration | B02 销售漏斗(流程+转化率) | ✅ 融合视图 |
| Overloading | B07 用户分层气泡图 | ✅ 多编码通道 |

301 个 case 中，约 60% 涉及多意图组合（2种以上信息意图），验证了组合规则体系的必要性。

---

## 九、参考文献

- **Javed, W. & Elmqvist, N.** "Exploring the Design Space of Composite Visualization" (2012) — [Paper](https://www.cs.au.dk/~elm/pdf/compvis.pdf)
- **Deng, D. et al.** "Revisiting the Design Patterns of Composite Visualizations" (IEEE VIS 2022, TVCG 2023) — [arXiv:2203.10476](https://arxiv.org/abs/2203.10476) | [IEEE](https://ieeexplore.ieee.org/document/9916137)
- **Pandey, A. et al.** "MEDLEY: Intent-based Recommendations to Support Dashboard Composition" (IEEE VIS 2022, TVCG 2023) — [arXiv:2208.03175](https://arxiv.org/abs/2208.03175)
- **Few, S.** *Information Dashboard Design: The Effective Visual Communication of Data* (Analytics Press, 2013) — [Book](https://www.analyticspress.com/idd.php)
- **Tufte, E. R.** *The Visual Display of Quantitative Information* (Graphics Press, 1983/2001) — [Book](https://www.edwardtufte.com/)
- **Dashboard Design Patterns** — [dashboarddesignpatterns.github.io](https://dashboarddesignpatterns.github.io/patterns.html)
