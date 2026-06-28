# 本地模型接入实现

> 本文档详细描述本地模型推理的实现机制，包括 sd.cpp 引擎的二进制管理与模型下载、Wan2GP 远程 Gradio 集成，以及前端如何调用这些功能。

---

## 目录

1. [架构概览](#1-架构概览)
2. [两种推理引擎](#2-两种推理引擎)
3. [引擎工作流程](#3-引擎工作流程)
4. [sd.cpp 引擎详解](#4-sdcpp-引擎详解)
5. [Wan2GP 远程引擎详解](#5-wan2gp-远程引擎详解)
6. [前端调用链路](#6-前端调用链路)
7. [新增本地模型](#7-新增本地模型)

---

## 1. 架构概览

本地模型功能**仅限 Electron 桌面端**。核心设计是两层分离：

```
UI 层 (渲染进程)                            Node 层 (主进程)
┌─────────────────────────┐     IPC       ┌─────────────────────────┐
│  src/components/         │   channel     │  electron/lib/           │
│  LocalModelManager.js    │  ────────→    │  localInference.js       │
│  (UI：下载管理、状态展示)   │              │  (二进制管理、模型下载、   │
│                          │              │   推理进程管控)           │
│  src/lib/                │              │                          │
│  localInferenceClient.js │  ←────────   │  localInferencePaths.js  │
│  (前端 IPC 封装)          │              │  (路径管理)               │
│                          │              │                          │
│  src/lib/localModels.js  │              │  modelCatalog.js         │
│  (前端模型数据 + Provider)│              │  (后端模型数据)            │
└─────────────────────────┘              └─────────────────────────┘
                                                    │
         Wan2GP 模式:                                ▼
┌─────────────────────────┐              ┌─────────────────────────┐
│  用户运行的              │   HTTP       │  electron/lib/          │
│  Wan2GP Gradio Server   │  ←───────→   │  wan2gpProvider.js      │
│  (独立 Python 进程)      │              │  (HTTP 客户端 + Gradio   │
│                         │              │   协议适配)              │
└─────────────────────────┘              └─────────────────────────┘
```

**IPC 桥接**：`electron/preload.js` 使用 `contextBridge` 暴露 `window.localAI` 对象，渲染进程通过它发起所有本地模型操作。关键的 IPC 通道：

| IPC Channel | 方向 | 用途 |
|-------------|------|------|
| `local-ai:binary-status` | 渲染→主 | 检查 sd.cpp 二进制是否存在 |
| `local-ai:download-binary` | 渲染→主 | 下载并解压 sd.cpp 引擎 |
| `local-ai:list-models` | 渲染→主 | 返回所有本地模型及其下载状态 |
| `local-ai:download-model` | 渲染→主 | 从 HuggingFace 下载模型权重 |
| `local-ai:download-auxiliary` | 渲染→主 | 下载辅助文件（Z-Image 专用） |
| `local-ai:generate` | 渲染→主 | 调用本地引擎生成图像 |
| `local-ai:cancel-generation` | 渲染→主 | 取消正在生成的进程 |
| `local-ai:progress` | 主→渲染 | 生成进度推送 (SSE-like) |
| `local-ai:download-progress` | 主→渲染 | 下载进度推送 |
| `wan2gp:*` | 双向 | Wan2GP 相关操作（见 §5） |

---

## 2. 两种推理引擎

| 特性 | sd.cpp 引擎 | Wan2GP 引擎 |
|------|-------------|-------------|
| **本质** | 本地编译的 C++ 二进制 | 远程 Gradio 服务器（Python） |
| **模型位置** | 磁盘上的 `.safetensors` / `.gguf` 文件 | 远程服务器上的 GPU 显存 |
| **网络要求** | 下载时需要，推理时离线 | 始终需要 |
| **支持的模型** | 图像：Z-Image (GGUF)、SD 1.5 (safetensors)、SDXL (safetensors) | 图像 + 视频：Flux、Qwen、Wan 2.2、Hunyuan、LTX |
| **通信协议** | 子进程 (spawn) + stdout/stderr 解析 | HTTP + Gradio SSE 流 (`/gradio_api/call/<fn>`) |
| **适用场景** | macOS 本地推理（小模型） | 有独立 GPU 服务器时使用 |
| **前端判定** | `provider: 'sdcpp'` | `provider: 'wan2gp'` |

---

## 3. 引擎工作流程

### 3.1 sd.cpp 完整流程

```
用户操作                        Electron 主进程
─────────────────              ─────────────────

1. 打开 Local Models 设置
   ├─ LocalModelManager.render()
   │   ├─ getBinaryStatus()          → electron/lib/localInference.js
   │   │   └─ ensureBundledBinaryInstalled()
   │   │       ├─ 检查 BINARY_PATH (BIN_DIR/sd-cli)
   │   │       ├─ 检查 app.isPackaged 下的预打包二进制
   │   │       └─ 失败 → 显示"未安装"提示
   │   │
   │   └─ listModels()              → localInference.js
   │       ├─ 读取 MODEL_CATALOG (modelCatalog.js)
   │       ├─ 逐个检查 getModelState()
   │       │   └─ 文件存在 → "downloaded"
   │       │      文件.part存在 → "partial"
   │       │      都不存在 → "not-downloaded"
   │       └─ 返回带状态的模型列表
   │
2. 用户点击"安装引擎"按钮
   ├─ downloadBinary()              → localInference.js
   │   ├─ 检查预打包二进制
   │   ├─ macOS arm64 → 从 GitHub Release 下载自定义 Metal 版本
   │   ├─ 其他平台 → 搜索 leejet/stable-diffusion.cpp 最新的 15 个 release
   │   │   └─ pickBinaryAssetForPlatform() 按优先级匹配 zip 名
   │   ├─ downloadFile() 支持:
   │   │   ├─ Range header (断点续传)
   │   │   ├─ 重定向跟随 (最多 10 跳)
   │   │   └─ 失败重试 (最多 5 次)
   │   ├─ extractZip() 解压
   │   ├─ macOS: xattr -cr (移除 Gatekeeper 检疫标记)
   │   └─ 移动二进制到 BINARY_PATH
   │
3. 用户点击模型的"下载"按钮
   ├─ downloadModel(modelId)        → localInference.js
   │   ├─ 从 catalog 获取 downloadUrl
   │   ├─ downloadFile(url, MODELS_DIR/filename)
   │   └─ 发送下载进度到渲染进程
   │
   ├─ Z-Image 需要额外两步:
   │   downloadAuxiliary('llm')     → 下载 Qwen3-4B GGUF (2.4 GB)
   │   downloadAuxiliary('vae')     → 下载 FLUX VAE (335 MB)
   │
4. 用户在 Studio 中选择使用本地模型 + 点击生成
   ├─ localAI.generate({            → localInference.js
   │     model: 'z-image-turbo',
   │     prompt: '...',
   │     aspect_ratio: '1:1',
   │     steps: 8,
   │     guidance_scale: 1.0,
   │     seed: -1
   │   })
   │
   └─ generate(params)              → localInference.js
       ├─ 验证二进制存在
       ├─ 验证模型文件存在
       ├─ Z-Image: 验证辅助文件存在
       ├─ 构建 sd-cli 命令行参数
       │   ├─ z-image/flux: --diffusion-model <path>
       │   ├─ sd1: -m <path>
       │   ├─ sdxl: -m <path> --sd-version sdxl
       │   └─ flux: --diffusion-model <path> --flux
       │
       ├─ spawn(BINARY_PATH, args)   → 启动子进程
       │   ├─ DYLD_LIBRARY_PATH=BIN_DIR (macOS 找 dylib)
       │   ├─ 解析 stdout/stderr 的进度行
       │   │   └─ parseGenerationProgressChunk()
       │   │       匹配 "step N/M" 或 "N/M - Xs/it"
       │   └─ 通过 IPC 发送 progress 事件
       │
       └─ 进程退出:
           ├─ exit code ≠ 0 → 抛出错误 (含尾部输出)
           ├─ 输出图片不存在 → 抛出错误
           └─ 成功 → 读取图片 → base64 dataUrl → 返回
```

### 3.2 Wan2GP 完整流程

```
用户操作                        Electron 主进程
─────────────────              ─────────────────

1. 打开 Local Models 设置
   ├─ Wan2gpConfigBar.render()
   │   ├─ getWan2gpConfig()         → wan2gpProvider.js
   │   │   └─ 读取 wan2gp.json (存于 userData/local-ai/)
   │   ├─ 预填 URL input
   │   └─ probe(url)                → wan2gpProvider.js
   │       ├─ GET /config → 验证是 Gradio 服务器
   │       └─ fetchApiNames()       → 发现可用端点
   │           ├─ 尝试 /info → named_endpoints
   │           ├─ 尝试 /api → api 列表
   │           └─ 尝试 /config → dependencies[].api_name
   │
   └─ listModels()                  → wan2gpProvider.js
       ├─ probe(url) → fnResolutionCache 缓存结果
       └─ withWan2gpAvailability()
           ├─ resolveFnNames(): 按优先级匹配 api_name
           │   ├─ 1. 精确匹配 model.fn
           │   ├─ 2. 精确匹配 model.fnAliases[]
           │   └─ 3. 模糊匹配 (api_name 包含 family 字符串)
           └─ 返回每个模型的 ready / fn 状态

2. 用户在 Studio 中选择 Wan2GP 模型 + 点击生成
   └─ generate(params)              → wan2gpProvider.js
       ├─ 从缓存获取解析后的 api_name
       ├─ 构建 payload:
       │   { data: [prompt, negative_prompt, W, H, steps, cfg, seed, imageDescriptor] }
       │
       ├─ POST /gradio_api/call/<fn>
       │   → 返回 { event_id }
       │
       ├─ GET /gradio_api/call/<fn>/<event_id> (SSE 流)
       │   ├─ event: generating → 解析 progress
       │   ├─ event: complete   → 解析结果数据
       │   └─ event: error      → 抛出错误
       │
       └─ resolveOutputUrl(): 从结果提取媒体 URL
           └─ 返回 { url, mediaType, seed }
```

---

## 4. sd.cpp 引擎详解

### 4.1 文件路径

```
用户数据目录 (userData)
└── local-ai/                          # 根目录（可被 OPEN_GENERATIVE_AI_LOCAL_AI_DIR 覆盖）
    ├── bin/
    │   └── sd-cli / sd-cli.exe        # 推理引擎二进制
    ├── models/
    │   ├── z_image_turbo-Q4_K.gguf    # 模型权重文件
    │   ├── DreamShaper_8_pruned.safetensors
    │   ├── sd_xl_base_1.0.safetensors
    │   ├── Qwen3-4B-...gguf           # Z-Image 辅助 LLM
    │   └── ae.safetensors             # Z-Image 辅助 VAE
    └── tmp/
        └── gen-{timestamp}.png        # 临时输出文件（生成后删除）
```

路径通过 `electron/lib/localInferencePaths.js` 的 `resolveLocalAiPaths()` 计算。可通过环境变量 `OPEN_GENERATIVE_AI_LOCAL_AI_DIR` 覆盖。

### 4.2 二进制管理

二进制来源有两种：

**方式 A**：打包进桌面应用的预编译二进制 (`build/local-ai/` → 构建时复制到 extraResources)

**方式 B**：从 GitHub Release 在线下载
- macOS arm64 → 自托管 Metal 优化版本 (`sd-cli-metal-macos-arm64.zip`)
- 其他平台 → 搜索 `leejet/stable-diffusion.cpp` 仓库
  - 遍历最近 15 个 release
  - `pickBinaryAssetForPlatform()` 按优先级匹配 zip 文件名
  - 下载 → 解压 → 在子目录中查找二进制 → 移动到 `BINARY_PATH`
  - macOS 额外移除 Gatekeeper 检疫标记

### 4.3 模型目录 (`electron/lib/modelCatalog.js`)

当前预配置 6 个模型：

| ID | 名称 | 类型 | 大小 | 依赖 | 下载源 |
|----|------|------|------|------|--------|
| `z-image-turbo` | Z-Image Turbo | z-image | 2.5 GB | Qwen3-4B + VAE | HuggingFace |
| `z-image-base` | Z-Image Base | z-image | 3.5 GB | Qwen3-4B + VAE | HuggingFace |
| `dreamshaper-8` | Dreamshaper 8 | sd1 | 2.1 GB | 无 | HuggingFace |
| `realistic-vision-v51` | Realistic Vision v5.1 | sd1 | 2.1 GB | 无 | HuggingFace |
| `anything-v5` | Anything v5 | sd1 | 2.1 GB | 无 | HuggingFace |
| `stable-diffusion-xl-base` | SDXL Base 1.0 | sdxl | 6.9 GB | 无 | HuggingFace |

每个模型包含：
- `id`, `name`, `description`, `type` (模型架构类型)
- `filename`, `sizeGB`, `downloadUrl` (下载信息)
- `defaultSteps`, `defaultGuidance`, `sampler` (默认推理参数)
- `aspectRatios`, `defaultWidth`, `defaultHeight` (输出尺寸)
- `requiresAuxiliary` (是否需额外文件)
- `tags`, `featured` (展示信息)

### 4.4 推理参数构建 (`generate()` 函数)

sd-cli 命令行参数构建逻辑：

```
sd-cli <modelFlag> <modelPath>
       -p <prompt>
       -o <tempOutputPath>
       --steps <steps>
       -H <height> -W <width>
       --cfg-scale <guidance>
       --seed <seed>
       --sampling-method <sampler>
       -v                                          # 详细输出
       [-n <negative_prompt>]
       [--diffusion-model <path>]                  # z-image / flux 专用
       [--llm <path> --vae <path> --scheduler <scheduler>]  # z-image 专用
       [--sd-version sdxl]                         # SDXL 专用
       [--flux]                                    # Flux 专用
       [--sd-version sd2]                          # SD2 专用
```

模型标志映射：
- `z-image` / `flux` → `--diffusion-model`（GGUF 文件作为扩散模型加载）
- `sd1` / `sd2` / `sdxl` → `-m`（标准 SD 检查点）

### 4.5 进度解析 (`localInferenceRuntime.js`)

从 sd-cli 的 stdout/stderr 中实时解析生成进度：

```
stdout 原始输出:  "step 3/8" 或 "3/8 - 2.5s/it"
         ↓
stripAnsiSequences()   ← 清除 ANSI 转义码
         ↓
extractProgressEvents()  ← 正则匹配 step N/M 或 N/M - Xs/it
         ↓
parseGenerationProgressChunk()
         ├─ 去重：只返回大于 lastStep 的事件
         ├─ 上下文切换检测：totalSteps 变化时重置 lastStep
         └─ 状态容器：保留尾部 1024 字符用于跨 chunk 匹配
         ↓
IPC 事件: { step, totalSteps, status: 'generating', progress: 0.125 }
```

---

## 5. Wan2GP 远程引擎详解

### 5.1 工作原理

Wan2GP 是一个独立的 Gradio 应用（Python），用户在 GPU 服务器上手动运行。本应用作为 HTTP 客户端与之通信：

```
用户运行: python wgp.py --listen --server-name 0.0.0.0
                    │
                    ▼
              http://<ip>:7860
                    │
┌───────────────────┴───────────────────┐
│  Gradio v4 协议                       │
│                                       │
│  POST  /gradio_api/call/<api_name>    │
│       → 返回 { event_id }             │
│                                       │
│  GET   /gradio_api/call/<fn>/<event_id>│
│       → SSE 数据流                     │
│  event: generating                    │
│  data: {"progress": 0.5}             │
│  ...                                  │
│  event: complete                      │
│  data: [output, ...]                  │
└───────────────────────────────────────┘
```

### 5.2 API 名称解析

Wan2GP 的 api_name 在不同版本间可能变化。系统通过分层匹配解决：

```
fetchApiNames(base)
  ├─ /info → named_endpoints (Gradio v4)
  ├─ /api → api 名称列表
  └─ /config → dependencies[].api_name (旧版本)
         ↓
resolveFnNames(apiNames)
  ├─ 精确匹配 fn (如 "flux")
  ├─ 精确匹配 fnAliases (如 "flux_dev", "flux_1_dev")
  └─ 模糊匹配: api_name 包含 family 字符串且匹配类型关键字
         ↓
fnResolutionCache 缓存结果
```

### 5.3 文件上传

Gradio 的文件上传通过 `<base>/upload` 端点完成：

```
renderer → IPC → wan2gpProvider.uploadFile({ name, type, bytes })
  └─ POST {base}/upload?upload_id=<id> (multipart/form-data)
       └─ Gradio 返回服务器路径
            └─ 缓存到 uploadedFiles Map
                 └─ generate() 时重建 Gradio 文件描述符 { path, url, orig_name, mime_type }
```

### 5.4 模型目录 (`wan2gpProvider.js`)

| ID | 名称 | 类型 | 默认 api_name | 别名 |
|----|------|------|---------------|------|
| `wan2gp:flux-dev` | Flux.1 Dev | image | `flux` | `flux_dev`, `flux_1_dev` |
| `wan2gp:qwen-image` | Qwen Image | image | `qwen_image` | `qwen`, `qwen_t2i` |
| `wan2gp:wan22-t2v` | Wan 2.2 T2V | video | `wan22_t2v` | `wan_2_2_t2v`, `wan_t2v` |
| `wan2gp:wan22-i2v` | Wan 2.2 I2V | video | `wan22_i2v` | `wan_2_2_i2v`, `wan_i2v` |
| `wan2gp:hunyuan-video` | Hunyuan Video | video | `hunyuan_video` | `hunyuan`, `hyvideo` |
| `wan2gp:ltx-video` | LTX Video | video | `ltx_video` | `ltx`, `ltx_t2v` |

---

## 6. 前端调用链路

### 6.1 IPC 封装 (`src/lib/localInferenceClient.js`)

前端不直接调用 `electron.ipcRenderer`，而是通过封装类：

```javascript
// 关键 API 概览
localAI.getBinaryStatus()       → Promise<{ exists, path, modelsDir, envVar }>
localAI.downloadBinary()        → Promise<void>
localAI.listModels()            → Promise<Model[]>
localAI.downloadModel(id)       → Promise<void>
localAI.downloadAuxiliary(key)  → Promise<void>
localAI.deleteModel(id)         → Promise<void>
localAI.generate(params)        → Promise<{ url: dataUrl|string, seed }>
localAI.cancelGeneration()      → Promise<void>
localAI.onProgress(callback)    → unsubscribeFn     // 生成进度订阅
localAI.onDownloadProgress(cb)  → unsubscribeFn     // 下载进度订阅

// Wan2GP 专用
localAI.getWan2gpConfig()       → Promise<{ url }>
localAI.setWan2gpUrl(url)       → Promise<void>
localAI.probeWan2gp(url)        → Promise<{ ok, version, ... }>
```

### 6.2 前端模型数据 (`src/lib/localModels.js`)

维护与后端 `electron/lib/modelCatalog.js` + `electron/lib/wan2gpProvider.js` 对应的前端模型目录。包含 `provider` 字段区分引擎：

```javascript
getLocalModelById(id)     → 查找模型
isWan2gpModelId(id)       → 是否是 Wan2GP 模型
isLocalModelId(id)        → 是否是本地模型（任一引擎）
```

### 6.3 Studio 集成（以 ImageStudio 为例）

在 `src/components/ImageStudio.js` 中：

```
UI 切换按钮
  └─ useLocalModel = true/false
        │
        ├─ 切换时:
        │   ├─ 隐藏 API Key 输入框
        │   └─ 显示本地模型选择器下拉
        │
        ├─ 文件上传:
        │   WiP 模式下使用 URL.createObjectURL(file)
        │   (不需要上传到远程)
        │
        └─ 生成时:
            ├─ localAI.generate({ model, prompt, ... })
            ├─ 订阅 localAI.onProgress()
            │   └─ 更新进度条 + 按钮文本
            ├─ 返回结果 → addToHistory() + showImageInCanvas()
            └─ 错误 → 恢复 UI + 显示错误信息
```

### 6.4 模型管理器 UI (`src/components/LocalModelManager.js`)

纯 DOM 方式构建的完整管理界面，分为三个区域：

| 区域 | 内容 |
|------|------|
| **引擎状态** | sd.cpp 二进制安装状态 + 安装/重试按钮 + 下载进度条 |
| **Wan2GP 配置** | URL 输入框 + Test + Save + 连接状态 |
| **模型列表** | 每个模型的卡片（名称、标签、大小、类型）+ 下载/删除按钮 + 进度条 + Z-Image 组件区域 |

---

## 7. 新增本地模型

### 7.1 新增 sd.cpp 模型

需要**同步修改三处文件**：

**① `electron/lib/modelCatalog.js`**（后端，主进程使用）：

```javascript
{
    id: 'my-new-model',
    name: 'My New Model',
    description: '...',
    type: 'sd1',                  // sd1 | sdxl | sd2 | z-image | flux
    filename: 'model.safetensors',
    sizeGB: 2.1,
    downloadUrl: 'https://huggingface.co/.../model.safetensors',
    aspectRatios: ['1:1', '4:3', '3:4', '16:9', '9:16'],
    defaultWidth: 512,
    defaultHeight: 512,
    defaultSteps: 20,
    defaultGuidance: 7.5,
    sampler: 'euler_a',
    tags: ['tag1', 'tag2'],
    featured: false,
    requiresAuxiliary: false,      // true = Z-Image 需要额外 LLM+VAE
}
```

**② `src/lib/localModels.js`**（前端，渲染进程使用）：

```javascript
{
    id: 'my-new-model',           // 必须与后端一致
    name: 'My New Model',
    description: '...',
    type: 'sd1',
    provider: 'sdcpp',            // sdcpp | wan2gp
    filename: 'model.safetensors',
    sizeGB: 2.1,
    aspectRatios: ['1:1', '4:3', '3:4', '16:9', '9:16'],
    defaultSteps: 20,
    defaultGuidance: 7.5,
    tags: ['tag1', 'tag2'],
    featured: false,
}
```

**③ （可选）`electron/lib/localInference.js`** 的 `generate()` 函数：

如果新模型类型需要不同的 sd-cli 参数，需添加对应的参数分支：

```javascript
if (model.type === 'my-new-type') {
    args.push('--my-flag');
}
```

### 7.2 新增 Wan2GP 模型

**① `electron/lib/wan2gpProvider.js`**：

```javascript
{
    id: 'wan2gp:my-model',
    name: 'My Model (Wan2GP)',
    description: '...',
    type: 'video',                // image | video
    family: 'mymodel',            // 用于模糊匹配
    provider: 'wan2gp',
    fn: 'my_model_fn',
    fnAliases: ['my_fn', 'my_model'],
    needsImage: false,            // true = 需要输入图片
    aspectRatios: ['16:9', '1:1', '9:16'],
    defaultSteps: 30,
    defaultGuidance: 6.0,
    tags: ['video'],
}
```

**② `src/lib/localModels.js`**（前端）：

```javascript
{
    id: 'wan2gp:my-model',
    name: 'My Model (Wan2GP)',
    description: '...',
    type: 'video',
    family: 'mymodel',
    provider: 'wan2gp',
    aspectRatios: ['16:9', '1:1', '9:16'],
    defaultSteps: 30,
    defaultGuidance: 6.0,
    tags: ['video'],
}
```

### 7.3 为 Studio 组件添加新的本地模型类型

如果新模型需要不同的 UI 参数面板或生成流程：

1. 在 `src/components/ImageStudio.js`（或对应 Studio）中：
   - 在模型选择区域添加对新 `type` / `provider` 的处理
   - 根据 `model.type` 显示/隐藏特定参数输入
   - 在 `localAI.generate()` 调用中传入新参数
2. 如果新模型产生视频而非图像，确保 Studio 在 `mediaType === 'video'` 时做相应处理

### 7.4 测试新模型

- **模型下载**：在 LocalModels 窗口中触发下载，检查 `local-ai/models/` 目录
- **推理执行**：查看 Electron 主进程日志中的 sd-cli 命令及输出
- **进度显示**：确认 stdout 的步骤日志被正确解析为进度事件
- **结果返回**：检查生成的文件是否被正确读取和编码为 data URL
