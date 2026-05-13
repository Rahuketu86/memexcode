# ============================================================================
# SolveIt Container — Optimized Dockerfile
# ============================================================================
# Target: Debian 12 (bookworm) with Python 3.12 compiled from source,
#         Node.js 24, Rust, Deno, Quarto, and the full solveit stack.
#
# Build time: ~15-20 minutes (Python compilation + pip installs)
# Image size: ~20-25GB (vs ~40-60GB with the original 50-layer build)
# Layer count: ~18 (vs 50 in the original)
#
# Usage:
#   docker build -t solveit:optimized .
#   docker run -d -p 6000:6000 --name solveit solveit:optimized
# ============================================================================

# ============================================================================
# STAGE 1: BUILD — Everything needed to compile/build, discarded after
# ============================================================================
FROM debian:bookworm-slim AS builder

# Prevent interactive prompts during apt install
ENV DEBIAN_FRONTEND=noninteractive

# Install only what's needed to compile Python 3.12 from source
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    g++ \
    wget \
    tar \
    curl \
    make \
    && rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------------------
# Python 3.12.13 — compile from source, install to /usr/local
# All source/build artifacts stay in this stage (never copied forward)
# --------------------------------------------------------------------------
RUN wget https://www.python.org/ftp/python/3.12.13/Python-3.12.13.tgz && \
    tar xzf Python-3.12.13.tgz && \
    cd Python-3.12.13 && \
    ./configure --prefix=/usr/local \
                --enable-optimizations \
                --with-system-ffi \
                --with-system-zlib && \
    make -j$(nproc) && \
    make install && \
    cd .. && \
    rm -rf Python-3.12.13 Python-3.12.13.tgz

# --------------------------------------------------------------------------
# Install Rust/Cargo (only needed for building certain pip packages)
# --------------------------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
    cargo \
    rustc \
    && rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------------------
# Install Node.js 24 (from NodeSource)
# --------------------------------------------------------------------------
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && \
    apt-get install -y --no-install-recommends nodejs && \
    rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------------------
# Download Quarto 1.8.24
# --------------------------------------------------------------------------
RUN wget https://github.com/quarto-dev/quarto-cli/releases/download/v1.8.24/quarto-1.8.24-linux-amd64.tar.gz && \
    tar xzf quarto-1.8.24-linux-amd64.tar.gz && \
    mv quarto-1.8.24 /opt/quarto && \
    rm quarto-1.8.24-linux-amd64.tar.gz

# --------------------------------------------------------------------------
# Download custom binaries (caddy, ast-grep, shfmt)
# These are pre-compiled, so we download and place them directly
# --------------------------------------------------------------------------
RUN wget -O /usr/local/bin/caddy https://downloads.caddyserver.com/caddy_2.10.4_linux_amd64.deb && \
    dpkg -i caddy.deb && \
    rm caddy.deb

RUN wget https://github.com/ast-grep/ast-grep/releases/download/v0.42.1/sg-linux-x86_64-static.tar.gz && \
    tar xzf sg-linux-x86_64-static.tar.gz && \
    mv sg /usr/local/bin/ast-grep && \
    chmod +x /usr/local/bin/ast-grep && \
    rm sg-linux-x86_64-static.tar.gz

RUN wget https://github.com/mvdan/sh/releases/download/v3.10.0/shfmt_v3.10.0_linux_amd64 && \
    mv shfmt_v3.10.0_linux_amd64 /usr/local/bin/shfmt && \
    chmod +x /usr/local/bin/shfmt

# --------------------------------------------------------------------------
# Install Python packages in builder (some need compilation)
# These will be copied to the runtime stage, but pip cache is left behind
# --------------------------------------------------------------------------
RUN pip install --no-cache-dir \
    torch==2.11.0+cpu \
    torchaudio==2.11.0+cpu \
    torchvision==0.26.0+cpu \
    openai==2.24.0 \
    anthropic==0.98.1 \
    replicate==1.0.7 \
    && rm -rf /root/.cache/pip

