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

## 十、Agent 生态集成

Agent Canvas 的三层架构和四步处理管线不是孤立的设计，它可以直接映射到当前主流 Agent 框架的工程能力上。以下分析基于 [Claude Code](https://code.claude.com/docs/en/overview) 文档体系，说明各层设计如何在工程层面落地。

### 10.1 Dynamic Workflows → 管线编排与 Token 成本优化

[Claude Code Workflows](https://code.claude.com/docs/en/workflows) 是 JavaScript 脚本，把编排逻辑从 agent 对话中抽出来——脚本自己持有循环、分支和中间结果，agent 上下文只保存最终答案。能编排数十到数百个 agent 并行运行。

**对 Agent Canvas 的价值**：我们的四步处理管线（意图解析 → 语义建模 → 数据实例化 → 视觉映射）可以做成一个 workflow 脚本：

```
workflow "generate-visualization":
  Phase 1 (intent-parse):   agent-A → Intent JSON     → 脚本变量
  Phase 2 (semantic-model):  agent-B → Semantic JSON    → 脚本变量
  Phase 3 (data-instantiate): agent-C → DataItem JSON    → 脚本变量
  Phase 4 (visual-map):     agent-D → Canvas Schema     → 最终输出
```

每一步的中间 JSON 存在脚本变量中，不回填到主 agent 上下文。这直接缓解了我们在利弊分析中指出的"token 消耗增加 2-3 倍"问题——三层中间表示不再占用主上下文。

同时 workflow 支持 adversarial review 模式（独立 agent 互相审查），这是视觉反馈闭环的 agent 级实现：一个 agent 渲染 Schema，另一个 agent 审查截图并反驳。

### 10.2 Hooks → 三层反馈的质量门禁

[Claude Code Hooks](https://code.claude.com/docs/en/hooks) 是在 Agent 生命周期特定节点自动执行的 shell 命令，支持 `PreToolUse`、`PostToolUse`、`Stop` 等事件，可以通过 exit code 2 阻止 agent 结束并返回反馈。

**对 Agent Canvas 的价值**：三层反馈系统（语义审查 / 数据审查 / 视觉审查）可以用 Stop hook 工程化落地：

| Hook 事件 | 审查层 | 校验脚本 | 阻止条件 |
|----------|--------|---------|---------|
| `Stop` | 语义层 | `check-semantic-coverage.sh` | 意图中的 what 未被语义元素完全覆盖 |
| `Stop` | 数据层 | `check-data-completeness.sh` | DataItem 缺失或值不合理 |
| `PostToolUse` (渲染后) | 视觉层 | `check-visual-quality.sh` | 截图审查发现布局/审美问题 |

agent 声称"完成"时，hook 脚本依次校验三层，任一层不通过就 exit code 2 阻止结束并返回修正建议。Claude Code 最多重试 8 次后强制结束，这与我们"最多迭代 N 次"的设计一致。

### 10.3 Subagent 模型路由 → 分层成本优化

[Claude Code Subagents](https://code.claude.com/docs/en/sub-agents) 支持为每个 subagent 指定不同模型——Haiku 做探索（便宜快），Sonnet 做实现（贵但好）。内置的 Explore subagent 在 v2.1.198 前固定用 Haiku。

**对 Agent Canvas 的价值**：三层架构的每一步对模型能力的需求不同，可以用模型路由降低成本：

| 处理步骤 | 模型选择 | 理由 |
|---------|---------|------|
| 意图解析 | Haiku | 分类任务，输入输出结构化 |
| 语义建模 | Sonnet/Opus | 需要深度理解用户意图 |
| 数据实例化 | Haiku | 结构化数据提取，模式固定 |
| 视觉映射 | Sonnet | 需要审美判断和映射决策 |
| 视觉审查 | Sonnet (多模态) | 需要视觉理解能力 |

简单场景（30% fast path）可以全部用 Haiku，复杂场景（70% full pipeline）在语义建模和视觉映射步骤升级到 Sonnet。

### 10.4 MCP → 数据层的标准化数据接入

[Model Context Protocol (MCP)](https://code.claude.com/docs/en/mcp) 是连接 AI 工具与外部数据源的开放标准。Agent Canvas 的 DataLayer 中 DataItem 的 value 有三个来源（用户输入、数据源、Agent 推理），其中"结构化数据源"通过 MCP 标准化接入。

**对 Agent Canvas 的价值**：DataLayer 可以通过 MCP 连接：

- **数据库 MCP**：PostgreSQL/MySQL → 查询 SQL 填充 DataItem
- **API MCP**：REST/GraphQL → 调用接口获取实时数据
- **文档 MCP**：Google Drive/Notion → 从文档中提取数据
- **监控 MCP**：Sentry/Datadog → 拉取指标数据

这让 Agent Canvas 不只是一个设计框架，而是可以接入真实数据源的完整系统。语义层的 entity/attribute 自然映射到 MCP 工具的参数，数据层的 DataItem 就是 MCP 调用的返回值。

### 10.5 CLAUDE.md / Auto Memory → 意图解析的偏好持久化

[Claude Code Memory](https://code.claude.com/docs/en/memory) 机制包括 CLAUDE.md（手动编写的持久指令）和 Auto Memory（agent 自动学习的工作偏好）。这些在每次会话开始时加载。

**对 Agent Canvas 的价值**：意图解析的 6 个维度中，who（受众）和 where（展示场景）对同一用户来说是稳定的偏好，不需要每次重新解析：

```markdown
# CLAUDE.md 示例

## 可视化偏好
- 受众：高管团队（CEO/VP 级别）
- 展示场景：会议室大屏 → 宽屏布局，大字体，高对比
- 数据来源：通过 MCP 连接 PostgreSQL (host: xxx)
- 色彩偏好：商务蓝主题
- 信息密度：稀疏（每屏不超过 5 个关键信息）
```

Auto Memory 还能自动学习用户的展示偏好——比如用户多次要求"数字大一点""趋势图用折线不要用柱状"，系统会自动记住这些模式。

### 10.6 Agent Teams → 组合策略的并行生成

[Claude Code Agent Teams](https://code.claude.com/docs/en/agent-teams) 让多个 Claude Code 实例协同工作，一个 lead 协调任务，teammates 各自独立运行并直接通信。

**对 Agent Canvas 的价值**：对于多区域组合的仪表盘（Level 3 组合），可以分配不同 teammates 各自负责一个区域的语义→数据→视觉映射：

```
Lead Agent:
  "生成 Q2 经营仪表盘，包含指标区、趋势区、构成区、明细区"

  → Teammate 1 (指标区): 3个KPI的语义建模+数据+视觉映射
  → Teammate 2 (趋势区): 月度趋势的语义建模+数据+视觉映射
  → Teammate 3 (构成区): 业务线构成的语义建模+数据+视觉映射
  → Teammate 4 (明细区): 区域明细表格的语义建模+数据+视觉映射

  Lead: 合并4个区域的 Canvas Schema → 布局引擎 → 最终渲染
```

teammates 可以互相审查——Teammate 1 检查 Teammate 2 的趋势数据是否与自己的 KPI 数据一致。这通过 hooks 的 `TeammateIdle` 和 `TaskCompleted` 事件实现质量门禁。

### 10.7 Plan Mode → "先理解再执行"的工程验证

[Claude Code Plan Mode](https://code.claude.com/docs/en/permission-modes) 让 agent 先只读探索、提出方案，用户审批后才执行修改。这与我们三层架构的核心理念一致——"先理解要表达什么含义（语义层），再决定怎么呈现（视觉层）"。

**对 Agent Canvas 的价值**：Plan Mode 的存在验证了"分离理解和执行"在工程上可行且有价值。Agent Canvas 可以将语义层建模作为 plan 阶段的产物——agent 先输出语义层 JSON，用户确认"这些是我要表达的含义"后，再进入数据层和视觉层。这为三层架构提供了天然的 human-in-the-loop 检查点。

---

## 十一、Agent Harness 工程方法论

> 上一章（第十章）分析了 Agent Canvas 如何映射到具体 agent 框架的工程特性。本章从更高维度出发，综合 Claude Blog、LangChain Blog、NVIDIA Blog 的最新研究成果，提炼出指导 Agent Canvas 持续演进的 Harness 工程方法论。

Agent Canvas 本质上是一个可视化生成 Harness——它围绕 LLM 构建工具和流程，将模型"参差不齐的智能"塑造为可靠的视觉输出能力。LangChain 的研究表明，[仅通过调优 Harness（而非更换模型），可以将 Terminal Bench 2.0 分数从 52.8 提升至 66.5（+13.7 分）](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)；[Nemotron 3 Ultra 通过 Harness 调优从 0.80 提升至 0.86，接近 Opus 4.8 的 0.87，成本仅 1/10](https://www.langchain.com/blog/tuning-the-harness-not-the-model-a-nemotron-3-ultra-playbook)。这些发现验证了 Agent Canvas 的核心策略：通过结构化 Harness 而非追求更强模型来提升可视化质量。

### 11.1 Harness 设计三原则

[Claude Blog 的 "Agent Harness Design: 3 Patterns"](https://claude.com/blog/agent-harness-design-3-patterns-for-harnessing-claudes-intelligence) 提出了三个模式，精准映射到 Agent Canvas 的设计决策：

**原则一：Lean on the model, not the harness（依赖模型而非 Harness）**

> "Agent harnesses encode assumptions about what Claude can't do on its own, but those assumptions grow stale as Claude gets more capable."

Agent Canvas 的三层架构正是这个理念：不在 Harness 里硬编码"如果检测到'对比'就生成 ComparisonTable"之类的规则，而是让模型自主完成语义理解→数据提取→视觉映射的决策链。Harness 只提供组件目录和映射约束（有界生成），具体的组合决策完全由模型完成。随着模型能力提升，Harness 中的硬编码规则可以逐步移除。

**原则二：Strip your harness down（精简 Harness）**

让 Agent 自己编排动作、管理上下文、持久化记忆。映射到 Agent Canvas：

- Agent 自己决定调用哪些语义元素、怎么拆数据项——不是模板填空
- Canvas Schema 是 Agent 自主编排的产物，不是 Harness 指定的输出格式
- 三层之间的中间 JSON 存储在脚本变量中（Dynamic Workflows），不污染主上下文

**原则三：Set boundaries carefully（谨慎设置边界）**

用声明式工具定义 UX/安全边界，最大化 cache hits。Agent Canvas 的有界生成正是这个原则的体现：预定义组件目录就是边界，Zod schema 验证就是声明式约束。文章强调"context engineering"——在需要时注入指导，而非在 system prompt 里堆规则。这暗示三层管线应该分层注入指导信息：

| 管线阶段 | 注入的指导 | 注入方式 |
|---------|-----------|---------|
| 语义层 | 语义元素目录 + 关系类型 | 独立的短 prompt 块 |
| 数据层 | 数据提取规则 + 类型约束 | Zod schema + 示例 |
| 视觉层 | 映射约束 + 组件目录 | 组件 API 文档 + 预设 |

### 11.2 循环工程化

[Claude Blog 的 "Loop Engineering"](https://claude.com/blog/getting-started-with-loops) 定义了四种循环模式，对 Agent Canvas 的视觉反馈闭环设计有直接启发：

| 循环类型 | 触发方式 | 停止条件 | Agent Canvas 对应 |
|---------|---------|---------|------------------|
| Turn-based | 用户 prompt | 模型自判完成 | 单次生成（无反馈） |
| Goal-based | 手动 prompt | 目标达成或轮次上限 | 视觉反馈闭环（当前设计） |
| Time-based | 时间间隔 | 取消或完成 | 批量生成任务 |
| Proactive | 事件/计划 | 每个任务独立完成 | 自动化仪表盘更新 |

关键洞察：Agent Canvas 的"渲染→截图→多模态审查→修正"应升级为 **Goal-based loop with deterministic stop criteria**。文章强调"确定性停止条件"（如 Lighthouse score ≥ 90）比"模型自己判断好不好"更有效。Agent Canvas 的视觉审查应建立量化标准：

```
视觉审查量化标准（示例）：
- 标题不遮挡数据标签（重叠面积 < 5%）
- 对齐偏差 < 4px
- 色彩对比度 ≥ WCAG AA 标准（4.5:1）
- 图表图例不溢出容器
- 文字大小不小于 12px
```

文章的 SKILL.md 验证模式可以直接用作视觉反馈层的指导原则：

```markdown
# verify-visualization
Never report a visualization as complete based on a successful render alone.
Verify it the way a human reviewer would:
1. Render the Canvas Schema and screenshot the result.
2. Check: no overlapping elements, no text overflow, no missing data labels.
3. Check: color contrast meets WCAG AA, alignment within 4px tolerance.
4. Use multimodal LLM to review overall visual quality.
If any step fails, fix the Schema and rerun from step 1.
```

Token 管理建议："Use scripts for deterministic work — Running a script is cheaper than reasoning through the steps"——这验证了确定性渲染引擎的设计：渲染不该让模型推理，应该跑脚本。

### 11.3 Eval 驱动的持续改进

LangChain 的三篇文章构建了一个完整的 Agent 持续改进方法论：

**改进循环（来自 [Harness Engineering](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)）**

LangChain 用一个简单配方将 coding agent 在 Terminal Bench 2.0 上提升了 13.7 分：

```
1. 运行 eval → 收集 Trace
2. 并行分析错误 → 人工综合发现 + 建议
3. 聚合反馈 → 定向修改 Harness（System Prompt / Tools / Middleware）
4. 重新运行 eval → 验证改进 + 检查回归
5. 回到 1
```

这个循环可以直接用于 Agent Canvas 的持续改进：建立可视化质量评估集（类似 [VisEval](https://arxiv.org/abs/2407.00981)），用 eval 分数驱动三层架构的迭代。

**三明治模式（来自 [Data Mining](https://www.langchain.com/blog/improving-agents-is-a-data-mining-problem)）**

> "Harness Engineering → Fine-Tuning → Harness Engineering"

```
阶段一：Harness 调优
  - 完善三层架构、组件库、映射规则
  - 快速迭代，即时反馈
  - 遇到智能天花板时进入阶段二

阶段二：模型微调
  - 收集好的生成 Trace，蒸馏专用模型
  - 例如：微调一个"可视化语义解析"专用小模型
  - 降低成本的同时保持质量

阶段三：Harness 再调优
  - 用微调后的模型重新调整 Harness
  - 验证泛化能力
```

**关键原则：Evals are training data for harness work（来自 [Harness Tuning](https://www.langchain.com/blog/tuning-the-harness-not-the-model-a-nemotron-3-ultra-playbook)）**

> "Evals are the training data for harness work."

eval 不仅是测量工具，更是改进 Harness 的"训练数据"。每个 agent 失败案例都应转化为一个 eval case，每次修复都应增加 eval 覆盖。这形成了一个正向循环：失败 → 建 eval → 改 Harness → eval 通过 → 覆盖度增加 → 未来回归被自动捕获。

**Trace Mining（来自 [LangSmith Engine](https://www.langchain.com/blog/introducing-langsmith-engine)）**

LangSmith Engine 自动化了整个改进循环：

```
监控生产 Trace → 聚类失败模式为命名 Issue → 诊断根因 → 三个解决方案：
  (1) 开 PR 修复代码/prompt
  (2) 创建自定义 online evaluator（防止回归）
  (3) 将失败 Trace 加入 offline eval 数据集
```

Agent Canvas 应记录每次生成过程的完整 Trace（用户需求 → 语义元素 → 数据项 → Canvas Schema → 渲染结果 → 审查反馈 → 修正记录），这些 Trace 是改进系统的金矿。"Traces are the currency of long-horizon agent improvement"。

**重要限制：Harness 调优有天花板**

> "Harness tuning has a ceiling — it can't add what isn't in the weights."

即使 Harness 再好，如果模型本身没有空间推理能力，三层架构也帮不了太多。这为模型路由策略提供了依据：语义理解用强模型，确定性渲染用脚本，视觉审查用多模态模型。

### 11.4 分层上下文工程

[Claude Blog 的 "Steering Claude Code"](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more) 定义了 7 种指令传递方法，每种在上下文成本和权威性上有不同权衡：

| 方法 | 加载时机 | 压缩行为 | 上下文成本 | Agent Canvas 对应 |
|------|---------|---------|-----------|------------------|
| CLAUDE.md（根） | 会话开始 | 压缩后重读 | 高（每行都消耗 token） | 用户可视化偏好 |
| CLAUDE.md（子目录） | 按需 | 丢失直到再访问 | 低 | 特定组件库的约定 |
| Rules | 开始/路径触发 | 压缩后重注入 | 中 | "所有图表必须验证输入" |
| Skills | 调用时加载主体 | 按预算重注入 | 低 | 视觉审查流程 |
| Subagents | 调用时加载 | 仅返回最终消息 | 零（主上下文） | 并行区域生成 |
| Hooks | 生命周期事件 | 完全绕过压缩 | 低 | 质量门禁 |

关键原则：**"Keep CLAUDE.md under 200 lines"**——每行都在每个会话中消耗 token，应只放最核心的全局信息。路径限定的 Rules 只在相关文件被触及时加载，适合"所有 API handler 必须用 Zod 验证输入"这类跨切关注点。

[LangChain 的 Deep Agents 文章](https://www.langchain.com/blog/improving-deep-agents-with-harness-engineering)进一步指出："The purpose of the harness engineer: prepare and deliver context so agents can autonomously complete work." 三种上下文注入中间件：

```
1. PreCompletionChecklistMiddleware
   - 拦截 agent 退出，提醒运行验证
   - 映射：视觉反馈闭环的 "Stop" hook

2. LocalContextMiddleware
   - 启动时注入目录结构、可用工具
   - 映射：语义层注入可用组件目录 + 数据源列表

3. LoopDetectionMiddleware
   - 追踪每文件编辑次数，N 次后提示 "考虑换个方法"
   - 映射：视觉修正循环的 "最大 N 次迭代" + "doom loop" 检测
```

文章还发现了一个重要的失败模式：**agents 不会自然进入 build-verify 循环**——它们倾向于"写完代码→重读一遍→觉得没问题→停止"，而不是"写完→运行测试→对比规格→修复"。这解释了为什么当前 agent 可视化方案大多是"盲生成"——agent 生成完代码就停了，不会主动截图检查。Agent Canvas 的视觉反馈闭环正是对这个失败模式的工程补丁。

### 11.5 模型与努力级别选择

[Claude Blog 的 "Choosing a model and effort level"](https://claude.com/blog/claude-model-and-effort-level-in-claude-code) 区分了两个正交的维度：

- **模型选择** = 哪组冻结权重处理请求（能力天花板）
- **努力级别** = 模型做多少工作（读多少文件、用多少工具、验证多深）

诊断启发式：

```
如果 Claude 有所有上下文且确实尝试了，但还是错了
  → 问题太难 → 选更大模型

如果 Claude 跳过了文件、没跑测试、半途而废
  → 不够努力 → 选更高努力级别
```

用类比理解：Fable 是见过几乎所有人没见过的问题的**专家**，Opus 是经验丰富的**行家**，Sonnet 是非常好的**通才**。努力级别决定他们中任何一个花多少时间在你的任务上。

**推理三明治（Reasoning Sandwich）**：LangChain 的实验发现 xhigh-high-xhigh 是最优配置——在规划和验证阶段用最高推理，在实现阶段用中等推理。映射到 Agent Canvas 管线：

| 管线阶段 | 推理级别 | 模型选择 | 理由 |
|---------|---------|---------|------|
| 意图解析 | high | Sonnet/Opus | 需要理解复杂意图 |
| 语义建模 | xhigh | Opus/Fable | 核心决策，影响后续所有步骤 |
| 数据实例化 | medium | Haiku/Sonnet | 结构化提取，规则明确 |
| 视觉映射 | high | Sonnet/Opus | 需要审美判断 |
| 视觉审查 | xhigh | 多模态模型 | 需要深度视觉理解 |
| 简单场景（fast path） | low | Haiku | 30% 的简单场景用便宜模型快速完成 |

**跨模型 Harness 适配**：LangChain 强调不同模型需要不同的 Harness 配置——Codex 和 Claude 的 prompting guide 不同，直接套用会降低表现。Agent Canvas 应为不同模型维护不同的 prompt 模板，而非一刀切。

### 11.6 成本可观测性与优化

[LangChain 的 "Your coding agent bill doubled"](https://www.langchain.com/blog/fix-your-coding-agent-bill) 揭示了行业现状：Uber 4 个月耗尽全年 AI 预算、Microsoft 取消 Claude Code 许可、Salesforce 面对 $300M 账单。核心问题是"碎片化"——不同工具各自记录活动，无法回答"构建这个功能实际花了多少"。

Agent Canvas 的三层架构天然提供了统一的成本追踪粒度：

```
成本追踪粒度：
  意图解析阶段: input_tokens + output_tokens → 模型 × 单价
  语义建模阶段: input_tokens + output_tokens → 模型 × 单价
  数据实例化阶段: MCP 调用成本 + 模型 token 成本
  视觉映射阶段: input_tokens + output_tokens → 模型 × 单价
  渲染阶段: 确定性脚本，零模型成本
  视觉审查阶段: 截图 + 多模态模型 token 成本
  修正循环: × N 次迭代
  ─────────────────────────
  总成本 = Σ(各阶段成本)
```

[NVIDIA 的 Agentic Inference 研究](https://developer.nvidia.com/blog/full-stack-optimizations-for-agentic-inference-with-nvidia-dynamo/)提供了底层优化数据：

- Claude Code 单 worker 缓存命中率 85-97%，Agent Teams 4 个 Opus teammates 聚合缓存命中率 97.2%
- 读写比 11.7:1（write-once-read-many 模式）——系统 prompt 和会话前缀只计算一次，后续全部从缓存读取
- Agent Hints 机制：Harness 向推理编排器传递结构化提示（priority、output sequence length、speculative prefill），让底层做 agent 感知的调度

这验证了 Dynamic Workflows 将中间 JSON 存储在脚本变量中的设计——最大化缓存命中，减少重复计算。

### 11.7 大规模并行生成

[Claude Blog 的 "Large-Scale Code Migrations"](https://claude.com/blog/ai-code-migration) 展示了 Agent Canvas 并行生成的终极形态。Anthropic 内部用 Claude Code 完成了 Bun 的百万行 Zig→Rust 迁移（5.9B token，约 $165K），核心方法论可以直接迁移到可视化生成：

**核心洞察："You fix the process (loop) that produced the code, not the code itself."**

当可视化质量不达标时，不应逐个修复——应修复生成流程本身。每次发现的视觉问题都应转化为：(1) 一个 eval case，(2) 一个 Hook 规则，(3) 一个 prompt 改进。

**五个适用条件（原文用于代码迁移，同样适用于可视化生成）**：

1. **工作是可并行的**——多区域仪表盘的不同区域可以独立生成
2. **上下文清晰且完整**——旧的可视化/数据就是最好的 spec
3. **有内置裁判**——测试套件/视觉审查提供客观验证标准
4. **队列自己生成**——当渲染失败或审查不通过，这成为下一个待修复项
5. **需要一致性和边界处理**——规则化确保 drift 无处藏身

**六步流程（适配可视化生成）**：

```
1. 前置：建立强裁判
   - 将现有可视化标准转化为可执行的验证规则
   - 用对抗性 agent 验证规则不会放过错误

2. 生成翻译指南
   - 让强模型分析"用户需求 → 高质量可视化"的映射模式
   - 产出一份 guide 供后续 agent 遵循

3. 并行执行
   - Dynamic Workflows 编排多个 agent 同时生成不同区域
   - 每个 agent 独立完成 语义→数据→视觉 管线

4. 对抗性审查
   - 独立 agent 用裁判规则逐项检查
   - 每个发现引用背后的规则

5. 修复队列
   - 审查发现自动成为待修复项
   - 修复后规则更新，后续 agent 自动遵守

6. 同等性验证
   - 最终可视化与用户意图做 diff
   - 确保语义覆盖完整
```

### 11.8 Agent 执行环境与沙盒

[LangChain 的 "Agents need their own computer"](https://www.langchain.com/blog/agents-need-their-own-computer) 提出了 Agent 可视化生成中一个被忽视的问题：Agent 需要一个真正的执行环境来验证自己的输出。

> "A system that can only produce text is like a contractor who can describe exactly how to fix your plumbing but has no hands, no tools, and no truck."

Agent Canvas 的视觉反馈闭环（渲染 → 截图 → 审查 → 修正）本质上就需要这样的执行环境：

```
Agent Canvas 执行环境需求：
1. 安全执行 — 渲染引擎运行的代码可能来自 Agent 生成的 Schema，需隔离
2. 状态控制 — 截图服务、渲染引擎、审查模型各有不同的资源限制
3. 可观测性 — 记录每次渲染的命令、文件变更、网络调用、输出结果
4. 快速迭代 — 渲染 + 截图的延迟直接影响反馈闭环的响应速度
```

文章的四个核心需求映射到 Agent Canvas：

| 需求 | LangChain 定义 | Agent Canvas 对应 |
|------|---------------|------------------|
| 安全执行 | 每个 Agent 工作区是硬件虚拟化机器 | 渲染引擎运行在隔离沙盒中，Schema 不可执行恶意代码 |
| 状态控制 | 凭证管理、资源限制、生命周期控制 | 渲染资源限制（内存/CPU/超时）、API 密钥代理注入 |
| 可观测性 | 审计日志：命令、文件、网络、包 | 渲染日志 + 截图版本 + 审查反馈完整链路 |
| 快速迭代 | 亚秒级启动、可复现、持久状态 | 渲染引擎预热 + Canvas Schema 版本化 |

文章还提出了一个重要的安全洞察——**Prompt Injection 在沙盒工作流中的风险**：沙盒提供执行隔离，但不改变 LLM 的一个基本特性——Agent 读到的任何内容都可能影响它下一步做什么。Agent Canvas 在视觉审查环节也需要注意：多模态 LLM 审查截图时，如果截图中包含恶意注入的文字（如"忽略之前的指令"），可能影响审查结果。缓解措施包括：将审查输出结构化（Zod schema 验证），不直接执行审查建议而是经过中间层过滤。

快照与分叉（Snapshot and Fork）模式对 Agent Canvas 的视觉反馈闭环特别有价值：

```
快照与分叉模式：
1. 渲染 Canvas Schema → 截图 → 快照 A
2. 从快照 A 分叉 3 个并行修正方案：
   - 方案 1: 调整布局
   - 方案 2: 调整颜色
   - 方案 3: 调整字号
3. 各自渲染 + 截图 → 审查
4. 选最佳方案作为最终输出
```

### 11.9 程序化编排模式

[LangChain 的 "Introducing Dynamic Subagents"](https://www.langchain.com/blog/introducing-dynamic-subagents-in-deep-agents) 和 ["How to Use RLMs in Deep Agents"](https://www.langchain.com/blog/how-to-use-rlms-in-deep-agents) 提出了六种程序化编排模式，这些模式可以直接用于 Agent Canvas 的复杂可视化生成：

| 模式 | LangChain 定义 | Agent Canvas 应用场景 |
|------|---------------|---------------------|
| Classify and act | 按类型路由到专家 | 混合需求分类：表格类→表格专家，图表类→图表专家，布局类→布局专家 |
| Fanout and synthesize | 并行处理后合并 | 多区域仪表盘：各区域独立生成，Lead 合并 |
| Adversarial verification | 先发现再独立验证 | 视觉审查：生成 agent 产出 + 独立审查 agent 验证，只保留通过双重确认的结果 |
| Generate and filter | 生成多个方案，评分选优 | 同一需求生成 3 个视觉方案，多模态 LLM 评分选最佳 |
| Tournament | 两两对决，淘汰制 | 主观审美标准下的方案选择 |
| Loop until done | 重复扫描直到无新发现 | 完整性检查：反复扫描直到所有语义元素都有对应视觉表达 |

[RLM 论文](https://arxiv.org/abs/2512.24601) 的核心洞察——**"确定性覆盖"（deterministic coverage）**——对 Agent Canvas 尤其重要：

> "Coverage is guaranteed by code, not model judgment. A `for b in batches` loop touches every batch by construction, whereas a plain model has a hard time performing iterations like this at scale."

在 Agent Canvas 中，这意味着：确保所有语义元素都有对应的 DataItem、所有 DataItem 都有对应的 Mark、所有语义关系都有对应的 Relation——这些覆盖性检查应该用代码循环保证，而不是依赖模型的自我检查。

LangChain 的 OOLONG 基准测试数据验证了程序化编排在大规模场景下的价值：

```
OOLONG 基准测试结果（AgNews 数据集）：
- 64K token:  普通 agent 0.58 vs RLM agent 0.67
- 128K token: 普通 agent 0.44 vs RLM agent 0.79

→ 上下文越长，程序化编排的优势越大
→ Agent Canvas 的多区域仪表盘（大量 DataItem）天然适合程序化编排
```

### 11.10 Prompt Caching 策略

[LangChain 的 "Prompt Caching with Deep Agents"](https://www.langchain.com/blog/deep-agents-prompt-caching) 提供了 Agent Canvas 成本优化的量化数据：

> "If I had to choose just one metric, I'd argue that the KV-cache hit rate is the single most important metric for a production-stage AI agent." — Manus AI

Deep Agents 的实际测试结果：

| 模型 | 缓存策略 | 成本降低 |
|------|---------|---------|
| claude-haiku-4-5 | Anthropic 显式断点 | -77% |
| gpt-5.4-mini | OpenAI 自动最长前缀 | -80% |
| gemini-3.5-flash | Gemini 隐式缓存 | -49% |

**对 Agent Canvas 的关键启示**：

1. **三层架构的静态前缀天然适合缓存** — 语义元素目录、组件 API 文档、映射规则在会话中不变，应作为缓存前缀

2. **缓存友好的 prompt 结构** — Deep Agents 的做法是将静态部分（工具描述、技能、系统 prompt）放在前面，动态部分（用户消息、中间结果）放在后面，并在两者之间设置缓存断点。Agent Canvas 的管线 prompt 应遵循同样结构：

```
[缓存断点 A] 系统指令 + 组件目录 + 映射规则（会话内不变）
[缓存断点 B] 用户偏好 + 历史上下文（低频变化）
[无缓存]     当前请求 + 中间 JSON（高频变化）
```

3. **长会话收益更大** — "caching pays off more the longer a conversation runs" — Agent Canvas 的视觉修正循环（多轮渲染-审查-修正）天然是长会话，缓存收益显著

4. **缓存降级最小化** — 当 Memory 更新或上下文压缩时，会导致缓存失效（cache bust）。Deep Agents 通过结构化 prompt 和显式断点使得即使部分前缀变化，仍能命中子集缓存。Agent Canvas 应将三层管线的指导信息分别设置缓存断点，确保一层变化不影响其他层的缓存

### 11.11 Managed Agent 基础设施

[Claude Blog 的 "Claude Managed Agents"](https://claude.com/blog/claude-managed-agents) 和 [Claude Blog 的 "CLAUDE.md files"](https://claude.com/blog/using-claude-md-files) 提供了 Agent Canvas 生产化部署的参考架构。

**Managed Agents 架构**：Anthropic 提供了一套组合式 API，包含：

- 沙盒化代码执行
- 长时间运行会话（自主运行数小时，断线后状态持久化）
- 多 Agent 协调（Agent 生成并指挥其他 Agent）
- 治理（作用域权限、身份管理、执行追踪）

内部测试显示，Managed Agents 在结构化文件生成任务上比标准 prompting loop 提升最多 10 分，最大提升出现在最难的问题上——这与 Agent Canvas 在复杂场景（70% 的中高复杂度 case）上收益最大的发现一致。

**对 Agent Canvas 的启示**：Agent Canvas 可以作为 Managed Agent 上的一个应用部署——用户定义可视化需求和数据源，Managed Agent 基础设施处理沙盒、会话持久化、权限管理，Agent Canvas 负责三层管线和视觉反馈闭环。$0.08/session-hour 的定价模型为 Agent Canvas 的商业化提供了参考。

**CLAUDE.md 最佳实践**（来自 [CLAUDE.md files 指南](https://claude.com/blog/using-claude-md-files)）：

```markdown
# Agent Canvas — CLAUDE.md 示例

## 项目概述
Agent Canvas 是一个可视化生成框架，Agent 生成 Canvas Schema (JSON)，
确定性渲染引擎负责视觉输出。

## 关键目录
- src/semantic/    — 语义层（8 种元素 + 5 种角色 + 8 种关系）
- src/data/        — 数据层（DataItem 三元组）
- src/visual/      — 视觉语法层（4 种原语）
- src/render/      — 确定性渲染引擎
- src/review/      — 视觉反馈闭环（截图 + 多模态审查）

## 编码规范
- 所有 Schema 用 Zod 验证
- 组件属性用语义化描述（emphasis: "high"），非样式化（color: "red"）
- 渲染引擎用 CSS Grid/Flexbox，不用绝对定位

## 常用命令
npm run dev        # 开发服务器
npm run render     # 渲染 Canvas Schema
npm run review     # 视觉审查
npm test           # 测试套件

## 工作流
1. 理解需求 → 输出语义层 JSON
2. 用户确认语义 → 提取数据项
3. 视觉映射 → 生成 Canvas Schema
4. 渲染 + 截图 → 多模态审查
5. 如有问题 → 修正 Schema → 回到 4
```

文章的核心建议"Start simple, expand deliberately"与 Agent Canvas 的分层使用策略一致——先实现 fast path（30% 简单场景），再逐步完善 full pipeline（70% 复杂场景）。`/init` 命令的思路也可以借鉴——Agent Canvas 可以提供一个 `/init` 命令分析用户的数据源和可视化需求，自动生成初始配置。

**[Skills 完整指南](https://claude.com/blog/complete-guide-to-building-skills-for-claude) 的启示**：Agent Canvas 的视觉审查流程应该封装为 Skill——一个可复用的 SKILL.md 定义"如何验证可视化质量"，在每次生成后自动触发。Skill 的按需加载特性（会话开始只加载名称和描述，调用时才加载主体）确保不会浪费上下文。

### 11.12 持续学习的三层模型

[Harrison Chase 的 "Continual learning for AI agents"](https://www.langchain.com/blog/continual-learning-for-ai-agents) 提出了一个清晰的 Agent 持续学习框架，直接映射到 Agent Canvas 的三层架构：

| 学习层 | 定义 | 技术 | Agent Canvas 对应 |
|--------|------|------|------------------|
| Model | 模型权重 | SFT、GRPO、LoRA | 微调"可视化语义解析"专用模型 |
| Harness | 驱动 agent 的代码+固定指令+工具 | 代码改进、prompt 调优、工具增减 | 三层管线代码、Zod schema、组件库 |
| Context | 配置 harness 的外部指令/技能 | 记忆更新、技能增删 | CLAUDE.md、用户偏好、组件文档 |

核心洞察：**学习不只发生在模型层**。Harness 层的改进（改 prompt、加工具、调代码）和 Context 层的改进（更新记忆、增加技能）往往比模型微调更快见效且成本更低。

[Meta-Harness 论文](https://yoonholee.com/meta-harness/) 的方法特别值得关注：agent 在循环中运行任务 → 评估结果 → 将日志存入文件系统 → 让 coding agent 审查这些 Trace 并建议 harness 代码修改。这形成了一个 **agent 改进 agent 的元循环**。

Agent Canvas 的持续学习策略应遵循三明治模式：

```
阶段一：Context 层学习（最快见效）
  - 从生成 Trace 中提取用户偏好 → 更新 CLAUDE.md
  - 从审查反馈中提取视觉规则 → 增加 Skill
  - 从失败 case 中提取约束 → 更新 Zod schema

阶段二：Harness 层学习（中等成本）
  - 用 Trace 分析三层管线的瓶颈
  - 调整 prompt 结构、增加中间件、优化缓存断点
  - 用 coding agent 自动审查 Trace 并建议代码修改（Meta-Harness）

阶段三：Model 层学习（最高成本）
  - 收集好的生成 Trace 作为 SFT 数据
  - 微调专用"可视化语义解析"模型
  - 回到阶段一验证泛化
```

学习可以在不同粒度进行：

| 粒度 | 示例 | Agent Canvas 应用 |
|------|------|------------------|
| Agent 级 | agent 自身持久记忆 | Agent Canvas 的全局视觉偏好 |
| 用户级 | 每个用户的个性化记忆 | 用户 A 偏好暗色主题，用户 B 偏好简洁布局 |
| 组织级 | 团队共享的上下文 | 研发团队偏好数据密集型，高管团队偏好摘要型 |

两种更新模式：
- **离线作业**（offline job）：定期扫描最近 Trace，提取洞察，更新 context——类似 OpenClaw 的 "dreaming"
- **热路径**（hot path）：agent 在执行任务时主动更新自己的记忆——用户说"下次用柱状图"时立即记录

### 11.13 Wiki Memory — Agent 的知识基

[Harrison Chase 的 "Wiki Memory"](https://www.langchain.com/blog/wiki-memory) 和 [OpenWiki Brains](https://www.langchain.com/blog/introducing-openwiki-brains-general-purpose-wiki-memory-for-agents) 提出了 Agent 记忆的一个新模式，对 Agent Canvas 有直接价值。

核心概念：Wiki Memory 不是原始数据的检索（RAG），而是 **agent 维护的、预计算的、结构化的知识层**。

> "Raw data contains a lot of knowledge, but it is often inefficient to expose directly to an agent. So instead, we run a process over that data and transform it into a denser representation."

> "For every domain there exists a knowledge base you would be well served to create. This knowledge base is not just the raw data. It is an intelligently compressed version of the raw data."

Agent Canvas 应该维护的可视化 Wiki：

```
Agent Canvas Wiki Memory 结构：

/wiki/
  /patterns/
    comparison.md      — "对比"语义的可视化模式：何时用雷达图/柱状图/热力图
    flow.md            — "流转"语义的可视化模式：何时用桑基图/漏斗图/流程图
    distribution.md    — "分布"语义的可视化模式：何时用散点图/箱线图/直方图
  /components/
    statcard.md        — StatCard 组件的使用指南、属性、最佳实践
    chart.md           — Chart 组件的配置、数据格式、常见陷阱
  /design-decisions/
    why-bounded-gen.md — 为什么选择有界生成
    why-three-layers.md — 为什么分三层
  /user-preferences/
    global.md          — 全局可视化偏好
    user-{id}.md      — 用户级个性化偏好
```

Wiki Memory 与 RAG 的关键区别：

| 维度 | RAG | Wiki Memory |
|------|-----|-------------|
| 时机 | 查询时检索原始块 | 预计算高层综合 |
| 格式 | 原始文本块 | 结构化、agent 友好 |
| 维护 | 被动更新索引 | Agent 主动维护和更新 |
| 成本 | 每次查询消耗 token | 预计算后查询成本低 |

文件是最佳载体——"inspectable, editable, versionable, and easy for agents to read and write"。Agent Canvas 的 Wiki 应该用 Markdown 文件存储，版本化在 Git 中，agent 可以主动读取和更新。

### 11.14 模型中立性

[LangChain 的 "Why Model Neutrality Matters More Than Cloud Neutrality"](https://www.langchain.com/blog/model-neutrality) 论证了为什么 Agent Canvas 必须保持模型中立。

核心论证：

> "The labs are selling you tokens. Tokens are a commodity. So their next move is to capture you at the harness... If they own the orchestration layer your business logic lives in, you keep consuming their tokens even when a better, cheaper model exists."

> "Model neutrality matters more than cloud neutrality did. The rate of change is fundamentally different — labs are leapfrogging each other every quarter. Models are selectively commoditizing — Anthropic is ahead on coding, OpenAI on multimodal. The right answer is often to use more than one model in the same workflow."

模型中立性对 Agent Canvas 的三个要求：

**1. 开源 Harness** — Agent Canvas 的三层管线代码应该开源，不绑定任何模型 API。Canvas Schema 是标准 JSON，不是任何模型的私有格式。

**2. 多模型支持** — 三层管线的每个阶段可以使用不同模型：

```
管线阶段          模型选择
─────────────────────────────
意图解析          Claude / GPT / Gemini / 开源模型
语义建模          强模型（Claude Opus / GPT-4）
数据实例化        便宜模型（Haiku / GPT-4-mini）
视觉映射          中等模型（Sonnet / GPT-4）
视觉审查          多模态模型（GPT-4o / Gemini）
```

**3. Profile-aware，非最低公约数** — 中立性不意味着假装所有模型一样。每个模型有自己的"个性"——不同的 prompt 模式、工具调用风格、强弱领域。Agent Canvas 应为不同模型维护不同的 profile（prompt 模板、工具描述格式、调用参数），而非一刀切。

[NemoClaw Blueprint](https://www.langchain.com/blog/langchain-and-nvidia-launch-the-nemoclaw-deep-agents-blueprint) 的核心发现验证了这个方向：

> "Agent performance improves when the model, harness, evals, and runtime are tuned together."

Nemotron 3 Ultra + 调优后的 Deep Agents harness = 0.86 分 / $4.48，而最强模型 = 0.87 分 / $43.48（10 倍成本差异）。关键不是哪个模型更强，而是 **harness 与模型协同调优**。

### 11.15 Trace 驱动的可观测性

["Your coding agents are a black box"](https://www.langchain.com/blog/your-coding-agents-are-a-black-box-heres-how-to-crack-them-open) 和 ["Agent observability needs feedback to power learning"](https://www.langchain.com/blog/agent-observability-needs-feedback-to-power-learning) 共同构建了 Agent Canvas 的可观测性框架。

**核心洞察：Trace 不是调试工具，而是学习的原料**

> "The deeper role of agent observability is to power learning. But traces alone do not create that loop. You also need feedback: signals that tell you whether the agent's behavior was useful, accepted, rejected, inefficient, risky, or wrong."

Agent Canvas 的完整可观测性框架：

```
Trace（发生了什么）          Feedback（是否正确）
─────────────────────         ──────────────────────
用户需求                      用户是否满意
语义元素 JSON                 语义是否准确
数据项 JSON                   数据是否完整
Canvas Schema                 Schema 是否通过验证
渲染截图                      视觉审查是否通过
审查反馈                      审查发现的问题
修正记录                      修正是否解决问题
token 消耗                    成本是否在预算内
执行时长                      延迟是否可接受
```

**Trace + Feedback → 三层学习**：

```
Trace + Feedback
    ↓
分析失败模式
    ↓
┌─────────────────────────────────────────┐
│ Context 层                              │
│ - 提取规则 → 增加到 CLAUDE.md / Skill   │
│ - 提取偏好 → 更新用户记忆                │
│ - 失败 case → 加入 eval 数据集           │
├─────────────────────────────────────────┤
│ Harness 层                              │
│ - 诊断管线瓶颈 → 优化 prompt 结构        │
│ - 发现重复失败 → 增加 Hook 规则          │
│ - 用 coding agent 审查 Trace → 改代码    │
├─────────────────────────────────────────┤
│ Model 层                                │
│ - 好的 Trace → SFT 数据                  │
│ - 坏的 Trace → DPO 负样本                │
│ - 蒸馏专用小模型                         │
└─────────────────────────────────────────┘
```

**标准化 Trace Schema**：LangSmith 的方法是将不同 agent（Claude Code、Codex、Cursor、Copilot）的 Trace 映射到统一 schema。Agent Canvas 的 Trace schema 应包含：

```typescript
interface AgentCanvasTrace {
  sessionId: string;
  timestamp: string;
  
  // 用户输入
  userRequest: string;
  userFeedback?: "positive" | "negative" | "neutral";
  
  // 三层管线
  semanticLayer: {
    elements: SemanticElement[];
    relations: SemanticRelation[];
    modelUsed: string;
    tokensIn: number;
    tokensOut: number;
    durationMs: number;
  };
  
  dataLayer: {
    items: DataItem[];
    source: "user_provided" | "mcp_fetched" | "agent_generated";
    modelUsed: string;
    tokensIn: number;
    tokensOut: number;
  };
  
  visualLayer: {
    schema: CanvasSchema;
    grammarPrimitives: { marks: number; relations: number; boundaries: number };
    modelUsed: string;
    tokensIn: number;
    tokensOut: number;
  };
  
  // 渲染与反馈
  render: {
    durationMs: number;
    screenshotUrl: string;
  };
  
  review: {
    passed: boolean;
    issues: string[];
    corrections: number;
    finalScore?: number;
    modelUsed: string;
  };
  
  // 成本
  totalCost: number;
  cacheHitRate: number;
}
```

["Your harness, your memory"](https://www.langchain.com/blog/your-harness-your-memory) 的核心警示也适用于 Agent Canvas：

> "If you use a closed harness, especially if it's behind an API, you don't own your memory. Memory creates lock-in. Without memory, your agents are easily replicable by anyone who has access to the same tools."

Agent Canvas 必须保持开放：Canvas Schema 是标准 JSON、Trace 是标准格式、Wiki Memory 是 Markdown 文件、记忆存储在用户控制的数据库中。这确保用户不会被锁定在特定模型或平台。

### 11.16 Dreaming 与 Outcomes — 自我改进的质量门禁

[Claude Blog 的 "New in Claude Managed Agents: dreaming, outcomes, and multiagent orchestration"](https://claude.com/blog/new-in-claude-managed-agents) 引入了两个直接适用于 Agent Canvas 的机制：

**Dreaming（做梦）** — 在会话之间运行的后台进程，审查 agent 的历史 session 和 memory store，提取模式，整理记忆，让 agent 随时间自我改进。Dreaming 可以发现单个 agent 看不到的模式：重复犯的错误、多个 agent 收敛到的工作流、团队共享的偏好。

```
Agent Canvas 的 Dreaming 设计：
1. 定期扫描最近的生成 Trace（如每天一次）
2. 聚类失败模式：
   - "柱状图标签重叠"出现 15 次 → 增加 Hook 规则
   - "暗色主题对比度不足"出现 8 次 → 更新组件预设
   - "多区域仪表盘对齐偏差"出现 5 次 → 增加 layout 验证步骤
3. 提取偏好模式：
   - 60% 的用户偏好暗色主题 → 设为默认
   - 80% 的仪表盘包含至少一个 StatCard → 加入默认模板
4. 整理 Memory Store：去重、压缩、保持高信号
5. 两种模式：自动更新（快速迭代）或人工审核后更新（安全优先）
```

这与 11.12 节的"持续学习三层模型"中的"离线作业"模式完全对应。Dreaming 是 Context 层学习的自动化实现。

**Outcomes（结果评估）** — 开发者编写一个 rubric（评分标准）描述"成功长什么样"，agent 朝这个目标工作。一个独立的 grader（评分器）在单独的上下文窗口中评估输出，不受 agent 推理过程的影响。当不达标时，grader 指出具体需要改什么，agent 再试一次。

```
Agent Canvas 的 Outcomes 设计：
1. 定义可视化质量 rubric：
   - 所有数据标签可读（不重叠、不遮挡）
   - 色彩对比度 ≥ WCAG AA（4.5:1）
   - 布局对齐偏差 < 4px
   - 所有语义元素都有视觉表达
   - 图表类型匹配数据语义（对比→柱状/雷达，流转→桑基/漏斗）
2. 生成 agent 产出 Canvas Schema → 渲染 → 截图
3. 独立 grader agent 在单独上下文中审查截图
4. 如不达标 → grader 指出具体问题 → 生成 agent 修正 → 回到 2
5. 达标 → 输出最终结果
```

Anthropic 内部测试：Outcomes 在结构化文件生成任务上将成功率提升最多 10 分，最大提升出现在最难的问题上。docx +8.4%，pptx +10.1%——这与 Agent Canvas 在复杂场景（70%）收益最大的发现一致。

关键设计决策：**grader 必须在独立上下文窗口中运行**——不接触生成 agent 的推理链，避免 confirmation bias。这验证了 Agent Canvas 视觉审查层的设计：多模态审查模型应该是独立调用，而不是让生成模型自己审查自己。

### 11.17 未知管理 — 从 Fable 5 学到的提示工程

[Claude Blog 的 "A field guide to Claude Fable 5: Finding your unknowns"](https://claude.com/blog/a-field-guide-to-claude-fable-finding-your-unknowns) 提出了一个对 Agent Canvas 直接有用的认知框架——**四种未知**：

| 类型 | 定义 | Agent Canvas 中的表现 |
|------|------|---------------------|
| Known Knowns | 用户明确告诉 agent 的 | "做一个销售仪表盘，包含营收、趋势、TOP10" |
| Known Unknowns | 用户知道但还没想清楚的 | "我要对比三个方案，但还没想好用什么图表" |
| Unknown Knowns | 太明显以至于不会写出来，但看到就会认出 | "数据标签不该挡住数据点" |
| Unknown Unknowns | 完全没考虑到的 | "响应式布局在小屏幕上柱状图变成了窄条" |

> "Claude Fable is the first model where I find the quality of the work is bottlenecked by my ability to clarify its unknowns."

**对 Agent Canvas 的启示**：

1. **盲点扫描（Blind Spot Pass）** — 在语义建模之前，让 agent 主动发现用户的 unknown unknowns：

```
用户："做一个销售概览"
Agent Canvas 的盲点扫描：
  "你提到了销售概览，但以下问题我还不清楚：
   - 时间范围？（本月？本季度？全年对比？）
   - 对比维度？（同比？环比？目标完成率？）
   - 受众？（高管需要摘要，运营需要明细）
   - 这些是你在提示中可能没想到的，但会影响可视化设计。"
```

2. **原型发散** — 对 unknown knowns（"看到才知道想要什么"），让 agent 生成多个视觉方案让用户选择，而不是一次性生成一个"最优"结果：

```
"我不确定你要的'对比'长什么样，这里有 4 个方向：
   A. 雷达图（多维度同时对比）
   B. 分组柱状图（逐维度对比）
   C. 热力图（矩阵式对比）
   D. 评分表（精确数值对比）
 请告诉我哪个方向最接近你的需求。"
```

3. **实现笔记** — 在视觉映射阶段，让 agent 边做边记录它的假设：

```
"我假设你想用柱状图展示月度趋势，因为：
   - 你说'趋势'（暗示时间序列）
   - 数据是月度的（12 个数据点，适合柱状图）
   - 如果这个假设不对，请告诉我。"
```

### 11.18 OKF 结构化 Wiki — 标准化知识格式

[LangChain 的 "OpenWiki 0.2 OKF support"](https://www.langchain.com/blog/openwiki-0-2-adds-okf-support) 引入了 [OKF（Open Knowledge Format）](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)——一个来自 Google Cloud 的结构化知识 Wiki 标准。这对 Agent Canvas 的 Wiki Memory（11.13 节）有直接价值。

OKF 的核心规范：

```yaml
---
type: <类型名称>           # 必填，标识文档概念
title: <显示名称>
description: <一行摘要>
resource: <底层资源的规范 URI>
tags: [<tag>, <tag>, ...]
timestamp: <ISO 8601 最后修改时间>
---

# 文档正文（Markdown）
```

OKF 定义两个约定：
- `index.md` — 每个目录的摘要文件，列出该目录下所有文件和子目录
- `logs.md` — 变更日志，记录每次更新的内容

**Agent Canvas Wiki Memory 的 OKF 化**：

```
/wiki/
  index.md                    — Wiki 总索引
  logs.md                     — 变更日志
  /patterns/
    index.md                  — 可视化模式索引
    comparison.md             — OKF: type=pattern, tags=[compare, bar, radar, heatmap]
    flow.md                   — OKF: type=pattern, tags=[flow, sankey, funnel]
    distribution.md           — OKF: type=pattern, tags=[distribute, scatter, boxplot]
  /components/
    index.md
    statcard.md               — OKF: type=component, tags=[card, metric, kpi]
    chart.md                  — OKF: type=component, tags=[chart, recharts]
  /decisions/
    index.md
    why-bounded-gen.md        — OKF: type=decision, tags=[architecture, safety]
```

**OKF 带来的好处**：

1. **确定性检索** — agent 可以按 tag 或 category 精确过滤，而不是依赖昂贵的 agentic search。例如"找所有 tag 包含 `compare` 的 pattern 文档"是确定性操作，不需要模型推理。

2. **增量更新** — `logs.md` 让 agent 检查"上次更新后什么变了"变得简单——只需读 changelog 而非扫描整个 Wiki。

3. **生态兼容** — OKF 是开放标准，可以使用 Google 的[开源 Wiki 可视化器](https://github.com/GoogleCloudPlatform/knowledge-catalog)检视 Wiki 结构，也可以用社区开发的渲染器、linter 等工具。

4. **进度追踪** — OKF 的 `timestamp` 字段让 Agent Canvas 可以追踪每个 Wiki 条目的新鲜度，优先更新过时的条目。

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
