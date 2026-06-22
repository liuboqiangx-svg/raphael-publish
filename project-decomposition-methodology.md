# Project Decomposition Methodology
## 从源码到交互式逻辑地图 + 小白指南的自动化 SOP

---

## 1. Overview

### 1.1 这份方法论是什么

本方法论描述了一套**从任意软件项目源码自动生成两份可视化教学产物**的标准操作流程（SOP）：

1. **`project-map.html`** —— 交互式 ECharts 网络图
   - 每个文件/命令是一个节点
   - 节点之间用连线表示构建、运行、依赖三种关系
   - 支持点击节点查看详情、点击路径卡片高亮路径、播放动画演示完整流程

2. **`project-guide.html`** —— 三标签页小白指南
   - 标签页 1：项目地图（文件树 + 每个文件的比喻、作用、触发时机、依赖）
   - 标签页 2：构建流程（从命令到产物的分步时间线）
   - 标签页 3：运行流程（从用户打开页面到交互完成的分步时间线）

### 1.2 面向谁

- 编程小白，想从零理解一个项目是怎么跑起来的
- 技术负责人，想给团队或客户快速讲解项目结构
- 教育者，想把项目讲解变成可交互、可重复使用的材料

### 1.3 覆盖哪些项目类型

| 项目类型 | 示例 |
|----------|------|
| frontend | React / Vue / Angular + Vite / Webpack |
| python-backend | FastAPI / Flask / Django |
| mobile | Flutter / iOS / Android |
| cli | Node.js CLI / Python CLI |
| data-pipeline | Airflow / Spark / dbt / ETL |
| game | Unity / Godot / Bevy |
| generic | 无法被明确分类的混合项目 |

---

## 2. Prerequisites

输入只需要一个项目目录：

```
/path/to/your-project/
```

可选输入：

| 输入 | 说明 |
|------|------|
| `projectName` | 项目名称，默认使用 `package.json` 的 `name` 或目录名 |
| `projectType` | 手动覆盖检测出的项目类型 |
| `metaphorSystem` | 手动覆盖比喻系统 |

禁止输入：
- 不需要运行项目
- 不需要访问网络或外部 API
- 不需要读取 `.env`、密钥、数据库等敏感文件

---

## 3. Phase 1：项目类型自动检测

### 3.1 检测策略

采用**置信度评分矩阵**。扫描项目根目录文件，对每个候选类型打分，超过阈值（默认 `0.5`）者胜出；否则归入 `generic`。

### 3.2 评分规则

| 候选类型 | 强信号（+0.4） | 弱信号（+0.2） | 排除规则 |
|----------|---------------|---------------|----------|
| **frontend** | `package.json` + `index.html` 同时存在 | React/Vue/Angular 在依赖中、`src/` 含 `.tsx`/`.jsx` | 根目录无 `main.py`、无移动框架 |
| **python-backend** | `requirements.txt`/`pyproject.toml` + `main.py`/`app.py`/`manage.py` | FastAPI/Flask/Django 在依赖中、使用 uvicorn/gunicorn | 根目录无 `package.json` |
| **mobile** | `pubspec.yaml` / `Podfile` / `build.gradle` | `ios/`、`android/`、`lib/` 目录 | - |
| **cli** | `package.json` 含 `bin` 字段 / `setup.py` 含 `entry_points` | `argparse`/`commander`/`yargs` 在依赖中 | 无 `index.html`、无 server 框架 |
| **data-pipeline** | `dags/` / `etl/` / `pipeline/` 目录 | pandas/polars/spark/dbt/kafka 在依赖中 | - |
| **game** | `Cargo.toml` 含 bevy/ggez / Unity 工程文件 / `game/` 目录 | 游戏循环代码、`sprites/`/`audio/` 目录 | - |
| **generic** | 无明确信号 | 混合信号 | - |

### 3.3 伪代码

```python
def detect_project_type(root_dir):
    scores = defaultdict(float)
    files = list_root_files(root_dir)

    if 'package.json' in files and 'index.html' in files:
        scores['frontend'] += 0.4
    if any(f in files for f in ['requirements.txt', 'pyproject.toml', 'setup.py']):
        if any(f in files for f in ['main.py', 'app.py', 'manage.py']):
            scores['python-backend'] += 0.4
    # ... 其他规则

    max_type = max(scores, key=scores.get, default='generic')
    max_score = scores.get(max_type, 0)

    if max_score >= 0.5:
        return max_type, max_score
    return 'generic', 0.0
```

---

