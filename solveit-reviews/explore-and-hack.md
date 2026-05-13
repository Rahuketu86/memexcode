# What to Explore & Hack — Live Container Exploration Guide

**Date:** 2026-05-13  
**Container:** `aabb4c8a0f0a`  
**Purpose:** Actionable ideas for exploring, modifying, and building on this live environment

---

## 1. The solveit Stack — Fix, Hack, Extend

The solveit app source lives at `/root/solveit/solveit/` but is **permission-denied for the `solveit` user**. First step:

```bash
chmod -R 755 /root/solveit
python3 -c "import solveit.main; print(solveit.main.__file__)"
```

Once that's unlocked, you have a **FastHTML app by Jeremy Howard** with 37+ dependencies in an active nbdev workflow.

### What You Can Do

- **Add features directly** — nbdev compiles notebooks → Python packages. Edit the source, update a notebook, rebuild
- **Hook up ghost text** — The metadata references `AAI_OPENAI_PROXY_URL` for inline suggestions powered by Deepseek
- **Study Jeremy Howard's patterns** — `fastcore`, `execnb`, `nbdev`, `fastlite` — unique codebase patterns
- **Build custom nbdev extensions** — The stack has `nbdev-numpy`, `nbdev-stdlib` — create your own preset
- **Run the playwright tests** — `pip install -e ".[dev]" && playwright install chromium && pytest`

---

## 2. Claude Code — The Most Underutilized Asset

3 Claude profiles (default, Aman, Kaushal), enterprise Anthropic subscription, Opus with 1M context, 45+ marketplace plugins auto-installed, custom MCP servers.

### What's Already Running

```bash
cat ~/.mcp.json                     # MCP server configuration
ls ~/.claude/projects/              # Per-project Claude settings
cat ~/.claude/.claude.json          # Feature flags, subscription details
python3 -c "
import json
with open('/app/data/.claude/.claude.json') as f:
    d = json.load(f)
print('Subs:', d.get('subscription',{}).get('plan','?'))
print('Org:', d.get('user',{}).get('orgName','?'))
print('Feature flags:', len(d.get('featureFlags',{})))
"
```

### What You Can Build

| Idea | Effort | Impact |
|---|---|---|
| **Custom MCP server** exposing ast-grep, Quarto, your notebooks to Claude Code | Medium | Huge — makes Claude Code aware of your entire codebase |
| **Multi-project context** — MCP server that indexes all 11 nbdev projects | Hard | Game-changing for cross-project reasoning |
| **Automated code review** — MCP server that runs ast-grep rules and feeds results to Claude | Easy-Medium | Catch bugs before they ship |
| **Session-aware agent** — Use the Pi coding agent sessions as context for Claude | Medium | Persistent memory across sessions |

---

## 3. The Voice Stack — STT/TTS Dockerfiles Sitting There

```bash
ls /app/data/voice-stack/
cat /app/data/voice-stack/stt/Dockerfile
cat /app/data/voice-stack/tts/Dockerfile
```

### What You Can Build

- **Build and run the containers** — `cd voice-stack && docker build -f stt/Dockerfile -t stt .`
- **Voice-to-note pipeline** — Record → Whisper (STT) → Claude (summarize) → nbdev (save as notebook)
- **Telegram voice bot** — You have `JCMS_TELEGRAM_BOT_TOKEN` and `MEMEX*_BOT_TOKEN`. A bot that listens for voice messages, transcribes them, saves to notes
- **Discord voice assistant** — Similar but for Discord bots you've already deployed

---

## 4. The LLM Infrastructure — 5+ Providers

You have these API keys configured:
| Provider | Key Prefix | Use Case |
|---|---|---|
| OpenAI | `sk-or-v1-...` | GPT-4, gpt-oss, etc. |
| Anthropic | `sk-ant-oat01-...` | Claude Opus, Sonnet |
| Groq | `gsk_t00YY...` | Fast inference (Llama, Mixtral) |
| Cerbras | `csk-4mrd...` | Specialized models |
| OpenRouter | `sk-or-v1-...` | Router for multiple models |
| Ollama | `http://bore.pub:5386/` | Local models |
| VLLM Internal | `http://memexpaas-vllm:8100/v1` | Self-hosted inference |
| VLLM External | `http://memexpaas-llm-api.rahulsaraf.online/v1` | Cloud-hosted inference |

### What You Can Build

