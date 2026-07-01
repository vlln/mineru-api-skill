# MinerU + mineru-api Skill 安装与使用手册

> 适用版本: MinerU 3.x (当前 3.1.9) | mineru-api-skill 0.1.0
> 最后更新: 2026-05-11

---

## 目录

1. [概述](#概述)
2. [环境要求](#环境要求)
3. [MinerU 服务端部署](#mineru-服务端部署)
   - [方式 A: Docker 部署（推荐 GPU 服务器）](#方式-a-docker-部署推荐-gpu-服务器)
   - [方式 B: pip 直接安装](#方式-b-pip-直接安装)
   - [方式 C: Docker Compose 多服务](#方式-c-docker-compose-多服务)
4. [mineru-api Skill 安装与配置](#mineru-api-skill-安装与配置)
5. [使用方式](#使用方式)
   - [通过 Claude Code 使用](#通过-claude-code-使用)
   - [命令行直接使用](#命令行直接使用)
   - [Raw HTTP API 调用](#raw-http-api-调用)
6. [完整示例](#完整示例)
7. [常见问题与故障排除](#常见问题与故障排除)
8. [附录: Docker 镜像体积分析](#附录-docker-镜像体积分析)

---

## 概述

[MinerU](https://github.com/opendatalab/MinerU) 是 OpenDataLab 开源的 PDF 转 Markdown 工具，支持从 PDF 中提取文字、表格、公式、图片，支持中/英/日/韩等多语言，并支持 OCR 识别扫描件。

**mineru-api-skill** 是一个 Agent Skills 封装，让 Claude Code 等兼容 Agent 能通过 HTTP API 远程调用 MinerU 解析 PDF，无需在本地运行 MinerU 引擎。

```
┌─────────────────────┐     HTTP API      ┌──────────────────┐
│  Claude Code / CLI  │ ────────────────→  │  MinerU Server   │
│  (mineru-api skill) │                    │  (GPU 机器)       │
└─────────────────────┘                    └──────────────────┘
```

---

## 环境要求

### 服务端（运行 MinerU 的机器）

| 项目 | 要求 |
|------|------|
| 操作系统 | Linux（Docker 仅支持 Linux/WSL2，不支持 macOS Docker） |
| GPU | 可选但强烈推荐；Volta 架构及以上（Compute Capability >= 7.0），显存 8GB+ |
| NVIDIA 驱动 | CUDA 12.9.1+（Docker 方式） |
| 内存 | 16GB+ |
| 磁盘 | 30GB+（含基础镜像和模型） |

### 客户端（使用 skill 的机器）

| 项目 | 要求 |
|------|------|
| Python | 3.8+（仅标准库，无需 pip 安装额外依赖） |
| 网络 | 能访问 MinerU 服务端的 HTTP 端口 |
| Claude Code | 可选，但 skill 也可脱离 Claude Code 独立运行 |

---

## MinerU 服务端部署

### 方式 A: Docker 部署（推荐 GPU 服务器）

#### 1. 构建镜像

```bash
# 下载 Dockerfile
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/docker/global/Dockerfile

# 构建镜像（耗时较长，约需下载 20GB+ 数据）
docker build -t mineru:latest -f Dockerfile .
```

**构建过程会下载：**
- `vllm/vllm-openai:v0.11.2` 基础镜像（约 13GB 压缩后）
- Noto 中文字体、libgl1 等系统依赖（约 100MB）
- `mineru[core]` 及 30+ Python 依赖包（约 500MB–1GB）
- Pipeline 模型（约 6GB）和 VLM 模型（约 2.2GB），总计约 8GB 模型文件

最终镜像体积约 **20–23GB**。

> **提示:** 在中国大陆网络环境下，从 HuggingFace 下载模型可能很慢。可以考虑：
> - 使用 modelscope 源：`mineru-models-download -s modelscope -m all`
> - 或修改 Dockerfile，将模型下载改为使用 modelscope

#### 2. 进入容器交互式使用

```bash
docker run --gpus all \
  --shm-size 32g \
  -p 30000:30000 -p 7860:7860 -p 8000:8000 -p 8002:8002 \
  --ipc=host \
  -it mineru:latest \
  /bin/bash
```

进入容器后，可以运行 `mineru` 命令行工具。

#### 3. 直接启动 API 服务

```bash
docker run --gpus all \
  --shm-size 32g \
  -p 8000:8000 \
  --ipc=host \
  -d \
  --name mineru-api \
  mineru:latest \
  /bin/bash -c "export MINERU_MODEL_SOURCE=local && mineru-api --host 0.0.0.0 --port 8000"
```

### 方式 B: pip 直接安装

适用于有 Python/CUDA 环境且不想用 Docker 的 GPU 机器。

```bash
# 1. 安装 mineru
pip install 'mineru[core]>=3.0.0'

# 2. 下载模型（二选一）
# HuggingFace（需要网络通畅）
mineru-models-download -s huggingface -m all
# ModelScope（中国大陆更快）
mineru-models-download -s modelscope -m all

# 3. 启动 API 服务
export MINERU_MODEL_SOURCE=local
mineru-api --host 0.0.0.0 --port 8000
```

> **说明:** `-m all` 下载 pipeline 和 VLM 两类模型。如果只用 pipeline 后端，可以用 `-m pipeline` 节省约 2.2GB 下载。

### 方式 C: Docker Compose 多服务

MinerU 提供多种服务形态，可以通过 compose profile 按需启动：

```bash
# 下载 compose 文件
wget https://gcore.jsdelivr.net/gh/opendatalab/MinerU@master/docker/compose.yaml

# 启动 OpenAI 兼容 server（适合 vlm-http-client 后端）
docker compose -f compose.yaml --profile openai-server up -d

# 启动 Web API 服务（mineru-api-skill 对接此服务）
docker compose -f compose.yaml --profile api up -d

# 启动 Router 服务（聚合多个 API 节点）
docker compose -f compose.yaml --profile router up -d

# 启动 Gradio WebUI
docker compose -f compose.yaml --profile gradio up -d
```

各服务端口说明：

| 服务 | 端口 | 用途 |
|------|------|------|
| `mineru-api` | 8000 | REST API（skill 对接此端口） |
| `mineru-openai-server` | 30000 | OpenAI 兼容的 VLM 推理接口 |
| `mineru-router` | 8002 | 多节点路由与负载均衡 |
| `mineru-gradio` | 7860 | WebUI 界面上传解析 |

> **注意:** vllm 引擎会预占显存，同一张 GPU 上无法同时启动多个 vllm 服务。

---

## mineru-api Skill 安装与配置

### 1. 安装 Skill

```bash
# 通过 skit 安装
skit install vlln/mineru-api-skill/skills/mineru-api
```

或者直接克隆仓库：

```bash
git clone <your-repo-url> mineru-api-skill
```

### 2. 配置 .env 文件

在项目工作目录下创建 `.env` 文件：

```bash
# MinerU API 服务端地址（必填）
MINERU_API_URL=http://<GPU服务器IP>:8000

# 请求超时秒数（可选，默认 600）
MINERU_API_TIMEOUT=600

# 异步任务轮询间隔秒数（可选，默认 3）
MINERU_API_POLL_INTERVAL=3
```

`.env.example` 已包含模板供参考：

```bash
cp .env.example .env
# 然后编辑 .env 中的 MINERU_API_URL
```

### 3. 验证连通性

```bash
# 检查 API 服务是否可达
./skills/mineru-api/scripts/mineru-api --check
```

期望输出：

```
mineru-api: http://<服务器IP>:8000 is healthy
```

---

## 使用方式

### 通过 Claude Code 使用

安装 skill 后，直接在 Claude Code 中用自然语言交互：

```
请用中文解析 /path/to/paper.pdf，输出到 output/ 目录
```

```
批量解析 /data/papers/ 下所有 PDF，用 pipeline 后端
```

```
检查 MinerU 服务器状态
```

Claude Code 会自动调用 `mineru-api` skill 并映射正确的参数。

### 命令行直接使用

skill 的 `scripts/mineru-api` 脚本可以脱离 Claude Code 独立运行，完全兼容 `mineru` CLI 参数。

#### 单文件解析

```bash
scripts/mineru-api -p paper.pdf -o output -l en -b pipeline
```

输出结构（与 `mineru` 一致）：

```
output/
└── paper/
    └── auto/
        └── paper.md
```

#### 指定页面范围

```bash
# 仅解析第 3–7 页（0-indexed）
scripts/mineru-api -p paper.pdf -o output -l en -s 2 -e 6
```

#### OCR 扫描件

```bash
scripts/mineru-api -p scanned.pdf -o output -l ch -m ocr -b pipeline
```

#### 异步模式（适用于大文件）

```bash
scripts/mineru-api --async -p large.pdf -o output -l en
```

异步模式下，脚本提交任务后周期性轮询状态，完成后自动下载结果。

#### 批量解析目录

```bash
scripts/mineru-api -p /data/papers/ -o extracted/ -l en
```

批量模式下使用同步模式，一个文件失败不会影响其他。

#### 使用远程 VLM 引擎

```bash
scripts/mineru-api -p paper.pdf -o output -b vlm-http-client -l en
```

需要在服务器端同时运行 `openai-server` 服务（端口 30000）。

### 参数速查

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-p` / `--path` | 必填 | PDF 文件路径或目录 |
| `-o` / `--output` | 必填 | 输出目录 |
| `-b` / `--backend` | `pipeline` | `pipeline` / `vlm-http-client` / `hybrid-auto-engine` 等 |
| `-m` / `--method` | `auto` | 解析方法: `auto` / `txt` / `ocr` |
| `-l` / `--lang` | `en` | 语言: `en` / `ch` / `ch_server` / `japan` / `korean` |
| `-s` / `--start` | `0` | 起始页（0-indexed） |
| `-e` / `--end` | `-1` | 结束页（-1 表示全部） |
| `-f` / `--formula` | `true` | 是否提取公式 |
| `-t` / `--table` | `true` | 是否提取表格 |
| `--async` | 关闭 | 异步提交并轮询 |
| `--check` | — | 仅检查服务健康状态 |

### 后端选择指南

| 场景 | 推荐后端 | 命令示例 |
|------|---------|----------|
| 常规 PDF，有 GPU | `pipeline` | `-b pipeline` |
| 扫描件（图片型 PDF） | `pipeline` + OCR | `-b pipeline -m ocr` |
| 高精度需求，有 GPU | `hybrid-auto-engine` | `-b hybrid-auto-engine` |
| 远程 GPU 推理 | `vlm-http-client` | `-b vlm-http-client` |
| 中文文档 | `pipeline` + 中文 | `-b pipeline -l ch` |

### Raw HTTP API 调用

当脚本不可用或需要编程集成时，可以直接调用 REST API：

#### 健康检查

```bash
curl http://<服务器IP>:8000/health
```

#### 同步解析

```bash
curl -X POST http://<服务器IP>:8000/file_parse \
  -F "files=@paper.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=en" \
  -F "parse_method=auto" \
  -F "formula_enable=true" \
  -F "table_enable=true" \
  -F "return_md=true"
```

响应：

```json
{
  "results": [
    {
      "md_content": "# Title\n\n..."
    }
  ]
}
```

#### 异步解析

```bash
# 提交任务
curl -X POST http://<服务器IP>:8000/tasks \
  -F "files=@large.pdf" \
  -F "backend=pipeline" \
  -F "lang_list=en"

# 返回: {"task_id": "xxx-xxx-xxx"}

# 查询状态
curl http://<服务器IP>:8000/tasks/xxx-xxx-xxx

# 状态为 "done" 后获取结果
curl http://<服务器IP>:8000/tasks/xxx-xxx-xxx/result
```

完整 API 端点：

| 方法 | 路径 | 用途 |
|------|------|------|
| `GET` | `/health` | 健康检查 |
| `POST` | `/file_parse` | 同步解析（multipart/form-data） |
| `POST` | `/tasks` | 提交异步任务 |
| `GET` | `/tasks/{id}` | 查询任务状态 |
| `GET` | `/tasks/{id}/result` | 获取异步任务结果 |

---

## 完整示例

### 场景: 从零开始部署并使用

**在 GPU 服务器上（192.168.1.100）：**

```bash
# 1. 安装 MinerU
pip install 'mineru[core]>=3.0.0'

# 2. 下载模型（中国大陆用 modelscope）
export MINERU_MODEL_SOURCE=huggingface
mineru-models-download -s huggingface -m all

# 3. 启动 API 服务（后台运行）
export MINERU_MODEL_SOURCE=local
nohup mineru-api --host 0.0.0.0 --port 8000 > mineru-api.log 2>&1 &
```

**在本地笔记本上：**

```bash
# 4. 安装 skill
skit install vlln/mineru-api-skill/skills/mineru-api

# 5. 配置服务器地址
cd /path/to/your/project
cat > .env << 'EOF'
MINERU_API_URL=http://192.168.1.100:8000
MINERU_API_TIMEOUT=600
MINERU_API_POLL_INTERVAL=3
EOF

# 6. 验证连通性
./skills/mineru-api/scripts/mineru-api --check
# 输出: mineru-api: http://192.168.1.100:8000 is healthy

# 7. 解析 PDF
./skills/mineru-api/scripts/mineru-api -p paper.pdf -o output -l en

# 8. 查看结果
cat output/paper/auto/paper.md
```

---

## 常见问题与故障排除

### 1. 构建 Docker 镜像太慢

Docker build 主要耗时在三个方面：

| 步骤 | 大小 | 瓶颈 |
|------|------|------|
| 拉取 `vllm/vllm-openai:v0.11.2` | 13GB | Docker Hub 带宽 |
| pip 安装 `mineru[core]` + 依赖 | ~800MB | PyPI 带宽 |
| `mineru-models-download -s huggingface -m all` | ~8GB | HuggingFace 带宽 |

优化建议：

- 在中国大陆用 ModelScope 替代 HuggingFace：`mineru-models-download -s modelscope -m all`
- 模型可挂载卷而非构建进镜像，减少 6–8GB
- 只下载需要的模型：`-m pipeline` 或 `-m vlm` 而非 `-m all`
- 预构建包含模型的基础镜像供团队复用

### 2. `mineru-api --check` 连接失败

```bash
mineru-api: http://xxx:8000 is unreachable
```

排查步骤：

1. 确认服务器上 mineru-api 正在运行：`ps aux | grep mineru-api`
2. 确认端口监听正常：`curl http://<服务器IP>:8000/health`
3. 检查防火墙：`ufw status` 或 `iptables -L`
4. 确认 `.env` 中 `MINERU_API_URL` 配置正确

### 3. 解析返回空结果

- 检查 PDF 是否为纯图片扫描件，尝试 `-m ocr`
- 检查语言设置是否匹配：中文 PDF 用 `-l ch`
- 检查服务器端模型是否下载完整

### 4. GPU 内存不足

- vllm 引擎会预占显存
- 启动参数添加 `--gpu-memory-utilization 0.5`（或更低）
- 同一 GPU 不能同时运行多个 vllm 服务
- 关闭不需要的服务：`docker compose -f compose.yaml --profile <profile> down`

### 5. `no module named 'huggingface_hub'`

mineru 依赖未完全安装，重新安装：

```bash
pip install 'mineru[core]>=3.0.0'
```

### 6. Docker 在 macOS 上性能差

Docker on macOS 无法访问 MPS/MLX 加速，Apple Silicon 设备不能获得 GPU 加速。macOS 用户建议：
- 使用 pip 方式直接安装（如果 Apple Silicon 有 MPS 支持）
- 或连接远程 Linux GPU 服务器的 API

---

## 附录: Docker 镜像体积分析

```
┌─────────────────────────────────────────────┐
│  Layer 1: vllm/vllm-openai:v0.11.2  (13GB)  │
│  ├── CUDA runtime + cuDNN                    │
│  ├── PyTorch + vllm inference engine          │
│  └── Python + system deps                     │
├─────────────────────────────────────────────┤
│  Layer 2: apt-get fonts/libgl1     (~0.1GB)  │
│  ├── fonts-noto-core                         │
│  ├── fonts-noto-cjk (中文字体)               │
│  ├── fontconfig                              │
│  └── libgl1 (OpenCV 支持)                    │
├─────────────────────────────────────────────┤
│  Layer 3: pip install mineru[core]  (~0.8GB) │
│  ├── mineru 核心包                           │
│  ├── opencv-python, numpy, pandas            │
│  ├── huggingface-hub, modelscope             │
│  ├── pdfminer, pypdf, pypdfium2              │
│  └── fastapi, uvicorn (API 服务)             │
├─────────────────────────────────────────────┤
│  Layer 4: mineru-models-download   (~8GB)    │
│  ├── Pipeline 模型 (PDF-Extract-Kit-1.0)     │
│  │   ├── models/MFR       (9.4GB)            │
│  │   ├── models/TabRec    (2.1GB)            │
│  │   ├── models/OCR       (1.0GB)            │
│  │   ├── models/Layout    (0.9GB)            │
│  │   └── models/Other     (1.0GB)            │
│  └── VLM 模型 (MinerU2.5-Pro-2604-1.2B)     │
│      └── model.safetensors (2.2GB)           │
├─────────────────────────────────────────────┤
│  最终镜像: ~22GB                             │
└─────────────────────────────────────────────┘
```