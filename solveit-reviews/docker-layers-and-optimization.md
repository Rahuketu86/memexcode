# Docker Layers: Analysis and Optimization Strategy

**Date:** 2026-05-13  
**Container:** `aabb4c8a0f0a` (containerd/overlayfs)  
**Current Layer Count:** 51 (50 read-only + 1 read-write upper)

---

## 1. What Are Docker Layers?

Docker images are stacks of **read-only layers**, each created by one `RUN`, `COPY`, or `ADD` instruction in the Dockerfile. The overlayfs filesystem unions them together:

```
Container filesystem (read-write upper layer)
  └── Overlayfs union of:
      ├── Layer 50: final image state
      ├── Layer 49: apt install quarto
      ├── Layer 48: apt install rustc cargo
      ├── Layer 47: wget + compile Python 3.12
      ├── ...
      ├── Layer  3: RUN apt install git curl tmux
      ├── Layer  2: RUN apt install gcc build-essential
      └── Layer  1: FROM debian:bookworm-slim
```

**Your container: 51 layers total** (50 lower read-only layers from the image, plus 1 writable upper layer for the running container's runtime changes).

You can verify this from inside the container:

```bash
# Count layers from mountinfo
cat /proc/self/mountinfo | grep overlay | awk -F',' '{
  for(i=1;i<=NF;i++){
    if($i~/^lowerdir=/){
      gsub("lowerdir=","",$i);
      n=split($i,parts,":");
      print "Lower layers: " n
    }
  }
}'
# Output: Lower layers: 50
```

---

## 2. Why 50 Layers? — Mapping Layers to Likely Dockerfile Steps

Each `RUN`, `COPY`, or `ADD` in the Dockerfile creates exactly one layer. Your 50 layers map to a Dockerfile with roughly this structure:

### Layer Breakdown by Phase

| Layers | Phase | Example Instructions |
|---|---|---|
| 1 | `FROM debian:bookworm-slim` | Base OS image (~75MB) |
| 2-9 | System apt packages (split across many RUNs) | Separate `RUN` for each package group |
| 10-12 | Python 3.12 source download + compile | wget → tar → configure → make install |
| 13 | Node.js 24 from NodeSource | apt install nodejs |
| 14 | Rust/Cargo from apt | apt install cargo rustc |
| 15-16 | Quarto download + extract | wget → tar → mv to /opt/quarto |
| 17 | Deno binary download | wget → mv to /usr/local/bin |
| 18-20 | Custom binaries | wget caddy, ast-grep, shfmt |
| 21 | User creation | RUN useradd -ms /bin/bash solveit |
| 22-23 | NVM + Node 20 setup | git clone nvm → nvm install node |
| 24-37 | Python pip packages (one per RUN) | torch, openai, fastapi, nbdev, scipy, matplotlib, pytest, etc. |
| 38-48 | Editable project installs (one per RUN) | pip install -e for each of 11 projects |
| 49-50 | Final config/setup | chmod, COPY Caddyfile |

### The Problem: Why This Pattern Is Inefficient

```dockerfile
# === BAD: One apt install per RUN = 8 layers ===
RUN apt-get install -y git
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y tmux
RUN apt-get install -y jq
RUN apt-get install -y htop
RUN apt-get install -y vim
RUN apt-get install -y tree

# === GOOD: All apt installs in one RUN = 1 layer ===
RUN apt-get update && apt-get install -y \
    git curl wget tmux jq htop vim tree \
    && rm -rf /var/lib/apt/lists/*
```

**Rule: Every line that changes the filesystem = 1 layer.** That's why 50 layers means a very fragmented Dockerfile.

---

## 3. The Hidden Cost — Space Wasted in Layers

Docker layers are additive. When Layer 5 deletes a file that Layer 2 created, the file still exists in Layer 2's delta. It's "hidden" but still takes space. Here's what's wasting space in your 50-layer image:

### Space-Waster #1: Python 3.12 Build Artifacts

```dockerfile
# This creates ~300-400MB of dead weight:
RUN wget https://python.org/ftp/python/3.12.13/Python-3.12.13.tgz && \
    tar xzf Python-3.12.13.tgz && \
    cd Python-3.12.13 && ./configure && make install && make clean
```

The Python source directory (~200-300MB), configure scripts, object files, and the tarball are all baked into the image, even though only the compiled `python3.12` binary is needed at runtime.

### Space-Waster #2: pip Cache Not Cleared

```dockerfile
# This caches all downloaded wheels:
RUN pip install torch
RUN pip install openai
RUN pip install scikit-learn
```

If `--no-cache-dir` wasn't used, pip stores downloaded packages in `/root/.cache/pip` — easily 2-3GB of compressed wheel files that are never needed again.

### Space-Waster #3: Separate Editable Installs

```dockerfile
# 11 separate layers, each creating a pth entry:
RUN pip install -e /app/data/solveit
RUN pip install -e /app/data/floor_assistant
RUN pip install -e /app/data/memexsolve
RUN pip install -e /app/data/JCMSSalesPortal
RUN pip install -e /app/data/memexpaas
RUN pip install -e /app/data/memexpaas_deploy
RUN pip install -e /app/data/memexpaas_browser
RUN pip install -e /app/data/memexpaas_mathagent
RUN pip install -e /app/data/memexhelper
RUN pip install -e /app/data/memexplatform_flows
RUN pip install -e /app/data/browserhacks
```

All of these could be a single `RUN` line.

### Space-Waster #4: Build-essential Installed for Runtime

```dockerfile
RUN apt-get install -y gcc g++ make autoconf automake clang-14
```

If the container only **runs** compiled code (not compiling at runtime), you don't need these packages. They add ~200MB.

---

## 4. Impact Summary — 50 Layers vs. Optimized

| Metric | Current (50 layers) | Optimized (~15 layers) |
|---|---|---|
| **Image size** | ~30-50GB | ~15-25GB |
| **Layer count** | 50 | ~15 |
| **Build time (first)** | Slow (many layers) | Faster (fewer layers) |
| **Build time (rebuild)** | Partial cache | Better cache (fewer small layers) |
| **Pull time** | 50 HTTP requests, one per layer | ~15 HTTP requests |
| **Container start** | 50 stat() calls for union mount | ~15 stat() calls |
| **`docker history` rows** | 50 lines | ~15 lines |
| **Debug layer isolation** | Easy to grep individual layers | Still reasonable |

---

## 5. Three Optimization Strategies

### Strategy A: Multi-Stage Builds (Best, Recommended)

Multi-stage builds separate the **build environment** from the **runtime environment**. Build artifacts (source code, compilers, temp files) stay in the build stage and are never copied to the final image.

```dockerfile
# === STAGE 1: BUILD ===
FROM debian:bookworm-slim AS builder

# Install everything needed to compile/build
RUN apt-get update && apt-get install -y \
    build-essential gcc g++ wget tar curl \
    && rm -rf /var/lib/apt/lists/*

# Compile Python 3.12 — everything stays in this stage
RUN wget https://python.org/ftp/python/3.12.13/Python-3.12.13.tgz && \
    tar xzf Python-3.12.13.tgz && \
    cd Python-3.12.13 && ./configure --prefix=/usr/local && \
    make -j$(nproc) && make install && \
    cd .. && rm -rf Python-3.12.13 Python-3.12.13.tgz

# Install Rust, Node.js, Quarto in builder
RUN apt-get install -y cargo rustc
RUN curl -fsSL https://deb.nodesource.com/setup_24.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*
RUN wget https://github.com/quarto-dev/quarto-cli/releases/download/v1.8.24/quarto-1.8.24-linux-amd64.tar.gz && \
    tar xzf quarto-*.tar.gz && mv quarto /opt/quarto && rm quarto-*.tar.gz

# Download custom binaries in builder
RUN wget https://github.com/caddyserver/caddy/releases/... && mv caddy /usr/local/bin/caddy

# === STAGE 2: RUNTIME ===
FROM debian:bookworm-slim

# Only runtime packages (no build-essential, no gcc, no clang)
RUN apt-get update && apt-get install -y \
    git curl wget sudo tmux htop jq \
    ffmpeg imagemagick graphviz pandoc \
    libssl-dev libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Copy ONLY compiled Python from builder — no source, no build tools
COPY --from=builder /usr/local /usr/local

# Copy Quarto from builder
COPY --from=builder /opt/quarto /opt/quarto

# Copy custom binaries from builder
COPY --from=builder /usr/local/bin/caddy /usr/local/bin/caddy

# Download deno (small binary, no build needed)
RUN wget https://github.com/denoland/deno/releases/download/v2.6.9/deno-x86_64-unknown-linux-gnu.zip && \
    unzip deno-*.zip && \
    mv deno /usr/local/bin/deno && rm deno-*.zip

# Create user and set up workspace
RUN useradd -ms /bin/bash -u 2000 solveit
USER solveit
WORKDIR /app/data

# All pip installs in ONE RUN, with --no-cache-dir
RUN pip install --no-cache-dir torch openai anthropic fastapi nbdev fastcore \
    ghapi scikit-learn scipy pandas matplotlib pytest \
    huggingface_hub replicate streamlit gradio redis sqlalchemy litellm cython \
    && rm -rf /root/.cache/pip

# All editable installs in ONE RUN
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

# Final configs
COPY Caddyfile /app/data/Caddyfile

ENTRYPOINT ["/usr/local/bin/python", "/usr/local/bin/solveit", "--reload", "false"]
```

**Why this works:** The `COPY --from=builder` only copies the compiled Python binary and standard library. The 300MB of Python source code, configure scripts, and object files never leave the builder stage.

### Strategy B: Combine RUN Instructions (Quick Fix)

If you don't want to rewrite as multi-stage, just combine small `RUN` instructions:

```dockerfile
# BEFORE — 8 layers (one apt install per RUN):
RUN apt-get update && apt-get install -y git
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y tmux
RUN apt-get install -y jq
RUN apt-get install -y htop
RUN apt-get install -y vim
RUN apt-get install -y tree

# AFTER — 1 layer (all in one RUN, clean up in same layer):
RUN apt-get update && apt-get install -y \
    git curl wget tmux jq htop vim tree && \
    rm -rf /var/lib/apt/lists/*

# BEFORE — 11 layers (one editable install per RUN):
RUN pip install -e /app/data/solveit
RUN pip install -e /app/data/floor_assistant
# ... (9 more)

# AFTER — 1 layer:
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
    -e /app/data/browserhacks
```

**This alone drops you from 50 → ~35 layers.**

### Strategy C: Image Squashing (Post-Build, Nuclear Option)

Squash collapses all existing layers into one after building:

```bash
# Method 1: --squash flag (Docker 23+, requires BuildKit)
docker build --squash -t solveit:v2 .

# Method 2: buildx with squash
docker buildx build --squash -t solveit:v2 .

# Method 3: save/import (any Docker version)
docker save solveit:v1 | docker import - solveit:v2-squashed
```

**What squashing does:**

```
Before squashing (5 layers):
  Layer 1: debian base (75MB)
  Layer 2: apt install git+curl+tmux (12MB)
  Layer 3: compile python (350MB)
  Layer 4: pip install torch+others (3GB)
  Layer 5: COPY Caddyfile + user setup (100MB)
  Total: 5 layers, ~3.4GB on disk

After --squash (1 layer):
  Layer 1: everything combined (3.4GB)
  Total: 1 layer, 3.4GB on disk
```

### Trade-offs Comparison

| Aspect | Many Layers (Current) | Combined RUNs | Squashed | Multi-Stage |
|---|---|---|---|---|
| **Image size** | 30-50GB | 30-50GB | 15-25GB | 15-25GB |
| **Layer count** | 50 | ~35 | 1 | ~15 |
| **Cache (rebuilds)** | Good | Good | Poor | Best |
| **Pull speed** | Slow (50 requests) | Better (35) | Fast (1) | Fast (15) |
| **Debuggability** | Easy | Easy | Hard | Easy |
| **Complexity** | Simple | Simple | Trivial | Moderate |

---

## 6. Specific Space Savings Estimate

| Optimization | Estimated Savings | Layers Saved |
|---|---|---|
| Multi-stage Python build (drop 300MB source) | -300-400MB | 2-3 |
| `--no-cache-dir` for all pip installs | -2-3GB | 0 (same layers, smaller) |
| Combine 11 editable pip installs → 1 | 0MB | -10 |
| Combine all apt installs → 2-3 RUNs | 0MB | -5 |
| Remove gcc/g++/make (not needed at runtime) | -200MB | 1 |
| Download deno/shfmt/caddy in one RUN | 0MB | -2 |
| **Totals** | **~3-5GB smaller** | **~20 fewer layers** |

---

## 7. Key Takeaways

1. **Every `RUN`, `COPY`, `ADD` = 1 layer.** Fragmented Dockerfiles create fragmentation in images.
2. **Build artifacts are permanently baked in** unless you use multi-stage builds.
3. **`--no-cache-dir` for pip** is mandatory in production images — pip caches are not needed at runtime.
4. **Combine small instructions.** 50 layers of 1-line RUNs is the most common anti-pattern.
5. **Squashing is good for rarely-rebuilt images** (production deployment, not CI).
6. **Multi-stage builds are the gold standard** — they solve both size and cleanliness problems.
