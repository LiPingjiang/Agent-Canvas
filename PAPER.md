# Agent Canvas: A Three-Layer Semantic Architecture for Agent-Generated Visualization

> **Draft v1.0** — A framework for enabling LLM-based Agents to generate professional-quality visualizations through semantic description rather than code generation.

---

## Abstract

Large Language Models (LLMs) are increasingly deployed as autonomous agents that assist users with information processing tasks, yet their ability to generate professional visualizations remains fundamentally limited. Existing approaches—Markdown rendering, HTML/CSS/JS code generation, Python plotting, and DSL configurations—each suffer from a modal mismatch between sequential text generation and spatial visual reasoning. We present **Agent Canvas**, a framework that separates the visualization generation process into a four-step processing pipeline backed by a three-layer conceptual abstraction: **Semantic Layer** (what meanings to express), **Data Layer** (what concrete data exists), and **Visual Grammar Layer** (what visual forms to use). The Visual Grammar Layer is built on four primitives—**Mark**, **Relation**, **Boundary**, and **ChannelMapping**—that compose to express any 2D visualization without presupposing chart types. We validate the framework against 301 real-world cases spanning 19/20 industry categories, 47 departments, and 15+ role types, achieving 99% coverage. We provide a cost-benefit analysis showing that the three-layer architecture improves output quality for 70% of cases (medium-to-high complexity) while recommending a fast path for the remaining 30% (simple cases). The framework's visual feedback loop—render, screenshot, multimodal LLM review, and correction—further ensures output quality through iterative refinement.

**Keywords**: information visualization, LLM agents, visual grammar, semantic abstraction, automatic visualization generation, bounded generation

---

## 1. Introduction

The rise of LLM-based agents has created a new paradigm: agents that can understand user intent in natural language and produce structured outputs. However, when the desired output is a **visualization**—a dashboard, a report, a chart, or an infographic—agents face a fundamental challenge that we call **modal mismatch**: LLMs operate on sequential text tokens, while visualizations are inherently spatial and relational.

### 1.1 The Problem

Current approaches to agent-generated visualization each have critical limitations:

**Markdown rendering** is highly reliable but extremely limited in expressiveness—it can only do simple text formatting, tables, and basic layout. It cannot produce charts, diagrams, or interactive dashboards.

