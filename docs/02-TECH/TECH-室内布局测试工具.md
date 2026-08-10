# TECH：室内布局测试工具技术设计

> 版本：v1.0 | 日期：2026-08-10 | 关联：PRD-室内布局测试工具.md

## 1. 技术选型

| 项 | 选择 | 理由 |
|----|------|------|
| 形态 | 纯前端单页面（SPA） | 无后端、无账号，打开即用 |
| 框架 | **原生 HTML + CSS + TypeScript（可选）或纯 JS + Canvas 2D** | 轻量、零依赖、无需构建工具；Canvas 2D 足够满足家具渲染与交互 |
| 渲染 | Canvas 2D | 家具数量少（≤100），Canvas 足够流畅；可导出 PNG |
| 构建 | 可选 Vite（若用 TS）；也可单 HTML 文件 | 保持极简：优先单文件 index.html 可直接双击打开 |
| 数据持久化 | localStorage | 自动保存当前方案，刷新不丢失 |
| 依赖 | 无第三方库（纯手写） | 碰撞/吸附算法手写可控，避免引入大型库 |

> **建议**：为方便 CEO 直接测试，MVP 采用**单 HTML 文件**（内嵌 CSS+JS），浏览器双击即用。若后续扩展 3D/多人分享再引入框架。

## 2. 目录结构

```
room-layout-planner/
├── index.html          # 单页应用（MVP：内嵌全部 CSS/JS）
├── assets/             # 图标等静态资源（可选，优先用 Unicode/emoji/CSS 图形）
└── docs/               # 设计文档（本目录）
    ├── README.md
    ├── 01-PRD/
    ├── 02-TECH/
    ├── 03-UI/
    └── 05-实施计划/
```

## 3. 核心数据结构

### 3.1 户型（房间）

```typescript
interface Room {
  id: string;
  name: string;            // 客厅/卧室...
  walls: Wall[];           // 闭合墙体
}

interface Wall {
  id: string;
  x1: number; y1: number;  // 起点（mm）
  x2: number; y2: number;  // 终点（mm）
  thickness: number;       // 墙厚（mm，默认120）
}
```

- 绘制方式：用户点击起点 → 移动预览 → 点击终点生成一段墙；**双击/闭合检测**（终点接近起点 <50mm 时自动闭合）。
- 房间多边形由墙体围合，用于"家具越界检测"（判断家具是否在房间内）。

### 3.2 家具

```typescript
interface FurnitureItem {
  id: string;
  type: string;            // 'sofa' | 'teaTable' | 'tvCabinet' | 'diningTable' | 'diningChair' | 'bed' | 'desk' | 'studyDesk' ...
  name: string;            // 沙发
  width: number;           // 长（mm）
  depth: number;           // 宽（mm）
  x: number; y: number;    // 中心点坐标（mm）
  rotation: number;        // 旋转角度（度，0-360）
  color: string;           // 显示颜色
}

// 家具类型元数据（默认尺寸等）
interface FurnitureTypeMeta {
  type: string;
  name: string;
  defaultWidth: number;    // 默认长 mm
  defaultDepth: number;    // 默认宽 mm
  color: string;
  icon: string;            // 面板图标（emoji/SVG）
}
```

### 3.3 应用状态

```typescript
interface AppState {
  rooms: Room[];
  furniture: FurnitureItem[];
  selectedId: string | null;
  snapEnabled: boolean;        // 吸附开关
  collisionMode: 'block' | 'warn';  // 碰撞处理：阻止放置 or 仅警告
  gridSize: number;            // 网格 100mm
  scale: number;               // px/mm 比例
  panX: number; panY: number;  // 视口平移
}
```

## 4. 核心算法

### 4.1 渲染

- 坐标系统：**业务坐标（mm）↔ 屏幕坐标（px）**，通过 `toScreen()` / `toWorld()` 转换。
- 每帧重绘：网格 → 墙体 → 家具（按类型绘制矩形+图标+名称）→ 选中框 → 碰撞高亮。
- 家具旋转渲染：`ctx.translate(x,y) + ctx.rotate(rad)` 后绘制矩形。

### 4.2 拖拽与旋转交互

| 交互 | 实现 |
|------|------|
| 拖入家具 | 面板 mousedown → 画布 drag 放落（或点击面板→家具出现在画布中心再拖动） |
| 移动家具 | 选中 → 鼠标按住拖动 → 更新 x/y → 实时吸附/碰撞检测 |
| 旋转 | 选中后：① 键盘 ←/→ 每步 5° ② 拖旋转手柄 ③ 右侧输入框精确输入 |
| 缩放平移 | 滚轮缩放（以鼠标为中心），空格+拖动/中键平移 |