## 4. Phase 2：静态结构抽取

### 4.1 抽取目标

在不执行代码的前提下，从项目中提取：

- **节点候选**：文件、目录、命令、外部角色（浏览器、用户）
- **边候选**：import / require / include 关系、脚本引用、构建工具配置中的入口关系

### 4.2 节点候选生成规则

| 来源 | 是否生成节点 | 节点 ID 示例 |
|------|-------------|--------------|
| 根目录配置文件 | 是 | `package.json`、`pyproject.toml` |
| 源码入口文件 | 是 | `src/main.tsx`、`main.py` |
| 核心应用文件 | 是 | `src/App.tsx`、`app.py` |
| 组件/模块目录下的文件 | 是 | `components/Header.tsx`、`routes/user.py` |
| 工具库文件 | 是 | `lib/markdown.ts`、`utils/logger.py` |
| 静态资源目录 | 是 | `public/`、`assets/`、`media/` |
| 测试目录 | 是 | `tests/`、`e2e/` |
| CI/DevOps 目录 | 是 | `.github/workflows/` |
| 构建产物目录 | 是 | `dist/`、`build/` |
| 命令脚本 | 是 | `pnpm build`、`python main.py` |
| 外部角色 | 是 | `浏览器`、`用户`、`运行完成` |

### 4.3 边候选生成规则

| 关系类型 | 如何发现 | 示例 |
|----------|---------|------|
| dependency（依赖） | 解析 `import` / `require` / `include` / `from ... import` | `App.tsx` → `Header.tsx` |
| build（构建） | 读取构建工具配置中的入口/输出 | `vite.config.ts` → `index.html` → `dist/` |
| run（运行） | 从 README、脚本命令、入口文件推导执行链 | `浏览器` → `index.html` → `main.tsx` → `App.tsx` |

### 4.4 输出

```json
{
  "rawNodes": [
    { "id": "package.json", "path": "package.json", "kind": "file" },
    { "id": "src/App.tsx", "path": "src/App.tsx", "kind": "file" }
  ],
  "rawEdges": [
    { "source": "src/App.tsx", "target": "src/components/Header.tsx", "kind": "import" }
  ]
}
```

---

## 5. Phase 3：语义分析与分类

### 5.1 通用节点分类（8 类）

所有项目类型共用以下分类和视觉编码：

| ID | 类别 | 颜色 | ECharts 符号 | 典型成员 | 布局区域 |
|----|------|------|-------------|----------|----------|
| 0 | 命令/入口 | `#f472b6` | diamond | `pnpm build`、浏览器、用户、CLI 命令 | 顶部 |
| 1 | 配置 | `#60a5fa` | circle | `package.json`、`pyproject.toml`、`tsconfig.json` | 左上 |
| 2 | 核心入口 | `#f87171` | circle | `index.html`、`main.py`、`app.py` | 中上 |
| 3 | 核心应用 | `#fb923c` | circle | `App.tsx`、`server.py`、主逻辑模块 | 中心 |
| 4 | 组件/子模块 | `#a78bfa` | circle | React 组件、路由模块、CLI 子命令 | 中下部 |
| 5 | 工具库 | `#2dd4bf` | circle | `lib/`、`utils/`、`helpers/` | 底部 |
| 6 | 资源/测试/DevOps | `#94a3b8` | circle | `public/`、`tests/`、`.github/` | 左右两侧 |
| 7 | 用户/输出 | `#fbbf24` | roundRect | `dist/`、`build/`、运行完成、API 响应 | 底部边缘 |

### 5.2 分类规则（按项目类型举例）

**frontend**:
- `package.json` / `vite.config.ts` → 1（配置）
- `index.html` → 2（核心入口）
- `src/main.tsx` → 2（核心入口）
- `src/App.tsx` → 3（核心应用）
- `src/components/*` → 4（组件）
- `src/lib/*` → 5（工具库）
- `public/` / `e2e/` → 6（资源/测试）
- `dist/` / `运行完成` → 7（用户/输出）

**python-backend**:
- `requirements.txt` / `pyproject.toml` → 1（配置）
- `main.py` / `app.py` → 2（核心入口）
- 路由聚合文件 / 应用工厂 → 3（核心应用）
- `routes/`、`models/`、`schemas/` → 4（组件/子模块）
- `services/`、`utils/`、`middleware/` → 5（工具库）
- `tests/`、`.github/` → 6（资源/测试）
- API 响应 / 运行完成 → 7（用户/输出）

### 5.3 边类型判定

