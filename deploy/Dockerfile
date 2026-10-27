# ============================================================
# OmniVoice — Railway-compatible multi-stage Dockerfile
# ============================================================
# ------------------------------------------------------------
# Build arguments
# ------------------------------------------------------------
ARG BASE_IMAGE=pytorch/pytorch:2.8.0-cuda12.8-cudnn9-runtime
# ------------------------------------------------------------
# Stage 1: Build frontend
# ------------------------------------------------------------
FROM oven/bun:1.4.2-alpine AS frontend-builder
WORKDIR /app
# Copy package manifests first for better layer caching.
COPY package.json bun.lock ./
COPY electron/package.json ./electron/
COPY config/ ./config/
# Install frontend dependencies.
RUN bun install --frozen-lockfile
# Copy application source.
COPY frontend/ ./frontend/
COPY electron/ ./electron/
# Build the web frontend.
RUN bun run --cwd electron build:web
# ------------------------------------------------------------
# Stage 2: NVIDIA ARM64 base
# Used only when BASE_IMAGE=cuda-arm64-base.
# ------------------------------------------------------------
FROM ubuntu:24.04 AS cuda-arm64-base
ENV DEBIAN_FRONTEND=noninteractive \
    PIP_BREAK_SYSTEM_PACKAGES=1 \
    NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=compute,utility
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-dev \
    python3-pip \
    pkg-config \
    cmake \
    libopus-dev \
    && rm -rf /var/lib/apt/lists/*
RUN python3 -m pip install --no-cache-dir \
    --index-url https://download.pytorch.org/whl/cu129 \
    torch==2.8.0+cu129 \
    torchaudio==2.8.0 \
    torchvision==0.23.0
# ------------------------------------------------------------
# Stage 3: Python runtime
# ------------------------------------------------------------
FROM ${BASE_IMAGE} AS runtime
ARG GPU_FLAVOR=cuda
ARG BUILD_COMMIT=unknown
WORKDIR /app
# ------------------------------------------------------------
# Environment
# ------------------------------------------------------------
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    HF_HOME=/app/omnivoice_data/huggingface \
    PYTHONPATH=/app/backend \
    OMNIVOICE_SERVER_MODE=1 \
    OMNIVOICE_BIND_HOST=0.0.0.0 \
    OMNIVOICE_BUILD_COMMIT=${BUILD_COMMIT} \
    PIP_BREAK_SYSTEM_PACKAGES=1 \
    UV_BREAK_SYSTEM_PACKAGES=1
# Railway provides PORT at runtime. The application entrypoint
# must read PORT or OMNIVOICE_PORT and use it for the web server.
# 3900 is the fallback port.
ENV PORT=3900 \
    OMNIVOICE_PORT=3900
# ------------------------------------------------------------
# System dependencies
# ------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    ffmpeg \
    libsndfile1 \
    curl \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*
# ------------------------------------------------------------
# Install uv
# ------------------------------------------------------------
RUN python3 -m pip install --no-cache-dir uv
# ------------------------------------------------------------
# Copy Python package configuration
# ------------------------------------------------------------
COPY pyproject.toml uv.lock README.md ./
COPY deploy/torch-constraints.txt ./deploy/torch-constraints.txt
COPY deploy/uv-overrides.txt ./deploy/uv-overrides.txt
# ------------------------------------------------------------
# Install Python dependencies
# Preserve the base image's preinstalled PyTorch stack.
# ------------------------------------------------------------
RUN uv pip install \
    --python "$(command -v python3)" \
    --no-cache \
    --constraint deploy/torch-constraints.txt \
    --override deploy/uv-overrides.txt \
    .
# ------------------------------------------------------------
# Validate security overrides
# ------------------------------------------------------------
RUN python3 -c "from importlib.metadata import version; from packaging.requirements import Requirement; \
    reqs = [Requirement(l.strip()) for l in open('deploy/uv-overrides.txt') if l.strip() and not l.lstrip().startswith('#')]; \
    bad = [f'{r.name} {version(r.name)} does not satisfy {r}' for r in reqs if not r.specifier.contains(version(r.name), prereleases=True)]; \
    assert not bad, bad; \
    print('uv overrides applied:', ', '.join(f'{r.name} {version(r.name)}' for r in reqs))"
# ------------------------------------------------------------
# Validate PyTorch GPU installation
# ------------------------------------------------------------
RUN python3 -c "import os, torch, torchaudio, torchvision; \
    flavor = os.environ.get('GPU_FLAVOR', 'cuda'); \
    accel = torch.version.hip if flavor == 'rocm' else torch.version.cuda; \
    print(f'torch={torch.__version__} torchaudio={torchaudio.__version__} torchvision={torchvision.__version__} {flavor}={accel}'); \
    assert accel, f'base image {flavor} torch was clobbered (now {torch.__version__})'; \
    import torchvision.ops; torchvision.ops.nms; \
    print('torchvision C++ ops resolve against this torch')"
# ------------------------------------------------------------
# cuDNN 8 compatibility for CTranslate2
# Required by certain WhisperX / faster-whisper configurations.
# ------------------------------------------------------------
RUN if [ "$GPU_FLAVOR" = "cuda" ] && [ "$(uname -m)" != "aarch64" ]; then \
      target="$(python3 -c "import os, sys; print(os.path.join(sys.prefix, 'lib', 'python%d.%d' % sys.version_info[:2], 'site-packages', 'cudnn8_compat'))")" && \
      uv pip install \
        --python "$(command -v python3)" \
        --no-cache \
        --no-deps \
        --target "$target" \
        nvidia-cudnn-cu12==8.9.7.29 && \
      python3 -c "import os, sys; d = os.path.join(sys.prefix, 'lib', 'python%d.%d' % sys.version_info[:2], 'site-packages', 'cudnn8_compat', 'nvidia', 'cudnn', 'lib'); \
        libs = [f for f in os.listdir(d) if '.so.8' in f]; \
        assert libs, 'cuDNN 8 libraries not found in ' + d; \
        print('cuDNN 8 compatibility libraries:', len(libs))"; \
    fi
# ------------------------------------------------------------
# Copy application backend and configuration
# ------------------------------------------------------------
COPY backend/ ./backend/
COPY omnivoice/ ./omnivoice/
COPY config/ ./config/
COPY alembic.ini ./
# ------------------------------------------------------------
# Copy compiled frontend
# ------------------------------------------------------------
COPY --from=frontend-builder /app/frontend/dist ./frontend/dist
# ------------------------------------------------------------
# Persistent application data
# Railway should mount a persistent volume at /app/omnivoice_data
# ------------------------------------------------------------
RUN mkdir -p /app/omnivoice_data
VOLUME ["/app/omnivoice_data"]
# ------------------------------------------------------------
# Networking
# ------------------------------------------------------------
EXPOSE 3900
# ------------------------------------------------------------
# Health check
# ------------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=5s --start-period=120s --retries=5 \
    CMD ["python3", "backend/docker_entrypoint.py", "--health-check"]
# ------------------------------------------------------------
# Start OmniVoice
# ------------------------------------------------------------
ENTRYPOINT ["python3", "backend/docker_entrypoint.py"]