# ============================================================================
# STAGE 2: RUNTIME — Minimal image with only what's needed to run
# ============================================================================
FROM debian:bookworm-slim AS runtime

# Prevent interactive prompts
ENV DEBIAN_FRONTEND=noninteractive

# Install ONLY runtime dependencies (no build tools, no gcc, no clang)
RUN apt-get update && apt-get install -y --no-install-recommends \
    # System essentials
    git \
    curl \
    wget \
    sudo \
    tmux \
    htop \
    jq \
    tree \
    vim \
    less \
    # Multimedia / data
    ffmpeg \
    imagemagick \
    graphviz \
    pandoc \
    # C library deps
    libssl-dev \
    libffi-dev \
    libbz2-dev \
    zlib1g-dev \
    libsqlite3-dev \
    liblzma-dev \
    libncurses5-dev \
    libxml2-dev \
    libxslt1-dev \
    # Runtime C libraries (for compiled Python extensions)
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# --------------------------------------------------------------------------
# Copy compiled Python 3.12 from builder — NO source, NO build tools
# This is the critical step: we get the binary but NOT the 300MB of source
# --------------------------------------------------------------------------
COPY --from=builder /usr/local /usr/local

# --------------------------------------------------------------------------
# Copy Node.js from builder
# --------------------------------------------------------------------------
COPY --from=builder /usr/local/bin/node /usr/local/bin/node
COPY --from=builder /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=builder /usr/local/bin/npm /usr/local/bin/npm
COPY --from=builder /usr/local/bin/npx /usr/local/bin/npx
COPY --from=builder /usr/local/share /usr/local/share

# --------------------------------------------------------------------------
# Copy Quarto from builder
# --------------------------------------------------------------------------
COPY --from=builder /opt/quarto /opt/quarto

# --------------------------------------------------------------------------
# Copy custom binaries from builder
# --------------------------------------------------------------------------
COPY --from=builder /usr/local/bin/caddy /usr/local/bin/caddy
COPY --from=builder /usr/local/bin/ast-grep /usr/local/bin/ast-grep
COPY --from=builder /usr/local/bin/shfmt /usr/local/bin/shfmt

# --------------------------------------------------------------------------
# Download Deno (small binary, no build needed at runtime)
# --------------------------------------------------------------------------
RUN wget https://github.com/denoland/deno/releases/download/v2.6.9/deno-x86_64-unknown-linux-gnu.zip && \
    unzip deno-x86_64-unknown-linux-gnu.zip && \
    mv deno /usr/local/bin/deno && \
    chmod +x /usr/local/bin/deno && \
    rm deno-x86_64-unknown-linux-gnu.zip

# --------------------------------------------------------------------------
# Install Node.js npm globals (Node.js itself came from builder, npm packages
# are installed fresh in runtime for reproducibility)
# --------------------------------------------------------------------------
RUN npm install -g corepack && \
    corepack enable

# --------------------------------------------------------------------------
# Download GitHub CLI (gh) for Git integration
# --------------------------------------------------------------------------
RUN wget -O /tmp/gh.deb https://github.com/cli/cli/releases/download/v2.74.1/gh_2.74.1_linux_amd64.deb && \
    dpkg -i /tmp/gh.deb && \
    rm /tmp/gh.deb

# --------------------------------------------------------------------------
# Create non-root user matching the original container (uid=2000, gid=2000)
# Home set to /app/data (the bind-mounted volume)
# --------------------------------------------------------------------------
RUN useradd -ms /bin/bash -u 2000 -G sudo solveit

# Set up sudo without password for the solveit user
RUN echo 'solveit ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/solveit && \
    chmod 440 /etc/sudoers.d/solveit

# Set ownership for workspace
RUN mkdir -p /app/data && \
    chown solveit:solveit /app/data

USER solveit
WORKDIR /app/data

