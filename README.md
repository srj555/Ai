# Running Gemma 4 E4B Locally with Claude Code

A complete guide to what was set up, how it works, how to re-create it, and what to expect on an enterprise Mac.

---

## What Was Done

Google's **Gemma 4 E4B** (Effective 4B) is a multimodal open model that runs fully locally — no cloud, no API costs. This guide sets it up so Claude Code can switch between Anthropic's Claude and local Gemma with a single command.

The architecture is:

```
Claude Code
    │
    │  (ANTHROPIC_BASE_URL=http://localhost:4000)
    ▼
LiteLLM Proxy  ←── translates Anthropic API format to OpenAI format
    │
    ├──► Ollama (localhost:11434)  →  Gemma 4 E4B  (local, offline)
    └──► Anthropic API             →  Claude Sonnet (cloud, your key)
```

**Three components installed:**

| Component | What it does | How it starts |
|-----------|-------------|---------------|
| **Ollama** | Runs Gemma locally on Apple Silicon | On-demand via `gemma-start` |
| **Gemma 4 E4B** | The actual model weights (9.6 GB) | Loaded by Ollama on demand |
| **LiteLLM Proxy** | Bridges Claude Code → Ollama | On-demand via `gemma-start` |

> **Neither Ollama nor the proxy auto-start at login.** This keeps the system cool and memory free until you actually need Gemma.

Shell functions/aliases in `~/.zshrc`:

| Command | What it does |
|---------|-------------|
| `gemma-start` | Start Ollama + LiteLLM proxy |
| `gemma-stop` | Stop both — use when done or system is hot |
| `gemma-proxy-status` | Check if everything is running |
| `claude-gemma` | Open Claude Code using Gemma 4 E4B |
| `claude` | Normal Claude Code (Anthropic, unchanged) |

---

## Minimum Requirements

### Hardware

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **RAM** | 6 GB free | 16 GB total (M2/M3) |
| **Disk** | 12 GB free | 20 GB+ free |
| **CPU/GPU** | Apple Silicon preferred | M2 Pro or better |

> **On M1 with 8 GB RAM:** Gemma consumes ~5–6 GB of unified memory while loaded. Other apps will become sluggish. Always run `gemma-stop` when done. M2/M3 with 16 GB is the comfortable sweet spot.

### Software Prerequisites

| Tool | Required Version | Install |
|------|-----------------|---------|
| macOS | 13 Sonoma or later | System update |
| Homebrew | Any | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/homebrew/install/HEAD/install.sh)"` |
| Python | 3.13 (not 3.14) | `brew install python@3.13` |
| Claude Code CLI | Latest | `npm install -g @anthropic-ai/claude-code` |
| Ollama | 0.23+ | via Homebrew (below) |

> **Why Python 3.13 specifically?** LiteLLM's dependency `orjson` does not yet support Python 3.14 (the current Homebrew default as of May 2026). Use 3.13 explicitly.

---

## Fresh Setup — Step by Step

Run these in order. Each step builds on the previous one.

### Step 1 — Install Ollama

```bash
brew install ollama

# Start it temporarily just for the model download
brew services start ollama

# Verify it's running
ollama list
```

### Step 2 — Download Gemma 4 E4B

This is a 9.6 GB download. Make sure you have at least 12 GB free first.

```bash
# Check free space before starting
df -h /

# Pull the model (Ollama resumes automatically if interrupted)
ollama pull gemma4:e4b

# Confirm it's installed
ollama list
```

Expected output:
```
NAME          ID              SIZE      MODIFIED
gemma4:e4b    c6eb396dbd59    9.6 GB    just now
```

Once downloaded, stop the service — you'll start it on-demand from now on:
```bash
brew services stop ollama
```

### Step 3 — Install LiteLLM in a Python 3.13 Virtual Environment

```bash
python3.13 -m venv ~/.litellm-venv
~/.litellm-venv/bin/pip install 'litellm[proxy]' --quiet

# Verify
~/.litellm-venv/bin/litellm --version
```

### Step 4 — Create the LiteLLM Config

