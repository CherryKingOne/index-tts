# IndexTTS Docker Image
# 支持 NVIDIA GPU 加速与 WebUI
#
# 构建命令:
#   docker build -t indextts:latest -f docker/Dockerfile .
#
# 运行命令:
#   docker run --gpus all -p 7860:7860 -v $(pwd)/checkpoints:/app/checkpoints -v $(pwd)/outputs:/app/outputs indextts:latest

# ============================================================
# Stage 1: Base image with CUDA + system tools
# ============================================================
FROM nvidia/cuda:12.8.1-runtime-ubuntu22.04 AS base

# Avoid interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install system dependencies (no Python here, uv will manage it)
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    ffmpeg \
    git \
    build-essential \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# ============================================================
# Stage 2: Install latest uv package manager
# ============================================================
FROM base AS production

# Install latest uv (supports extra-build-dependencies and Python version management)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

# Copy all project files (required for hatchling to find the package)
COPY . .

# Install Python dependencies with CUDA-enabled PyTorch
# uv will automatically install the Python version from .python-version
RUN uv sync --extra webui \
    --index-strategy unsafe-best-match \
    --no-cache

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