#### Provider Benchmarking Script
```python
# benchmark_providers.py
import asyncio, aiohttp, time

PROVIDERS = {
    "openai": {"url": "https://api.openai.com/v1", "key": os.getenv("OPENAI_API_KEY")},
    "anthropic": {"url": "https://api.anthropic.com/v1", "key": os.getenv("ANTHROPIC_API_KEY")},
    "groq": {"url": "https://api.groq.com/openai/v1", "key": os.getenv("GROQ_API_KEY")},
    "openrouter": {"url": "https://openrouter.ai/api/v1", "key": os.getenv("OPENROUTER_API_KEY")},
    "cerbras": {"url": "https://api.cerbras.net/v1", "key": os.getenv("CEREBRAS_API_KEY")},
    "vllm_internal": {"url": os.getenv("VLLM_INTERNAL_URL"), "key": os.getenv("VLLM_API_KEY")},
}

prompt = "What is the time complexity of quicksort in the worst case?"
models = {
    "openai": "gpt-4.1-mini",
    "anthropic": "claude-sonnet-4-20250514",
    "groq": "llama-3.3-70b-versatile",
    "openrouter": "google/gemini-2.5-pro",
    "cerbras": "llama-3.3-70b",
    "vllm_internal": "your-model-here",
}

async def benchmark(provider, model, prompt):
    async with aiohttp.ClientSession() as session:
        start = time.time()
        # ... send request, measure latency, store results
```

#### LiteLLM Proxy
```bash
# litellm is already installed, just configure it
pip install litellm[proxy]
litellm --config config.yaml --port 4000
```

```yaml
# config.yaml
model_list:
  - model_name: gpt-4
    litellm_params:
      model: openai/gpt-4.1-mini
      api_key: YOUR_OPENAI_KEY
  - model_name: claude
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: YOUR_ANTHROPIC_KEY
  - model_name: fast
    litellm_params:
      model: groq/llama-3.3-70b-versatile
      api_key: YOUR_GROQ_KEY
  - model_name: local
    litellm_params:
      model: openai/vllm-local
      api_base: $VLLM_INTERNAL_URL
      api_key: $VLLM_API_KEY

route_by_latency: true
```

Then every call to `http://localhost:4000/v1/chat/completions` automatically routes to the best provider.

---

## 5. The 11 nbdev Projects — Cross-Project Integration

```
pip3 list --editable 2>/dev/null | grep Editable
```

| Project | DB | What It Does |
|---|---|---|
| `solveit` | — | Main FastHTML IDE/app |
| `floor_assistant` | `floor.db` | Floor plan assistant, Vite static site |
| `memexsolve` | `memexsolve/memexsolve.db` | Core memex system |
| `JCMSSalesPortal` | `JCMSSalesPortal/leads.db` | CRM with Telegram bot + Google OAuth |
| `memexpaas` | `memexpaas.db` | PaaS plugin specification |
| `memexpaas_deploy` | — | Deployment orchestrator |
| `memexpaas_browser` | — | Browser automation agent |
| `memexpaas_mathagent` | — | Math agent |
| `memexhelper` | — | Utility functions |
| `memexplatform_flows` | `memexplatform_flows/flows.db` | Workflow engine |
| `browserhacks` | — | Browser automation utilities |

### What You Can Build

- **Unified memex dashboard** — All projects speak SQLite + FastHTML. One solveit app that surfaces data from ALL databases
- **Cross-project search** — Index all SQLite databases into a single search endpoint using `fastlite`
- **Shared auth** — All 11 projects use Google OAuth. Build a single auth gateway
- **Flow executor** — `memexplatform_flows` stores workflow definitions. Build a UI that runs them and shows results across other projects

---

## 6. tmux Sessions — Live Process Recon

```bash
tmux list-sessions          # See 4 running sessions
tmux capture-pane -t memexsolve -p | head -30   # What's running in memexsolve?
tmux capture-pane -t jcms -p | head -30         # What's running in jcms?
tmux capture-pane -t floor-assistant -p | head -30
tmux capture-pane -t memexpaas -p | head -30
```

### What You Can Build

- **Session monitor dashboard** — FastHTML app showing all tmux sessions, CPU/memory usage, restart controls
- **Auto-restart dead sessions** — Cron job or systemd service watching tmux panes
- **Web terminal** — Allow attaching to tmux sessions via browser (there are tools for this)

---

## 7. Cloudflare + Tailscale Infrastructure

