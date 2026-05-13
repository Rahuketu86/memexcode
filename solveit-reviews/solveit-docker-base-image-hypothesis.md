# SolveIt Container: File/Dotfile Organization & Docker Base Image Hypothesis

**Date:** 2026-05-13  
**Container:** `aabb4c8a0f0a` (Docker/containerd)  
**Review Scope:** File/dotfile organization, software stack, and base image hypothesis

---

## 1. Host & Infrastructure

| Property | Value |
|---|---|
| **CPU** | AMD EPYC 9554P 64-Core (48 physical cores, Zen 5/Turin) |
| **RAM** | ~1.05 TB (1,056,438,656 kB) |
| **GPU** | None (no nvidia-smi, no CUDA) |
| **Virtualization** | KVM (full virtualization) |
| **Disk** | 2.5 TB HDD (`/dev/sda1`, ext4, rotational=1) |
| **Disk Usage** | ~64% used, 8% inodes consumed |
| **Container Runtime** | containerd (overlayfs snapshotter) |
| **Container Mount** | Bind mount from host `/data/volumes/61jstpii` |
| **Host OS** | Ubuntu (kernel `6.8.0-106-generic`) |
| **DNS** | `genesishoping.com` |
| **Network** | Cloudflare Tunnel + Tailscale |

---

## 2. Docker Base Image Hypothesis

### Strong Hypothesis: **Custom Debian 12 (bookworm) image with manually compiled Python 3.12**

#### Evidence Summary:

