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
| **Ollama** | Runs Gemma locally on Apple Silicon | `brew services start ollama` (auto at login) |
| **Gemma 4 E4B** | The actual model weights (9.6 GB) | Loaded by Ollama on demand |
| **LiteLLM Proxy** | Bridges Claude Code → Ollama | launchd agent (auto at login) |

Shell aliases added to `~/.zshrc`:
- `claude-gemma` — Claude Code using Gemma 4 E4B
- `gemma-proxy-status` — check if everything is running

---

## Minimum Requirements

### Hardware

| Resource | Minimum | Your Mac |
|----------|---------|----------|
| **RAM** | 6 GB free | 8 GB total (M1) |
| **Disk** | 12 GB free | 460 GB (20 GB free at setup) |
| **CPU/GPU** | Apple Silicon preferred | M1 ✓ |

> **Note:** On M1 with 8 GB RAM, Gemma 4 E4B uses unified memory. Expect ~5–6 GB consumed while the model is loaded. Other apps may become sluggish. M2/M3 with 16 GB is the comfortable sweet spot.

### Software Prerequisites

| Tool | Required Version | Install |
|------|-----------------|---------|
| macOS | 13 Sonoma or later | System update |
| Homebrew | Any | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/homebrew/install/HEAD/install.sh)"` |
| Python | 3.13 (not 3.14) | `brew install python@3.13` |
| Claude Code CLI | Latest | `npm install -g @anthropic-ai/claude-code` |
| Ollama | 0.23+ | via Homebrew (below) |

> **Why Python 3.13 specifically?** LiteLLM's dependency `orjson` does not yet support Python 3.14 (the current Homebrew default). Use 3.13 explicitly.

---

## Fresh Setup — Step by Step

Run these in order. Each step builds on the previous one.

### Step 1 — Install Ollama

```bash
brew install ollama
brew services start ollama

# Verify it's running
ollama list
```

### Step 2 — Download Gemma 4 E4B

This is a 9.6 GB download. Make sure you have at least 12 GB free first.

```bash
# Check free space before starting
df -h /

# Pull the model (Ollama resumes if interrupted)
ollama pull gemma4:e4b

# Confirm it's installed
ollama list
```

Expected output:
```
NAME          ID              SIZE      MODIFIED
gemma4:e4b    c6eb396dbd59    9.6 GB    just now
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
      model: ollama/gemma4:e4b
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

### Step 5 — Create the launchd Auto-Start Agent

This makes the proxy start automatically at every login.

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
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/litellm-proxy.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/litellm-proxy.log</string>
</dict>
</plist>
EOF

# Replace YOUR_USERNAME with your actual username
sed -i '' "s/YOUR_USERNAME/$(whoami)/g" ~/Library/LaunchAgents/com.litellm.proxy.plist

# Load it
launchctl load ~/Library/LaunchAgents/com.litellm.proxy.plist
```

### Step 6 — Add Shell Aliases

Add these to your `~/.zshrc`:

```bash
cat >> ~/.zshrc << 'EOF'

# Gemma 4 E4B via LiteLLM proxy — model switching helpers
alias claude-gemma='ANTHROPIC_BASE_URL=http://localhost:4000 ANTHROPIC_API_KEY=sk-no-key claude --model gemma4-e4b'
alias claude-sonnet='claude'

gemma-proxy-status() {
  if curl -s http://localhost:4000/v1/models &>/dev/null; then
    echo "LiteLLM proxy: running"
    curl -s http://localhost:4000/v1/models | python3 -m json.tool 2>/dev/null | grep '"id"' | sed 's/.*"id": "\(.*\)".*/  - \1/'
  else
    echo "LiteLLM proxy: stopped — check /tmp/litellm-proxy.log"
  fi
}
EOF

source ~/.zshrc
```

### Step 7 — Verify Everything Works