| 原始关系 | 判定为 |
|----------|--------|
| 静态 import / require / include | `dependency`（虚线灰） |
| 构建工具读取配置并生成产物 | `build`（实线蓝） |
| 用户/命令触发程序执行，数据在模块间流动 | `run`（实线绿） |

### 5.4 构建路径（buildPath）推导

构建路径是一条从「人类触发构建」到「最终产物」的有序节点序列。

**frontend 默认 buildPath**:
```
pnpm build → package.json → tsconfig.json → vite.config.ts → tailwind.config.js
→ index.html → src/main.tsx → src/App.tsx
→ [components] → [lib tools] → src/index.css → src/defaultContent.ts
→ dist/
```

**python-backend 默认 buildPath**:
```
pyproject.toml / requirements.txt → Dockerfile（如有） → .github/workflows/
→ build artifact / deployed service
```

### 5.5 运行路径（runPath）推导

运行路径是一条从「外部触发」到「可观察结果」的有序节点序列。

**frontend 默认 runPath**:
```
浏览器 → index.html → src/main.tsx → src/App.tsx → src/defaultContent.ts
→ 用户输入 → EditorPanel → src/App.tsx → src/lib/markdown.ts
→ src/lib/themes/index.ts → src/lib/markdownIndexer.ts → PreviewPanel
→ 运行完成
```

**python-backend 默认 runPath**:
```
HTTP 请求 → main.py → app.py → routes/ → services/ → 数据库
→ services/ → routes/ → HTTP 响应 → 运行完成
```

---

## 6. Phase 4：比喻系统分配

### 6.1 比喻系统选择

按项目类型选择一套**自洽**的比喻系统：

| 项目类型 | 根比喻 |
|----------|--------|
| frontend | 厨房做菜 |
| python-backend | 餐厅后厨 + 服务员 |
| mobile | 剧院演出 |
| cli | 瑞士军刀 / 工具箱 |
| data-pipeline | 工厂流水线 |
| game | 游乐园 |
| generic | 办公室 / 工作坊（中性比喻） |

### 6.2 比喻映射表（frontend 示例）

| 节点 | 比喻 | shortLogic |
|------|------|------------|
| `package.json` | 菜谱 + 配料表 | 脚本/依赖清单 |
| `vite.config.ts` | 厨房电器设置 | Vite 配置 |
| `index.html` | 空盘子 | 浏览器入口 |
| `src/main.tsx` | 开火按钮 | 挂载应用 |
| `src/App.tsx` | 主厨 / 总指挥 | 状态与布局总管 |
| `EditorPanel.tsx` | 切菜板 / 编辑器 | 输入处理 |
| `PreviewPanel.tsx` | 成品展示盘 | 结果呈现 |
| `src/lib/markdown.ts` | Markdown 切菜机 | 解析转换 |
| `src/lib/wechatCompat.ts` | 微信口味调节师 | 平台兼容 |
| `dist/` | 最终菜品 | 构建产物 |
| `e2e/app.spec.ts` | 试吃员 | 端到端测试 |

### 6.3 比喻一致性规则

1. **同一项目内，同一类别的节点使用同一类比喻**。例如所有 frontend 配置都是「厨房电器/手册」相关。
2. **共享节点（如 index.html、main.tsx、App.tsx）的比喻要同时适用于构建和运行语境**。
3. **比喻不掩盖技术事实**：每个节点仍保留 `logic` 字段，用一句话准确说明它是做什么的。
4. **如无法生成恰当比喻，使用类别级 fallback**：
   - 配置 → 「项目设置书」
   - 工具库 → 「辅助工具」
   - 组件 → 「功能模块」

---

## 7. Phase 5：统一数据模型 ProjectGraph

### 7.1 Schema