**HTML/CSS/JS generation** (as used by Claude Artifacts and Vercel v0) is flexible but unreliable: the generated code may contain syntax errors, the aesthetic quality depends on the quality of generated CSS, and the approach requires sandboxed execution for security. Studies show that complex layout scenarios have only a 20-40% success rate ([VisEval](https://arxiv.org/abs/2407.00981)).

**Python plotting** (e.g., matplotlib via [MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/)) requires a runtime environment, produces static images with uncontrollable aesthetics, and suffers from the same code-generation reliability issues.

**DSL configurations** (e.g., ECharts JSON, [Vega-Lite](https://idl.cs.washington.edu/files/2017-VegaLite-InfoVis.pdf) specs) are reliable but limited to predefined chart types—they cannot express custom layouts, composite visualizations, or non-statistical diagrams like flowcharts and organizational charts.

### 1.2 The Root Cause

The root cause is not a lack of technical capability but a **modal mismatch** between how LLMs reason (sequentially, probabilistically) and how visualizations work (spatially, precisely). This manifests in four ways:

1. **Spatial reasoning deficit**: LLMs struggle with spatial relationships like alignment, proximity, and containment ([LaySPA](https://arxiv.org/abs/2509.16891), [LayoutVLM](https://arxiv.org/abs/2504.09125)).
2. **Lack of visual feedback**: Without seeing the rendered output, agents cannot self-correct visual errors. [MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/) demonstrated that visual feedback improves chart quality by 12-13 points.
3. **Probabilistic generation vs. visual precision**: Visualizations require exact pixel-level alignment, but LLMs generate probabilistically, leading to inconsistent spacing, overlapping elements, and misaligned components.
4. **Layout constraint solving**: Complex multi-component layouts require solving spatial constraint problems that LLMs handle poorly ([VisEval](https://arxiv.org/abs/2407.00981) data: complex scenario success rate only 20-40%).

### 1.3 Our Approach

We propose **Agent Canvas**, a framework based on two key principles:

**Separation of concerns**: The agent focuses on "what to show" (generating semantic JSON Schema), while a deterministic rendering engine handles "how to render" (professional components + automatic layout). This is analogous to the separation between HTML (structure) and CSS (presentation), but at a higher semantic level.

**Bounded generation**: The agent can only use predefined components and layout patterns, not arbitrary code. This ensures reliability and visual consistency while still allowing sufficient flexibility for diverse visualization needs.

The framework introduces a **three-layer conceptual abstraction** that bridges user intent and visual rendering:

- **Semantic Layer**: 8 semantic element types (entity, attribute, measure, set, event, process, narrative, constraint) with 5 semantic relations
- **Data Layer**: DataItem triples (entity × attribute → value) that instantiate semantic elements
- **Visual Grammar Layer**: 4 visual grammar primitives (Mark, Relation, Boundary, ChannelMapping) that compose to express any visualization

### 1.4 Contributions

This work makes the following contributions:

1. **A three-layer architecture** that explicitly separates semantic understanding, data instantiation, and visual mapping—making the implicit semantic layer in academic visualization models explicit for agent use.
2. **Four visual grammar primitives** (Mark, Relation, Boundary, ChannelMapping) that replace chart-type-based thinking with composable primitives, achieving 99% coverage of 301 real-world cases.
3. **A composition strategy** with 7 layout patterns and a decision tree for multi-intent visualization pages.
4. **A cross-element linking overlay** for expressing relationships between corresponding points across multiple charts.
5. **A visual feedback loop** using multimodal LLM review of screenshots for iterative quality assurance.
6. **A validated case library** of 301 cases across 19/20 industry categories, providing a benchmark for visualization coverage.

### 1.5 Paper Organization

Section 2 reviews related work. Section 3 analyzes the problem in depth. Section 4 presents the three-layer architecture. Section 5 defines the visual grammar primitives. Section 6 describes the composition strategy. Section 7 covers cross-element linking. Section 8 validates coverage with 301 cases. Section 9 provides the cost-benefit analysis. Section 10 discusses limitations and future work. Section 11 concludes.

---

## 2. Related Work

### 2.1 Visualization Reference Models

**Card's Reference Model** ([Card et al., 1999](https://www.cs.umd.edu/~ben/Card-Mackinlay-Shneiderman-Readings%20in%20Information%20Visualization-1999-v2.pdf)) describes a pipeline from Data Table → Visual Structures → Views, with human interaction at each stage. The model implicitly assumes a human designer who transforms data into visual structures—a step that must be made explicit for automated agents.

**Ed Chi's Data State Reference Model** ([Chi, 2000](https://ics.uci.edu/~kobsa/courses/ICS280/InfoViz2000/ed-chi.pdf)) introduces four data states (Value, Analytical Abstraction, Visualization Abstraction, View) and three transformations (Data Transformation, Visualization Transformation, Visual Mapping). Chi noted that the Analytical Abstraction stage is most frequently skipped, leading to understanding errors. Our Semantic Layer corresponds to this often-skipped stage.

**Grammar of Graphics** ([Wilkinson, 2005](https://link.springer.com/book/10.1007/0-387-28695-0)) provides a formal grammar for statistical graphics based on Marks and Aesthetics. While powerful for statistical charts, it lacks primitives for expressing relationships between marks (e.g., flowcharts, organizational hierarchies) and visual boundaries (e.g., card enclosures, region dividers). Our framework extends GoG with Relation and Boundary primitives.

**Vega-Lite** ([Satyanarayan et al., 2017](https://idl.cs.washington.edu/files/2017-VegaLite-InfoVis.pdf)) builds on GoG to provide a JSON-based grammar for interactive graphics. It excels at statistical visualizations but shares GoG's limitations for non-statistical diagrams. Vega-Lite also lacks an explicit semantic layer between user intent and data specification.

### 2.2 Agent-Generated Visualization

**Claude Artifacts** generates complete HTML/CSS/JS code for visualizations. While flexible, it suffers from reliability issues (code errors), aesthetic inconsistency, and security concerns (requires sandboxing). It has no intermediate representation between user intent and code.

**Vercel v0** generates React component code with Tailwind CSS styling. It uses an AutoFix mechanism for code-level error correction but has no visual feedback loop. Like Claude Artifacts, it lacks a semantic intermediate layer.

**A2UI** uses declarative JSON for UI generation, offering higher reliability than code generation. However, it is limited to predefined UI components and cannot express diverse visualization types.

**MatPlotAgent** ([Yang et al., 2024](https://aclanthology.org/2024.findings-acl.701/)) uses a multi-agent system with visual feedback for matplotlib chart generation. It demonstrated that visual feedback improves quality by 12-13 points, validating the importance of our visual feedback loop. However, it is limited to matplotlib charts and cannot produce dashboards or composite visualizations.

**VisEval** ([Chen et al., 2024](https://arxiv.org/abs/2407.00981)) evaluates LLM visualization capabilities and finds that complex layout scenarios have only 20-40% success rate, highlighting the need for bounded generation and deterministic rendering.

### 2.3 Composite Visualization

**Javed & Elmqvist (2012)** surveyed composite visualization patterns, identifying partitioning, overlay, and nesting as fundamental composition operations. Our composition strategy builds on this taxonomy.

**Deng et al. (IEEE VIS 2022)** extended composite visualization taxonomies with analysis of how multiple views coordinate. Our cross-element linking overlay design draws from this work.

### 2.4 Task and Data Taxonomies

**Shneiderman's task taxonomy** (Overview, Zoom, Filter, Details-on-demand, Relate, History, Extract) and **Munzner's What-Why-How framework** provide frameworks for understanding visualization tasks. Our 11 atomic visualization patterns are informed by both, extending them with operational patterns (Decide, Simulate, Navigate) that support human action rather than just description.

### 2.5 Layout Generation

**LayoutVLM** (Stanford, 2025) and **SKE-Layout** (CVPR 2025) study spatial reasoning in vision-language models, demonstrating that LLMs have significant deficits in spatial layout tasks. This validates our approach of delegating spatial reasoning to a deterministic rendering engine rather than the agent.

---

## 3. Problem Analysis

### 3.1 Why Agent Visualization Quality is Poor

The quality deficit in agent-generated visualization stems from four interrelated factors:

**Factor 1: Modal Mismatch.** LLMs process information as sequential token streams. Visualizations require simultaneous reasoning about spatial position, alignment, hierarchy, and visual encoding. When an LLM generates HTML code for a dashboard, it must mentally simulate the rendered layout while writing code sequentially—a task that exceeds its spatial reasoning capabilities ([LayoutVLM](https://arxiv.org/abs/2504.09125), [LaySPA](https://arxiv.org/abs/2509.16891)).

**Factor 2: Missing Intermediate Representation.** Existing agent visualization tools jump directly from user intent to code or configuration, skipping the semantic understanding step. When a user says "compare three solutions by performance, cost, and operations," the agent must simultaneously: (a) understand that this is a comparison task, (b) identify the entities (solutions) and attributes (performance, cost, operations), (c) extract or assign values, and (d) decide on a visual form (radar chart? grouped bar? comparison table?). Mixing these four cognitive tasks leads to errors.

**Factor 3: No Visual Feedback.** Without seeing the rendered result, the agent cannot detect visual problems: overlapping text, misaligned columns, color contrast issues, or unreadable font sizes. [MatPlotAgent](https://aclanthology.org/2024.findings-acl.701/) proved that adding visual feedback improves quality by 12-13 points, yet most agent visualization tools lack this capability.

**Factor 4: Unbounded Generation Space.** When the agent generates free-form code (HTML/CSS/JS or Python), the output space is theoretically infinite. This makes quality control impossible—the agent might produce any of billions of possible HTML structures, most of which are visually suboptimal. Restricting the agent to a bounded set of components and layout patterns dramatically reduces the output space and enables quality guarantees.

### 3.2 Requirements for a Solution

Based on this analysis, a successful agent visualization framework must:

1. **Separate semantic understanding from visual rendering** — reduce cognitive load by splitting the task into distinct stages
2. **Provide bounded generation** — restrict the agent to predefined, high-quality components
3. **Include visual feedback** — enable self-correction through rendered-output review
4. **Support multi-scenario adaptation** — allow the same semantic understanding to produce different visual outputs for different contexts (dashboard, mobile, report)
5. **Cover diverse visualization types** — not just statistical charts but also diagrams, tables, reports, and composite dashboards
6. **Be agent-friendly** — use intuitive abstractions that LLMs can reason about effectively

---

## 4. Three-Layer Architecture

### 4.1 Overview

The three-layer architecture separates the visualization generation process into three distinct abstraction levels, each answering a different question:

```
User Requirement (Natural Language)
    ↓
┌──────────────────────────────────────────────────┐
│  Semantic Layer                                   │
│  "What meanings to express?"                      │
│  8 element types + 5 relation types               │
│  No concrete values, no visual properties         │
└──────────────────┬───────────────────────────────┘
                   ↓  Semantic → Data Mapping
┌──────────────────────────────────────────────────┐
│  Data Layer                                       │
│  "What concrete data exists?"                     │
│  DataItem triples: entity × attribute → value     │
│  Concrete values, no visual properties            │
└──────────────────┬───────────────────────────────┘
                   ↓  Data → Visual Mapping (ChannelMapping)
┌──────────────────────────────────────────────────┐
│  Visual Grammar Layer                             │
│  "What visual forms to use?"                      │
│  4 primitives: Mark / Relation / Boundary /       │
│  ChannelMapping                                   │
└──────────────────────────────────────────────────┘
```

### 4.2 Semantic Layer

The Semantic Layer identifies "meaning units" from user intent. It contains no concrete data values and no visual properties—it purely describes what the user wants to express.

**8 Semantic Element Types:**

| Type | Description | Example |
|------|-------------|---------|
| entity | The object being discussed | "Solution A", "Revenue", "East Region" |
| attribute | A dimension or aspect of an entity | "Performance", "Monthly trend", "Revenue share" |
| measure | A quantifiable value | 8 points, ¥3.8M, 92% |
| set | A collection of related entities | "High-value users", "TOP 10 products" |
| event | An occurrence at a point in time | "Launch", "Outage", "Promotion" |
| process | An ordered sequence of steps | "Approval flow", "Deployment pipeline" |
| narrative | A structured discourse | "Background", "Analysis", "Conclusion" |
| constraint | A limiting condition | "One page", "Emphasize Q2" |

**5 Semantic Roles:** subject (the main entity), attribute (a dimension being examined), measure (a quantifiable value), relation (a connection between entities), context (background information).

**8 Semantic Relation Types:** compare, belong_to, flow_to, cause, part_of, sequence, correlate, reference.

**Key Design Decision**: The original framework used 10 "information elements" (DataPoint, DataSeries, Comparison, Composition, Relation, Process, Hierarchy, Distribution, Narrative, Decision). Analysis revealed three problems: (1) classification criteria were inconsistent (some by data structure, some by intent, some by relationship), (2) elements were over-bound to rendering components, and (3) elements couldn't express "same data, multiple presentations." The 8 semantic element types solve these by being purely semantic—comparison, composition, and hierarchy become relations rather than element types, and sequence/ordering become constraints rather than separate types.

### 4.3 Data Layer

The Data Layer instantiates semantic elements into concrete data items. Each DataItem is a triple (entity, attribute, value):

```typescript
interface DataItem {
  entityId: string;        // e.g., "plan_a"
  entityLabel: string;     // e.g., "Solution A"
  attribute: string;       // e.g., "performance"
  attributeLabel: string;  // e.g., "Performance"
  value: any;              // e.g., 8
  dataType: "number" | "string" | "boolean" | "date" | "array";
  unit?: string;           // e.g., "points", "¥", "%"
}
```

**Mapping Rule**: N subjects × M attributes in the Semantic Layer expand to N×M DataItems in the Data Layer. Semantic relations instantiate as data relations bound to specific DataItems.

### 4.4 Visual Grammar Layer

The Visual Grammar Layer maps DataItems to visual primitives. This layer is defined by four primitives (detailed in Section 5) that compose to express any 2D visualization.

### 4.5 Layer-to-Layer Mapping

**Semantic → Data**: Cross-product expansion of entities × attributes, with semantic relations instantiated as data relations. Values come from user input, data sources, or agent reasoning.

**Data → Visual**: ChannelMapping determines which data fields map to which visual channels (position, size, color, shape, etc.). The same DataItem set can produce radically different visualizations through different mappings.

### 4.6 Academic Validation

The three-layer architecture corresponds to established academic models:

| Our Layer | Ed Chi Data State Model (2000) | Card Reference Model (1999) | Vega-Lite (2017) | Grammar of Graphics (2005) |
|-----------|-------------------------------|---------------------------|-----------------|---------------------------|
| Semantic Layer | Analytical Abstraction | (implicit, undefined) | (missing) | Variables + Algebra |
| Data Layer | Value | Data Table | Data + Transform | Data |
| Visual Grammar Layer | Visualization Abstraction → View | Visual Structures → Views | Mark + Encoding + Scale + Layout | Geometry + Aesthetics + Coordinates |

**Key finding**: Academic models all have an intermediate abstraction between data and visuals, but it is typically compressed or implicit. Ed Chi's Analytical Abstraction corresponds to our Semantic Layer, but he noted this stage is most frequently skipped and most error-prone. Card's model jumps from Data Table to Visual Structures with implicit semantic transformation. Vega-Lite and GoG also lack an explicit semantic layer.

**Our contribution is not inventing three layers, but making the implicit semantic layer explicit**. This is not necessary for human visualization tools (humans perform semantic-to-data transformation mentally), but it is essential for agents—agents lack human implicit reasoning and require explicit representation at every step.

### 4.7 Processing Pipeline

The three-layer abstraction is realized through a four-step processing pipeline:

```
Step 1: Intent Parsing → Intent{what, who, where, why, dataShape, constraints}
Step 2: Semantic Modeling → SemanticElement[] + SemanticRelation[]
Step 3: Data Instantiation → DataItem[] + DataRelation[]
Step 4: Visual Mapping → ChannelMapping → Mark[] + Relation[] + Boundary[]
Step 5: Composition & Layout → CompositionPlan → spatial allocation + hierarchy
Step 6: Rendering → Canvas Schema → React rendering engine
Step 7: Visual Feedback → screenshot → multimodal review → correction
```

The three-layer feedback system enables error detection at each level:
- Semantic review: "Did we miss any meanings the user wanted to express?"
- Data review: "Is the data correct and complete?"
- Visual review: "Does the rendering look good and is it correct?"

---

## 5. Visual Grammar Primitives

### 5.1 Motivation

Traditional visualization frameworks are built around "chart types"—bar chart, line chart, pie chart, etc. This approach has two problems for agent use: (1) the agent must decide the chart type early in the process, before fully understanding the data, and (2) chart types don't compose well—you can't easily express "a scatter plot with a trend line overlay" or "a dashboard with linked views."

We instead define four **visual grammar primitives** that compose to express any 2D visualization. No primitive presupposes a rendering form; the form emerges from the combination.

### 5.2 Primitive 1: Mark

A Mark is the atomic unit of visual expression. It has a position and mappable visual channels, but **no preset rendering form**.

```typescript
interface Mark {
  id: string;
  position: { x: number, y: number };
  channels: {
    size?: number;
    color?: string;
    shape?: "circle" | "bar" | "square" | "triangle" | "text" | ...;
    orientation?: number;
    texture?: string;
    opacity?: number;
  };
  label?: string;
  value?: any;
  metadata?: Record<string, any>;
}
```

**Key insight**: The same data point "April revenue ¥3.8M" has the same Mark regardless of visualization context. The rendering form is determined by ChannelMapping:
- Precise reading → shape: "text", display "¥380万"
- Size comparison → shape: "bar", size mapped to value
- Trend analysis → shape: "circle", position.x = time, connected by line
- Matrix highlight → shape: "square", color depth mapped to value

### 5.3 Primitive 2: Relation

Relation describes how Marks connect to each other. Five basic types:

| Type | Semantics | Visual Expression | Example |
|------|-----------|------------------|---------|
| Connection | "is related to" | Line (solid/dashed/dotted) | Service dependency graph |
| Flow | "flows to" | Directed arrow | Data pipeline, user journey |
| Containment | "belongs to" | Spatial enclosure or hollow arrow | Org chart, file directory |
| Sequence | "ordered" | Position encoding (no explicit line) | Timeline, ranking |
| Alignment | "corresponds to" | Shared position on a dimension | Table rows, multi-chart alignment |

**Critical finding**: DataSeries and Distribution (two of the original 10 information elements) use identical Marks. Their only difference is that DataSeries has Sequence + Connection relations while Distribution does not. **Different relations, not different element types**.

### 5.4 Primitive 3: Boundary

Boundary visually encloses a group of Marks to express "these belong together."

```typescript
interface Boundary {
  shape: "rect" | "circle" | "ellipse" | "freeform" | "none";
  marks: string[];  // enclosed Mark IDs
  label?: string;
  style?: "solid" | "dashed" | "dotted" | "filled" | "shadow";
  nestLevel?: number;
}
```

Boundaries serve four purposes: grouping (enclosing related Marks), partitioning (dividing page into semantic regions), emphasis (highlighting with color/shadow), and hierarchy (nested boundaries for hierarchical structure).

### 5.5 Primitive 4: ChannelMapping

ChannelMapping defines how data fields map to Mark's visual channels—the bridge between data semantics and visual presentation.

```typescript
interface ChannelMapping {
  markSet: string[];
  position?: {
    x?: { field: string; scale: "linear" | "ordinal" | "time" | "log" };
    y?: { field: string; scale: "linear" | "ordinal" | "time" | "log" };
  };
  size?: { field: string; scale: "linear" | "sqrt" };
  color?: { field: string; scale: "categorical" | "sequential" | "diverging" };
  shape?: { field: string; scale: "categorical" };
  text?: { field: string };
}
```

**Mapping determines rendering form**: The same 9 DataItems (3 solutions × 3 dimensions) produce:
- Radar chart: angle ← attribute, radius ← value, color ← entity
- Grouped bar chart: x ← entity (grouped), size ← value, color ← attribute
- Heat matrix: x ← entity, y ← attribute, color ← value
- Comparison table: text ← value, Alignment relation, rect Boundary

### 5.6 Relationship to Grammar of Graphics

Our framework is inspired by [Grammar of Graphics](https://link.springer.com/book/10.1007/0-387-28695-0) (Wilkinson, 2005) but has key differences:

| Dimension | Grammar of Graphics | Agent Canvas Visual Grammar |
|-----------|--------------------|---------------------------|
| Core objects | Mark + Aesthetic | Mark + Relation + Boundary + ChannelMapping |
| Relation expression | None (implicit via position/alignment) | Explicit Relation types (Connection/Flow/Containment/Sequence/Alignment) |
| Boundary expression | None | Explicit Boundary primitive |
| Scope | Statistical charts | Charts + diagrams + reports + dashboards |
| Agent friendliness | Low (requires statistical knowledge) | High (intuitive primitives, derivable mappings) |

**Key distinction**: We add Relation and Boundary primitives that GoG lacks. GoG primarily solves statistical chart problems but cannot express flowcharts, organizational charts, kanban boards, or other "non-statistical" visualizations. Relation (explicit connections between marks) and Boundary (visual enclosure) fill this gap.

---

## 6. Composition Strategy

### 6.1 The Composition Problem

Real-world visualization needs rarely involve a single chart. A CEO dashboard might combine KPI cards, trend lines, pie charts, and a data table. The composition problem is: how to arrange multiple visualization elements on a single page while maintaining visual hierarchy and readability.

### 6.2 Three Composition Levels

**Level 1: Intra-chart composition** — Multiple encodings within a single chart (e.g., dual-axis overlay, size + color encoding, annotation + trend). Maximum 4 intents per chart.

**Level 2: Inter-chart composition** — Multiple charts arranged on a page. Seven layout patterns:

| Pattern | Description | Example |
|---------|-------------|---------|
| Side-by-side | Horizontal arrangement of equal-weight charts | Q1 vs Q2 comparison |
| Vertical stack | Vertical arrangement | Report-style flow |
| Hub-and-spoke | Central chart with supporting charts around | Central KPI with detail breakdowns |
| Grid | Regular grid of equal-weight charts | Small multiples dashboard |
| Split panel | Main + side panel | Chart + filter controls |
| Small multiples | Repeated chart structure with varying data | 10 cities' DAU trends |
| Linked views | Multiple charts with cross-element linking | Scatter plot + histogram with brushing |

**Level 3: Inter-region composition** — Semantic regions (metrics area, trend area, composition area, comparison area, detail area, narrative area, navigation area, decision area) arranged to form a complete page.

### 6.3 Primary-Secondary Hierarchy

Composition is governed by a primary-secondary hierarchy with three levels:

- **L1 (Focus)**: 50-60% of space, largest visual weight
- **L2 (Support)**: 30-40% of space, moderate visual weight
- **L3 (Supplement)**: 5-10% of space or collapsed

Differentiation is achieved through five dimensions: span (column width), font size, color saturation, card shadow, and chart size.

### 6.4 Composition Decision Tree

The composition decision tree guides selection of layout patterns based on: number of intents (1 → single chart, 2-4 → intra-chart composition, 5+ → inter-chart), intent relationships (comparison → side-by-side, sequence → vertical stack, hierarchy → hub-and-spoke, independence → grid), and page constraints (width, height, density).

---

## 7. Cross-Element Linking

### 7.1 The Linking Problem

Traditional visualization treats each chart as an independent unit. But real-world analysis often requires expressing relationships between corresponding points across multiple charts—e.g., highlighting the same user in a scatter plot and a bar chart, or showing how a data point in one view flows to another view.

### 7.2 Link Overlay Design

We introduce a **link overlay layer** that sits above the visual grammar layer, connecting Marks across different chart boundaries:

**7 Link Types:**

| Link Type | Semantics | Visualization Means |
|-----------|-----------|-------------------|
| entity_match | Same entity in different views | Color consistency, highlight synchronization |
| entity_flow | Entity moves from one view to another | Connecting lines, animated flow |
| field_mapping | Same field mapped differently | Axis alignment, shared scale |
| comparison_link | Corresponding points for comparison | Connecting lines, paired markers |
| correspondence | One-to-one or one-to-many mapping | Brushing and linking |
| causal | Cause-effect across views | Annotated arrows |
| aggregation | Detail-to-summary relationship | Zoom links, drill-down |

### 7.3 Position Registry

The link overlay requires a Position Registry mechanism in the rendering engine: each registered Mark reports its screen position, enabling the overlay layer to draw connecting lines between Marks in different chart containers.

### 7.4 Linked Views Pattern

The "Linked Views" layout pattern (added to the 7 composition patterns) specifically supports cross-element linking: multiple charts share a coordinate or entity, and interactions in one chart (hover, select, brush) propagate to linked charts.

This design draws from [Roberts (2007)](https://doi.org/10.1016/j.cag.2006.10.016) on coordinated multiple views, [Becker & Cleveland (1987)](https://www.jstor.org/stable/2289529) on brushing and linking, and [Collins & Carpendale (2007)](https://doi.org/10.1109/TVCG.2007.70571) on VisLink.

---

## 8. Validation: 301-Case Coverage

### 8.1 Case Collection Methodology

We collected 301 visualization cases through systematic coverage of:

- **19/20 industry categories** (GB/T 4754-2017 national economy classification)
- **47 departments** (12 general enterprise + 15 internet + 20 industry-specific)
- **15+ role types** (from CEO to algorithm engineer, SRE, growth hacker)
- **10 enterprise types** (large enterprise, startup, public institution, non-profit, hospital, school, etc.)
- **30+ visualization forms** (dashboards, reports, confusion matrices, K-line charts, technical radar, etc.)

Cases were validated against four reference frameworks: [ISCO-08](https://www.ilo.org/publications/major-publications/international-standard-classification-occupations) international occupation classification, [GB/T 4754-2017](http://www.stats.gov.cn/) national economy classification, [Shneiderman](https://www.cs.umd.edu/~ben/Shneiderman-1996.pdf) task taxonomy, and [Munzner](https://www.crcpress.com/Visualization-Analysis-and-Design/Munzner/p/book/9781466508910) What-Why-How framework.

### 8.2 11 Atomic Visualization Patterns

From the 301 cases, we identified 11 atomic visualization patterns that compose to express all cases:

**Descriptive patterns** (describing states):

| Pattern | Core Question | Frequency |
|---------|--------------|-----------|
| Compare | How do A and B differ? | 55% |
| Track | How does it change over time? | 51% |
| Narrate | What is the story? | 41% |
| Arrange | How is it organized? | 33% |
| Distribute | How is data distributed? | 32% |
| Compose | What parts make up the whole? | 28% |
| Flow | Where does it come from and go? | 20% |
| Connect | Who is related to whom? | 14% |

**Operational patterns** (supporting action):

| Pattern | Core Question | Frequency |
|---------|--------------|-----------|
| Decide | Which should I choose? | 7% |
| Navigate | How do I get from here to there? | 7% |
| Simulate | What if...? | 4% |

### 8.3 Visual Grammar Coverage

We evaluated each of the 301 cases against the four visual grammar primitives (Mark + Relation + Boundary + ChannelMapping):

| Capability Dimension | Covered Cases | Percentage |
|---------------------|---------------|------------|
| Standard 2D charts | 162 | 54% |
| Structural relation diagrams | 48 | 16% |
| Tables/matrices | 32 | 11% |
| Text/reports | 28 | 9% |
| Dashboards/composites | 22 | 7% |
| Maps/geographic | 5 | 2% |
| K-line/candlestick | 2 | 1% |
| 3D/stereoscopic | 2 | 1% (not covered) |
| **Total** | **301** | **99%** |

**Coverage: 299/301 = 99%**. The only uncovered cases are 2 cases requiring 3D rendering (climate change 3D simulation, architectural 3D rendering), which exceed the 2D visual grammar scope.

Edge cases that require extension but are coverable:
- Geographic maps: ChannelMapping with geo coordinate system
- K-line charts: Composite Mark (one Mark carrying OHLC values)
- Sankey diagrams: Relation(Flow) with band morphology
- Real-time streaming: Mark lifecycle management
- Confusion matrix: Table + ChannelMapping(color=value) combination
- Technical radar: Boundary(ring) + ChannelMapping(angle=category, radius=status)
- Treemap: Containment + ChannelMapping(size=area) nesting

### 8.4 Traditional Chart Decomposition

All traditional chart types decompose into Mark + Relation + Boundary + ChannelMapping combinations:

| Chart Type | Mark Form | Relation | Boundary | ChannelMapping |
|-----------|----------|----------|----------|---------------|
| Bar chart | bar | — | optional (group box) | x=category, size=value |
| Line chart | circle | Connection + Sequence | — | x=time, y=value |
| Pie chart | bar (sector) | Containment | circle (whole) | angle=proportion |
| Scatter plot | circle | — | — | x=var1, y=var2 |
| Heatmap | square | Alignment | rect (matrix) | x=cat1, y=cat2, color=value |
| Stacked bar | bar | Containment + Alignment | rect (bar) | x=category, y=value, color=subcategory |
| Sankey | bar (flow band) | Flow | — | x=stage, width=volume |
| Funnel | bar (trapezoid) | Sequence + Flow | trapezoid | y=stage, size=volume |
| Radar | circle + polygon | Alignment | polygon | angle=dimension, radius=value |
| Flowchart | rect (node) | Flow | optional (swimlane) | — |
| Network graph | circle (node) | Connection | optional (community) | force-directed |
| Org chart | rect (node) | Containment | — | hierarchical layout |
| Table | text (cell) | Alignment | rect (table) | row=record, col=field |
| KPI card | text (number) | Containment | rect (card) | text=value |
| Map | circle/square | Containment | freeform (geo) | position=lat/lng |
| Decision tree | rect (node) | Flow (branches) | — | hierarchical |
| K-line | bar (candle) + line | Sequence | — | x=time, y=price range |

---

## 9. Cost-Benefit Analysis

### 9.1 Benefits

**Multi-scenario adaptation**: The same Semantic Layer and Data Layer can produce different visual outputs for different contexts. For example, "Q2 business data" can be rendered as a dashboard (Grid + large fonts), a mobile card (Stack + mini charts), or a report (Report + tables)—only the ChannelMapping changes. Approximately 18% of the 301 cases involve multi-scenario adaptation and directly benefit.

**Layered error detection**: The three-layer feedback system enables error detection at each level. Semantic review catches missing meanings (e.g., user mentioned "year-over-year" but no YoY entity exists). Data review catches incorrect values. Visual review catches layout issues. The original single-layer feedback could detect errors but couldn't distinguish understanding errors from rendering errors.

**Delayed rendering decisions**: The agent makes rendering decisions (bar chart vs. scatter plot) only at the Visual Mapping stage, not during decomposition. This reduces early-error risk—if a mapping choice is wrong, only the mapping needs changing, not the entire understanding.

### 9.2 Costs

**Increased agent steps**: The original framework uses 3 steps (intent → elements → Schema); the three-layer framework uses 5 steps (intent → semantic → data → visual mapping → Schema). Each step introduces potential error. For simple scenarios ("show revenue ¥12.34M"), the three-layer architecture is over-engineering.

**Token consumption**: The agent outputs Semantic Layer JSON + Data Layer JSON + Visual Layer JSON, approximately 2-3× more tokens than directly outputting a Canvas Schema. For high-frequency simple scenarios, this is a significant overhead.

### 9.3 Case Verification

We verified three representative scenarios:

**Case 1 — Simple: "Show total revenue ¥12.34M"**

The original framework handles this in 1 step with 1 JSON object. The three-layer framework requires 3 steps with 3 JSON objects. **Conclusion: Three layers are over-engineering. The fast path is better.**

**Case 2 — Medium: "Q2 dashboard with 3 KPIs + trend chart + composition pie, for CEO on large screen"**

The original framework requires re-decomposition when switching to "send to group chat" (mobile). The three-layer framework keeps semantic and data layers unchanged—only the visual mapping changes. **Conclusion: Three layers are clearly superior. Multi-scenario adaptation and layered feedback are valuable.**

**Case 3 — Complex: "Compare recommendation system A/B test results across CTR/revenue/retention, highlight significant differences"**

The original framework struggles with multiple element types and complex relationships—the agent easily confuses Comparison elements. The three-layer framework's Semantic Layer clearly identifies "2 entities × 3 dimensions comparison relations," the Data Layer correctly expands to 6 data items, and the Visual Layer can flexibly choose table/radar/bar. **Conclusion: Three layers are significantly superior. Semantic Layer prevents understanding errors; Data Layer ensures data completeness.**

### 9.4 Net Effect

**The net effect is positive.** Approximately 70% of the 301 cases are medium-to-high complexity, and the scenarios where agent visualization quality is poorest are precisely the complex scenarios—simple scenarios are already handled well by agents. The three-layer architecture specifically improves the "hard" scenarios.

**Recommendation: Tiered usage.** For simple scenarios (single metric, single chart, ~30% of cases), provide a "fast path" where the agent skips the Semantic Layer and goes directly from intent to data to visual. For medium and complex scenarios (multi-metric, multi-intent composition, multi-scenario adaptation, ~70% of cases), use the full three-layer pipeline.

---

## 10. Discussion

### 10.1 Limitations

**3D visualization**: The current framework covers 2D visualizations only. Two cases (0.7%) requiring 3D rendering cannot be expressed. Future work should extend Mark with a z-coordinate and add 3D Boundary primitives.

**Real-time data**: The current framework is designed for static or batch-generated visualizations. Real-time streaming data requires Mark lifecycle management (creation, update, expiration) and is left as future work.

**Interactivity**: While the framework supports basic interactions (hover, click), rich interactive behaviors (drag-and-drop, zoom-pan, dynamic filtering) require further design.

**Empirical validation**: The 301-case validation is coverage-based (can the framework express each case?) rather than quality-based (does the framework produce better visualizations than alternatives?). Controlled experiments comparing Agent Canvas against Claude Artifacts, Vercel v0, and baseline approaches are needed.

### 10.2 Implementation Considerations

The framework is designed for implementation with React 18+ / TypeScript, Tailwind CSS, Recharts/ECharts, Zod for schema validation, Playwright for screenshots, and multimodal LLMs (GPT-4o / Claude 3.5 Sonnet) for visual review. The rendering engine uses CSS Grid/Flexbox rather than absolute positioning to reduce spatial reasoning demands.

### 10.3 Comparison with Existing Approaches

| Dimension | Claude Artifacts | Vercel v0 | A2UI | Agent Canvas |
|-----------|-----------------|-----------|------|-------------|
| Agent output | HTML/CSS/JS code | React component code | Declarative JSON | Canvas Schema (JSON) |
| Generation complexity | High (full frontend code) | High (React+TSX) | Low | Low (structure + data) |
| Reliability | Medium (code errors) | Medium (needs AutoFix) | High | High (Schema validation + deterministic rendering) |
| Aesthetic quality | Unstable | Good (shadcn/ui) | Medium | Stable (professional component presets) |
| Visual feedback | None | AutoFix (code-level) | None | Yes (screenshot + multimodal review) |
| Intermediate representation | None | None | None | Yes (Semantic Layer + Data Layer) |
| Multi-scenario adaptation | Requires regeneration | Requires regeneration | Requires regeneration | Same semantics → multiple schemas |
| Security | Requires sandbox | Requires sandbox | Naturally safe | Naturally safe (JSON non-executable) |

**Core differentiators**: Agent Canvas has a semantic intermediate layer that decouples "understanding requirements" from "generating visuals"; has a visual feedback loop for self-correction; and enables multi-scenario adaptation from a single semantic representation.

---

## 11. Conclusion

We presented Agent Canvas, a framework for enabling LLM-based agents to generate professional-quality visualizations through semantic description rather than code generation. The framework's three-layer architecture (Semantic Layer → Data Layer → Visual Grammar Layer) makes explicit the semantic understanding step that is implicit in traditional visualization models—a step that humans perform mentally but agents must perform explicitly. The four visual grammar primitives (Mark, Relation, Boundary, ChannelMapping) replace chart-type-based thinking with composable primitives, achieving 99% coverage of 301 real-world cases across 19/20 industry categories.

The framework's bounded generation approach (predefined components only) and visual feedback loop (render → screenshot → multimodal review → correction) address the fundamental reliability and quality challenges of agent-generated visualization. The cost-benefit analysis shows that while the three-layer architecture adds overhead for simple cases, it significantly improves quality for the 70% of cases that are medium-to-high complexity—precisely the scenarios where existing approaches fail.

Future work includes: (1) extending to 3D visualization, (2) supporting real-time streaming data, (3) conducting controlled experiments against existing approaches, and (4) implementing and open-sourcing the full framework.

---

## References

1. **Card, S. K., Mackinlay, J. D. & Shneiderman, B.** *Readings in Information Visualization: Using Vision to Think*. Morgan Kaufmann, 1999. [Link](https://www.cs.umd.edu/~ben/Card-Mackinlay-Shneiderman-Readings%20in%20Information%20Visualization-1999-v2.pdf)

2. **Chi, E. H.** "A Taxonomy of Visualization Techniques Using the Data State Reference Model." *IEEE Symposium on Information Visualization (InfoVis)*, 2000. [Link](https://ics.uci.edu/~kobsa/courses/ICS280/InfoViz2000/ed-chi.pdf)

3. **Wilkinson, L.** *The Grammar of Graphics*. Springer, 2005. [Link](https://link.springer.com/book/10.1007/0-387-28695-0)

4. **Satyanarayan, A., Moritz, D., Wongsuphasawat, K. & Heer, J.** "Vega-Lite: A Grammar of Interactive Graphics." *IEEE Transactions on Visualization and Computer Graphics (InfoVis)*, 2017. [Link](https://idl.cs.washington.edu/files/2017-VegaLite-InfoVis.pdf)

5. **Shneiderman, B.** "The Eyes Have It: A Task by Data Type Taxonomy for Information Visualizations." *IEEE Symposium on Visual Languages*, 1996. [Link](https://www.cs.umd.edu/~ben/Shneiderman-1996.pdf)

6. **Munzner, T.** *Visualization Analysis and Design*. CRC Press, 2014. [Link](https://www.crcpress.com/Visualization-Analysis-and-Design/Munzner/p/book/9781466508910)

7. **Javed, W. & Elmqvist, N.** "Exploring the Design Space of Composite Visualizations." *IEEE Pacific Visualization Symposium*, 2012. [Link](https://doi.org/10.1109/PacificVis.2012.6183556)

8. **Deng, Z., Weng, D., Chen, J., Liu, R., Wang, Z., Bao, J., Zheng, Y., Wu, Y. & Ying, W.** "Composing Visual Elements: A Survey of Visualization Composition." *IEEE Transactions on Visualization and Computer Graphics (VIS)*, 2022. [Link](https://doi.org/10.1109/TVCG.2022.3209445)

9. **Roberts, J. C.** "State of the Art: Coordinated & Multiple Views in Exploratory Visualization." *Computer Graphics Forum*, 2007. [Link](https://doi.org/10.1016/j.cag.2006.10.016)

10. **Becker, R. A. & Cleveland, W. S.** "Brushing Scatterplots." *Technometrics*, 1987. [Link](https://www.jstor.org/stable/2289529)

11. **Collins, C. & Carpendale, S.** "VisLink: Revealing Relationships Amongst Visualizations." *IEEE Transactions on Visualization and Computer Graphics*, 2007. [Link](https://doi.org/10.1109/TVCG.2007.70571)

12. **Yang, J., Chen, X., Qian, J., et al.** "MatPlotAgent: Method and Evaluation for LLM-Based Agentic Scientific Data Visualization." *ACL Findings*, 2024. [Link](https://aclanthology.org/2024.findings-acl.701/)

13. **Chen, Z., Liu, J., Shen, Y., et al.** "VisEval: A Benchmark for Evaluating LLMs on Visualization Generation." *arXiv preprint*, 2024. [Link](https://arxiv.org/abs/2407.00981)

14. **International Labour Organization.** *International Standard Classification of Occupations (ISCO-08)*. ILO, 2012. [Link](https://www.ilo.org/publications/major-publications/international-standard-classification-occupations)

15. **National Bureau of Statistics of China.** *GB/T 4754-2017: Industrial Classification for National Economic Activities*. 2017.

16. **Stanford HAI.** "LayoutVLM: Evaluating Spatial Reasoning in Vision-Language Models." 2025. [Link](https://arxiv.org/abs/2504.09125)

17. **LaySPA.** "Enhancing Spatial Reasoning in Large Language Models." 2025. [Link](https://arxiv.org/abs/2509.16891)

18. **SKE-Layout.** "Spatial Knowledge Enhanced Layout Generation." *CVPR*, 2025.

---

## Appendix A: Three-Layer Architecture vs. Original Architecture

| Dimension | Original Architecture | Three-Layer Architecture |
|-----------|----------------------|------------------------|
| Layers | 2 (intent → Mark) | 4 (intent → semantic → data → visual) |
| Decomposition output | 10 information elements | 8 semantic elements → DataItems |
| Rendering decision timing | Bound to component during decomposition | Decided at visual mapping stage |
| Same requirement, multiple scenarios | Requires re-decomposition | Semantic/Data unchanged, only mapping changes |
| Agent cognitive load | High (mixes three layers) | Low (each layer does one thing) |
| Feedback levels | 1 (post-render) | 3 (semantic / data / visual) |

## Appendix B: Semantic Element Evolution

| Original 10 Elements | Semantic Layer Equivalent | Change Rationale |
|----------------------|--------------------------|-----------------|
| DataPoint | entity + attribute + measure | Split into "what entity, what attribute, what value" |
| DataSeries | entity + attribute(ordered) + measure[] | Sequentiality expressed via constraints.ordered |
| Comparison | entity[] + attribute[] + compare relation | Comparison is a relation, not an element type |
| Composition | entity + set[] + part_of relation | Composition is a relation, not an element type |
| Relation | entity[] + relation(various) | Relations unified as semantic relations |
| Process | event[] + flow_to relation | Process is ordered flow of events |
| Hierarchy | entity[] + belong_to relation | Hierarchy is a special case of belonging |
| Distribution | entity[] + attribute(continuous) | Distribution doesn't need a separate element type |
| Narrative | narrative element | Unchanged |
| Decision | entity[] + attribute[] + measure[] + narrative | Decision is a combination of multiple elements |

**Key change**: DataSeries, Distribution, Comparison, Composition, and Hierarchy are no longer independent element types—their differences are in semantic relations (compare / part_of / flow_to / belong_to) and constraints (ordered / hierarchical), not in the elements themselves. Only 8 pure semantic types remain.