```bash
# Check proxy is up and both models are listed
gemma-proxy-status

# Send a real message to Gemma
curl -s http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-no-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemma4-e4b","messages":[{"role":"user","content":"Say hello"}],"max_tokens":20}' \
  | python3 -m json.tool | grep '"content"'
```

---

## Daily Usage

```bash
# Use Gemma 4 E4B (local, free, offline)
claude-gemma

# Use Claude Sonnet (Anthropic cloud, normal)
claude

# Check proxy health
gemma-proxy-status

# Proxy logs (if something is wrong)
tail -f /tmp/litellm-proxy.log
```

You can also switch mid-session by passing `--model` explicitly:

```bash
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
- Use the Ollama Mac app (has a menu bar icon, starts automatically)

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
  # On personal Mac — package the model
  ollama show --modelfile gemma4:e4b
  # Copy ~/.ollama/models/ to office Mac via USB or shared drive
  ```

### Problem 3 — launchd Agent May Require Admin Approval

On MDM-managed Macs (Jamf, etc.), launchd user agents in `~/Library/LaunchAgents/` usually work without admin rights. But if the plist is blocked:

**Alternative — start the proxy manually each session:**
```bash
# Add to ~/.zshrc instead of using launchd
nohup ~/.litellm-venv/bin/litellm --config ~/.litellm-config.yaml --port 4000 > /tmp/litellm-proxy.log 2>&1 &
```

Or run it in a persistent terminal tab.

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

Change the port in both the plist and your aliases if needed (e.g., use `4001`).

---

## Files Created During This Setup

| File | Purpose |
|------|---------|
| `~/.ollama/` | Ollama model storage (9.6 GB for gemma4:e4b) |
| `~/.litellm-config.yaml` | LiteLLM proxy model routing config |
| `~/.litellm-venv/` | Python 3.13 virtual environment for LiteLLM |
| `~/Library/LaunchAgents/com.litellm.proxy.plist` | Auto-start agent for LiteLLM |
| `~/.zshrc` | Shell aliases added at the bottom |

---

## Uninstalling / Reverting

```bash
# Stop and remove the proxy service
launchctl unload ~/Library/LaunchAgents/com.litellm.proxy.plist
rm ~/Library/LaunchAgents/com.litellm.proxy.plist

# Remove LiteLLM
rm -rf ~/.litellm-venv ~/.litellm-config.yaml

# Remove Gemma model (reclaims 9.6 GB)
ollama rm gemma4:e4b

# Stop Ollama service
brew services stop ollama

# Remove aliases from ~/.zshrc manually
# (the block between "# Gemma 4 E4B" comments)
```

---

## Quick Troubleshooting Reference

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `claude-gemma` hangs | Proxy not running | `gemma-proxy-status` then check logs |
| `Connection refused :4000` | launchd agent failed to start | `launchctl load ~/Library/LaunchAgents/com.litellm.proxy.plist` |
| Proxy running but Gemma slow | Model loading cold | Wait 10–15s on first prompt; warm after that |
| `ollama pull` fails | Network/cert issue | See Enterprise Problem 2 and 4 above |
| Mac fans spin up during inference | Normal | Gemma uses ~5–6 GB unified memory and full GPU |
| `ModuleNotFoundError: litellm` | Wrong Python used | Use `~/.litellm-venv/bin/litellm` explicitly |
| Responses cut off | `max_tokens` too low | Add `--max_tokens 2048` or set in LiteLLM config |

---

## Performance Expectations (M1, 8 GB)

| Scenario | Speed |
|----------|-------|
| First prompt (cold load) | 10–20 seconds to first token |
| Subsequent prompts (warm) | ~15–25 tokens/second |
| Parallel with other apps | Slower; close memory-heavy apps |

For comparison, an M3 Pro with 36 GB RAM typically achieves ~60–80 tokens/second on this model.

---

*Set up on: macOS 14.5, Apple M1, 8 GB RAM — Ollama 0.23.2, LiteLLM 1.83.14, Gemma 4 E4B*