```typescript
interface ProjectGraph {
  metadata: {
    projectName: string;           // 项目名称
    projectType: string;           // 检测出的类型，如 "frontend"
    metaphorSystem: string;        // 比喻系统名称，如 "cooking"
    rootMetaphor: string;          // 一句话总比喻
    generatedAt: string;           // ISO 时间戳
    version: string;               // 数据模型版本，如 "1.0.0"
  };

  nodes: Array<{
    id: string;                    // 唯一标识，通常等于文件路径或命令
    name: string;                  // 显示名称
    category: number;              // 0-7
    x: number;                     // ECharts 坐标 0-1600
    y: number;                     // ECharts 坐标 0-1000
    symbolSize: number;            // 节点大小
    symbol?: string;               // ECharts 符号覆盖
    shortLogic: string;            // 一句话功能
    logic: string;                 // 完整说明
    metaphor: string;              // 比喻名称
    role: string;                  // 它在项目中的角色
    trigger: string;               // 何时被触发
    deps: string[];                // 依赖的节点 ID 列表
    detail: string;                // guide 中展开显示的 HTML 内容
  }>;

  edges: Array<{
    source: string;                // 源节点 ID
    target: string;                // 目标节点 ID
    label: string;                 // 关系说明
    type: "build" | "run" | "dependency";
  }>;

  paths: {
    build: {
      name: string;                // "构建路径"
      description: string;         // 卡片中显示的一句话概括
      nodeIds: string[];           // 有序节点序列
      color: string;               // #60a5fa
    };
    run: {
      name: string;                // "运行路径"
      description: string;
      nodeIds: string[];
      color: string;               // #34d399
    };
  };

  guide: {
    buildSteps: Array<{
      title: string;
      subtitle: string;
      content: string;             // 支持 HTML
    }>;
    runSteps: Array<{
      title: string;
      subtitle: string;
      content: string;
    }>;
    fileTree: FileTreeNode;
  };
}

interface FileTreeNode {
  id: string;
  name: string;
  type: "folder" | "file";
  tagClass?: string;
  metaphor?: string;
  role?: string;
  trigger?: string;
  deps?: string[];
  detail?: string;
  children?: FileTreeNode[];
}
```

### 7.2 坐标布局规则

采用**语义网格布局**，而非自动力导向布局：

- **X 轴**：表示时间/阶段
  - 左侧：构建配置、命令
  - 中间：入口、核心应用、组件
  - 右侧：用户、输出、运行完成
- **Y 轴**：表示层级/深度
  - 上方：命令、配置、入口
  - 中间：核心应用、组件
  - 下方：工具库、输出

具体坐标分配：

| 区域 | X 范围 | Y 范围 | 用途 |
|------|--------|--------|------|
| 命令/入口 | 0-300 | 顶部 | `pnpm build`、浏览器、用户 |
| 配置 | 0-300 | 上部 | 各类 config 文件 |
| 核心入口 | 300-600 | 上部 | `index.html`、`main.py` |
| 核心应用 | 300-600 | 中部 | `App.tsx`、`app.py` |
| 组件/子模块 | 100-900 | 中下部 | 组件、路由、子命令 |
| 工具库 | 400-1000 | 底部 | `lib/`、`utils/` |
| 资源/测试 | 1000-1400 | 上部/两侧 | `public/`、`tests/`、`.github/` |
| 用户/输出 | 0-1200 | 底部边缘 | `dist/`、运行完成 |

---

## 8. Phase 6：HTML 产物生成

### 8.1 统一渲染策略

两个 HTML 文件共享同一份 `ProjectGraph` 数据：

```html
<script>
  const PROJECT_GRAPH = { /* ... */ };
</script>
```

HTML 模板包含固定 CSS/JS 骨架，仅数据部分不同。

### 8.2 project-map.html 渲染

- **标题**：`PROJECT_GRAPH.metadata.projectName + " 项目逻辑地图"`
- **节点**：`PROJECT_GRAPH.nodes` 直接作为 ECharts `series.data`
- **连线**：`PROJECT_GRAPH.edges` 直接作为 ECharts `series.links`
- **路径卡片**：使用 `PROJECT_GRAPH.paths.build` 和 `PROJECT_GRAPH.paths.run` 的描述和颜色
- **动画播放**：使用 `paths.build.nodeIds` 和 `paths.run.nodeIds` 作为播放序列
- **详情面板**：使用 `nodes[].logic`、`incoming`、`outgoing` 生成

### 8.3 project-guide.html 渲染

- **标签页 1（项目地图）**：把 `nodes` 按文件路径组织成树形结构 `guide.fileTree`
- **标签页 2（构建流程）**：把 `paths.build.nodeIds` 中的每个节点转换成时间线一步，标题用 `name`，内容用 `logic`
- **标签页 3（运行流程）**：把 `paths.run.nodeIds` 中的每个节点转换成时间线一步
- **总比喻说明**：在页面顶部展示 `metadata.rootMetaphor`

### 8.4 交互功能清单

| 功能 | project-map.html | project-guide.html |
|------|------------------|-------------------|
| 节点/文件详情 | ✅ 点击节点右侧显示 | ✅ 点击文件展开 |
| 构建路径高亮 | ✅ 点击顶部卡片 | ✅ 标签页 2 时间线 |
| 运行路径高亮 | ✅ 点击顶部卡片 | ✅ 标签页 3 时间线 |
| 播放动画 | ✅ 顶部播放控制区 | ❌（静态指南） |
| 拖拽/缩放 | ✅ ECharts roam | ❌ |