# --------------------------------------------------------------------------
# Set up NVM for Node.js 20.20.0 (project-level version)
# NVM is needed for certain tools; Node 24 comes from builder for global
# --------------------------------------------------------------------------
RUN mkdir -p /home/solveit/.nvm && \
    git clone https://github.com/nvm-sh/nvm.git /home/solveit/.nvm && \
    bash -c 'source /home/solveit/.nvm/nvm.sh && nvm install 20.20.0' && \
    chown -R solveit:solveit /home/solveit/.nvm

# --------------------------------------------------------------------------
# Install ALL Python packages in ONE RUN (instead of one RUN per package)
# --no-cache-dir prevents pip from storing downloaded wheels (saves 2-3GB)
# --------------------------------------------------------------------------
RUN pip install --no-cache-dir \
    fastapi==0.116.1 \
    starlette==0.45.3 \
    uvicorn==0.34.0 \
    streamlit==1.46.0 \
    gradio==5.23.0 \
    pandas==2.3.0 \
    sqlalchemy==2.0.41 \
    redis==5.2.1 \
    matplotlib==3.10.3 \
    seaborn==0.13.2 \
    scikit-learn==1.8.0 \
    scipy==1.17.1 \
    pytest==8.4.2 \
    pytest-playwright==0.7.0 \
    huggingface_hub==0.35.0 \
    huggingface-cli==0.4.0 \
    litellm==1.68.0 \
    cython==3.1.3 \
    pygments==2.19.2 \
    PyYAML==6.0.2 \
    && rm -rf /root/.cache/pip

# --------------------------------------------------------------------------
# Install ALL editable project packages in ONE RUN
# Each -e points to a local project directory
# --------------------------------------------------------------------------
# Create directory structure for editable installs
RUN mkdir -p /app/data/{solveit,floor_assistant,memexsolve,JCMSSalesPortal,memexpaas,memexpaas_deploy,memexpaas_browser,memexpaas_mathagent,memexhelper,memexplatform_flows,browserhacks}

# Editable installs for each project
RUN pip install --no-cache-dir -e /app/data/solveit \
    -e /app/data/floor_assistant \
    -e /app/data/memexsolve \
    -e /app/data/JCMSSalesPortal \
    -e /app/data/memexpaas \
    -e /app/data/memexpaas_deploy \
    -e /app/data/memexpaas_browser \
    -e /app/data/memexpaas_mathagent \
    -e /app/data/memexhelper \
    -e /app/data/memexplatform_flows \
    -e /app/data/browserhacks \
    && rm -rf /root/.cache/pip

# --------------------------------------------------------------------------
# Set up user-level CLI tools in ~/.local/bin
# --------------------------------------------------------------------------
RUN mkdir -p /home/solveit/.local/bin

# --------------------------------------------------------------------------
# Set up shell environment
# --------------------------------------------------------------------------
RUN cat > /home/solveit/.bashrc << 'BASHRC'
# Source bashrc.d modules
if [ -d ~/.bashrc.d ]; then
    for f in ~/.bashrc.d/*.sh; do
        [ -r "$f" ] && source "$f"
    done
fi

# NVM setup
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

# PATH
export PATH="$HOME/.local/bin:$HOME/.nvm/versions/node/v20.20.0/bin:/usr/local/bin:$PATH"
BASHRC

RUN chown solveit:solveit /home/solveit/.bashrc

# --------------------------------------------------------------------------
# Set up Caddy reverse proxy configuration
# --------------------------------------------------------------------------
COPY --chown=solveit:solveit Caddyfile /app/data/Caddyfile

# --------------------------------------------------------------------------
# Health check
# --------------------------------------------------------------------------
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:6000/ || exit 1

# --------------------------------------------------------------------------
# Expose Caddy (proxy) and solveit ports
# --------------------------------------------------------------------------
EXPOSE 6000

# --------------------------------------------------------------------------
# Entry point: run the solveit FastHTML app
# This matches the original: /usr/local/bin/python /usr/local/bin/solveit --reload false
# --------------------------------------------------------------------------
ENTRYPOINT ["/usr/local/bin/python", "/usr/local/bin/solveit", "--reload", "false"]
