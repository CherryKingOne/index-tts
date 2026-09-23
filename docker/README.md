# IndexTTS Docker 部署指南

本文档介绍如何使用 Docker 部署和启动 IndexTTS 服务。

## 前置要求

| 组件 | 说明 |
|------|------|
| Docker Engine | >= 20.10 |
| Docker Compose | >= 2.0 |
| NVIDIA Driver | >= 520.61.05 |
| NVIDIA Container Toolkit | 最新稳定版 |

### 安装 NVIDIA Container Toolkit

```bash
# Ubuntu / Debian
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

验证 GPU 可用性：

```bash
docker run --gpus all nvidia/cuda:12.8.1-base-ubuntu22.04 nvidia-smi
```

## 快速启动

### 1. 克隆仓库

```bash
git clone https://github.com/CherryKingOne/index-tts.git
cd index-tts
```

### 2. 启动 WebUI

```bash
docker compose -f docker/docker-compose.yml up -d
```

### 3. 访问 WebUI

浏览器打开: **http://localhost:7860**

> 首次启动会自动下载模型文件到 `./checkpoints` 目录，请耐心等待。

## Docker Compose 命令参考

```bash
# 启动服务 (后台运行)
docker compose -f docker/docker-compose.yml up -d

# 查看实时日志
docker compose -f docker/docker-compose.yml logs -f

# 停止服务
docker compose -f docker/docker-compose.yml down

# 重启服务
docker compose -f docker/docker-compose.yml restart

# 查看运行状态
docker compose -f docker/docker-compose.yml ps

# 重新构建镜像后启动
docker compose -f docker/docker-compose.yml up -d --build
```

## 目录挂载说明

| 宿主机路径 | 容器内路径 | 说明 |
|-----------|-----------|------|
| `${PWD}/checkpoints` | `/app/checkpoints` | 模型文件目录 |
| `${PWD}/outputs` | `/app/outputs` | 生成音频输出目录 |
| `${PWD}/prompts` | `/app/prompts` | 预设配置目录 |

> 使用 `${PWD}` 动态获取绝对路径，不受执行目录限制。

## 配置说明

### 端口修改

编辑 `docker/docker-compose.yml` 中的 `ports` 字段：

```yaml
ports:
  - "8080:7860"   # 将宿主机 8080 端口映射到容器 7860
```

### 模型版本

默认使用 v2.5，可通过修改 `command` 参数切换：

```yaml
command: ["--host", "0.0.0.0", "port", "7860", "--model_dir", "/app/checkpoints", "--version", "2"]
```

### 高级参数

在 `command` 数组中添加其他参数：

| 参数 | 说明 |
|------|------|
| `--fp16` | 使用 FP16 半精度推理 |
| `--deepspeed` | 启用 DeepSpeed 加速 |
| `--cuda_kernel` | 启用 CUDA Kernel 加速 |
| `--accel` | 启用 GPT2 加速引擎 |
| `--torch_compile` | 启用 torch.compile 优化 |

## 直接运行 Docker 镜像

如果不使用 Docker Compose：

```bash
# 构建镜像
docker build -t indextts:latest -f docker/Dockerfile .

# 运行容器
docker run -d \
  --name indextts-webui \
  --gpus all \
  -p 7860:7860 \
  -v $(pwd)/checkpoints:/app/checkpoints \
  -v $(pwd)/outputs:/app/outputs \
  -v $(pwd)/prompts:/app/prompts \
  indextts:latest \
  --host 0.0.0.0 --port 7860 --model_dir /app/checkpoints --version 2.5
```

## 常见问题

### Q: 启动后无法访问 WebUI？

```bash
# 检查容器运行状态
docker compose -f docker/docker-compose.yml ps

# 查看容器日志
docker compose -f docker/docker-compose.yml logs -f indextts
```

### Q: GPU 无法识别？

请确认：
1. NVIDIA Driver 已正确安装：`nvidia-smi`
2. NVIDIA Container Toolkit 已安装并重启了 docker
3. Docker Compose 文件中 `deploy.resources.reservations.devices` 已正确配置

### Q: 模型下载慢？

可手动下载模型并挂载到 `./checkpoints` 目录，或使用镜像源加速。

## 更新部署

```bash
# 拉取最新代码
git pull

# 重新构建并启动
docker compose -f docker/docker-compose.yml up -d --build
```
