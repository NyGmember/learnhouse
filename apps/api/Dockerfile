# ── Build Stage ─────────────────────────────────────────────────────────────
FROM python:3.12-slim-bookworm AS builder

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    build-essential \
    libpq-dev \
    libffi-dev \
    libssl-dev \
    pkg-config \
    ca-certificates \
    && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
    && rm -rf /var/lib/apt/lists/*

ENV PATH="/root/.cargo/bin:${PATH}"

COPY --from=ghcr.io/astral-sh/uv:0.10.7 /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

# Dependencies first, in their own layer for efficient docker caching
COPY pyproject.toml uv.lock* ./
RUN uv sync --no-install-project

# Copy the project into the image
COPY . /app

# Remove Enterprise Edition folder for public builds
ARG LEARNHOUSE_PUBLIC=false
RUN if [ "$LEARNHOUSE_PUBLIC" = "true" ]; then rm -rf /app/ee; fi

# Sync again so the lockfile is verified against the copied project
RUN uv sync

# ── Production Runner Stage ──────────────────────────────────────────────────
FROM python:3.12-slim-bookworm AS runner

# Install runtime dependencies: curl/netcat for healthchecks, ffmpeg for video streaming, libpq5 for PostgreSQL
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    netcat-openbsd \
    ffmpeg \
    libpq5 \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY --from=ghcr.io/astral-sh/uv:0.10.7 /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONIOENCODING=utf-8 \
    PORT=8000 \
    LEARNHOUSE_PORT=8000 \
    HOSTNAME=0.0.0.0

WORKDIR /app

# Copy built application and virtual environment from builder
COPY --from=builder /app /app

# Ensure entrypoint script is executable
RUN chmod +x /app/docker-entrypoint.sh

# Expose default API port
EXPOSE 8000

# Health check endpoint
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8000/api/v1/health || exit 1

# Use entrypoint script
ENTRYPOINT ["./docker-entrypoint.sh"]