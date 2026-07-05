# =============================================================================
# Dockerfile — AI Dev Workspace (Single Container)
# =============================================================================
# Base: nousresearch/hermes-agent:latest
#   → Provides: hermes-agent (Python), gateway (:8642), dashboard (:9119)
#
# Added on top:
#   - Node.js 22 + npm
#   - Bun runtime
#   - pnpm
#   - OmniRoute AI Gateway     (port 4000)
#   - Hermes Workspace UI      (port 3000)
#   - Aion UI                  (port 3005)
#   - Claude Code, Antigravity, OpenCode, CommandCode CLIs
#
# Process manager: supervisord (OmniRoute + Hermes Workspace + Aion UI)
# hermes gateway + dashboard are started by entrypoint.sh before supervisord.
# =============================================================================

FROM nousresearch/hermes-agent:latest

LABEL maintainer="Bruno <brunomiguel@outsourc-e.com>"
LABEL description="AI Dev Workspace — hermes-agent + all UI apps + CLI tools in one container"

# Avoid prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# ---------------------------------------------------------------------------
# System Dependencies (additional, not in hermes-agent base)
# ---------------------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    wget \
    ca-certificates \
    gnupg \
    build-essential \
    jq \
    unzip \
    openssh-client \
    supervisor \
    libreoffice \
    poppler-utils \
    tesseract-ocr \
    tesseract-ocr-spa \
    tesseract-ocr-por \
    && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------------------------
# Node.js 22 (via NodeSource — hermes-agent base is Debian-based)
# ---------------------------------------------------------------------------
RUN curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# ---------------------------------------------------------------------------
# Bun runtime
# ---------------------------------------------------------------------------
RUN npm install -g bun

# ---------------------------------------------------------------------------
# pnpm (used by Hermes Workspace)
# ---------------------------------------------------------------------------
RUN npm install -g pnpm

# Install python packages globally
RUN pip install agent-reach || true

# ---------------------------------------------------------------------------
# CLI Tools (globally available in the workspace)
# ---------------------------------------------------------------------------

# Install global CLI tools
RUN npm install -g \
    @anthropic-ai/claude-code@latest \
    antigravity@latest \
    opencode-ai@latest \
    command-code@latest \
    @openai/codex \
    @google/gemini-cli \
    officecli \
    postiz \
    agent-reach \
    zcode \
    cursor-cli \
    remotion \
    @remotion/cli \
    hyperframes \
    || true

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
    https://github.com/outsourc-e/hermes-workspace.git /opt/hermes-workspace

WORKDIR /opt/hermes-workspace
RUN pnpm install
RUN pnpm build

# ---------------------------------------------------------------------------
# App 3: Aion UI (port 3005)
# ---------------------------------------------------------------------------
ARG AIONUI_VERSION=main

RUN git clone --depth 1 --branch ${AIONUI_VERSION} \
    https://github.com/iofficeai/aionui.git /opt/aionui

WORKDIR /opt/aionui
# Remove conflicting direct dependency override to prevent EOVERRIDE error
RUN jq 'del(.overrides["@codemirror/language"])' package.json > package.json.tmp \
    && mv package.json.tmp package.json
RUN bun install
RUN node scripts/prepareAioncore.js
RUN bun run package

# ---------------------------------------------------------------------------
# Supervisord Configuration
# ---------------------------------------------------------------------------
RUN mkdir -p /var/log/supervisor

COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# ---------------------------------------------------------------------------
# Entrypoint (initializes hermes-agent config, then starts supervisord)
# ---------------------------------------------------------------------------
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# ---------------------------------------------------------------------------
# Volumes & Workspace
# ---------------------------------------------------------------------------
RUN mkdir -p /workspace /data/omniroute /data/hermes-workspace /data/aionui \
    && chown -R hermes:hermes /workspace /data /var/log/supervisor


WORKDIR /workspace

# Disable hermes-agent built-in s6 services — we manage them via supervisord.
#
# s6-overlay v3 compiles service definitions into a binary bundle at BASE IMAGE
# build time. Modifying source files in s6-rc.d has no effect unless the bundle
# is recompiled. Strategy:
#   1. Remove services from user/contents.d so s6-rc excludes them from the bundle
#   2. Recompile the s6-rc database so the change is active at runtime
RUN \
    # Step 1: Remove from user bundle manifest
    rm -f /etc/s6-overlay/s6-rc.d/user/contents.d/main-hermes \
          /etc/s6-overlay/s6-rc.d/user/contents.d/dashboard 2>/dev/null || true; \
    \
    # Step 2: Find the compiled database and recompile
    COMPILED=$(find / -maxdepth 8 -path '*/s6-overlay/compiled' -type d 2>/dev/null | head -1); \
    if [ -n "$COMPILED" ]; then \
      echo "[s6-disable] Recompiling bundle at: $COMPILED"; \
      s6-rc-compile "${COMPILED}.tmp" /etc/s6-overlay/s6-rc.d \
        && rm -rf "$COMPILED" \
        && mv "${COMPILED}.tmp" "$COMPILED" \
        && echo "[s6-disable] Bundle recompiled — main-hermes and dashboard excluded"; \
    else \
      echo "[s6-disable] WARNING: compiled bundle not found, services may still start"; \
    fi || true



# ---------------------------------------------------------------------------
# Expose Ports
# ---------------------------------------------------------------------------
# 8642 — hermes-agent gateway (OpenAI-compatible API)
# 9119 — hermes-agent dashboard (internal, optional exposure)
# 4000 — OmniRoute AI Gateway
# 3000 — Hermes Workspace UI
# 3005 — Aion UI
EXPOSE 8642 9119 4000 3000 3005

# ---------------------------------------------------------------------------
# Health Check
# ---------------------------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=10s --start-period=90s --retries=5 \
    CMD curl -f http://localhost:8642/health 2>/dev/null && \
        curl -f http://localhost:3000/ 2>/dev/null || exit 1

# ---------------------------------------------------------------------------
# Entrypoint
# ---------------------------------------------------------------------------
CMD ["/entrypoint.sh"]