```bash
cat > ~/.litellm-config.yaml << 'EOF'
model_list:
  - model_name: gemma4-e4b
    litellm_params:
      model: ollama/gemma4-cc
      api_base: http://localhost:11434

  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-6
      api_key: os.environ/ANTHROPIC_API_KEY

litellm_settings:
  drop_params: true
  set_verbose: false

general_settings: {}
EOF
```

> Note: the model name is `gemma4-cc`, not `gemma4:e4b` — this is a custom model created in Step 5b that fixes JSON-wrapped responses.

### Step 5 — Create the launchd Agent (On-Demand, Not Auto-Start)

The plist file is used for easy start/stop control but **does not auto-start at login** (`RunAtLoad` and `KeepAlive` are both `false`).

```bash
cat > ~/Library/LaunchAgents/com.litellm.proxy.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.litellm.proxy</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/YOUR_USERNAME/.litellm-venv/bin/litellm</string>
        <string>--config</string>
        <string>/Users/YOUR_USERNAME/.litellm-config.yaml</string>
        <string>--port</string>
        <string>4000</string>
    </array>
    <key>RunAtLoad</key>
    <false/>
    <key>KeepAlive</key>
    <false/>
    <key>StandardOutPath</key>
    <string>/tmp/litellm-proxy.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/litellm-proxy.log</string>
</dict>
</plist>
EOF

# Replace YOUR_USERNAME with your actual username
sed -i '' "s/YOUR_USERNAME/$(whoami)/g" ~/Library/LaunchAgents/com.litellm.proxy.plist
```

### Step 5b — Fix JSON-Wrapped Responses (Required)

Out of the box, Gemma 4 E4B wraps its replies in JSON like `{"response": "..."}` instead of plain text. Fix this by creating a custom Ollama model with a system prompt that instructs it to respond naturally:

```bash
cat > /tmp/gemma4-cc.modelfile << 'EOF'
FROM gemma4:e4b
SYSTEM "You are a helpful AI coding assistant. Always respond in plain, natural language. Never wrap your responses in JSON, never use keys like 'response:' or structured output unless the user explicitly asks for it."
EOF

ollama create gemma4-cc -f /tmp/gemma4-cc.modelfile
```

> This creates a thin wrapper model called `gemma4-cc` (same weights, just with a fixed system prompt). The LiteLLM config already references this name. If you ever need to reset it, just re-run the `ollama create` command.

After creating, stop Ollama again:
```bash
brew services stop ollama
```

### Step 6 — Add Shell Functions

Add these to your `~/.zshrc`:

```bash
cat >> ~/.zshrc << 'EOF'

# Gemma 4 E4B — on-demand start/stop (does NOT auto-start at login)
gemma-start() {
  echo "Starting Ollama..."
  brew services start ollama
  echo "Starting LiteLLM proxy..."
  launchctl load ~/Library/LaunchAgents/com.litellm.proxy.plist
  sleep 3
  echo "Ready. Use 'claude-gemma' to start a session."
}

gemma-stop() {
  echo "Stopping LiteLLM proxy..."
  launchctl unload ~/Library/LaunchAgents/com.litellm.proxy.plist
  echo "Stopping Ollama..."
  brew services stop ollama
  echo "All stopped. System will cool down."
}

gemma-proxy-status() {
  if curl -s http://localhost:4000/v1/models &>/dev/null; then
    echo "LiteLLM proxy: running"
    curl -s http://localhost:4000/v1/models | python3 -m json.tool 2>/dev/null | grep '"id"' | sed 's/.*"id": "\(.*\)".*/  - \1/'
  else
    echo "LiteLLM proxy: stopped"
  fi
}

alias claude-gemma='ANTHROPIC_BASE_URL=http://localhost:4000 ANTHROPIC_API_KEY=sk-no-key claude --model gemma4-e4b'
alias claude-sonnet='claude'
EOF

source ~/.zshrc
```

### Step 7 — Verify Everything Works

