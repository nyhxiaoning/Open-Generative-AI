# 本地模型使用教程

> 本文档教你如何在桌面版 Open Generative AI 中使用本地模型生成图像和视频。

---

## 目录

- [适用条件](#适用条件)
- [第一步：打开本地模型设置](#第一步打开本地模型设置)
- [第二步（方式 A）：安装 sd.cpp 引擎](#第二步方式-a安装-sdcpp-引擎)
- [第二步（方式 B）：连接 Wan2GP 远程服务器](#第二步方式-b连接-wan2gp-远程服务器)
- [第三步：下载模型](#第三步下载模型)
- [第四步：在 Studio 中使用本地模型](#第四步在-studio-中使用本地模型)
- [常见问题](#常见问题)

---

## 适用条件

| 条件 | 说明 |
|------|------|
| **必须使用桌面版** | 浏览器 Web 版不支持本地推理，只有 Electron 桌面应用才有此功能 |
| **sd.cpp 引擎** | macOS 可用（推荐 Apple Silicon），Windows/Linux 也可用，但依赖 CPU/GPU |
| **Wan2GP 引擎** | 需要一台有 NVIDIA GPU 的独立服务器运行 Gradio 服务 |

> 如果你只是在浏览器中打开 `http://localhost:3000`，本地模型功能不可用。请运行 `npm run electron:dev` 或打开已打包的 `.dmg` / `.exe` 应用。

---

## 第一步：打开本地模型设置

1. 打开桌面版应用
2. 点击右上角 **设置图标**（⚙️）
3. 在弹出的设置窗口中，你会看到两个 Tab：
   - **API Key** — 输入 MuAPI 密钥（使用远程模型时用）
   - **Local Models** — 本地模型管理（桌面版特有）
4. 点击 **Local Models** 标签

你会看到如下界面：

```
┌─ Settings ─────────────────────────────────┐
│  API Key  |  [Local Models]                 │
│                                             │
│  ── INFERENCE ENGINE ────────────────────── │
│  ┌──────────────────────────────────────┐   │
│  │ sd.cpp inference engine              │   │
│  │ ⚠ Not installed         [Install]   │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │ Wan2GP server (optional)            │   │
│  │ [http://________] [Test] [Save]     │   │
│  └──────────────────────────────────────┘   │
│                                             │
│  ── MODELS ──────────────────────────────── │
│  (模型列表)                                  │
└─────────────────────────────────────────────┘
```

---

## 第二步（方式 A）：安装 sd.cpp 引擎

sd.cpp 引擎是你电脑本地运行的推理程序。**只需要安装一次**。

### 操作步骤

1. 在 Local Models 设置页，找到 "sd.cpp inference engine" 卡片
2. 如果显示 ⚠ **Not installed** 状态，点击右边的 **[Install]** 按钮
3. 等待下载完成，进度条会显示百分比：

```
┌──────────────────────────────────────┐
│ sd.cpp inference engine              │
│ Downloading 67%                    ██│
│ ██████████████████░░░░░░░░░░░░░ 67%  │
└──────────────────────────────────────┘
```

4. 下载完成后会自动解压，状态变为 ✅ **Installed**
5. 此时引擎已就绪，可以下载模型了

### 下载速度慢？

引擎二进制约 20-30 MB，通常在几秒内完成。如果下载失败，点击 **Retry** 按钮重试。

### 引擎在哪里？

安装在你的用户数据目录下：

```
macOS:   ~/Library/Application Support/Open-Generative-AI/local-ai/bin/sd-cli
Windows: %APPDATA%/Open-Generative-AI/local-ai/bin/sd-cli.exe
Linux:   ~/.config/Open-Generative-AI/local-ai/bin/sd-cli
```

你也可以在启动前设置环境变量 `OPEN_GENERATIVE_AI_LOCAL_AI_DIR` 来改变存储路径。

---

## 第二步（方式 B）：连接 Wan2GP 远程服务器

如果你有一台带 NVIDIA GPU 的服务器，可以运行 Wan2GP 来获得更强的模型（包括视频生成）。

> 方式 A 和 方式 B 可以共存。两种引擎安装后，你可以在 Studio 中自由选择。

### 在服务器上启动 Wan2GP

在 GPU 服务器上执行：

```bash
git clone https://github.com/deepbeepmeep/Wan2GP
cd Wan2GP
pip install -r requirements.txt
python wgp.py --listen --server-name 0.0.0.0
```

启动后你会看到类似输出：

```
Running on local URL:  http://0.0.0.0:7860
```

### 在桌面应用中连接

1. 在 Local Models 设置页，找到 "Wan2GP server" 区域
2. 在输入框中填入服务器地址：
   - 如果是**本机运行** Wan2GP：`http://127.0.0.1:7860`
   - 如果是**远程服务器**：`http://<服务器IP>:7860`
3. 点击 **[Test]** 按钮验证连接
4. 如果显示 ✅ **Connected · Gradio v4.x**，说明连接成功
5. 点击 **[Save]** 保存配置

### 连接失败怎么办？

| 错误 | 原因 | 解决 |
|------|------|------|
| "Failed to fetch" | 服务器地址不对 | 确认 IP 和端口号 |
| "Connection refused" | 服务器没启动 | 在服务器上运行 Wan2GP |
| "Timed out" | 防火墙阻挡 | 检查服务器防火墙是否开放 7860 端口 |
| "Not a Gradio server" | 地址不是 Wan2GP | 确认路径正确 |

---

## 第三步：下载模型

引擎安装完成后（或 Wan2GP 连接成功后），你会看到可用模型列表。

### sd.cpp 模型

每个模型卡片显示：
- 模型名称和描述
- 类型标签（如 `SD1` / `SDXL` / `Z-IMAGE`）
- 文件大小（如 `2.1 GB`）
- 标签（如 `fast` / `photorealistic` / `anime`）

**下载单个模型：**

1. 找到你想用的模型（推荐从 **Z-Image Turbo** 开始，这是最快且最推荐的本地模型）
2. 如果模型状态不是 "downloaded"，点击 **[Download]** 按钮
3. 等待下载完成。模型大小从 2 GB 到 7 GB 不等，下载时间取决于你的网络：

```
┌─ Z-Image Turbo ───────────────────────────┐
│ ⚡ FEATURED                                    │
│ WaveSpeed's featured local model...       │
│ Z-IMAGE  3.4 GB  turbo fast local        │
│                                           │
│ ████████████░░░░░░░░░░  Downloading 52%   │
└──────────────────────────────────────────────┘
```

4. 下载完成后，状态会显示绿色 ✅ 及 **downloaded**

**Z-Image 模型特殊说明：**

Z-Image Turbo 和 Z-Image Base 除了模型本身，还需要额外下载两个辅助文件：
- **Qwen3-4B Text Encoder** — 约 2.4 GB
- **FLUX VAE** — 约 335 MB

在模型卡片下方会看到：

```
┌─ Required Components ─────────────────────┐
│ ✅ Qwen3-4B Text Encoder (2.4 GB)   Ready │
│ ✅ FLUX VAE (335 MB)                Ready │
└────────────────────────────────────────────┘
```

如果没有 Ready，点击 **[Get]** 按钮分别下载。**这两个文件所有 Z-Image 模型共享，只需下载一次。**

**删除模型：**

点击已下载模型右侧的红色垃圾桶 🗑️ 图标，确认后即可删除。

### Wan2GP 模型

Wan2GP 模型不需要下载（模型运行在 GPU 服务器上），每个模型卡片会显示在线状态：
- 如果服务器已连接且模型可用 → 显示 ✅ **Available**
- 如果服务器未连接或模型不可用 → 显示 ⚠️ **Offline**

---

## 第四步：在 Studio 中使用本地模型

模型下载完成后，就可以在生成界面中使用了。以 ImageStudio 为例：

### 切换本地模式

1. 进入 ImageStudio（图像生成界面）
2. 在提示词输入框上方的工具栏中，找到切换按钮：
   - 显示 **API** → 远程模式（调用 MuAPI）
   - 显示 **Local** → 本地模式（绿色/青色高亮）
3. 点击按钮切换到 **Local** 模式

### 选择本地模型

1. 点击模型选择按钮（默认显示本地模型名称）
2. 在下拉列表中会列出所有已下载的 sd.cpp 模型 + Wan2GP 可用模型
3. 选择你要用的模型

### 输入提示词并生成

1. 在输入框中输入描述文字（如 "a cat sitting on a windowsill, afternoon light"）
2. 可选：填写负面提示词（Negative Prompt）、调整步数（Steps）和引导比例（Guidance Scale）
3. 点击 **Generate** 按钮

### 观察生成进度

生成过程中会显示进度条：

```
──────────────────────────────────────
Generating Locally          47%
█████████████████░░░░░░░░░ 47%
                        [Cancel]
──────────────────────────────────────
```

- 进度条实时反映 sd.cpp 的生成进度（每隔几步刷新一次）
- 底部的 **[Cancel]** 按钮可以随时取消当前生成
- Wan2GP 的进度取决于服务器端生成速度

### 查看结果

生成完成后，图像会像远程生成一样显示在画布中，并自动添加到历史记录。

### 注意：文件上传

在本地模式下上传参考图片（用于图生图），图片不会上传到任何服务器，而是直接使用 `URL.createObjectURL()` 创建本地 URL。所有处理都在你的电脑上完成。

---

## 常见问题

### Q：本地模型和远程模型有什么区别？

| | 本地模型 | 远程 MuAPI |
|---|---|---|
| 需要网络 | 推理时不需要（sd.cpp） | 始终需要 |
| 速度 | 取决于你的硬件（CPU/GPU） | 取决于服务器 |
| 模型数量 | 有限（目前 6 个 sd.cpp + 6 个 Wan2GP） | 200+ |
| 视频生成 | 仅 Wan2GP 支持 | 丰富 |
| 免费？ | 完全免费 | 需要 API Key 和额度 |

### Q：为什么本地模式切换按钮不显示？

你使用的是 **Web 浏览器**（http://localhost:3000）而不是 **桌面应用**。请运行 `npm run electron:dev` 或在 Releases 下载桌面版。

### Q：下载模型时进度卡住不动了？

- 模型文件较大（2-7 GB），大文件下载可能需要几分钟
- 如果长时间不动，可以点击 **Retry** 重试
- 下载支持断点续传，重启下载会从上次中断位置继续

### Q：Z-Image 模型辅助文件下载失败？

Z-Image 需要 Qwen3-4B（2.4 GB）和 FLUX VAE（335 MB）。如果只下载了模型而没有下载这两个辅助文件，生成时会报错。在模型卡片下方的 Required Components 区域分别点击 [Get] 下载即可。

### Q：生成时报错 "output not returned"？

可能原因：
- 模型文件损坏 → 删除模型重新下载
- 磁盘空间不足 → 检查硬盘剩余空间
- Z-Image 缺少辅助文件 → 检查 Required Components 是否全部 Ready
- 某些模型需要大量内存（如 SDXL 约 6.9 GB 模型 + 运行时内存），内存不足会导致进程崩溃

### Q：生成速度很慢？

- sd.cpp 使用 CPU 推理（macOS 上 Apple Silicon 用 Metal GPU 加速）
- Z-Image Turbo 只需要 8 步，是最快的选择
- Dreamshaper 8 / Anything v5（SD 1.5 模型）只需要 20 步，平衡质量和速度
- SDXL 需要 30 步且模型更大，推荐只在性能好的机器上使用

### Q：Wan2GP 连接成功后模型还是显示 Offline？

- 确认 Wan2GP 服务器的模型已正确加载
- 尝试点击 **[Save]** 重新保存配置
- 在 Wan2GP 服务器端查看日志，确认没有报错
- 某些模型需要大量显存（如 Wan 2.2 视频模型需要至少 16 GB VRAM）

### Q：可以同时用本地模型和远程模型吗？

可以。Local/API 切换按钮让你在两种模式间随时切换：
- **Local 模式**：使用本地模型，无需 API Key
- **API 模式**：使用 MuAPI 远程模型，需要 API Key

两者互不干扰，你的 API Key 和本地模型各自独立管理。

---

## 快速启动清单

如果是第一次使用，按这个顺序操作：

1. ✅ 下载并打开 **桌面版** 应用
2. ✅ 打开设置 → **Local Models**
3. ✅ 点击 **Install** 安装 sd.cpp 引擎
4. ✅ 下载 **Z-Image Turbo** 模型（点 Download）
5. ✅ 下载辅助文件：**Qwen3-4B** + **FLUX VAE**（点 Get）
6. ✅ 回到 **ImageStudio**，点击 **API** 切换到 **Local**
7. ✅ 输入提示词 → 点击 **Generate**
8. ✅ 等待进度条完成 → 查看生成结果 🎉

整个过程约需要下载：~20 MB（引擎）+ ~2.5 GB（Z-Image）+ ~2.4 GB（Qwen3-4B）+ ~335 MB（VAE）。文件较多，请耐心等待下载完成。
