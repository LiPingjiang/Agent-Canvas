# Agent Canvas — Agent AGI 可视化生成工具设计文档

## 一、问题分析：为什么 Agent 生成的可视化效果不好？

### 1.1 根本矛盾：文本生成 vs 视觉评判

[AeSlides](https://arxiv.org/abs/2604.22840)（2025）论文精准总结了这一矛盾：**"生成过程以文本为中心，质量标准由视觉美学决定"**。

LLM 的本质是下一个 token 的概率预测，它在文本序列上运作。而可视化是空间性的——需要理解上下左右、层级关系、对齐分布、色彩搭配、视觉平衡。这就像让一个只能用经纬度坐标描述事物的人去画一幅油画，理论上可行，但效率极低且极易出错。

### 1.2 四层核心问题

**第一层：空间推理能力缺失**

[LaySPA](https://arxiv.org/abs/2509.16891)（2025）的研究表明，LLM 在布局设计任务中缺乏显式的空间推理能力。当被要求"把这个卡片放在图表右侧"时，LLM 无法可靠地将这种空间关系转化为正确的 CSS 布局代码。具体表现为：

- 元素重叠、溢出、错位
- 响应式布局在不同屏幕尺寸下崩溃
- 复杂嵌套结构中层级关系混乱
- 多元素对齐和分布不均匀

[LayoutVLM](https://arxiv.org/abs/2412.02193)（Stanford, CVPR 2025）和 [SKE-Layout](https://openaccess.thecvf.com/content/CVPR2025/html/Wang_SKE-Layout_Spatial_Knowledge_Enhanced_Layout_Generation_with_LLMs_CVPR_2025_paper.html)（CVPR 2025）的研究进一步证实，即使是最先进的多模态模型，在物理上合理的布局生成方面仍有显著局限。

**第二层：缺乏视觉反馈闭环**

[MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/)（ACL 2024 Findings）的核心贡献是证明了视觉反馈机制的价值：通过让多模态 LLM 审查渲染结果并提供反馈，可视化质量平均提升约 12-13 分（百分制）。这说明当前绝大多数 Agent 可视化方案是"盲生成"——生成代码后没有查看渲染结果的能力。

这就像一个闭着眼睛的画家，无法自我纠错。

**第三层：概率生成 vs 视觉精确性冲突**

代码生成是概率性的——LLM 每次生成的代码可能有细微差异。但视觉设计对精确性要求极高：一个像素的偏移就可能导致元素错位，一个颜色的细微差异就可能导致视觉不和谐。这种"概率性生成"与"精确性要求"之间的冲突，随着可视化复杂度的增加而指数级放大。

[VisEval](https://arxiv.org/abs/2407.00981) 和 [PlotCraft](https://arxiv.org/abs/2511.00010)（2025）的基准研究表明：当前最强 LLM 在单图表任务成功率约 60-80%，但复杂多面板场景骤降至 20-40%。

**第四层：布局约束求解能力不足**

CSS Flexbox/Grid、绝对定位、响应式设计——这些布局系统本质上是约束求解问题。LLM 不擅长在多维约束空间中找到最优解。[Vercel v0](https://v0.app/) 通过 RAG + AutoFix 机制来缓解这个问题（检索相似布局模式 + 自动修复生成错误），但根本问题仍未解决。

### 1.3 现有方案的量化对比

| 方案 | Agent 任务复杂度 | 可靠性 | 审美质量 | 灵活性 | 交互性 | 视觉反馈 |
|------|-----------------|--------|---------|--------|--------|---------|
| Markdown 渲染 | 极低 | 极高 | 低 | 极低 | 无 | 无 |
| HTML/CSS/JS (Artifacts) | 高 | 中 | 不稳定 | 极高 | 有 | 通常无 |
| Python 绘图 (matplotlib) | 中 | 中 | 低-中 | 中 | 无 | 通常无 |
| DSL 配置 (Vega-Lite/ECharts) | 低-中 | 高 | 中 | 低(域限定) | 有限 | 通常无 |
| 组件库 (Gradio/Streamlit) | 中 | 中-高 | 中 | 中 | 有 | 无 |
| SVG/Canvas 直接绘制 | 极高 | 低 | 不稳定 | 极高 | 有 | 通常无 |

**关键发现**：可靠性与灵活性之间存在强烈的反相关关系。Markdown 最可靠但最不灵活，SVG/Canvas 最灵活但最不可靠。没有一种方案能在所有维度上都表现优秀。

---

## 二、Agent 可视化的本质

### 2.1 什么是 Agent 可视化？

Agent 可视化是将 Agent 的语义理解转化为人类可快速感知的视觉形式的过程。这个过程包含四个阶段：

```
语义理解 → 结构决策 → 视觉执行 → 结果验证
(LLM 擅长)  (LLM 部分擅长) (LLM 不擅长) (需要多模态)
```

**阶段一：语义理解** — Agent 理解要展示什么信息、信息之间的关系、目标受众。这是 LLM 的强项。

**阶段二：结构决策** — 决定信息的组织方式：哪些内容放在一起、用什么组件展示、整体布局结构。LLM 在这个阶段表现中等——能做出合理的宏观决策，但在细节布局上容易出错。

**阶段三：视觉执行** — 将结构决策转化为具体的视觉实现：精确的像素位置、颜色值、字体大小、间距、对齐。这是 LLM 的弱项，因为需要空间推理和审美判断。

**阶段四：结果验证** — 查看渲染结果，判断是否正确、美观，必要时修正。需要多模态能力（视觉理解）。

### 2.2 核心洞察：分离关注点

现有方案的根本问题是把这四个阶段混在一起，让 LLM 一次性完成从语义到像素的全部转化。就像让一个人同时担任编剧、导演、摄影和后期——虽然理论上有全能型人才，但效率和可靠度远不如分工协作。

**解决方案的核心思路是分离关注点**：

- 让 Agent 专注于**阶段一和阶段二**（语义理解 + 结构决策）——生成一个语义化的结构描述
- 让**确定性渲染引擎**处理阶段三（视觉执行）——将结构描述转化为精确的视觉实现
- 让**视觉反馈闭环**处理阶段四（结果验证）——渲染、截图、多模态审查、修正

### 2.3 从行业趋势看方向

2025 年的行业趋势正在收敛到这个方向：

**[A2UI](https://a2ui.org/)（Agent-to-User Interface）协议** — [Google 主导的开放标准](https://github.com/a2ui-project/a2ui)，Agent 发送声明式 JSON 描述 UI 意图，客户端用原生组件渲染。核心理念："传输的是界面意图而非可执行代码"。支持 React、Angular、Flutter 等多平台。

**[Vercel JSON-Render](https://github.com/vercel-labs/json-render)** — Vercel 推出的生成式 UI 框架，开发者用 Zod schemas 定义组件目录，LLM 生成受约束的 JSON 规范，确定性渲染器负责实例化。

**[Portal UX Agent](https://arxiv.org/abs/2511.00843)** — 提出"有界生成"概念：LLM 在高层规划 UI，确定性渲染器从审核过的组件和模板中组装最终界面。

**[Vercel v0](https://v0.app/)** — 从"AI 搭建网页"进化为全栈 Agent，核心是 RAG + AutoFix 机制：检索相似组件模式、生成代码、自动检测和修复错误。

这些项目都指向同一个方向：**约束生成空间 + 确定性渲染 + 视觉反馈**。

---

## 三、设计理念

### 3.1 三大设计原则

**原则一：语义驱动，而非代码驱动**

Agent 不写代码，而是描述"要展示什么"和"怎么组织"。系统提供一组语义化的组件原语（Card、Chart、Timeline、Table、Kanban...），Agent 通过组合这些原语来表达视觉意图。

类比：与其让 Agent 写 HTML/CSS（像 [Claude Artifacts](https://claude.com/blog/build-artifacts)），不如让 Agent 写一个 JSON schema 描述"一个包含标题、统计卡片组和趋势图表的仪表盘"。前者需要处理几百行 CSS，后者只需要几十行 JSON。

**原则二：有界生成 + 确定性渲染**

生成空间是有界的——Agent 只能使用预定义的组件和属性，不能生成任意代码。渲染是确定性的——相同的 schema 永远产生相同的视觉输出，且输出保证美观（因为组件本身经过专业设计）。

这解决了两个核心问题：
- 可靠性：有界生成消除了"生成出无法运行的代码"的可能性
- 审美：确定性渲染保证每个组件都有专业级的视觉表现

**原则三：视觉反馈闭环**

系统支持渲染 schema → 截图 → 多模态 LLM 审查 → 反馈修正的闭环。这是 [MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/) 证明有效的机制，但当前主流方案（[Artifacts](https://claude.com/blog/build-artifacts)、[v0](https://v0.app/) 等）大多缺乏。

### 3.2 与现有方案的关键差异

```
现有方案的流程：
  Agent → 生成 HTML/CSS/JS 代码 → 浏览器渲染 → (结束，无反馈)

Agent Canvas 的流程：
  Agent → 生成 Canvas Schema (JSON) → 确定性渲染 → 截图 → 多模态审查 → (修正 Schema) → 最终输出
```

核心差异：
1. Agent 生成的是**结构化 schema** 而非**原始代码**——降低了生成复杂度和出错率
2. 渲染由**确定性引擎**完成——保证了视觉质量和一致性
3. 内置**视觉反馈闭环**——Agent 可以看到自己的输出并修正

---

## 四、架构设计

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Agent Canvas System                         │
│                                                                     │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────┐            │
│  │  Agent    │───▶│  Canvas DSL  │───▶│   Renderer   │            │
│  │  (LLM)   │    │  (JSON Schema)│    │  (React Engine)│           │
│  └──────────┘    └──────────────┘    └──────┬───────┘            │
│       ▲               │                     │                     │
│       │               ▼                     ▼                     │
│       │         ┌──────────┐         ┌───────────┐               │
│       │         │ Validator│         │ Screenshot│               │
│       │         │ (Zod)    │         │ Service   │               │
│       │         └──────────┘         └─────┬─────┘               │
│       │                                    │                      │
│       │              ┌─────────────────────┘                      │
│       │              ▼                                            │
│       │         ┌───────────┐                                    │
│       └─────────│ Visual    │  (反馈: "图表标题遮挡了数据标签")      │
│                 │ Reviewer  │                                    │
│                 │ (VLM)     │                                    │
│                 └───────────┘                                    │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Component Library                         │  │
│  │  Layout │ Content │ Chart │ Diagram │ Interactive │ Media    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                      Theme System                            │  │
│  │  Light │ Dark │ Custom │ Typography │ Spacing │ Color        │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 核心模块

#### 4.2.1 Canvas DSL — 语义化描述语言

Agent 生成的核心产物。一个 JSON 结构，描述"要展示什么"而非"怎么渲染"。

设计要点：
- **树形结构**：组件以树形嵌套组合，天然映射视觉层级
- **强类型**：每个组件有明确的属性定义，用 Zod/TypeScript 约束
- **语义化属性**：属性描述意图（如 `emphasis: "high"`），而非样式（如 `color: "red"`）
- **声明式布局**：通过 `layout` 属性声明布局意图（如 `layout: "grid"` + `columns: 3`），不写 CSS

示例 — 一个数据仪表盘：

```json
{
  "type": "Dashboard",
  "props": {
    "title": "销售数据概览",
    "subtitle": "2025年Q2季度报告"
  },
  "children": [
    {
      "type": "StatCard",
      "props": {
        "label": "总营收",
        "value": "¥1,234,567",
        "trend": { "direction": "up", "percentage": 12.5 },
        "emphasis": "high"
      }
    },
    {
      "type": "StatCard",
      "props": {
        "label": "新增用户",
        "value": "8,901",
        "trend": { "direction": "down", "percentage": 3.2 }
      }
    },
    {
      "type": "Chart",
      "props": {
        "chartType": "line",
        "title": "营收趋势",
        "data": [
          { "label": "4月", "value": 380000 },
          { "label": "5月", "value": 420000 },
          { "label": "6月", "value": 434567 }
        ],
        "xAxis": "label",
        "yAxis": "value"
      }
    },
    {
      "type": "Table",
      "props": {
        "title": "区域销售明细",
        "columns": ["区域", "销售额", "环比增长"],
        "rows": [
          ["华东", "¥456,789", "+15%"],
          ["华北", "¥321,098", "+8%"],
          ["华南", "¥256,789", "-2%"]
        ]
      }
    }
  ],
  "layout": {
    "type": "grid",
    "columns": 2,
    "gap": 16
  }
}
```

Agent 不需要知道卡片用什么阴影、图表用什么颜色、表格怎么对齐——这些由渲染引擎和主题系统决定。

#### 4.2.2 Component Library — 组件库

预构建的、经过专业设计的组件集合。每个组件保证：
- 开箱即用的专业级视觉表现
- 响应式适配
- 主题感知（自动适配 Light/Dark 主题）
- 可组合（可以嵌套在其他组件中）

组件分类：

**布局组件（Layout）**

| 组件 | 用途 | 关键属性 |
|------|------|---------|
| Container | 通用容器，可嵌套 | padding, background, border |
| Grid | 网格布局 | columns, gap, responsive |
| Flex | 弹性布局 | direction, align, justify, gap |
| Stack | 堆叠布局（简化版Flex） | direction, gap, divider |
| Tabs | 标签页 | tabs[], activeIndex |
| Accordion | 折叠面板 | items[], defaultExpanded |
| Carousel | 轮播 | items[], autoPlay, interval |
| SplitPanel | 分屏面板 | direction, ratio, resizable |

**内容组件（Content）**

| 组件 | 用途 | 关键属性 |
|------|------|---------|
| Heading | 标题 | text, level (h1-h6) |
| Text | 文本段落 | content, format (plain/markdown), align |
| List | 列表 | items[], ordered, icon |
| Table | 表格 | columns, rows, sortable, striped |
| Card | 卡片容器 | title, subtitle, headerAction |
| StatCard | 统计卡片 | label, value, trend, icon |
| Badge | 徽章 | text, variant |
| Tag | 标签 | text, color, closable |
| Quote | 引用 | content, author, source |
| CodeBlock | 代码块 | code, language, highlight |
| Callout | 提示框 | type (info/warn/error/success), title, content |
| Timeline | 时间线 | items[], layout |
| Steps | 步骤条 | steps[], current |

**可视化组件（Chart & Diagram）**

| 组件 | 用途 | 关键属性 |
|------|------|---------|
| Chart | 通用图表 | chartType (line/bar/pie/scatter/area/radar), data, series |
| Gauge | 仪表盘 | value, min, max, thresholds |
| Progress | 进度条 | value, max, label, variant |
| Heatmap | 热力图 | data, xLabels, yLabels |
| TreeMap | 矩形树图 | data, colorScale |
| MindMap | 思维导图 | root, layout |
| FlowChart | 流程图 | nodes, edges, direction |
| OrgChart | 组织架构图 | data, layout |
| Kanban | 看板 | columns, cards |
| Gantt | 甘特图 | tasks, timeline |
| RelationGraph | 关系图 | nodes, edges, layout |

**交互组件（Interactive）**

| 组件 | 用途 | 关键属性 |
|------|------|---------|
| Button | 按钮 | label, variant, icon, action |
| Toggle | 开关 | label, checked |
| Slider | 滑块 | label, min, max, value |
| Select | 选择器 | label, options, value |
| SearchInput | 搜索框 | placeholder, value |
| Filter | 过滤器 | label, options, multiSelect |

**媒体组件（Media）**

| 组件 | 用途 | 关键属性 |
|------|------|---------|
| Image | 图片 | src, alt, fit, rounded |
| Icon | 图标 | name, size, color |
| Avatar | 头像 | src, name, size |
| Gallery | 图片画廊 | images[], columns, gap |
| Video | 视频 | src, poster, controls |

#### 4.2.3 Renderer — 确定性渲染引擎

基于 React 的渲染引擎，负责将 Canvas Schema 转化为实际的视觉输出。

核心职责：
- **Schema 解析**：将 JSON schema 解析为 React 组件树
- **组件实例化**：根据 type 字段查找对应组件，传入 props
- **布局计算**：根据 layout 属性计算组件位置和尺寸
- **主题应用**：根据当前主题应用样式
- **响应式适配**：根据容器尺寸自适应布局

渲染引擎的关键设计决策：
- 使用 CSS Grid 和 Flexbox 作为布局基础（而非绝对定位），减少空间推理需求
- 组件内部使用 CSS-in-JS（如 styled-components 或 Emotion）确保样式隔离和主题感知
- 布局引擎支持自动换行、自动间距、自动对齐——Agent 只需声明意图

```
渲染流程:
  Schema JSON
    → Schema 验证 (Zod)
    → 组件树构建 (递归解析 children)
    → 布局计算 (Grid/Flex/Stack)
    → 主题注入 (CSS Variables)
    → React 渲染
    → DOM 输出
    → (可选) Screenshot
```

#### 4.2.4 Visual Feedback Loop — 视觉反馈闭环

这是 Agent Canvas 区别于 [Claude Artifacts](https://claude.com/blog/build-artifacts)、[Vercel v0](https://v0.app/) 等方案的关键差异化能力。

工作流程：

```
1. Agent 生成 Canvas Schema
2. Renderer 渲染 Schema → React 组件 → DOM
3. Screenshot Service 截图 → 图片
4. Visual Reviewer (多模态 LLM) 审查截图:
   - 检查布局问题（重叠、溢出、错位）
   - 检查视觉问题（对比度、间距、对齐）
   - 检查内容问题（文字截断、数据遗漏）
   - 检查审美问题（色彩搭配、视觉层级）
5. 如果发现问题，生成修正建议
6. Agent 根据建议修正 Schema
7. 回到步骤 2（最多迭代 N 次）
```

Visual Reviewer 的审查维度：

- **布局正确性**：元素是否重叠、溢出、错位
- **视觉层级**：标题、正文、辅助信息的层级是否清晰
- **信息完整性**：是否有文字被截断、数据被遮挡
- **间距一致性**：元素间距是否均匀、合理
- **色彩和谐**：颜色搭配是否协调、对比度是否足够
- **响应式适配**：在不同尺寸下是否正常显示

#### 4.2.5 Theme System — 主题系统

确保所有组件有一致的视觉语言。

- **预设主题**：Light、Dark、Blue（商务）、Warm（温暖）、Tech（科技感）
- **设计 Token**：颜色、字体、间距、圆角、阴影等设计变量
- **语义色彩**：success（绿）、warning（黄）、error（红）、info（蓝）—— Agent 只需指定语义，系统自动选择正确颜色
- **自动适配**：组件自动感知当前主题并调整样式

Agent 不需要知道 `#1890ff` 是什么颜色，只需说 `emphasis: "high"` 或 `status: "success"`。

### 4.3 技术栈选择

```
前端渲染层:  React 18+ + TypeScript
组件样式:    Tailwind CSS + CSS Variables (主题系统)
图表渲染:    Recharts / ECharts (封装为 Canvas 组件)
图标库:      Lucide React
Schema 验证: Zod
构建工具:    Vite
截图服务:    Playwright (headless browser screenshot)
截图审查:    多模态 LLM (GPT-4o / Claude 3.5 Sonnet vision)
```

---

## 五、Canvas DSL 详细规范

### 5.1 Schema 结构

```typescript
// 顶层 Schema
interface CanvasSchema {
  type: string;           // 组件类型，如 "Dashboard", "Report", "Page"
  props?: Record<string, any>;  // 组件属性
  children?: CanvasSchema[];    // 子组件
  layout?: LayoutSpec;          // 布局规范
  theme?: ThemeSpec;            // 主题覆盖
}

// 布局规范
interface LayoutSpec {
  type: "grid" | "flex" | "stack" | "absolute" | "flow";
  columns?: number;       // grid 模式
  gap?: number;           // 间距 (px)
  direction?: "row" | "column";  // flex/stack 模式
  align?: "start" | "center" | "end" | "stretch";
  justify?: "start" | "center" | "end" | "between" | "around";
  wrap?: boolean;         // 是否换行
  responsive?: {          // 响应式断点
    mobile?: Partial<LayoutSpec>;
    tablet?: Partial<LayoutSpec>;
    desktop?: Partial<LayoutSpec>;
  };
}

// 主题规范
interface ThemeSpec {
  preset?: "light" | "dark" | "blue" | "warm" | "tech";
  overrides?: {
    primaryColor?: string;
    backgroundColor?: string;
    textColor?: string;
    borderRadius?: number;
    fontFamily?: string;
  };
}
```

### 5.2 组件定义规范

每个组件用 Zod schema 定义，既是类型约束也是文档：

```typescript
// StatCard 组件定义
const StatCardSchema = z.object({
  type: z.literal("StatCard"),
  props: z.object({
    label: z.string().describe("统计项名称"),
    value: z.string().describe("统计值，已格式化的字符串"),
    trend: z.object({
      direction: z.enum(["up", "down", "flat"]),
      percentage: z.number().optional(),
      label: z.string().optional()
    }).optional().describe("趋势信息"),
    icon: z.string().optional().describe("图标名称"),
    emphasis: z.enum(["high", "medium", "low"]).optional()
      .describe("强调级别，影响视觉突出程度"),
    sparkline: z.array(z.number()).optional()
      .describe("迷你趋势线数据")
  })
});
```

### 5.3 预设页面模板

为降低 Agent 的决策负担，提供预设页面模板：

```typescript
// 模板示例
const templates = {
  dashboard: {
    type: "Page",
    layout: { type: "stack", gap: 24 },
    slots: ["header", "stats", "charts", "table"]
  },
  report: {
    type: "Page",
    layout: { type: "stack", gap: 32 },
    slots: ["title", "summary", "sections", "conclusion"]
  },
  comparison: {
    type: "Page",
    layout: { type: "stack", gap: 24 },
    slots: ["title", "sideBySide", "summary"]
  },
  timeline: {
    type: "Page",
    layout: { type: "stack", gap: 24 },
    slots: ["title", "timeline", "details"]
  }
};
```

Agent 可以选择基于模板生成（填充 slot），也可以自由组合组件。

---

## 六、Agent 交互协议

### 6.1 Agent 工作流程

```
Agent 收到可视化任务
  ↓
1. 理解需求 — 识别要展示的信息、目标受众、展示场景
  ↓
2. 选择模板或自由布局 — 决定页面结构
  ↓
3. 选择组件 — 为每块信息选择合适的展示组件
  ↓
4. 填充数据 — 将信息映射到组件属性
  ↓
5. 生成 Canvas Schema — 输出 JSON
  ↓
6. (可选) 触发视觉反馈闭环 — 渲染、截图、审查、修正
  ↓
7. 输出最终 Schema — 交给渲染引擎展示
```

### 6.2 Schema 生成约束

为提高生成可靠性，Agent 遵循以下约束：

- 只能使用预定义的组件类型，不能自创组件
- 每个组件的 props 必须符合 Zod schema 验证
- children 只能嵌套在支持嵌套的组件中（如 Container、Card、Grid）
- 布局属性必须使用预定义的 LayoutSpec
- 颜色不直接指定，使用语义化标签（emphasis、status、variant）

### 6.3 错误处理与降级

```
Schema 验证失败
  → 返回验证错误信息给 Agent
  → Agent 修正 Schema
  → 重新验证

组件不存在
  → 使用 Fallback 组件（显示错误提示但不崩溃）
  → 记录错误日志

渲染异常
  → 降级为纯文本展示
  → 记录错误日志
```

---

## 七、对比分析

### 7.1 与 [Claude Artifacts](https://claude.com/blog/build-artifacts) 对比

| 维度 | Claude Artifacts | Agent Canvas |
|------|-----------------|--------------|
| Agent 产物 | HTML/CSS/JS 代码 | JSON Schema |
| 生成复杂度 | 高（需处理完整前端代码） | 低（只需描述结构和数据） |
| 可靠性 | 中（代码可能有语法/逻辑错误） | 高（Schema 验证 + 确定性渲染） |
| 审美质量 | 不稳定（取决于生成代码质量） | 稳定（组件预设专业设计） |
| 灵活性 | 极高（任意 HTML/CSS/JS） | 中高（受限于组件库，但组件丰富） |
| 视觉反馈 | 无（生成后不审查） | 有（截图 + 多模态审查闭环） |
| 安全性 | 需要沙盒隔离 | 天然安全（JSON 不可执行） |

Claude Artifacts 的优势在于灵活性——可以生成任意复杂的 Web 应用。Agent Canvas 的优势在于可靠性和审美一致性——用更低的生成复杂度获得更稳定的视觉输出。

### 7.2 与 [A2UI](https://a2ui.org/) 协议对比

A2UI 是一个更通用的协议（面向所有 UI 交互），而 Agent Canvas 专注于**可视化展示**场景。关键差异：

- A2UI 面向跨平台（Flutter/Angular/React），Agent Canvas 专注 Web 渲染
- A2UI 使用扁平 JSON + 邻接表模型（优化流式生成），Agent Canvas 使用树形 JSON（更直观）
- A2UI 不含视觉反馈闭环，Agent Canvas 内置
- A2UI 是协议标准，Agent Canvas 是完整工具（含组件库 + 渲染引擎 + 反馈系统）

### 7.3 与 [Vercel v0](https://v0.app/) / [JSON-Render](https://github.com/vercel-labs/json-render) 对比

Vercel 的方案和 Agent Canvas 理念最接近——都是"约束生成 + 确定性渲染"。关键差异：

- v0 面向**开发者建站**（生成 React 组件代码），Agent Canvas 面向**Agent 可视化**（生成展示页面）
- v0 最终产物是代码（可复制到项目中），Agent Canvas 最终产物是渲染结果（直接展示）
- v0 使用 RAG + AutoFix 修复代码错误，Agent Canvas 使用视觉反馈闭环修正视觉问题
- v0 依赖 shadcn/ui 组件库，Agent Canvas 有专为可视化设计的组件库

---

## 参考文献

### 论文

- **AeSlides: Incentivizing Aesthetic Layout in LLM-Based Slide Generation via Verifiable Rewards** (2025) — [arXiv:2604.22840](https://arxiv.org/abs/2604.22840) | [Project](https://ympan0508.github.io/aeslides/)
- **LaySPA: LLMs as Layout Designers — A Spatial Reasoning Perspective** (2025) — [arXiv:2509.16891](https://arxiv.org/abs/2509.16891)
- **LayoutVLM: Differentiable Optimization of 3D Layout via Vision-Language Models** (Stanford, CVPR 2025) — [arXiv:2412.02193](https://arxiv.org/abs/2412.02193) | [Project](https://ai.stanford.edu/~sunfanyun/layoutvlm/)
- **SKE-Layout: Spatial Knowledge Enhanced Layout Generation with LLMs** (CVPR 2025) — [Paper](https://openaccess.thecvf.com/content/CVPR2025/html/Wang_SKE-Layout_Spatial_Knowledge_Enhanced_Layout_Generation_with_LLMs_CVPR_2025_paper.html)
- **MatPlotAgent: Method and Evaluation for LLM-Based Agentic Scientific Data Visualization** (ACL 2024 Findings) — [arXiv:2402.11453](https://arxiv.org/abs/2402.11453) | [ACL Anthology](https://aclanthology.org/2024.findings-acl.701/)
- **VisEval: A Benchmark for Data Visualization in the Era of Large Language Models** (2025) — [arXiv:2407.00981](https://arxiv.org/abs/2407.00981)
- **PlotCraft: Pushing the Limits of LLMs for Complex and Interactive Data Visualization** (2025) — [arXiv:2511.00010](https://arxiv.org/abs/2511.00010)
- **Portal UX Agent — A Plug-and-Play Engine for Rendering UIs from Natural Language** (2025) — [arXiv:2511.00843](https://arxiv.org/abs/2511.00843)
- **Shneiderman, B.** "The Eyes Have It: A Task by Data Type Taxonomy for Information Visualizations" (1996) — [Paper](https://www.cs.umd.edu/~ben/papers/Shneiderman1996eyes.pdf)
- **Munzner, T.** *Visualization Analysis and Design* (AK Peters, 2014) — [Book](https://www.cs.ubc.ca/~tmm/vadbook/)
- **Bertin, J.** *Sémiologie Graphique* (1967) — [Info](https://pages.graphics.cs.wisc.edu/VisSnacks/resources/bertin/)

### 项目与产品

- **A2UI (Agent-to-User Interface)** — Google 主导的开放协议 — [a2ui.org](https://a2ui.org/) | [GitHub](https://github.com/a2ui-project/a2ui)
- **Vercel JSON-Render** — 生成式 UI 框架 — [GitHub](https://github.com/vercel-labs/json-render) | [Docs](https://json-render.dev/)
- **Vercel v0** — AI 全栈应用构建器 — [v0.app](https://v0.app/)
- **Claude Artifacts** — Anthropic 的交互式内容创作 — [Blog](https://claude.com/blog/build-artifacts)

---

## 八、实施路线图

### Phase 1: MVP — 核心框架 + 基础组件（2-3 周）

目标：验证"Schema 驱动 + 确定性渲染"的可行性

- [ ] 定义 Canvas DSL 规范（Zod schema）
- [ ] 实现 Schema 验证器
- [ ] 实现基础渲染引擎（React + Tailwind）
- [ ] 实现核心组件：Container、Grid、Stack、Card、Text、Heading、Table、Chart
- [ ] 实现 Light/Dark 主题
- [ ] 基础文档和示例

### Phase 2: 组件完善 + 模板系统（2-3 周）

目标：覆盖常见可视化场景

- [ ] 扩展组件库：Timeline、Kanban、StatCard、Callout、Progress、Gauge
- [ ] 实现图表组件：Line、Bar、Pie、Scatter、Area、Radar
- [ ] 实现图组件：FlowChart、MindMap、RelationGraph
- [ ] 实现预设模板：Dashboard、Report、Comparison、Timeline
- [ ] 响应式布局支持

### Phase 3: 视觉反馈闭环（2 周）

目标：实现自动质量保障

- [ ] 实现截图服务（Playwright headless）
- [ ] 实现 Visual Reviewer（多模态 LLM 审查）
- [ ] 实现反馈修正流程（审查 → 建议 → 修正 Schema）
- [ ] 迭代次数控制和质量阈值

### Phase 4: Agent 集成 + 优化（2-3 周）

目标：与 Agent 系统集成

- [ ] 实现 Agent 提示词模板（指导 Agent 生成 Canvas Schema）
- [ ] 实现 Schema 生成约束（Zod schema 作为 function calling 参数）
- [ ] 性能优化（渲染速度、截图速度）
- [ ] 错误处理和降级机制
- [ ] 完整文档和示例库

### Phase 5: 高级特性（持续迭代）

- [ ] 动画和过渡效果
- [ ] 用户交互（点击、筛选、排序）
- [ ] 实时数据更新
- [ ] 自定义组件扩展机制
- [ ] 多页面导航
- [ ] 导出为 PDF / 图片 / HTML

---

## 九、关键设计决策记录

### 决策 1：为什么选择 JSON Schema 而非代码生成？

**决策**：Agent 生成 JSON Schema，而非 HTML/CSS/JS 代码。

**理由**：
1. JSON 是 LLM 最擅长的格式（训练数据中大量 JSON）
2. Schema 验证可以在渲染前捕获所有结构错误
3. 有界生成空间显著降低出错率（从"无限可能的代码"到"有限的组件组合"）
4. 确定性渲染保证视觉质量不受 Agent 代码质量影响
5. 安全性：JSON 不可执行，无需沙盒

**代价**：灵活性降低——无法生成组件库不支持的视觉效果。通过丰富的组件库和可扩展机制来缓解。

### 决策 2：为什么选择 React 而非其他框架？

**决策**：渲染引擎基于 React。

**理由**：
1. 组件化模型天然适配 Schema → 组件树的映射
2. 生态成熟，组件库丰富（可复用 Recharts、shadcn/ui 等）
3. TypeScript 支持优秀，类型安全
4. [Vercel v0](https://v0.app/)、[Claude Artifacts](https://claude.com/blog/build-artifacts) 等同类方案都选择 React，验证了可行性

### 决策 3：为什么内置视觉反馈闭环？

**决策**：系统内置"渲染 → 截图 → 多模态审查 → 修正"的闭环。

**理由**：
1. [MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/) 证明视觉反馈可提升约 12-13 分（百分制）
2. 这是当前主流方案（[Artifacts](https://claude.com/blog/build-artifacts)、[v0](https://v0.app/)）的核心缺失
3. 闭环机制可以自动捕获布局错误、视觉问题，显著提高最终质量
4. 多模态 LLM 的视觉理解能力已足够支持这一场景

**代价**：增加延迟（每次迭代需要渲染 + 截图 + 审查）。通过"可选启用 + 迭代次数限制"来平衡。

### 决策 4：语义化属性 vs 样式化属性

**决策**：组件属性使用语义化描述（如 `emphasis: "high"`），而非样式化描述（如 `color: "red"`）。

**理由**：
1. 降低 Agent 的决策负担——不需要理解色彩理论
2. 保证主题一致性——语义标签在不同主题下自动映射到不同样式
3. 避免 Agent 生成不和谐的样式组合

**代价**：灵活性降低。通过"高级样式覆盖"机制（ThemeSpec.overrides）为需要精细控制的场景提供逃生舱。