```bash
# Start everything
gemma-start

# Check proxy is up and models are listed
gemma-proxy-status

# Send a test message — should be plain text, no JSON
curl -s http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-no-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemma4-e4b","messages":[{"role":"user","content":"Say hello"}],"max_tokens":20}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['choices'][0]['message']['content'])"
```

Expected output: `Hi there! How can I help you today?` — plain text, no braces, no keys.

```bash
# Stop when done testing
gemma-stop
```

---

## Daily Usage

### Starting a Gemma session

```bash
gemma-start        # starts Ollama + proxy (~3 seconds)
claude-gemma       # opens Claude Code with Gemma 4 E4B
```

### When you're done (or system gets hot)

```bash
gemma-stop         # stops everything, frees ~5–6 GB memory
```

### Normal Claude (unchanged)

```bash
claude             # uses Anthropic API directly, no proxy involved
```

### Other useful commands

```bash
# Check if proxy is running
gemma-proxy-status

# Watch proxy logs live
tail -f /tmp/litellm-proxy.log

# Pass model inline without the alias
ANTHROPIC_BASE_URL=http://localhost:4000 ANTHROPIC_API_KEY=sk-no-key claude --model gemma4-e4b "explain this file"
```

---

## Enterprise / Office Mac Setup

Running this on a managed corporate Mac has additional complications. Here is what to expect and how to work around each.

### Problem 1 — Homebrew May Be Blocked

Many enterprise Macs block Homebrew or restrict installing into `/opt/homebrew`.

**Check first:**
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/homebrew/install/HEAD/install.sh)"
```

**Workarounds:**
- Ask IT to whitelist Homebrew, or
- Install Ollama directly from the official `.dmg` at `ollama.com` — no Homebrew needed
- Use the Ollama Mac app (has a menu bar icon for easy start/stop — ideal for enterprise)

### Problem 2 — Outbound Port Restrictions

Corporate networks often block outbound connections on non-standard ports. The model download (`ollama pull`) needs HTTPS access to Ollama's registry.

**Check:**
```bash
curl -v https://registry.ollama.ai 2>&1 | grep "Connected\|SSL\|403\|blocked"
```

If blocked, you have two options:
- Ask IT to whitelist `registry.ollama.ai` on port 443
- Download the model on your personal Mac and copy it to the office Mac:
  ```bash
  # On personal Mac — find the model files
  ls ~/.ollama/models/
  # Copy ~/.ollama/models/ to office Mac via USB or shared drive
  # Then on office Mac:
  # Place files in ~/.ollama/models/ and run: ollama list
  ```

### Problem 3 — launchd Agent May Require Admin Approval

On MDM-managed Macs (Jamf, etc.), launchd user agents in `~/Library/LaunchAgents/` usually work without admin rights. But if the plist is blocked:

**Alternative — start the proxy manually each session:**
```bash
nohup ~/.litellm-venv/bin/litellm \
  --config ~/.litellm-config.yaml \
  --port 4000 > /tmp/litellm-proxy.log 2>&1 &
echo "Proxy PID: $!"
```

Replace `gemma-start` with this command and `gemma-stop` with:
```bash
pkill -f "litellm.*4000" && brew services stop ollama
```

### Problem 4 — SSL/TLS Inspection (MITM Proxy)

Some enterprise networks inspect HTTPS traffic with a corporate certificate. `ollama pull` may fail with a certificate error.

**Symptom:**
```
Error: failed to pull model: certificate signed by unknown authority
```

**Fix:**
```bash
# Find your corporate cert and add it
export SSL_CERT_FILE=/path/to/corporate-ca.pem
export REQUESTS_CA_BUNDLE=/path/to/corporate-ca.pem
ollama pull gemma4:e4b
```

Ask IT for the corporate CA certificate path — it's usually in `/etc/ssl/certs/` or installed in the macOS Keychain.

### Problem 5 — Disk Quota

Enterprise Macs sometimes have user home directory quotas. The model is 9.6 GB.

**Check your quota:**
```bash
quota -s 2>/dev/null || df -h ~
```

If you're quota-limited, store the model on a local non-home volume:
```bash
# Tell Ollama to store models elsewhere
export OLLAMA_MODELS=/Volumes/SomeLocalDisk/ollama-models
# Add this export to ~/.zshrc permanently
```

### Problem 6 — No `python3.13` Available

If IT-managed Python is too old or only Python 3.14 is available:

```bash
# Option A: Install python@3.13 via Homebrew (if allowed)
brew install python@3.13