### 4.3 吸附算法（Snap）

**触发条件**：拖动家具时，若 `snapEnabled=true`。

| 吸附目标 | 规则 |
|---------|------|
| 网格吸附 | 家具中心/边缘与最近网格线距离 < 阈值（默认 15px）→ 对齐网格线 |
| 墙体吸附 | 家具边缘与墙体边缘距离 < 阈值 → 对齐墙边（家具靠墙摆放） |
| 家具间吸附 | 家具边缘与另一家具边缘距离 < 阈值 → 对齐（家具并排） |

- 吸附距离阈值可在设置面板调整（0-50px，默认 15px）。
- 吸附优先：墙 > 家具 > 网格。
- 实现：拖动时计算候选吸附位置集合 → 取偏移最小的目标 → 应用偏移。

### 4.4 碰撞检测（Collision）

**检测对象**：
1. 家具 ↔ 家具：AABB（轴对齐包围盒）+ 旋转后 SAT（分离轴定理）精确检测
2. 家具 ↔ 墙体/房间边界：点/矩形与多边形内外检测（射线法）→ 家具中心或四角越界检测

**处理策略**（`collisionMode`）：
| 模式 | 行为 |
|------|------|
| `block`（默认） | 发生碰撞时家具显示红色 + 松手回弹到上一个合法位置（禁止放置） |
| `warn` | 允许放置，但碰撞区域红色高亮提示 + 状态栏计数 |

**SAT 要点**：旋转矩形碰撞用分离轴定理，取两矩形 4 条边法线投影，若有任一轴无重叠 → 不碰撞；否则碰撞。家具数量少（≤100），O(n²) 检测可接受。

## 5. 家具默认尺寸表（与 PRD 一致，代码常量）

```typescript
const FURNITURE_TYPES: FurnitureTypeMeta[] = [
  { type: 'sofa',        name: '沙发',     defaultWidth: 2100, defaultDepth: 900,  color: '#4a7fb5' },
  { type: 'teaTable',    name: '茶几',     defaultWidth: 1200, defaultDepth: 600,  color: '#8d6e63' },
  { type: 'tvCabinet',   name: '电视柜',   defaultWidth: 1800, defaultDepth: 400,  color: '#5d4037' },
  { type: 'diningTable', name: '餐桌',     defaultWidth: 1400, defaultDepth: 800,  color: '#a1887f' },
  { type: 'diningChair', name: '餐椅',     defaultWidth: 450,  defaultDepth: 450,  color: '#7cb342' },
  { type: 'bed',         name: '床',       defaultWidth: 1800, defaultDepth: 2000, color: '#90a4ae' },
  { type: 'desk',        name: '电脑桌',   defaultWidth: 1200, defaultDepth: 600,  color: '#b39ddb' },
  { type: 'studyDesk',   name: '学习桌',   defaultWidth: 1000, defaultDepth: 600,  color: '#ffb74d' },
];
```

## 6. UI 布局（见 UI 效果图）

```
┌──────────┬──────────────────────────────┬────────────┐
│ 家具面板  │        画布（Canvas）          │  属性面板   │
│ (左侧)   │  工具栏：画墙/选家具/移动/旋转    │  (右侧)    │
│ 沙发     │  网格 + 墙体 + 家具             │ 尺寸/角度  │
│ 茶几     │                              │ 吸附开关   │
│ ...      │                              │ 碰撞模式   │
└──────────┴──────────────────────────────┴────────────┘
底部状态栏：坐标 / 缩放 / 家具数 / 碰撞提示
```

## 7. 数据持久化

- 每次操作后 debounce 500ms 自动保存 `localStorage['room-layout-v1']`。
- 打开页面时自动恢复上次方案。
- 导出 JSON：下载文件 `layout-YYYYMMDD-HHmm.json`；导入：文件选择器读取恢复。

## 8. 风险与对策

| 风险 | 对策 |
|------|------|
| 吸附算法过于敏感/抖动 | 吸附阈值可调；应用偏移后做死区处理（偏移 > 阈值才吸附） |
| 旋转后碰撞误判 | 用 SAT 精确检测，不用 AABB 近似 |
| 复杂户型绘制繁琐 | 提供模板 + 墙厚默认 120 + 网格辅助 |
| 单文件过大 | 控制在 ~150KB 内；图标用 emoji/SVG 内联 |