```bash
echo "Private domain: $PRIVATE_DOMAIN"
echo "Public domains: $PUBLIC_DOMAINS"
echo "Tailscale IP: $MEMEXPAAS_DEPLOY_TAILSCALE_IP"
```

Domains mapped:
- `6000` → `qawesome-rahulsaraf-apps.solveit.pub`
- `6001` → `memexsolve.rahulsaraf.online`
- `6002` → `memexpaas-rahulsaraf-apps.solveit.pub`
- `6500` → `floorassistant.rahulsaraf.online`
- `8000` → `jcmsleads.rahulsaraf.online`

### What You Can Build

- **Domain health monitor** — Script that checks all 5 domains return HTTP 200, alerts on failure
- **Auto-cert management** — Cloudflare gives you certs. Build a tool that rotates/renews them
- **Rate limiter dashboard** — Cloudflare has rate limiting. Expose it via a simple API
- **Tailscale tailnet management** — Add/remove devices programmatically

---

## 8. The Toolchain — 70+ Tools in `~/.local/bin`

| Tool | Hack Idea |
|---|---|
| `litellm` / `litellm-proxy` | Run as LLM router, benchmark all providers |
| `tiny-agents` / `smolagent` | Build a code-writing agent that tests itself |
| `ipymini` / `ipyai-mcp-bridge` | Jupyter ↔ Claude bridge — live notebook + AI |
| `playwright` | Auto-scraper that generates nbdev docs from web pages |
| `sqlformat` / `alembic` | Migrate all 11 project DBs to unified schema |
| `gunicorn` / `fastmcp` | Self-host multi-service dashboard on port 6000 |
| `streamlit` / `gradio` | Quick ML/dashboards for memex data |
| `hf` / `huggingface-cli` | Pull/run models locally via HuggingFace |

---

## 9. The Hardware — 48 Cores, 1TB RAM, 0 GPU

This is a **CPU powerhouse** that can't run GPU workloads but is incredible for:

- **Massive parallel code analysis** — Run ast-grep across all 11 repos with 48 workers
- **Voice processing** — STT/TTS is CPU-friendly
- **Compilation** — Python, Rust, C, Go — everything compiles fast
- **Local JupyterHub** — Serve notebooks to multiple users on 48 cores
- **Dense data processing** — 1TB RAM means you can load enormous datasets into memory

### Concrete Idea: Parallel Code Review

```bash
# Run ast-grep across all 11 projects simultaneously
find /app/data -name "*.py" -not -path "*/.git/*" | \
  xargs -P 48 ast-grep scan -p your-rules.yaml
```

48 processes scanning your codebase in parallel could finish in seconds.

---

## 10. The Data Stores — SQLite Across All Projects

```bash
find /app/data -name "*.db" -not -path "*/.git/*" 2>/dev/null
# floor.db
# memexpaas.db
# memexsolve/memexsolve.db
# JCMSSalesPortal/leads.db
# memexplatform_flows/flows.db
```

### What You Can Build

- **Unified DB explorer** — FastHTML app that lets you browse all 5 databases from one interface
- **Cross-DB relationships** — Are there contacts in JCMS that also appear in memexsolve notes? Find them
- **FTS search across all DBs** — Add SQLite full-text search to all tables, query across everything
- **Export to graph** — Dump all DBs to a Neo4j/SQLite Graph store for relationship analysis

---

## Quick Wins (Ranked by Effort → Impact)

| Priority | Action | Effort | Impact |
|---|---|---|---|
| **1** | `chmod -R 755 /root/solveit` | 5 seconds | Unlock live-editing solveit |
| **2** | Write provider benchmark script | 1 hour | Understand which LLM is actually best |
| **3** | Run `litellm-proxy` | 30 min | Unified LLM endpoint, automatic routing |
| **4** | `docker build -f voice-stack/stt/Dockerfile -t stt .` | 15 min | Voice stack goes live |
| **5** | `tmux capture-pane -t memexsolve -p` | 30 seconds | See what's actually running |
| **6** | Export all 5 SQLite DBs + find cross-DB relationships | 2 hours | Discover hidden connections in your data |
| **7** | Build unified memex dashboard | 1-2 days | One UI for all 11 projects |
| **8** | Custom MCP server for Claude Code | 3-5 days | Claude Code with full project awareness |
| **9** | Voice-to-note pipeline | 2-3 days | Record → transcribe → summarize → save as notebook |
| **10** | Domain health monitor + auto-restart | 1 day | Production reliability for 5 domains |