# Option B: Use pyenv (no admin rights needed)
curl https://pyenv.run | bash
pyenv install 3.13.0
pyenv local 3.13.0
python3.13 -m venv ~/.litellm-venv
```

### Problem 7 — Port 4000 Already in Use

Check before starting:
```bash
lsof -i :4000
```

If occupied, change the port in `~/.litellm-config.yaml` and `~/Library/LaunchAgents/com.litellm.proxy.plist` to e.g. `4001`, then update the `claude-gemma` alias to match:
```bash
alias claude-gemma='ANTHROPIC_BASE_URL=http://localhost:4001 ANTHROPIC_API_KEY=sk-no-key claude --model gemma4-e4b'
```

---

## Files Created During This Setup

| File | Purpose |
|------|---------|
| `~/.ollama/models/` | Ollama model storage (9.6 GB for gemma4:e4b + gemma4-cc) |
| `~/.litellm-config.yaml` | LiteLLM proxy model routing config |
| `~/.litellm-venv/` | Python 3.13 virtual environment for LiteLLM |
| `~/Library/LaunchAgents/com.litellm.proxy.plist` | On-demand launchd definition (not auto-start) |
| `~/.zshrc` | `gemma-start`, `gemma-stop`, `gemma-proxy-status`, `claude-gemma` added at the bottom |

---

## Uninstalling / Reverting

```bash
# Stop everything first
gemma-stop

# Remove the launchd plist
launchctl unload ~/Library/LaunchAgents/com.litellm.proxy.plist 2>/dev/null
rm ~/Library/LaunchAgents/com.litellm.proxy.plist

# Remove LiteLLM
rm -rf ~/.litellm-venv ~/.litellm-config.yaml

# Remove Gemma models (reclaims ~9.6 GB)
ollama rm gemma4-cc
ollama rm gemma4:e4b

# Uninstall Ollama
brew uninstall ollama

# Remove aliases — edit ~/.zshrc and delete the block starting with
# "# Gemma 4 E4B — on-demand start/stop"
```

---

## Quick Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| System running hot / fans spinning | Ollama loaded and idle | Run `gemma-stop` immediately |
| `claude-gemma` hangs or errors | Proxy not running | Run `gemma-start`, then retry |
| `Connection refused :4000` | Proxy failed to start | Check `tail -20 /tmp/litellm-proxy.log` |
| Proxy running but Gemma slow to respond | Model loading cold start | Wait 10–20s on first prompt; fast after that |
| Responses wrapped in `{"response": "..."}` | Gemma's default JSON mode | Re-run Step 5b to recreate `gemma4-cc` modelfile |
| `ollama pull` fails | Network / cert issue | See Enterprise Problem 2 and 4 above |
| Mac sluggish during inference | ~5–6 GB memory in use | Normal on 8 GB M1; run `gemma-stop` when done |
| `ModuleNotFoundError: litellm` | Wrong Python version used | Use `~/.litellm-venv/bin/litellm` explicitly |
| Responses cut off mid-sentence | `max_tokens` too low | Add `max_tokens: 2048` under `litellm_settings` in the config |
| `gemma-start` / `gemma-stop` not found | Shell not reloaded | Run `source ~/.zshrc` first |

---

## Performance Expectations (M1, 8 GB)

| Scenario | Speed |
|----------|-------|
| First prompt (cold model load) | 10–20 seconds to first token |
| Subsequent prompts (model warm) | ~15–25 tokens/second |
| While other apps are open | Slower; close Chrome/Slack/Xcode if possible |

For comparison, an M3 Pro with 36 GB RAM typically achieves ~60–80 tokens/second on this model.

---

*Set up on: macOS 14.5, Apple M1, 8 GB RAM — Ollama 0.23.2, LiteLLM 1.83.14, Gemma 4 E4B*