---

## 9. 校验与质量检查

每个阶段结束后必须通过的自动化检查：

| 阶段 | 检查项 | 通过标准 | 失败处理 |
|------|--------|---------|----------|
| Phase 1 | 类型检测置信度 | `score >= 0.5` | 告警，默认 `generic`，允许人工覆盖 |
| Phase 2 | 悬空边检查 | 所有 `edge.source` 和 `edge.target` 都存在于 `nodes` | 删除悬空边并记录日志 |
| Phase 2 | 孤立节点检查 | 除外部角色外，所有节点至少有一条边 | 标记孤立节点待人工确认 |
| Phase 3 | buildPath 起点 | 第一个节点 category 必须为 0（命令/入口） | 报错 |
| Phase 3 | runPath 终点 | 最后一个节点 category 必须为 7（用户/输出） | 报错 |
| Phase 3 | buildPath 无环 | buildPath 节点序列无重复 | 告警并去重 |
| Phase 4 | 比喻完整性 | 所有节点 `metaphor` 非空 | 使用类别级 fallback |
| Phase 4 | 比喻一致性 | 同类别节点的比喻词汇属于同一语义场 | 标记不一致项 |
| Phase 5 | JSON Schema 校验 | 符合 ProjectGraph Schema | 报告违规字段 |
| Phase 5 | 路径节点存在性 | `paths.build.nodeIds` 和 `paths.run.nodeIds` 都在 `nodes` 中 | 删除无效项并记录 |
| Phase 6 | JS 控制台无错 | 浏览器打开 `project-map.html` 无报错 | 修复模板语法 |
| Phase 6 | 标签页非空 | `guide.buildSteps` 和 `guide.runSteps` 长度 > 0 | 补充默认步骤 |

---

## 10. 自定义指南

### 10.1 新增项目类型

1. 在 Phase 1 检测矩阵中增加该类型的信号文件。
2. 在 Phase 4 比喻系统表中增加一套自洽比喻。
3. 在 Phase 5 分类规则中补充该类项目的默认分类映射。
4. 在 Phase 6 模板中确保颜色/符号通用（不需要改）。

### 10.2 覆盖默认比喻

允许用户在项目根目录放置 `project-decomposition.config.json`：

```json
{
  "metaphorOverrides": {
    "src/App.tsx": "乐队指挥",
    "src/lib/markdown.ts": "乐谱翻译器"
  }
}
```

### 10.3 调整布局

用户可在配置中指定关键节点的坐标：

```json
{
  "layoutOverrides": {
    "src/App.tsx": { "x": 500, "y": 450 }
  }
}
```

---

## Appendix A：完整 ProjectGraph 示例（raphael-publish 片段）

```json
{
  "metadata": {
    "projectName": "Raphael Publish",
    "projectType": "frontend",
    "metaphorSystem": "cooking",
    "rootMetaphor": "把项目比作做一道菜：菜谱决定做什么，食材是第三方库，厨师是核心代码，最终菜品是构建产物。",
    "generatedAt": "2026-06-22T00:00:00Z",
    "version": "1.0.0"
  },
  "nodes": [
    {
      "id": "package.json",
      "name": "package.json",
      "category": 1,
      "x": 150,
      "y": 170,
      "symbolSize": 30,
      "shortLogic": "脚本/依赖清单",
      "logic": "项目身份证。scripts 定义命令菜单，dependencies 列出所有第三方库。",
      "metaphor": "菜谱 + 配料表",
      "role": "定义项目命令和依赖",
      "trigger": "任何 pnpm/npm 命令启动时首先读取",
      "deps": [],
      "detail": "..."
    }
  ],
  "edges": [
    { "source": "pnpm build", "target": "package.json", "label": "读取 build 脚本", "type": "build" }
  ],
  "paths": {
    "build": {
      "name": "构建路径",
      "description": "pnpm build → package.json → tsc/vite → index.html → main.tsx → App.tsx → 组件/lib → dist/",
      "nodeIds": ["pnpm build", "package.json", "vite.config.ts", "index.html", "src/main.tsx", "src/App.tsx", "dist/"],
      "color": "#60a5fa"
    },
    "run": {
      "name": "运行路径",
      "description": "浏览器 → index.html → main.tsx → App.tsx → 用户输入 → EditorPanel → lib → PreviewPanel → 运行完成",
      "nodeIds": ["浏览器", "index.html", "src/main.tsx", "src/App.tsx", "PreviewPanel", "运行完成"],
      "color": "#34d399"
    }
  },
  "guide": {
    "buildSteps": [
      { "title": "1. 输入构建命令", "subtitle": "pnpm build", "content": "你在终端输入的构建命令..." }
    ],
    "runSteps": [
      { "title": "1. 浏览器请求页面", "subtitle": "浏览器", "content": "用户在浏览器中打开项目地址..." }
    ],
    "fileTree": { "id": "root", "name": "raphael-publish", "type": "folder", "children": [] }
  }
}
```

