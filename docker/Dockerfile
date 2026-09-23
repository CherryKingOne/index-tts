# IndexTTS Docker Image
# 支持 NVIDIA GPU 加速与 WebUI
#
# 构建命令:
#   docker build -t indextts:latest -f docker/Dockerfile .
#
# 运行命令:
#   docker run --gpus all -p 7860:7860 -v $(pwd)/checkpoints:/app/checkpoints -v $(pwd)/outputs:/app/outputs indextts:latest

# ============================================================
# Stage 1: Base image with CUDA + Python
# ============================================================
FROM nvidia/cuda:12.8.1-runtime-ubuntu22.04 AS base

# Avoid interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.10 \
    python3.10-venv \
    python3-pip \
    curl \
    ffmpeg \
    git \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Set Python 3.10 as default
RUN update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1 \
    && update-alternatives --install /usr/bin/python python /usr/bin/python3.10 1

# ============================================================
# Stage 2: Install uv package manager
# ============================================================
FROM base AS uv-installer

# Install uv (fast Python package manager)
COPY --from=ghcr.io/astral-sh/uv:0.5.0 /uv /usr/local/bin/uv

# ============================================================
# Stage 3: Production image
# ============================================================
FROM uv-installer AS production

WORKDIR /app

# Copy project configuration files first (for better layer caching)
COPY pyproject.toml uv.lock ./

# Install Python dependencies with CUDA-enabled PyTorch
# Use --extra webui to include Gradio for WebUI support
RUN uv sync --extra webui \
    --index-strategyunsafe-best-match \
    --no-cache

# Copy the rest of the application code
COPY . .

# Create directories for models and outputs
RUN mkdir -p /app/checkpoints /app/outputs /app/prompts

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV PATH="/app/.venv/bin:$PATH"

# Expose the WebUI port
EXPOSE 7860

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:7860/ || exit 1

# Default entry point: start WebUI
ENTRYPOINT ["uv", "run", "--extra", "webui", "python", "webui.py"]

# Default arguments (can be overridden)
CMD ["--host", "0.0.0.0", "--port", "7860", "--model_dir", "/app/checkpoints", "--version", "2.5"]
