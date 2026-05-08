# 天府七中 G2020 级蹭饭图 — Code Wiki

> 本文档是对项目仓库的完整技术分析，涵盖架构设计、模块职责、关键函数说明、依赖关系及运行方式。

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术架构总览](#2-技术架构总览)
3. [项目目录结构](#3-项目目录结构)
4. [模块职责详解](#4-模块职责详解)
   - 4.1 [入口层 — index.html](#41-入口层--indexhtml)
   - 4.2 [配置层 — js/config.js](#42-配置层--jsconfigjs)
   - 4.3 [核心逻辑层 — js/main.js](#43-核心逻辑层--jsmainjs)
   - 4.4 [样式层 — css/style.css](#44-样式层--cssstylecss)
   - 4.5 [数据层 — data/classN.js](#45-数据层--dataclassnjs)
5. [核心机制深度解析](#5-核心机制深度解析)
   - 5.1 [口令验证流程](#51-口令验证流程)
   - 5.2 [地理编码队列](#52-地理编码队列)
   - 5.3 [标记渲染与班级筛选联动](#53-标记渲染与班级筛选联动)
   - 5.4 [多班同校合并算法](#54-多班同校合并算法)
   - 5.5 [双模式搜索系统](#55-双模式搜索系统)
   - 5.6 [版本号隔离机制](#56-版本号隔离机制)
   - 5.7 [容错与 API 额度检测](#57-容错与-api-额度检测)
6. [关键函数索引](#6-关键函数索引)
7. [全局状态变量清单](#7-全局状态变量清单)
8. [依赖关系图](#8-依赖关系图)
9. [数据格式规范](#9-数据格式规范)
10. [项目运行与部署](#10-项目运行与部署)
11. [扩展指南](#11-扩展指南)

---

## 1. 项目概述

**天府七中 G2020 级蹭饭图**是一个展示 2020 届高中毕业生就读大学地理分布的可视化 Web 工具。以天地图卫星影像为底图，将全年级四个班同学的大学去向以彩色标记呈现在全国地图上。

### 核心功能

| 功能 | 说明 |
|------|------|
| 身份验证 | SHA-256 哈希口令验证，源码不存明文 |
| 班级筛选 | 1–4 班复选框，支持任意组合，多班同校自动合并 |
| 地理编码 | 预置坐标优先 → 冲突检测 → 天地图 API 编码，限速 250ms/条 |
| 双模式搜索 | 同学姓名搜索（优先）→ 大学/城市搜索 |
| 数据统计 | 城市 Top 8 + 院校 Top 8 水平柱状图 |
| 数据缺失 | 按班级分组展示缺失去向的同学名单 |
| 响应式适配 | 双断点（768px / 480px）覆盖平板与手机 |

---

## 2. 技术架构总览

```
┌──────────────────────────────────────────────────────────────┐
│                        浏览器运行时                           │
│                                                              │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐  ┌────────────┐  │
│  │ index    │  │ js/       │  │ css/     │  │ data/      │  │
│  │ .html    │  │ config.js │  │ style.css│  │ class1-4.js│  │
│  │ (DOM结构)│  │ (配置中心) │  │ (样式体系)│  │ (原始数据) │  │
│  └────┬─────┘  └─────┬─────┘  └──────────┘  └─────┬──────┘  │
│       │              │                            │         │
│       │    ┌─────────▼──────────┐                 │         │
│       │    │   js/main.js       │◄────────────────┘         │
│       │    │   (核心业务逻辑)    │                           │
│       │    │  · 口令验证         │                           │
│       │    │  · 地图初始化       │                           │
│       │    │  · 地理编码队列     │                           │
│       │    │  · 标记渲染/搜索    │                           │
│       │    │  · 统计/缺失面板    │                           │
│       │    └────────┬───────────┘                           │
│       │             │                                       │
│  ┌────▼─────────────▼──────────────────────────────────┐    │
│  │              外部 API 依赖                            │    │
│  │  · 天地图 JavaScript API v4.0 (地图引擎)             │    │
│  │  · 天地图 Geocoder 服务 (地理编码)                    │    │
│  │  · Web Crypto API (SHA-256 哈希)                     │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### 技术选型

| 层次 | 技术 | 说明 |
|------|------|------|
| 标记语言 | HTML5 | 语义化结构，ARIA 无障碍属性 |
| 样式 | 纯 CSS | CSS 变量体系、Flexbox、backdrop-filter、双断点响应式 |
| 脚本 | 纯原生 JavaScript | ES5 兼容风格，零第三方框架依赖 |
| 地图引擎 | 天地图 JavaScript API v4.0 | 卫星影像底图（WMTS 瓦片服务） |
| 地理编码 | 天地图 Geocoder | T.Geocoder 服务 |
| 密码学 | Web Crypto API | crypto.subtle.digest('SHA-256') |
| 图标 | 自定义 SVG Pin | feDropShadow 滤镜，data URI 内联 |

---

## 3. 项目目录结构

```
/workspace/
├── index.html              # 主页面（DOM 结构 + API 引用 + 启动引导脚本）
├── README.md               # 项目说明文档
├── favicon.ico             # 浏览器标签页图标
├── css/
│   └── style.css           # 全局样式（~1015 行，含响应式适配）
├── js/
│   ├── config.js           # 项目总配置（集中管理所有可调参数）
│   └── main.js             # 主逻辑（~1482 行，含全部业务代码）
└── data/
    ├── class1.js           # 1 班数据（class1Data + class1MissingData）
    ├── class2.js           # 2 班数据（class2Data + class2MissingData）
    ├── class3.js           # 3 班数据（class3Data + class3MissingData）
    └── class4.js           # 4 班数据（class4Data + class4MissingData）
```

---

## 4. 模块职责详解

### 4.1 入口层 — index.html

**文件位置**: [index.html](file:///workspace/index.html)

**职责**: 全部 DOM 结构定义与脚本加载编排。

**DOM 结构组成**:

| 区域 | HTML 元素 | 功能 |
|------|-----------|------|
| 口令验证遮罩 | `.password-gate` | 全屏遮罩，验证通过后淡出 |
| 顶部导航栏 | `.header` | 品牌标题 + 搜索框 + 操作按钮 |
| 搜索结果导航条 | `.search-nav` | 多结果时显示 ← / → 导航 |
| 地图容器 | `#map-container` | 天地图渲染区域 |
| 数据缺失按钮 | `.missing-panel` | 金色边框按钮，有缺失时显示 |
| 班级筛选面板 | `.class-panel` | 右下角浮动面板，含复选框 |
| 加载遮罩 | `.loading-overlay` | 地理编码进行时显示 |
| 关于模态框 | `#aboutModal` | 项目说明弹窗 |
| 数据统计模态框 | `#statsModal` | 城市/院校 Top 8 统计 |
| 数据缺失模态框 | `#missingDataModal` | 缺失同学名单 |
| Toast 提示 | `#toast` | 底部短暂浮条通知 |

**脚本加载顺序**（关键，有依赖关系）:

```
1. js/config.js           → 定义 CONFIG 全局对象
2. data/class1~4.js       → 定义 classNData / classNMissingData 全局变量
   （由 index.html 中 IIFE 根据 CONFIG.classCount 动态 document.write）
3. js/main.js             → 定义 onTMapCallback 及所有业务逻辑
4. 天地图 API v4.0        → 动态创建 <script> 标签加载
5. bootstrapTMap 轮询     → 30 次重试（100ms 间隔）等待 T 全局对象就绪后调用 onTMapCallback
```

---

### 4.2 配置层 — js/config.js

**文件位置**: [js/config.js](file:///workspace/js/config.js)

**职责**: 集中管理所有可调配置参数，其余代码通过读取 `CONFIG` 全局对象来运行。

**配置项清单**:

| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `pageTitle` | string | `'天府第七中学 G2020 级蹭饭图'` | 导航栏标题文字 |
| `passwordPrompt` | string | `'这是哪所学校？'` | 口令验证界面提示问题 |
| `validPasswordHashes` | string[] | 3 个 SHA-256 哈希值 | 合法口令哈希数组，任一匹配即通过 |
| `tiandituTK` | string | `'7403bbf...'` | 天地图 API 密钥 |
| `classCount` | number | `4` | 班级总数（核心配置） |
| `classColors` | object | `{1:'#a74bb6', 2:'#10b981', 3:'#f59e0b', 4:'#a3292b'}` | 各班代表颜色 |
| `classNames` | object | `{1:'1 班', ...}` | 各班 UI 显示名称 |
| `colorNameMap` | object | 颜色→中文名映射 | 用于关于弹窗等 UI 文案 |
| `mergedMarkerColor` | string | `'#9ca3af'` | 多班同校合并标记颜色（灰色） |
| `geocodeInterval` | number | `250` | 地理编码请求间隔（ms） |
| `geocodeTimeout` | number | `8000` | 地理编码请求超时（ms） |
| `maxConsecutiveFailures` | number | `5` | 连续失败阈值 |
| `tileErrorThreshold` | number | `8` | 瓦片加载失败阈值 |
| `maxGeoCacheSize` | number | `500` | 地理编码缓存最大条目数 |
| `searchDebounceDelay` | number | `300` | 搜索防抖延迟（ms） |

**配置验证**: 文件末尾的 IIFE `validateConfig()` 在加载时立即执行，检查必需配置项是否存在、类型是否正确，验证失败则抛出异常。

---

### 4.3 核心逻辑层 — js/main.js

**文件位置**: [js/main.js](file:///workspace/js/main.js)

**职责**: 全部业务逻辑，约 1482 行，包含以下功能模块：

#### 模块划分

```
js/main.js
├── 全局状态声明 (L1~L92)         — 地图实例、缓存、版本号、队列等
├── 地理编码队列 (L93~L196)       — enqueueGeocode / processNextGeocode
├── 班级面板动态生成 (L198~L257)   — buildClassPanel / injectClassColorCSS
├── 口令验证 (L259~L350)          — sha256Hex / initPasswordGate / setupBrandTitle
├── DOMContentLoaded 初始化 (L379~L436) — 事件绑定、数据加载
├── 地图初始化 (L438~L495)        — onTMapCallback（天地图 API 回调）
├── 地图 UI 事件 (L497~L540)      — setupMapEventListeners
├── 标记加载/移除 (L542~L631)     — getSelectedClasses / groupSelectedByUniversity / renderSelectedMarkers
├── 标记创建 (L633~L731)          — createPinIcon / addMarkerToMap / createMarker / buildInfoWindowHTML
├── 搜索系统 (L733~L1109)         — performSearch / findStudentMatchesByName / findUniversityMatches / goToSearchResult 等
├── 模态框管理 (L1111~L1118)      — openModal / closeModal
├── 数据缺失面板 (L1120~L1204)    — updateMissingDataToggle / buildMissingDataContent
├── 数据统计 (L1206~L1281)        — buildStatsContent
└── 辅助函数 (L1283~L1482)        — getClassName / escapeHTML / getUniversityBaseName / parseStudentCoordinate 等
```

---

### 4.4 样式层 — css/style.css

**文件位置**: [css/style.css](file:///workspace/css/style.css)

**职责**: 全局样式体系，约 1015 行。

**CSS 变量体系** (`:root` 定义):

| 变量 | 值 | 用途 |
|------|----|------|
| `--hdr-h` | `62px` | 导航栏高度 |
| `--hdr-bg` | `rgba(10, 20, 42, 0.97)` | 导航栏背景色 |
| `--panel-bg` | `rgba(255, 255, 255, 0.97)` | 面板背景色 |
| `--panel-shadow` | `0 8px 36px rgba(0,0,0,0.18)` | 面板阴影 |
| `--radius` | `14px` | 全局圆角 |
| `--brand` | `#3b82f6` | 品牌蓝 |
| `--text` | `#1e293b` | 主文本色 |
| `--muted` | `#64748b` | 辅助文本色 |
| `--ease` | `0.2s cubic-bezier(0.4, 0, 0.2, 1)` | 全局过渡曲线 |

**样式模块划分**:

| 模块 | 行范围 | 说明 |
|------|--------|------|
| CSS 变量 + Reset | L1~L28 | 全局变量定义与 box-sizing 重置 |
| 口令验证过渡 | L31~L44 | 验证通过后导航栏/面板滑入动画 |
| 顶部导航栏 | L46~L219 | 品牌区、搜索框、操作按钮样式 |
| 搜索结果导航条 | L127~L200 | 多结果导航条样式 |
| 地图容器 | L221~L230 | 全屏地图容器 |
| 班级筛选面板 | L232~L314 | 右下角浮动面板 + 自定义复选框 |
| 加载遮罩 | L316~L348 | 地理编码进行时的加载蒙层 |
| 模态框（通用） | L350~L420 | 弹窗基础样式 + pop-in 动画 |
| 关于模态框内容 | L422~L446 | 项目说明弹窗内容样式 |
| 数据缺失面板 | L448~L530 | 金色边框按钮 + 缺失名单样式 |
| 数据统计内容 | L532~L570 | 柱状图 + 跑马灯样式 |
| Toast 提示 | L572~L594 | 底部浮条通知样式 |
| 地图控件微调 | L596~L615 | 天地图缩放控件 + 信息窗样式覆盖 |
| 口令验证遮罩 | L617~L729 | 验证卡片 + shake 抖动动画 |
| 响应式 ≤768px | L731~L936 | 平板断点适配 |
| 响应式 ≤480px | L938~L1015 | 小屏手机断点适配 |

**视觉效果亮点**:
- 导航栏：深色半透明 + `backdrop-filter: blur(20px)` 毛玻璃
- 品牌标题：`linear-gradient` 渐变剪裁文字
- 模态框：`pop-in` 弹性缩放入场动画
- 口令输入框：`shake` 关键帧抖动
- 自定义复选框：圆形 `<span>` 替代原生 checkbox
- 跑马灯：`marquee-scroll` 关键帧，长校名自动滚动

---

### 4.5 数据层 — data/classN.js

**文件位置**: [data/class1.js](file:///workspace/data/class1.js) ~ [data/class4.js](file:///workspace/data/class4.js)

**职责**: 每个文件导出两个全局变量，提供班级原始数据。

| 全局变量 | 类型 | 说明 |
|----------|------|------|
| `classNData` | `Array<Object>` | 已知去向的同学列表 |
| `classNMissingData` | `Array<Object>` | 缺失去向的同学列表 |

**数据统计**:

| 班级 | 已知去向 | 缺失数据 | 合计 |
|------|----------|----------|------|
| 1 班 | 39 | 1 | 40 |
| 2 班 | 33 | 3 | 36 |
| 3 班 | 41 | 4 | 45 |
| 4 班 | 43 | 0 | 43 |

---

## 5. 核心机制深度解析

### 5.1 口令验证流程

```
用户输入口令
    │
    ▼
sha256Hex(input)  ← Web Crypto API 计算 SHA-256
    │
    ▼
遍历 CONFIG.validPasswordHashes 逐一比对
    │
    ├─ 匹配成功 → .header/.class-panel 添加 .reveal 类（滑入动画）
    │             → .password-gate 添加 .dismiss 类（淡出消失）
    │
    └─ 匹配失败 → 显示错误信息 + 输入框 shake 抖动动画
```

**安全设计**: 源码中仅存储 SHA-256 哈希值，不包含任何口令明文。支持多个合法哈希值（全称、简称等）。

**关键函数**:
- [sha256Hex()](file:///workspace/js/main.js#L266-L280) — SHA-256 哈希计算
- [initPasswordGate()](file:///workspace/js/main.js#L288-L341) — 口令验证初始化与事件绑定

---

### 5.2 地理编码队列

```
数据加载 → 按大学分组 → 检查缓存/预置坐标 → 加入队列
                                                    │
                                    processNextGeocode() 逐条执行
                                                    │
                      天地图 Geocoder 搜索 "城市名市 + 大学名"
                                                    │
                         ├─ 成功 → 缓存结果 → 回调添加标记
                         ├─ 失败 → 缓存 null → 回调跳过
                         └─ 超时 → 不缓存   → 回调跳过 + 错误计数
                                                    │
                              间隔 250ms → 处理下一条
```

**缓存策略**:
- `geoCache` 对象以大学全名为键，值为 `{ point: T.LngLat | null, timestamp: number }`
- 成功和"未找到结果"均缓存，超时不缓存（允许重试）
- 缓存满时（≥500 条），使用 LRU 策略淘汰最旧条目（`findOldestCacheKey`）

**限速保护**: 每条请求间隔 250ms（`CONFIG.geocodeInterval`），避免触发天地图 API 频率限制。

**防重复回调**: 每条请求使用 `completed` 标志位防止超时回调与正常回调双重触发。

**关键函数**:
- [enqueueGeocode()](file:///workspace/js/main.js#L94-L118) — 入队地理编码请求
- [processNextGeocode()](file:///workspace/js/main.js#L139-L196) — 逐条执行队列
- [findOldestCacheKey()](file:///workspace/js/main.js#L121-L136) — LRU 缓存淘汰
- [parseGeocodeResult()](file:///workspace/js/main.js#L1320-L1350) — 解析天地图返回结果

---

### 5.3 标记渲染与班级筛选联动

```
用户勾选/取消班级复选框
    │
    ▼
renderSelectedMarkers()
    │
    ├─ 重置错误状态（consecutiveGeocodeFailures 等）
    ├─ 递增 markerRenderVersion（使过期回调失效）
    ├─ clearActiveMarkers()（清空所有现有标记）
    │
    ▼
groupSelectedByUniversity(selectedClasses)
    │
    ├─ 遍历已勾选班级的所有同学
    ├─ 按 university 字段分组
    ├─ 统计 totalStudents / classNums[] / studentsByClass{}
    ├─ 解析预置坐标（parseStudentCoordinate）
    ├─ 检测坐标冲突（isSameCoordinate）
    │
    ▼
对每个分组:
    ├─ 有预置坐标且无冲突 → 直接 addMarkerToMap()
    └─ 否则 → enqueueGeocode() → 回调中 addMarkerToMap()
```

**关键函数**:
- [renderSelectedMarkers()](file:///workspace/js/main.js#L609-L631) — 核心渲染入口
- [groupSelectedByUniversity()](file:///workspace/js/main.js#L558-L597) — 大学分组算法
- [addMarkerToMap()](file:///workspace/js/main.js#L649-L661) — 添加标记到地图
- [createMarker()](file:///workspace/js/main.js#L663-L692) — 创建单个标记

---

### 5.4 多班同校合并算法

```
groupSelectedByUniversity() 输出:
{
  university: "北京大学",
  city: "北京",
  coordinate: { lng, lat } | null,
  coordinateConflict: boolean,
  totalStudents: 5,
  classNums: [1, 2],           ← 多个班级
  studentsByClass: {
    1: ["郑睿恒", "张楚媛"],
    2: ["文帝文"]
  }
}
```

**合并规则**:
1. 遍历所有勾选班级中每个同学的 `university` 字段
2. 按大学名分组，统计 `totalStudents`、`classNums[]`、`studentsByClass{}`
3. 若 `classNums.length > 1` → 标记颜色设为灰色 `#9ca3af`，信息窗内按班级分节显示
4. 若仅一个班级 → 使用该班专属颜色

**关键函数**:
- [buildInfoWindowHTML()](file:///workspace/js/main.js#L695-L731) — 构建信息窗 HTML（含班级分组显示）

---

### 5.5 双模式搜索系统

```
用户输入搜索词
    │
    ▼
performSearch()
    │
    ├─ 第一步：findStudentMatchesByName(query)
    │   · 全年级数据中模糊匹配姓名（不区分大小写）
    │   · 命中 → focusOnStudentMatch()
    │
    ├─ 第二步：findUniversityMatches(query)
    │   · 按大学名称/城市名模糊匹配
    │   · 命中 → focusOnUniversityMatch()
    │
    └─ 都未匹配 → showToast('未找到...')
```

**搜索结果处理**:

| 场景 | 处理方式 |
|------|----------|
| 单一结果 | 地图飞行至目标 + 打开信息窗 + Toast 提示 |
| 多个结果 | 显示搜索导航条（← / →），逐一浏览 |
| 精确匹配 | 优先定位精确匹配项（姓名全名 / 大学金名） |

**临时标记管理**:

| 场景 | 处理方式 |
|------|----------|
| 搜索结果班级全被勾选覆盖 | 直接打开现有标记信息窗 |
| 搜索结果包含未勾选班级 | 临时替换信息窗为"合并全量数据版"，图标变灰 |
| 搜索结果大学无现有标记 | 创建临时标记（searchPinnedMarker） |
| 关闭搜索 | 移除临时标记，恢复被替换的信息窗和图标 |

**关键函数**:
- [performSearch()](file:///workspace/js/main.js#L734-L757) — 搜索入口
- [findStudentMatchesByName()](file:///workspace/js/main.js#L759-L776) — 姓名搜索
- [findUniversityMatches()](file:///workspace/js/main.js#L778-L819) — 大学搜索
- [goToSearchResult()](file:///workspace/js/main.js#L864-L888) — 定位到搜索结果
- [openOrPinSearchResult()](file:///workspace/js/main.js#L895-L1030) — 打开/创建搜索结果标记
- [clearSearchPinnedMarker()](file:///workspace/js/main.js#L1059-L1079) — 清理临时标记

---

### 5.6 版本号隔离机制

项目使用三个版本号变量来防止异步回调污染当前视图：

| 版本号变量 | 递增时机 | 作用 |
|-----------|----------|------|
| `markerRenderVersion` | 切换班级筛选时 | 使飞行中的地理编码回调判定自身过期并丢弃 |
| `searchNavRequestId` | 每次调用 goToSearchResult 时 | 使导航切换时滞留的地理编码回调失效 |
| `geocodeCancelledVersion` | 强制清空地理编码队列时 | 使所有在途请求的回调失效 |

**工作原理**: 每次发起异步操作前捕获当前版本号，回调执行时比对版本号是否一致，不一致则丢弃。

```javascript
// 示例：markerRenderVersion 的使用
const renderVersion = ++markerRenderVersion;
enqueueGeocode(university, city, function (point) {
  if (renderVersion !== markerRenderVersion) return; // 版本号不匹配，丢弃
  addMarkerToMap(point, group);
});
```

---

### 5.7 容错与 API 额度检测

**双通道检测机制**:

| 通道 | 检测方式 | 阈值 | 处理 |
|------|----------|------|------|
| 瓦片加载失败 | 监听天地图瓦片 `<img>` error 事件 | 10 秒窗口内 ≥8 次 | handleQuotaExceeded() |
| 地理编码连续失败 | 监听 geocoder 返回非成功状态 | ≥5 次连续失败 | handleSevereFailure() |

**两个处理函数的共同行为**:
1. 递增 `geocodeCancelledVersion`（使在途回调失效）
2. 清空 `geocodeQueue` 队列
3. 重置 `geocodingActive` 和 `pendingGeocodesCount`
4. 隐藏加载遮罩
5. 弹出 Toast 提示（仅一次，通过标志位防止重复）

**超时保护**: 每条地理编码请求设置 8 秒超时（`CONFIG.geocodeTimeout`），超时后不缓存结果，允许后续重试。

**关键函数**:
- [handleQuotaExceeded()](file:///workspace/js/main.js#L1352-L1363) — API 额度已满处理
- [handleSevereFailure()](file:///workspace/js/main.js#L1366-L1379) — 连续严重失败处理

---

## 6. 关键函数索引

### 口令验证

| 函数 | 位置 | 说明 |
|------|------|------|
| `sha256Hex(input)` | [L266](file:///workspace/js/main.js#L266) | 计算字符串的 SHA-256 哈希（hex 格式） |
| `initPasswordGate()` | [L288](file:///workspace/js/main.js#L288) | 初始化口令验证遮罩及事件绑定 |
| `setupBrandTitle()` | [L343](file:///workspace/js/main.js#L343) | 设置导航栏品牌标题 |

### 地图初始化

| 函数 | 位置 | 说明 |
|------|------|------|
| `onTMapCallback()` | [L441](file:///workspace/js/main.js#L441) | 天地图 API 加载完成回调，初始化地图 |
| `setupMapEventListeners()` | [L498](file:///workspace/js/main.js#L498) | 绑定班级复选框、搜索、导航等地图相关事件 |

### 地理编码

| 函数 | 位置 | 说明 |
|------|------|------|
| `enqueueGeocode(university, city, callback)` | [L94](file:///workspace/js/main.js#L94) | 将大学加入编码队列 |
| `processNextGeocode()` | [L139](file:///workspace/js/main.js#L139) | 逐条执行队列中的地理编码请求 |
| `findOldestCacheKey(cache)` | [L121](file:///workspace/js/main.js#L121) | 查找缓存中最旧的键（LRU 淘汰） |
| `parseGeocodeResult(result)` | [L1320](file:///workspace/js/main.js#L1320) | 解析天地图地理编码返回结果 |
| `handleQuotaExceeded()` | [L1352](file:///workspace/js/main.js#L1352) | API 额度已满处理 |
| `handleSevereFailure()` | [L1366](file:///workspace/js/main.js#L1366) | 连续严重失败处理 |

### 标记渲染

| 函数 | 位置 | 说明 |
|------|------|------|
| `getSelectedClasses()` | [L545](file:///workspace/js/main.js#L545) | 获取当前勾选的班级列表 |
| `groupSelectedByUniversity(selectedClasses)` | [L558](file:///workspace/js/main.js#L558) | 合并已勾选班级的大学去向 |
| `clearActiveMarkers()` | [L600](file:///workspace/js/main.js#L600) | 清空当前地图标记 |
| `renderSelectedMarkers()` | [L609](file:///workspace/js/main.js#L609) | 按当前复选框状态重算并渲染标记 |
| `createPinIcon(color)` | [L636](file:///workspace/js/main.js#L636) | 生成彩色 SVG Pin 图标 data URL |
| `addMarkerToMap(point, group)` | [L649](file:///workspace/js/main.js#L649) | 在地图上添加标记 |
| `createMarker(point, group, color, merged)` | [L663](file:///workspace/js/main.js#L663) | 创建单个标记（含 hover/click 事件） |
| `buildInfoWindowHTML(group, color, merged)` | [L695](file:///workspace/js/main.js#L695) | 构建信息窗口 HTML |

### 搜索系统

| 函数 | 位置 | 说明 |
|------|------|------|
| `performSearch()` | [L734](file:///workspace/js/main.js#L734) | 搜索入口（姓名优先 → 大学） |
| `findStudentMatchesByName(query)` | [L759](file:///workspace/js/main.js#L759) | 在全年级数据中模糊匹配姓名 |
| `findUniversityMatches(query)` | [L778](file:///workspace/js/main.js#L778) | 按大学名称/城市名模糊匹配 |
| `focusOnStudentMatch(query, matches)` | [L821](file:///workspace/js/main.js#L821) | 定位到姓名搜索结果 |
| `focusOnUniversityMatch(query, matches)` | [L840](file:///workspace/js/main.js#L840) | 定位到大学搜索结果 |
| `goToSearchResult()` | [L864](file:///workspace/js/main.js#L864) | 定位到当前搜索结果并更新导航条 |
| `openOrPinSearchResult(target, point)` | [L895](file:///workspace/js/main.js#L895) | 打开/创建搜索结果标记 |
| `clearSearchPinnedMarker()` | [L1059](file:///workspace/js/main.js#L1059) | 清理搜索临时标记 |
| `showSearchNav()` | [L1039](file:///workspace/js/main.js#L1039) | 显示搜索结果导航条 |
| `hideSearchNav(clearPinnedMarker)` | [L1048](file:///workspace/js/main.js#L1048) | 隐藏并清空搜索导航条 |
| `updateSearchNavInfo()` | [L1082](file:///workspace/js/main.js#L1082) | 刷新导航条文字与按钮状态 |

### 班级面板

| 函数 | 位置 | 说明 |
|------|------|------|
| `buildClassPanel()` | [L203](file:///workspace/js/main.js#L203) | 动态生成班级复选框面板 |
| `injectClassColorCSS()` | [L222](file:///workspace/js/main.js#L222) | 动态注入班级颜色 CSS 变量 |

### 数据统计与缺失

| 函数 | 位置 | 说明 |
|------|------|------|
| `buildStatsContent()` | [L1207](file:///workspace/js/main.js#L1207) | 构建数据统计面板内容 |
| `updateMissingDataToggle()` | [L1126](file:///workspace/js/main.js#L1126) | 更新数据缺失按钮显示/隐藏 |
| `buildMissingDataContent(selectedClasses)` | [L1156](file:///workspace/js/main.js#L1156) | 构建数据缺失模态框内容 |

### UI 辅助

| 函数 | 位置 | 说明 |
|------|------|------|
| `openModal(id)` | [L1112](file:///workspace/js/main.js#L1112) | 打开模态框 |
| `closeModal(id)` | [L1116](file:///workspace/js/main.js#L1116) | 关闭模态框 |
| `showToast(msg)` | [L1416](file:///workspace/js/main.js#L1416) | 显示底部 Toast 提示（3 秒后消失） |
| `updateLoadingOverlay()` | [L1315](file:///workspace/js/main.js#L1315) | 更新加载遮罩显示状态 |

### 数据处理辅助

| 函数 | 位置 | 说明 |
|------|------|------|
| `getClassName(classNum)` | [L1285](file:///workspace/js/main.js#L1285) | 获取班级显示名称 |
| `getUniversityBaseName(university)` | [L1433](file:///workspace/js/main.js#L1433) | 提取大学本名（去除括号后缀，保留独立办学实体） |
| `escapeHTML(text)` | [L1451](file:///workspace/js/main.js#L1451) | HTML 转义（防 XSS） |
| `parseStudentCoordinate(student)` | [L1381](file:///workspace/js/main.js#L1381) | 解析同学数据中的经纬度坐标 |
| `toLngLat(coordinate)` | [L1390](file:///workspace/js/main.js#L1390) | 将坐标对象转为 T.LngLat |
| `isSameCoordinate(a, b)` | [L1394](file:///workspace/js/main.js#L1394) | 比较两个坐标是否相同 |
| `addMapOverlay(overlay)` | [L1399](file:///workspace/js/main.js#L1399) | 兼容天地图 addOverLay/addOverlay |
| `removeMapOverlay(overlay)` | [L1407](file:///workspace/js/main.js#L1407) | 兼容天地图 removeOverLay/removeOverlay |
| `updateCountBadges()` | [L1291](file:///workspace/js/main.js#L1291) | 更新各班人数徽标 |
| `updateTotalCount()` | [L1302](file:///workspace/js/main.js#L1302) | 更新已勾选班级总人数 |
| `positionMissingPanel()` | [L1465](file:///workspace/js/main.js#L1465) | 定位数据缺失面板（班级筛选面板上方） |
| `buildAboutContent()` | [L352](file:///workspace/js/main.js#L352) | 填充关于弹窗中的动态内容 |

---

## 7. 全局状态变量清单

| 变量 | 类型 | 用途 |
|------|------|------|
| `map` | `T.Map` | 天地图 Map 实例 |
| `geocoder` | `T.Geocoder` | 天地图 Geocoder 实例 |
| `geoCache` | `Object` | 地理编码结果缓存 `{ "大学名": { point, timestamp } }` |
| `activeMarkers` | `Array<T.Marker>` | 当前地图上的所有标记 |
| `activeMarkerByUniversity` | `Object` | 大学名 → Marker 索引 |
| `searchPinnedMarker` | `T.Marker \| null` | 搜索临时标记 |
| `patchedExistingMarker` | `Object \| null` | 被临时替换信息窗的已有标记（含原始信息窗和图标） |
| `searchOpenedExistingMarker` | `T.Marker \| null` | 搜索时直接打开信息窗的已有标记 |
| `markerRenderVersion` | `number` | 标记渲染版本号（防过期回调） |
| `searchResults` | `Array` | 搜索结果列表 |
| `searchResultIndex` | `number` | 当前搜索结果索引 |
| `searchNavRequestId` | `number` | 搜索导航请求版本号 |
| `geocodeQueue` | `Array` | 地理编码请求队列 |
| `geocodingActive` | `boolean` | 地理编码是否正在执行 |
| `pendingGeocodesCount` | `number` | 待处理地理编码请求数 |
| `geocodeCancelledVersion` | `number` | 地理编码队列取消版本号 |
| `consecutiveGeocodeFailures` | `number` | 连续地理编码失败计数 |
| `tileErrorCount` | `number` | 瓦片加载失败计数 |
| `tileErrorWindowTimer` | `number \| null` | 瓦片错误窗口重置计时器 |
| `quotaExceededNotified` | `boolean` | 是否已报 API 额度已满 |
| `severeFailureNotified` | `boolean` | 是否已报连续严重失败 |
| `isTouchDevice` | `boolean` | 触摸设备检测结果 |
| `activeInfoWindowMarker` | `T.Marker \| null` | 当前打开信息窗的标记引用 |
| `CLASS_COUNT` | `number` | 班级数量（来自 CONFIG） |
| `CLASS_COLORS` | `Object` | 各班颜色（来自 CONFIG） |
| `MERGED_MARKER_COLOR` | `string` | 合并标记颜色（来自 CONFIG） |
| `ALL_CLASS_DATA` | `Object` | 全班数据引用 `{ classNum: classNData }` |
| `ALL_MISSING_DATA` | `Object` | 各班缺失数据引用 `{ classNum: classNMissingData }` |
| `toastTimer` | `number \| null` | Toast 计时器 |
| `tmapInitialized` | `boolean` | 天地图是否已初始化 |

---

## 8. 依赖关系图

### 文件加载依赖

```
index.html
  │
  ├── js/config.js ─────────── 定义 CONFIG 全局对象
  │     │
  │     └── validateConfig() ── 加载时立即验证
  │
  ├── data/class1~4.js ─────── 定义 classNData / classNMissingData
  │     （由 index.html IIFE 根据 CONFIG.classCount 动态加载）
  │
  ├── js/main.js ───────────── 依赖 CONFIG / classNData / classNMissingData
  │     │
  │     ├── DOMContentLoaded ── 读取 CONFIG / window['classNData']
  │     ├── onTMapCallback ──── 依赖 T 全局对象（天地图 API）
  │     └── 业务函数 ────────── 依赖 CONFIG / T / DOM 元素
  │
  └── 天地图 API v4.0 ──────── 外部依赖，动态加载
        │
        └── bootstrapTMap() ─── 轮询等待 T 就绪后调用 onTMapCallback
```

### 运行时依赖

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│  CONFIG      │────►│  main.js    │────►│  天地图 API   │
│  (配置中心)  │     │  (业务逻辑)  │     │  (地图引擎)   │
└─────────────┘     └──────┬──────┘     └──────────────┘
                           │
                    ┌──────┼──────┐
                    │      │      │
                    ▼      ▼      ▼
              ┌──────┐ ┌──────┐ ┌──────────┐
              │ DOM  │ │ Data │ │ Web Crypto│
              │ 元素  │ │ 文件  │ │ API      │
              └──────┘ └──────┘ └──────────┘
```

### 函数调用链（核心流程）

```
DOMContentLoaded
  ├── buildClassPanel() ──────────── injectClassColorCSS()
  ├── setupBrandTitle()
  ├── buildAboutContent()
  ├── initPasswordGate() ─────────── sha256Hex()
  ├── 数据加载 (ALL_CLASS_DATA / ALL_MISSING_DATA)
  ├── updateCountBadges()
  ├── positionMissingPanel()
  └── 事件绑定 (按钮/模态框/ESC/resize)

onTMapCallback (天地图 API 就绪后)
  ├── 创建卫星瓦片图层
  ├── 初始化地图 (T.Map)
  ├── 添加比例尺控件
  ├── 初始化地理编码器 (T.Geocoder)
  ├── setupMapEventListeners()
  │     ├── 班级复选框 change → renderSelectedMarkers()
  │     ├── 搜索按钮/回车 → performSearch()
  │     ├── 搜索导航 ← / → → goToSearchResult()
  │     └── 触摸设备地图点击 → 关闭信息窗
  └── 瓦片错误监听 → handleQuotaExceeded()

renderSelectedMarkers()
  ├── ++markerRenderVersion
  ├── clearActiveMarkers()
  ├── groupSelectedByUniversity()
  │     └── parseStudentCoordinate() / isSameCoordinate()
  └── 对每个分组:
        ├── 有坐标 → addMarkerToMap() → createMarker() → buildInfoWindowHTML()
        └── 无坐标 → enqueueGeocode() → processNextGeocode() → 回调 addMarkerToMap()

performSearch()
  ├── findStudentMatchesByName() → focusOnStudentMatch()
  ├── findUniversityMatches() → focusOnUniversityMatch()
  └── 无结果 → showToast()

goToSearchResult()
  ├── ++searchNavRequestId
  ├── clearSearchPinnedMarker()
  ├── 有坐标 → map.centerAndZoom() + openOrPinSearchResult()
  └── 无坐标 → enqueueGeocode() → 回调 openOrPinSearchResult()
```

---

## 9. 数据格式规范

### classNData — 已知去向的同学

```javascript
var class1Data = [
  {
    "name": "张三",              // 必填：姓名
    "university": "北京大学",    // 必填：大学全名
    "city": "北京",              // 必填：所在城市（不含"市"后缀，地理编码时自动补全）
    "latitude": 39.9063,         // 可选：纬度（十进制度，-90 ~ 90）
    "longitude": 116.3914        // 可选：经度（十进制度，-180 ~ 180）
  }
];
```

**坐标规则**:
- `latitude` / `longitude` 提供后优先使用，跳过地理编码请求
- 若同一大学多人提供了不同坐标，触发冲突回退机制（`coordinateConflict = true`）
- 坐标校验：`parseStudentCoordinate()` 检查数值有限性及范围

### classNMissingData — 缺失去向的同学

```javascript
var class1MissingData = [
  { "name": "李四" }
];
```

### 大学名称智能合并规则

`getUniversityBaseName()` 函数用于统计时合并同类大学：

| 输入 | 输出 | 说明 |
|------|------|------|
| 北京大学（医学部） | 北京大学 | 去除括号后缀 |
| 电子科技大学（清水河校区） | 电子科技大学 | 去除校区后缀 |
| 中国地质大学（武汉） | 中国地质大学（武汉） | 保留（独立办学实体白名单） |
| 中国石油大学（北京） | 中国石油大学（北京） | 保留（独立办学实体白名单） |
| 中国矿业大学（北京） | 中国矿业大学（北京） | 保留（独立办学实体白名单） |

---

## 10. 项目运行与部署

### 本地运行

```bash
cd /workspace
python -m http.server 8080
# 浏览器打开 http://localhost:8080
```

### 部署

整个项目为纯静态资源，可直接部署到任意静态文件服务器（GitHub Pages、Nginx、Apache、Vercel、Netlify 等）。无需构建步骤，无需后端服务。

### 配置步骤

1. **获取天地图密钥**: 访问 [天地图开放平台](https://lbs.tianditu.gov.cn/) 注册并申请 JavaScript API 密钥
2. **替换密钥**: 修改 `js/config.js` 中的 `tiandituTK` 字段
3. **管理口令**: 使用浏览器控制台计算 SHA-256 哈希值，添加到 `CONFIG.validPasswordHashes` 数组
4. **更新数据**: 编辑 `data/class1.js` ~ `data/class4.js`

### 浏览器兼容性

| 依赖项 | 最低要求 |
|--------|----------|
| Web Crypto API | Chrome 37+、Firefox 34+、Safari 11+、Edge 79+ |
| CSS backdrop-filter | Chrome 76+、Firefox 103+、Safari 9+、Edge 79+ |
| CSS 变量 | Chrome 49+、Firefox 31+、Safari 9.1+、Edge 15+ |
| 天地图 JS API v4.0 | 支持主流现代浏览器 |

**推荐**: Chrome 90+ / Firefox 90+ / Safari 15+ / Edge 90+

---

## 11. 扩展指南

### 添加新班级（第 5 班）

1. 创建 `data/class5.js`，导出 `class5Data` 和 `class5MissingData`
2. 修改 `js/config.js`:
   - `classCount: 5`
   - `classColors` 中添加 `{ 5: '#颜色值' }`
   - `classNames` 中添加 `{ 5: '5 班' }`
3. 在 `css/style.css` 的 `:root` 中添加 `--c5` 颜色变量（可选，`injectClassColorCSS()` 会自动注入）
4. 无需修改 `index.html`（班级数据文件由 IIFE 根据 `CONFIG.classCount` 动态加载）
5. 无需修改 `js/main.js`（所有循环均基于 `CLASS_COUNT` 变量）

### 修改口令

```javascript
// 浏览器控制台生成新哈希
crypto.subtle.digest('SHA-256', new TextEncoder().encode('你的口令'))
  .then(buf => Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, '0')).join(''))
  .then(console.log);
```

将得到的 64 位十六进制哈希值添加到 `CONFIG.validPasswordHashes` 数组即可。

### 修改地图中心/缩放

在 `js/main.js` 的 `onTMapCallback()` 函数中修改：
```javascript
map.centerAndZoom(new T.LngLat(105.4, 37.9), 5);  // 经度, 纬度, 缩放级别
```