---

## Appendix B：项目类型检测矩阵

| 类型 | 根文件/目录信号 | 依赖信号 | 默认 metaphorSystem |
|------|----------------|---------|---------------------|
| frontend | `package.json` + `index.html` | react/vue/angular | cooking |
| python-backend | `pyproject.toml`/`requirements.txt` + `app.py` | fastapi/flask/django | restaurant |
| mobile | `pubspec.yaml`/`Podfile`/`build.gradle` | flutter/react-native | theater |
| cli | `package.json` 含 `bin`/`setup.py` 含 `entry_points` | commander/yargs/argparse | toolbox |
| data-pipeline | `dags/`/`etl/`/`pipeline/` | pandas/spark/dbt | factory |
| game | Unity 文件/`Cargo.toml` 含 bevy/`game/` | bevy/godot/unity | amusement-park |
| generic | 无 | 无 | workshop |

---

## Appendix C：比喻速查表

### 厨房做菜（frontend）

| 技术概念 | 比喻 |
|----------|------|
| 项目整体 | 做一道菜 |
| package.json | 菜谱 + 配料表 |
| node_modules | 食材仓库 |
| vite.config.ts | 厨房电器设置 |
| tsconfig.json | 厨房规则手册 |
| index.html | 空盘子 |
| main.tsx | 开火按钮 |
| App.tsx | 主厨 / 总指挥 |
| components | 厨具 / 小工 |
| lib | 调料 / 配方 |
| public | 装饰配料 |
| dist | 最终菜品 |
| tests | 试吃员 |
| CI/CD | 自动上菜机器人 |

### 餐厅后厨（python-backend）

| 技术概念 | 比喻 |
|----------|------|
| 项目整体 | 餐厅运营 |
| requirements.txt | 供应商合同 |
| app.py | 迎宾台 |
| routes | 服务员 |
| services | 厨师 |
| models | 菜单 / 菜品规格 |
| middleware | 传菜员 |
| database | 食材冷库 |
| API response | 上桌的餐点 |
| tests | 神秘顾客 |

### 工厂流水线（data-pipeline）

| 技术概念 | 比喻 |
|----------|------|
| 项目整体 | 工厂生产 |
| config | 工厂蓝图 |
| raw data | 原材料 |
| extractor | 上料机 |
| transformer | 加工机床 |
| loader | 包装机 |
| scheduler | 排班系统 |
| output dataset | 包装产品 |
| monitoring | 质检员 |

### 瑞士军刀（cli）

| 技术概念 | 比喻 |
|----------|------|
| 项目整体 | 瑞士军刀 |
| CLI entry | 打开刀鞘 |
| main command | 主刀片 |
| subcommands | 螺丝刀 / 开瓶器 / 锉刀 |
| options/flags | 使用角度 / 力度 |
| output | 完成的小任务 |
| help text | 使用说明书 |

### 游乐园（game）

| 技术概念 | 比喻 |
|----------|------|
| 项目整体 | 游乐园 |
| game loop | 游乐设施控制室 |
| player | 游客 |
| input system | 检票闸机 |
| physics engine | 过山车轨道 |
| renderer | 灯光舞台 |
| audio | 园区广播 |
| score | 游玩积分 |
| save file | 纪念照片 |

---

## 使用建议

1. **先用一个熟悉项目跑通全流程**，再应用到新项目。
2. **比喻系统不是唯一答案**。如果某个比喻对目标受众不合适，用 Appendix C 中同类型的其他比喻替换。
3. **不要为了追求自动化而牺牲准确性**。自动检测和推导完成后，人工复核 buildPath 和 runPath 的顺序。
4. **保持产物独立**。生成的 HTML 文件应内嵌所有资源，不依赖外部构建工具。

---

*版本：1.0.0*
*基于 raphael-publish 项目实践总结*
