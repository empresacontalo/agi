# =============================================================================
# Dockerfile — AI Dev Workspace (Single Container)
# =============================================================================
# Container único com TODOS os apps e CLIs:
#   - OmniRoute (port 4000)
#   - Hermes Workspace (port 3000)
#   - Aion UI (port 3005)
#   - Claude Code CLI
#   - Antigravity CLI
#   - OpenCode CLI
#   - CommandCode CLI
#
# Usa supervisord para gerenciar múltiplos processos.
# Aion UI e Hermes autodetectam os CLIs no mesmo container.
# =============================================================================

FROM node:22-slim

LABEL maintainer="Bruno <brunomiguel@outsourc-e.com>"
LABEL description="AI Dev Workspace — all apps + CLI tools in one container"

# Avoid prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# ---------------------------------------------------------------------------
# System Dependencies
# ---------------------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    git \
    curl \
    wget \
    ca-certificates \
    gnupg \
    build-essential \
    python3 \
    python3-pip \
    jq \
    unzip \
    openssh-client \
    supervisor \
    && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------------------------
# CLI Tools Installation (globally available)
# ---------------------------------------------------------------------------

# Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code@latest

# Antigravity CLI (via npm)
RUN npm install -g antigravity@latest

# OpenCode CLI
RUN npm install -g opencode-ai@latest

# CommandCode CLI
RUN npm install -g command-code@latest

# ---------------------------------------------------------------------------
# App 1: OmniRoute — AI Gateway (port 4000)
# ---------------------------------------------------------------------------
ARG OMNIROUTE_VERSION=main

RUN git clone --depth 1 --branch ${OMNIROUTE_VERSION} \
    https://github.com/diegosouzapw/OmniRoute.git /opt/omniroute

WORKDIR /opt/omniroute
RUN npm install --production 2>/dev/null || npm install
RUN npm run build

# ---------------------------------------------------------------------------
# App 2: Hermes Workspace (port 3000)
# ---------------------------------------------------------------------------
ARG HERMES_VERSION=main

RUN git clone --depth 1 --branch ${HERMES_VERSION} \
    https://github.com/outsourc-e/hermes-workspace.git /opt/hermes

RUN npm install -g pnpm
WORKDIR /opt/hermes
RUN pnpm install
RUN pnpm build

# ---------------------------------------------------------------------------
# App 3: Aion UI (port 3005)
# ---------------------------------------------------------------------------
ARG AIONUI_VERSION=main

RUN git clone --depth 1 --branch ${AIONUI_VERSION} \
    https://github.com/iofficeai/aionui.git /opt/aionui

# Aion UI uses Bun workspaces and the "workspace:" protocol
RUN npm install -g bun

WORKDIR /opt/aionui
# Remove conflicting direct dependency override to prevent EOVERRIDE error in npm install
RUN jq 'del(.overrides["@codemirror/language"])' package.json > package.json.tmp && mv package.json.tmp package.json
RUN bun install
RUN node scripts/prepareAioncore.js
RUN bun run package

# ---------------------------------------------------------------------------
# Supervisord Configuration
# ---------------------------------------------------------------------------
RUN mkdir -p /var/log/supervisor

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ---------------------------------------------------------------------------
# Volumes & Workspace
# ---------------------------------------------------------------------------
RUN mkdir -p /workspace /data/omniroute /data/hermes /data/aionui

WORKDIR /workspace

# ---------------------------------------------------------------------------
# Expose Ports
# ---------------------------------------------------------------------------
EXPOSE 4000 3000 3005

# ---------------------------------------------------------------------------
# Health Check
# ---------------------------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:4000/health 2>/dev/null || \
        curl -f http://localhost:3000/ 2>/dev/null || \
        curl -f http://localhost:3005/ 2>/dev/null || exit 1

# ---------------------------------------------------------------------------
# Entrypoint
# ---------------------------------------------------------------------------
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
