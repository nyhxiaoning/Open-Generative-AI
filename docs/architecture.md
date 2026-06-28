# Open Generative AI — 项目架构文档

> 本文档描述项目的整体架构、目录结构、模块职责及数据流。

---

## 目录

1. [项目定位](#1-项目定位)
2. [架构概览](#2-架构概览)
3. [目录结构](#3-目录结构)
4. [渲染方式：两套 UI 系统](#4-渲染方式两套-ui-系统)
5. [核心模块详解](#5-核心模块详解)
6. [数据流与 API 通信](#6-数据流与-api-通信)
7. [构建与部署](#7-构建与部署)
8. [新增功能指南](#8-新增功能指南)

---

## 1. 项目定位

Open Generative AI 是一个**开源 AI 媒体生成平台**，提供 200+ 个 SOTA 模型的统一访问入口，覆盖图像、视频、音频、电影级别制作、口播同步、AI 剪辑、Marketing 素材生成等多个领域。项目定位为自托管的 AI 媒体工作室，通过 [MuAPI](https://muapi.ai) 作为后端推理网关，无需本地 GPU 即可调用最前沿的模型。

核心特征：

- **无内容审查**：完全开放的生成模型调用
- **多 Studio**：按媒体类型分 Studio，每个 Studio 独立管理
- **双前端**：同时支持 Next.js（浏览器 SPA）和 Vite+Electron（桌面应用）
- **可扩展**：通过 npm workspaces + git submodule 集成第三方包

---

## 2. 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                       用户界面层 (UI)                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Next.js 15 (App Router)                               │   │
│  │  ┌─────────┐ ┌──────────┐ ┌───────────┐ ┌──────────┐  │   │
│  │  │layout.js│ │page.js   │ │studio/    │ │agents/   │  │   │
│  │  │(根布局)  │ │(/ → /studio)│ │[[...slug]]│ │(agent CRUD)│  │   │
│  │  └─────────┘ └──────────┘ └───────────┘ └──────────┘  │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │  StandaloneShell (components/StandaloneShell.js)  │   │   │
│  │  │  路由分发 → packages/studio/src/components/*.jsx │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Vite 5 + Electron (桌面端)                              │   │
│  │  index.html → src/main.js (手写路由)                      │   │
│  │  electron/main.js → 创建 BrowserWindow                    │   │
│  │  electron/preload.js → 暴露 IPC 桥接 (localAI)            │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      核心逻辑层 (Lib)                            │
│  ┌──────────┐ ┌──────────────┐ ┌───────────┐ ┌───────────────┐ │
│  │ muapi.js │ │ models.js    │ │ promptUtils│ │pendingJobs.js │ │
│  │ (API客户端)│ │ (模型数据)    │ │ (提示词工具) │ │ (作业持久化)  │ │
│  └──────────┘ └──────────────┘ └───────────┘ └───────────────┘ │
│  ┌──────────┐ ┌──────────────┐ ┌──────────────────────────────┐ │
│  │ i18n.js  │ │ uploadHistory│ │ localInferenceClient.js      │ │
│  │ (国际化)  │ │ (上传记录)    │ │ (Electron 本地推理桥接)      │ │
│  └──────────┘ └──────────────┘ └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API 代理层 (Middleware)                      │
│  middleware.js                                                  │
│  /api/v1/* → rewrite → https://api.muapi.ai/api/v1/*            │
│  /api/workflow/* → rewrite → https://api.muapi.ai/api/workflow/*│
│  /api/app/* → rewrite → https://api.muapi.ai/api/app/*          │
│  例外: /api/v1/creative-agent, /api/v1/get_upload_url,          │
│         /api/v1/upload-binary 由本地 Route Handler 处理          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MuAPI 后端 (外部)                             │
│  https://api.muapi.ai                                           │
│  200+ 模型推理端点 (Flux, Midjourney, Kling, Veo 2 等)           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 目录结构

```
Open-Generative-AI/
├── app/                          # Next.js App Router 页面
│   ├── layout.js                 #   根布局 (全局 CSS、字体)
│   ├── page.js                   #   首页 → redirect(/studio)
│   ├── globals.css               #   全局样式 (Tailwind)
│   ├── studio/[[...slug]]/page.js #   主 Studio SPA 页面
│   ├── agents/                   #   AI Agent CRUD 页面
│   │   ├── layout.js
│   │   ├── [agent_id]/           #     单个 Agent 详情/对话
│   │   ├── create/               #     创建 Agent
│   │   └── edit/[id]/            #     编辑 Agent
│   ├── assistant/page.js         #   AI 助手页面
│   ├── workflow/[id]/            #   工作流运行页面
│   └── api/                      #   API Route Handlers (Next.js)
│       ├── agents/               #     Agent API 代理
│       ├── api/v1/               #     通用 API 代理
│       ├── app/                  #     App API 代理
│       ├── upload-binary/        #     二进制上传处理
│       ├── v1/                   #     本地处理的 MuAPI 端点
│       │   ├── creative-agent/   #       Creative Agent 自处理
│       │   ├── get_upload_url/   #       获取上传 URL
│       │   └── upload-binary/    #       二进制上传
│       └── workflow/             #     工作流 API 代理
│
├── components/                   # Next.js 客户端组件 (根级别)
│   ├── StandaloneShell.js        #   主 Shell：路由、Tab 导航、API Key 管理
│   └── ApiKeyModal.js            #   API Key 输入弹窗
│
├── src/                          # Vite（Electron/独立）前端
│   ├── main.js                   #   入口 (手写路由、挂载 DOM)
│   ├── style.css                 #   全局样式
│   ├── styles/                   #   样式模块
│   │   ├── global.css
│   │   ├── studio.css
│   │   └── variables.css
│   ├── components/               #   Vite 版本 Studio 组件 (自包含)
│   │   ├── Header.js             #     导航头
│   │   ├── Sidebar.js            #     侧边栏
│   │   ├── ImageStudio.js        #     图像生成 Studio
│   │   ├── VideoStudio.js        #     视频生成 Studio
│   │   ├── CinemaStudio.js       #     影院级生成 Studio
│   │   ├── LipSyncStudio.js      #     口播同步 Studio
│   │   ├── AgentStudio.js        #     AI Agent Studio
│   │   ├── WorkflowStudio.js     #     工作流 Studio
│   │   ├── McpCliStudio.js       #     MCP CLI 工具
│   │   ├── UploadPicker.js       #     文件上传组件
│   │   ├── CameraControls.js     #     相机控制
│   │   ├── AuthModal.js          #     认证弹窗
│   │   ├── SettingsModal.js      #     设置弹窗
│   │   ├── LocalModelManager.js  #     本地模型管理
│   │   └── ...                   #     其他
│   └── lib/                      #   核心库
│       ├── muapi.js              #     MuAPI 客户端类 (统一封装)
│       ├── models.js             #     模型定义 (重导出自 packages/studio)
│       ├── promptUtils.js        #     提示词工具函数
│       ├── pendingJobs.js        #     等待作业持久化 (localStorage)
│       ├── localModels.js        #     本地模型数据
│       ├── localInferenceClient.js#     Electron 本地推理客户端
│       ├── uploadHistory.js      #     上传历史 (localStorage)
│       ├── uploadProxyTarget.js  #     上传代理目标配置
│       └── i18n.js               #     国际化
│
├── packages/                     # npm workspaces 子包
│   ├── studio/                   #   ★ 核心 Studio 组件库
│   │   ├── src/
│   │   │   ├── index.js          #     统一导出 (所有 Studio 组件)
│   │   │   ├── components/       #     Studio 组件的 React 实现
│   │   │   │   ├── ImageStudio.jsx
│   │   │   │   ├── VideoStudio.jsx
│   │   │   │   ├── CinemaStudio.jsx
│   │   │   │   ├── AudioStudio.jsx
│   │   │   │   ├── ClippingStudio.jsx
│   │   │   │   ├── LipSyncStudio.jsx
│   │   │   │   ├── VibeMotionStudio.jsx
│   │   │   │   ├── RecastStudio.jsx
│   │   │   │   ├── MarketingStudio.jsx
│   │   │   │   ├── WorkflowStudio.jsx
│   │   │   │   ├── AgentStudio.jsx
│   │   │   │   ├── DesignAgentStudio.jsx
│   │   │   │   ├── AppsStudio.jsx
│   │   │   │   └── McpCliStudio.jsx
│   │   │   ├── models.js         #     200+ 模型定义 (核心数据集)
│   │   │   ├── muapi.js          #     Studio 版 MuAPI 客户端
│   │   │   └── tailwind.css
│   │   └── package.json
│   ├── Vibe-Workflow/            #   git submodule: 可视化工作流编排
│   ├── Open-Poe-AI/              #   git submodule: AI Agent 框架
│   └── Open-AI-Design-Agent/     #   git submodule: AI 设计代理
│
├── electron/                     # Electron 桌面端
│   ├── main.js                   #   主进程入口
│   ├── preload.js                #   预加载脚本 (IPC 桥接)
│   └── lib/                      #   本地推理引擎
│       ├── localInference.js     #     sd.cpp 引擎集成
│       ├── localInferencePaths.js #     路径管理
│       ├── localInferenceAssets.js#     资源文件管理
│       ├── localInferenceRuntime.js#    运行时管理
│       ├── modelCatalog.js       #    本地模型目录
│       ├── wan2gpProvider.js     #     Wan2GP 远程 Gradio 集成
│       └── wan2gpModelAvailability.js
│
├── components/                   # 独立 React 组件 (Next.js 用)
├── tests/                        # 测试
├── scripts/                      # 工具脚本
├── build/                        # 构建资源
│   ├── installer.nsh             #   Windows 安装器脚本
│   ├── linux/apparmor.profile    #   Linux AppArmor 配置
│   └── local-ai/                 #   内置本地推理二进制
│
├── public/                       # 静态资源
├── docs/                         # 文档
│   └── assets/
│
├── middleware.js                  # Next.js Middleware (API 代理)
├── next.config.mjs               # Next.js 配置
├── vite.config.mjs               # Vite 配置 (Electron)
├── tailwind.config.js            # Tailwind 配置
├── postcss.config.js             # PostCSS 配置
├── index.html                    # Vite 入口 HTML
├── Dockerfile                    # Docker 构建 (多阶段)
└── docker-compose.yml            # Docker Compose
```

---

## 4. 渲染方式：两套 UI 系统

项目维护两套独立的 UI 渲染方案：

### 4.1 Next.js (Web SPA)

- **入口**：`app/layout.js` → `app/page.js` → `app/studio/[[...slug]]/page.js`
- **路由**：`components/StandaloneShell.js` 解析 URL slug，动态渲染对应 Studio
- **组件**：所有 Studio 组件来自 `packages/studio/src/components/*.jsx`（通过 npm workspace 导入）
- **API 通信**：通过 Next.js Middleware 自动代理到 `api.muapi.ai`

### 4.2 Vite + Electron (桌面应用)

- **入口**：`index.html` → `src/main.js`（手写的简单路由）
- **路由**：内存中的 `navigate()` 函数，动态 import 各 Studio 组件
- **组件**：直接使用 `src/components/*.js`（结构精简的独立实现）
- **Electron 主进程**：`electron/main.js` 负责创建窗口、注册本地推理 IPC

> **为什么有两套？**
> 历史原因：最初使用 Vite 构建纯前端 SPA + Electron 桌面端。后来引入 Next.js 以支持 SSR、API Routes 和更好的 Web 部署体验。`packages/studio` 是 Next.js 版本的组件库，`src/components/` 是 Vite 版本的独立实现。两者功能对等但代码隔离。

---

## 5. 核心模块详解

### 5.1 Studio 组件体系 (`packages/studio/src/components/`)

每个 Studio 对应一种媒体类型，共享相似接口模式：

| Studio | 文件 | 功能 |
|--------|------|------|
| ImageStudio | `ImageStudio.jsx` (~60KB) | 文生图 (T2I) + 图生图 (I2I) |
| VideoStudio | `VideoStudio.jsx` (~79KB) | 文生视频 (T2V) + 图生视频 (I2V) |
| CinemaStudio | `CinemaStudio.jsx` (~42KB) | 电影级视频生成 |
| AudioStudio | `AudioStudio.jsx` (~46KB) | 音频生成 |
| LipSyncStudio | `LipSyncStudio.jsx` (~42KB) | 口播同步 (图像+音频→视频) |
| ClippingStudio | `ClippingStudio.jsx` (~45KB) | AI 视频剪辑 |
| VibeMotionStudio | `VibeMotionStudio.jsx` (~35KB) | 动态运动效果 |
| RecastStudio | `RecastStudio.jsx` (~31KB) | AI 视频重制 |
| MarketingStudio | `MarketingStudio.jsx` (~29KB) | 营销素材生成 |
| WorkflowStudio | `WorkflowStudio.jsx` (~42KB) | 可视化工作流 |
| AgentStudio | `AgentStudio.jsx` (~13KB) | AI Agent 对话 |
| DesignAgentStudio | `DesignAgentStudio.jsx` (~1KB) | AI 设计代理 (懒加载) |
| AppsStudio | `AppsStudio.jsx` (~27KB) | 第三方应用探索 |
| McpCliStudio | `McpCliStudio.jsx` (~7KB) | MCP CLI 工具 |

**共享模式**：每个 Studio 都包含：
1. 模型选择器（ModelSelector）
2. 参数面板（提示词、宽高比、分辨率等）
3. 生成按钮 → 调用 `muapi.generateImage()` 或类似
4. 结果展示区
5. 作业轮询（`pollForResult`）+ 本地持久化（`pendingJobs.js`）

### 5.2 MuAPI Client (`src/lib/muapi.js`)

统一的 API 客户端，负责所有与 MuAPI 后端的通信：

- **身份验证**：从 `localStorage` 读取 API Key，通过 `x-api-key` 头传递
- **端点解析**：根据模型 ID 从 `models.js` 中查找 `endpoint`，构建请求 URL
- **请求模式**：
  - 异步提交 → 返回 `request_id`
  - 轮询 `GET /api/v1/predictions/{requestId}/result` 直到完成
- **支持的操作**：
  - `generateImage()` — 文生图 / 图生图
  - `generateVideo()` — 文生视频
  - `generateI2I()` — 图生图（含 `imageField` 映射）
  - `generateI2V()` — 图生视频
  - `processV2V()` — 视频生视频
  - `processLipSync()` — 口播同步
  - `uploadFile()` — 文件上传
- **重试**：服务器错误（5xx）自动重试，最多 60 次（图像）/ 900 次（视频）

### 5.3 模型数据 (`packages/studio/src/models.js`)

200+ 个模型的元数据定义文件（~11MB 的 `models_dump.json` 版本），包含：

- 模型 ID、名称、端点
- 参数规范（支持的宽高比、分辨率范围）
- 图像字段映射（`imageField`、`videoField`等）
- 是否支持 prompt、种子等

### 5.4 API 代理 (`middleware.js`)

Next.js Middleware 将大多数 `/api/*` 请求透明地 rewrite 到 MuAPI 后端：

```
客户端请求                       Middleware
/api/v1/generate-image  ───→  https://api.muapi.ai/api/v1/generate-image
/api/v1/predictions/xxx ───→  https://api.muapi.ai/api/v1/predictions/xxx
```

**例外路径**（由本地 Route Handler 处理）：
- `/api/v1/creative-agent/*` — 自定义 Creative Agent 逻辑
- `/api/v1/get_upload_url` — 获取上传预签名 URL
- `/api/v1/upload-binary` — 二进制文件上传

### 5.5 本地推理 (Electron Only)

桌面端附加功能，无需联网即可使用模型：

- **sd.cpp 引擎**：内置二进制推理引擎（`build/local-ai/`），下载模型权重到本地
- **Wan2GP 引擎**：远程 Gradio 服务器集成
- **IPC 通信**：`electron/preload.js` 通过 `contextBridge` 暴露 `window.localAI` API
- **前端客户端**：`src/lib/localInferenceClient.js` 封装 Electron IPC 调用

### 5.6 Agent CRUD (`app/agents/`)

完整的 AI Agent 管理功能，类似 ChatGPT 的对话界面：

- `[agent_id]/` — 单个 Agent 的对话界面
- `create/` — 创建新 Agent
- `edit/[id]/` — 编辑 Agent 配置

Agent 数据通过 `app/api/agents/` 路由代理到 MuAPI 后端。

### 5.7 作业持久化 (`src/lib/pendingJobs.js`)

在生成结果返回前，将作业信息保存到 `localStorage`，实现：

- 页面刷新后恢复等待中的作业
- 按 Studio 类型筛选
- 作业完成时自动移除

---

## 6. 数据流与 API 通信

### 6.1 典型生成流程 (以图像生成为例)

```
用户操作
    │
    ▼
Studio 组件 (ImageStudio.jsx)
    ├─ 选择模型、填写提示词、配置参数
    │
    ▼
MuapiClient.generateImage({
    model:     "flux-pro",
    prompt:    "...",
    aspect_ratio: "16:9",
    onRequestId: (id) => savePendingJob({ requestId, studioType: 'image' })
})
    │
    ├── POST {baseUrl}/api/v1/{endpoint}
    │    headers: { x-api-key, Content-Type: application/json }
    │    body: { prompt, aspect_ratio, ... }
    │
    ▼
Next.js Middleware (middleware.js)
    │  rewrite: /api/v1/{endpoint} → https://api.muapi.ai/api/v1/{endpoint}
    ▼
MuAPI 后端
    │ 返回: { request_id: "abc-123" }
    ▼
MuapiClient.pollForResult(requestId, key)
    │
    ├── 循环 60 次，间隔 2 秒
    │   GET {baseUrl}/api/v1/predictions/{requestId}/result
    │   headers: { x-api-key }
    │
    ├── 返回: { status: "completed", outputs: ["https://..."], ... }
    │
    ▼
Studio 组件
    ├─ 移除 pendingJob
    └─ 显示生成结果 (图片)
```

### 6.2 文件上传流程

```
Studio 组件
    │
    ▼
UploadPicker.js
    │
    ├── 方式一: 直接上传到 MuAPI
    │   muapi.uploadFile(file)
    │   POST {baseUrl}/api/v1/upload_file → 获取托管 URL
    │
    ├── 方式二: 通过本地 API 代理 (大文件)
    │   POST /api/v1/upload-binary → 本地上传到服务器
    │
    └── 返回 URL → 作为 image_url / video_url 传入生成请求
```

---

## 7. 构建与部署

### 7.1 构建管道

```
                         ┌──────────────────────┐
                         │    git clone + setup   │
                         │   git submodule init   │
                         │   npm install          │
                         │   npm run build:packages│
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        ┌────────────────────┐          ┌──────────────────────┐
        │  Next.js 构建       │          │  Electron/Vite 构建   │
        │  npm run build      │          │  npm run vite:build   │
        │  → .next/           │          │  → dist/              │
        └─────────┬──────────┘          └──────────┬───────────┘
                  │                                │
                  ▼                                ▼
        ┌────────────────────┐          ┌──────────────────────────┐
        │  Docker / npm start │          │  electron-builder 打包   │
        │  → 端口 3000        │          │  → release/*.dmg/.exe   │
        └────────────────────┘          └──────────────────────────┘
```

### 7.2 部署方式

| 方式 | 命令 | 产物 |
|------|------|------|
| Docker | `docker compose up -d` | 容器，端口 3001→3000 |
| 裸机 (Next.js) | `npm run build && npm start` | 监听 3000 端口 |
| 桌面 (macOS) | `npm run electron:build` | `release/*.dmg` |
| 桌面 (Windows) | `npm run electron:build:win` | `release/*.exe` |
| 桌面 (Linux x64) | `npm run electron:build:linux` | `release/*.AppImage` / `.deb` |

---

## 8. 新增功能指南

### 8.1 新增一个 Studio (常见需求)

**步骤 1 — 创建组件**：

在 `packages/studio/src/components/` 下创建 `YourStudio.jsx`：

```jsx
'use client';
import { useState } from 'react';
import { muapi } from '../muapi';

export default function YourStudio() {
  const [prompt, setPrompt] = useState('');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);

  const handleGenerate = async () => {
    setLoading(true);
    try {
      const res = await muapi.generateImage({ model: 'your-model', prompt });
      setResult(res.url);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      {/* UI */}
    </div>
  );
}
```

**步骤 2 — 注册导出**：

在 `packages/studio/src/index.js` 中添加：

```js
export { default as YourStudio } from './components/YourStudio';
```

**步骤 3 — 添加到路由**：

在 `components/StandaloneShell.js` 中：
- 在 `TABS` 数组中添加 tab 配置
- 从 `studio` 包导入 `YourStudio`

**步骤 4 — 添加 API 端点 (如需)**：

如果新功能需要自定义后端逻辑（不走 MuAPI 代理），在 `app/api/` 下创建 Route Handler。

**步骤 5 — 文档**：

在 `docs/` 下新建 `docs/your-feature.md`，记录设计目的、API 参数和使用方法。

### 8.2 新增一个 API 端点

```bash
# 创建路由文件
mkdir -p app/api/your-endpoint
touch app/api/your-endpoint/route.js
```

如果端点需要走 MuAPI 代理，确保其在 `middleware.js` 的 `matcher` 范围内。如果需要绕过代理（本地处理），添加路径到 `isHandledByRoute` 排除列表。

### 8.3 新增一个工具库函数

在 `src/lib/` 下创建文件，遵循已有模块的导出模式。注意 Vite 版本 (`src/lib/`) 和 Next.js 版本 (`packages/studio/src/`) 的 lib 层是独立的，需要各改各的。