| Evidence | Details |
|---|---|
| **`/etc/os-release`** | Debian GNU/Linux 12 (bookworm), VERSION_ID=12 |
| **`dpkg` packages** | 703 packages, all Debian bookworm versions |
| **glibc** | GLIBC 2.36-9+deb12u13 (Debian's glibc, not musl/Alpine) |
| **Python** | Python 3.12.13 at `/usr/local/bin/python3.12` — compiled from source (not Debian apt) |
| **System Python** | Python 3.11.2 (Debian default) still available at `/usr/bin/python3.11` |
| **Python build** | GCC 12.2.0 compiled, installed to `/usr/local` with full header files |
| **Package manager** | `apt` with `debian.sources` + `nodesource.list` |
| **Overlay layers** | 70+ layers in overlayfs (heavily layered build) |
| **No distro markers** | No Ubuntu/Alpine/Amazon package repos; purely Debian |

### Why NOT the standard `python:3.12-slim` or `python:3.12-bookworm`?

1. **Node.js 24.15.0** installed from NodeSource (`deb.nodesource.com/node_24.x`), not built into the image
2. **Python 3.12.13** (latest patch) — the official `python:3.12-bookworm` image ships 3.12.x but likely an older patch
3. **Rust/Cargo** (0.66.0) and **clang-14** installed from apt
4. **Quarto 1.8.24** at `/opt/quarto/`
5. **Deno** installed manually to `~/.deno/bin/deno`
6. **Node 20.20.0** via NVM (plus global Node 24 from NodeSource)
7. **70+ overlay layers** suggests a custom multi-stage build with many `RUN` steps
8. **No `/root/.bashrc` or `/root/.profile`** — root user was never configured
9. **Python installed to `/usr/local`** with source compilation, not system package

### Likely Build Pattern:

```dockerfile
FROM debian:bookworm-slim

# System packages
RUN apt-get update && apt-get install -y \
    build-essential gcc g++ clang make \
    git curl wget sudo tmux htop tree jq \
    ffmpeg imagemagick graphviz pandoc \
    libssl-dev libffi-dev libxml2-dev libxslt1-dev \
    zlib1g-dev libbz2-dev libsqlite3-dev \
    python3.11  # system Python for compatibility

# Custom Python 3.12 compiled from source
RUN wget https://www.python.org/ftp/python/3.12.13/Python-3.12.13.tgz \
    && tar xzf Python-3.12.13.tgz \
    && cd Python-3.12.13 && ./configure --enable-optimizations \
    && make install

# Node.js 24 from NodeSource
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
RUN apt-get install -y nodejs

# Rust/Cargo (from apt)
RUN apt-get install -y cargo rustc clang-14

# Quarto
# (downloaded and extracted to /opt/quarto)

# Create non-root user
RUN useradd -ms /bin/bash -u 2000 solveit
USER solveit
WORKDIR /app/data

# User-level tools
RUN mkdir -p ~/.nvm ~/.local/bin
# ... nvm install, deno download, pip install, etc.
```

### Supporting Indicators:

- **No gosu** in the actual image (gosu was found in dpkg but not actually used as entrypoint)
- **PID 1** runs: `/usr/local/bin/python /usr/local/bin/solveit --reload false` — custom Python script
- **`/.dockerenv`** present — standard Docker container
- **No systemd** — minimal base image
- **`/root` is empty** — user never configured as root
- **`SUDO_USER=root`** in env — the container likely starts as root and then switches (or the solveit command was run via sudo)
- **Caddy** v2 (47MB binary) installed at `/usr/local/bin/caddy` — reverse proxy
- **ast-grep** v0.42.1 (47MB binary) installed at `/usr/local/bin/ast-grep` — code analysis
- **gh** CLI (39MB binary) installed at `/usr/local/bin/gh` — GitHub CLI

---

## 3. File/Dotfile Organization

### Directory Structure

```
/app/data/                          # $HOME for user solveit
├── .bashrc                         # Minimal: sources ~/.bashrc.d/*.sh
├── .bashrc.d/
│   ├── claude-aliases.sh           # aclaude, kclaude aliases
│   └── vllm.sh                     # VLLM_API_KEY, VLLM_INTERNAL/EXTERNAL_URL
├── .claude/                        # Claude Code config & state
│   ├── .claude.json                # User config (43 startups, 2.1.88 last release)
│   ├── .credentials.json           # Anthropic OAuth tokens
│   ├── history.jsonl               # 170KB history
│   ├── backups/, cache/, sessions/, etc.
│   ├── projects/                   # Per-project Claude settings
│   └── settings.json               # Default model: "opus[1m]"
├── .claude-aman/                   # Second Claude profile (Aman?)
├── .claude-kaushal/                # Third Claude profile (Kaushal?)
├── .config/
│   ├── caddy/                      # Caddy reverse proxy config
│   ├── git/                        # Git config
│   ├── matplotlib/                 # Matplotlib config
│   ├── mlflow/                     # MLflow config
│   ├── safecmd/                    # safecmd config
│   └── .wrangler/                  # Cloudflare Wrangler
├── .cache/
│   ├── claude/                     # Claude Code cache
│   ├── claude-cli-nodejs/          # Node.js Claude CLI cache (19 subdirs)
│   ├── deno/                       # Deno runtime cache
│   ├── quarto/                     # Quarto cache
│   ├── typst/                      # Typst cache
│   ├── pip/                        # pip cache
│   └── gh/                         # GitHub CLI cache
├── .local/
│   ├── bin/                        # 70+ user-installed CLI tools
│   │   ├── claude, mcp, mcp-dialoghelper
│   │   ├── floor-help, floor-run, jcms-help, jcms-serve
│   │   ├── memexhelp, memexroutes, memexrun
│   │   ├── tiny-agents, ipymini
│   │   ├── playwright, ngrok, gradio, streamlit
│   │   ├── litellm, litellm-proxy
│   │   └── ... (cython, cygdb, alembic, etc.)
│   ├── share/pyskills/             # Python skill definitions
│   └── state/                      # XDG state dir
├── .pi/                            # Pi coding agent
│   ├── auth.json                   # Auth tokens
│   ├── bin/fd                      # File finder binary
│   ├── extensions/                 # Pi extensions (empty)
│   ├── models.json                 # Model configuration
│   ├── settings.json               # Pi settings
│   └── sessions/                   # Session history (JSONL files)
├── .nvm/                           # NVM for Node.js v20.20.0
├── .deno/                          # Deno runtime
├── .cargo/                         # Rust package cache
├── solveit-reviews/                # This review output
├── Caddyfile                       # Caddy reverse proxy config (:6000)
├── floor.db                        # Floor assistant SQLite DB
├── memexpaas.db                    # Memex PaaS SQLite DB
├── README.test
├── claude_refresh.py
├── Remote_Kernel_v3.py
├── .mcp.json                       # MCP server config
├── .sesskey                        # Session UUID
├── .devops_token                   # Azure DevOps PAT
├── *.ipynb                         # Jupyter notebooks (test/demo files)
│
├── dialogs/                        # Conversation/project directories
│   └── b10Xp6TI7aVVLp8GyipIkw/
│       ├── floor_assistant/         # Floor assistant project (nbdev)
│       ├── memexcode/               # Memex code repo (nbdev)
│       ├── memexpaas/               # Memex PaaS plugin spec (nbdev)
│       ├── memexpaas_browser/       # Browser agent project
│       ├── memexpaas_deploy/        # Deployment orchestrator
│       ├── memexpaas_claude/        # Claude integration
│       ├── memexpass_mathagent/     # Math agent project (nbdev)
│       ├── memexcode-nbs/           # Notebooks
│       └── memexpaas_deploy-nbs-Testing/
│
├── JCMSSalesPortal/                # JCMS Sales Portal (nbdev, FastHTML)
├── memexsolve/                     # Memex solve system (nbdev)
├── memexhelper/                    # Memex helper tools
├── memexplatform_flows/            # Platform flows DB
├── browserhacks/                   # Browser automation hacks
├── voice-stack/                    # Voice stack (STT/TTS Dockerfiles)
│   ├── stt/Dockerfile
│   ├── tts/Dockerfile
│   └── test.sh
├── _solveit_static_site/           # (empty) Caddy static site root
└── _quarto/                        # (empty) Quarto project dir
```

### Key Dotfile Analysis

#### `.bashrc` — Minimalist Design
```bash
if [ -d ~/.bashrc.d ]; then
    for f in ~/.bashrc.d/*.sh; do
        [ -r "$f" ] && source "$f"
    done
fi
```
- Purely modular — loads everything from `~/.bashrc.d/`
- No aliases, no PS1, no PATH manipulation in `.bashrc` itself
- PATH is controlled by the shell init (likely set by the container)

#### `.bashrc.d/` — Environment Modules
| File | Contents |
|---|---|
| `claude-aliases.sh` | `aclude`/`kclaude` for multi-profile Claude, `deploy-api` wrapper |
| `vllm.sh` | VLLM_API_KEY + internal/external API URLs for local LLM serving |

#### `.claude/` — Claude Code Configuration
- **Default model:** `opus[1m]` (Claude Opus with 1M context)
- **MCP Server:** `dialoghelper` (custom MCP for project dialog management)
- **Profile management:** Three separate profiles — `.claude/`, `.claude-aman/`, `.claude-kaushal/`
- **Enterprise org:** JPL (Rahul Saraf, rahul1.saraf@ril.com)
- **Subscription:** Enterprise contract billing (stripe_subscription_contracted)
- **Auto-install:** 45+ plugins from Claude Code official marketplace
- **Feature flags:** 100+ internal feature flags (tengu_*)

#### `.pi/` — Pi Coding Agent
- Session-based architecture with JSONL session logs
- Session history organized by project paths
- Custom extensions directory (currently empty)

#### `.local/bin/` — User Toolchain (70+ tools)
Grouped by purpose:
- **Claude/AI:** `claude`, `tiny-agents`, `smolagent`, `litellm`, `litellm-proxy`
- **MCP:** `mcp`, `mcp-dialoghelper`, `fastmcp`, `ipyai-mcp-bridge`
- **Project runnners:** `memexrun`, `memexroutes`, `memexhelp`, `floor-run`, `jcms-serve`, `mpserve`
- **Dev:** `cython`, `cygdb`, `cythonize`, `alembic`, `mako-render`
- **Web:** `playwright`, `ngrok`, `gradio`, `streamlit`, `gunicorn`
- **API:** `openai`, `anthropic`, `huggingface-cli`, `hf`
- **AI/ML:** `ipyku-launcher`, `ipymini`

### Editable Project Installs (pip)

| Package | Path |
|---|---|
| `solveit` (0.0.105) | `/root/solveit/solveit` (editable, but permission-denied for user) |
| `floor_assistant` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/floor_assistant` |
| `JCMSSalesPortal` | `/app/data/JCMSSalesPortal` |
| `memexcode` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/memexcode` |
| `memexhelper` | `/app/data/memexhelper` |
| `memexpaas` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/memexpaas` |
| `memexpaas_browser` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/memexpaas_browser` |
| `memexpaas_deploy` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/memexpaas_deploy` |
| `memexpass_mathagent` | `/app/data/dialogs/b10Xp6TI7aVVLp8GyipIkw/memexpass_mathagent` |
| `memexplatform_flows` | `/app/data/memexplatform_flows` |
| `memexsolve` | `/app/data/memexsolve` |
| `browserhacks` | `/app/data/browserhacks` |

### Environment Variables — Key Categories

| Category | Variables |
|---|---|
| **LLM APIs** | `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `CEREBRAS_API_KEY`, `OPENROUTER_API_KEY`, `OLLAMA_API_BASE` |
| **GitHub** | `GITHUB_TOKEN` |
| **Cloudflare** | `CLOUDFLARE_API_TOKEN`, `CF-Access-Client-*` |
| **Hugging Face** | `HF_TOKEN`, `KAGGLE_KEY` |
| **Telegram Bots** | `JCMS_TELEGRAM_BOT_TOKEN`, multiple `MEMEX*_BOT_TOKEN` |
| **Azure DevOps** | `AZURE_DEVOPS`, `MEMEXASSIST_AZURE_DEVOPS_TEAMS__*` |
| **Google OAuth** | Multiple `*_GOOGLE_CLIENT_ID/SECRET` pairs |
| **Internal LLM** | `VLLM_INTERNAL_URL`, `VLLM_EXTERNAL_URL`, `VLLM_API_KEY` |
| **Memex Config** | `MEMEX_DB_PATH`, `MEMEXASSIST_*`, `MEMEXPLATFORM_*` |
| **Solveit** | `PRIVATE_DOMAIN`, `PUBLIC_DOMAINS`, `REMOTE_URL`, `SITE_URL` |
| **Pi Agent** | `PI_CODING_AGENT=true` |

### tmux Sessions (Running)

| Session | Windows | Created |
|---|---|---|
| `floor-assistant` | 1 | May 12, 14:28 |
| `jcms` | 1 | May 12, 14:26 |
| `memexpaas` | 1 | May 13, 12:05 |
| `memexsolve` | 2 | May 13, 08:08 |

---

## 4. Software Stack Summary

### System Packages (703 dpkg packages)
- **Core:** Debian 12 (bookworm), glibc 2.36, bash 5.2
- **Build:** gcc-12, g++-12, clang-14, make, autoconf, automake, cython, cygdb
- **Languages:** Python 3.11 (system), Python 3.12 (compiled), Node.js 24, Rust/Cargo 0.66
- **DevOps:** git, git-lfs, ssh, curl, wget, aria2, jq, tmux, htop, tree
- **Multimedia:** ffmpeg, imagemagick-6.q16, graphviz
- **Data:** pandoc, hdf5, libxml2-dev, libxslt1-dev, mariadb client dev
- **ML/AI:** torch 2.11.0+cpu (CPU-only), torchaudio, torchvision

### Python Ecosystem (~371 pip packages)
- **Web frameworks:** python-fasthtml (0.13.4), fastapi, starlette, streamlit
- **AI/ML:** torch 2.11.0+cpu, scikit-learn 1.8.0, scipy 1.17.1, transformers (via tiktoken)
- **APIs:** openai 2.24.0, anthropic 0.98.1, replicate 1.0.7
- **nbdev/fast.ai stack:** nbdev 2.4.x, fastcore, ghapi, fastlite, fastmigrate, execnb
- **Data:** pandas, sqlalchemy, redis, jq, matplotlib, seaborn
- **Testing:** pytest, pytest-playwright
- **Custom packages:** solveit, floor_assistant, memexsolve, memexpaas, JCMSSalesPortal, browserhacks, memexhelper

### Node.js Ecosystem
- Node.js 24.15.0 (NodeSource)
- NVM with Node.js 20.20.0 (project-level)
- npm 11.12.1
- Deno (binary in ~/.deno/bin)
- Quarto 1.8.24 (Dart-based document system)

### Custom Binaries
- **Caddy** v2 (47MB) — reverse proxy
- **ast-grep** v0.42.1 (47MB) — code analysis
- **sg** (362KB) — ast-grep CLI
- **shfmt** (2.9MB) — shell formatter
- **solveit** (299B) — Python script wrapper
- **exhash** (2.9MB) — custom hash tool
- **lnhashview** (535KB) — hash viewer

---

## 5. Operational Observations

### Architecture Pattern
1. **nbdev-based development** — Multiple projects use the nbdev ecosystem (FastHTML notebooks → Python packages)
2. **Claude Code as primary IDE** — Enterprise Anthropic subscription, multiple Claude profiles, MCP servers
3. **Pi coding agent** — Secondary AI coding tool with session-based architecture
4. **tmux multiplexing** — Heavy terminal session management for running multiple services
5. **Caddy reverse proxy** — Single Caddyfile routes services to subdomains

### Container Lifecycle
- **Entry process:** `solveit --reload false` (FastHTML web app)
- **User:** `solveit` (uid=2000, gid=2000) with sudo access
- **$HOME:** `/app/data` (bind-mounted from host)
- **Shell:** bash (interactive sessions via tmux)
- **Terminal:** `xterm-ghostty` (using Ghostty terminal emulator)

### Notable Setup Details
1. **Permission mismatch:** `solveit` package source at `/root/solveit/solveit/` is root-owned and inaccessible to the `solveit` user
2. **Three Claude profiles:** Default, "Aman", and "Kaushal" — likely multi-person or multi-project usage
3. **Enterprise Anthropic:** Contract billing, 1M context Opus, 43+ marketplace plugins auto-installed
4. **AI infrastructure:** Local VLLM serving + multiple external APIs (OpenAI, Anthropic, Groq, Cerbras, OpenRouter)
5. **Development workflow:** nbdev notebooks → pip editable installs → tmux sessions → Claude Code review
6. **Cloud infrastructure:** Cloudflare Tunnel + Tailscale for secure external access

---

## 6. Recommendations

1. **Fix `/root/solveit/solveit/` permissions** — The editable install targets this path but the `solveit` user cannot read it. Either move it to a user-accessible location or fix ownership.
2. **Consider Python version pinning** — Two Pythons (3.11 system, 3.12 compiled) coexist. Ensure all tooling explicitly uses 3.12.
3. **NVM not loaded** — `NVM_DIR` env var is unset despite NVM being installed. The `nvm.sh` sourcing appears to be missing from the PATH-based Node.js setup.
4. **Security review** — Many API keys in environment variables (some visible in container inspection). Consider using a secrets manager or `.env` files.
5. **Container image optimization** — 70+ overlay layers suggests a very long Dockerfile. Consider multi-stage builds and squashing.
6. **Root directory hygiene** — `/root` is empty but the solveit package source is there. Consider moving it under `/app/data` for consistency.
