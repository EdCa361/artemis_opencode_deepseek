# Running ARTEMIS on OpenCode Go (DeepSeek V4.1 Flash)

Adaptation and setup guide for connecting [google/artemis](https://github.com/google/artemis) to
[OpenCode Go](https://opencode.ai/docs/go) — the OpenAI-compatible gateway from the OpenCode team —
instead of Google Gemini as the model provider.

Verified on Windows 11 against `google/artemis` @ `351ca84` (September 2026) with
`deepseek-v4.1-flash` (multimodal, tool calling, 1M-token context).

> **Quick start (pre-adapted fork — recommended):** this branch already contains every
> change described in §5, so you can skip the manual port:
>
> ```powershell
> git clone https://github.com/EdCa361/artemis_opencode_deepseek.git C:\tools\artemis
> ```
>
> The repository's default branch IS this branch (alternatively:
> `git clone -b feat/opencode-go-deepseek ...`). Then follow §2 (prerequisites), run the
> commands in §3 (toolchain install — still required), and continue from §4. Skip §5
> (already applied). Each user needs **their own OpenCode Go subscription and API key** —
> do not share keys.

---

## 1. Why OpenCode Go

- The **Gemini free tier allows 20 requests/day/model** (`generate_content_free_tier_requests`).
  A single ARTEMIS task consumes 10–30+ model calls (one per UI step, plus the step
  summarizer lens), so the free tier is not viable for real device automation.
- **OpenCode Go ($10/month)** provides roughly **26,000 requests / 5 h and ~130,000/month**
  for DeepSeek V4.1 Flash, plus prompt caching (measured ~44–60% cache-hit ratios in our runs).
- The Go gateway exposes an **OpenAI-compatible endpoint**, and its documentation
  explicitly covers external agents: clients should identify themselves with their own
  User-Agent and send a stable `x-opencode-session` header (both implemented by this branch).
- Measured example run (battery-level task, flash profile): 7 LLM calls, ~78k prompt tokens,
  60% cached, all requests to `https://opencode.ai/zen/go/v1/chat/completions` → `200 OK`.

> Plan note: OpenCode Go is documented for "OpenCode and other coding agents". ARTEMIS is a
> mobile-automation agent; its traffic is a normal stream of chat-completion requests, but
> mind the described intended use. The plan's monthly per-model budget is shared with your
> OpenCode editor sessions.

## 2. Prerequisites

| Item | Notes |
| --- | --- |
| Windows 10/11 + PowerShell | `start.bat` exists for the UI; this guide uses manual, non-interactive steps |
| Python ≥ 3.12 | Provisioned automatically by `uv` |
| [uv](https://astral.sh/uv) | Astral's Python package manager |
| ADB | `winget install Google.PlatformTools`, or Android SDK platform-tools in PATH |
| ffmpeg + scrcpy | `winget install Gyan.FFmpeg Genymobile.scrcpy` (optional: recording/replay) |
| Android device/emulator | USB debugging enabled; verified with `adb devices -l` |
| OpenCode Go key | Subscribe at <https://opencode.ai/auth>, copy the API key |

## 3. Fresh install (original repository)

```powershell
git clone https://github.com/google/artemis.git C:\tools\artemis
Set-Location C:\tools\artemis

# uv bootstrap (skip if uv is already installed)
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Python runtime + dependencies (first sync downloads ~2 GB, one-time cost)
uv sync

# Configuration template
Copy-Item .env.example .env

# Optional video toolchain (screen recording, visual replays)
winget install --id Gyan.FFmpeg -e --accept-source-agreements --accept-package-agreements
winget install --id Genymobile.scrcpy -e --accept-source-agreements --accept-package-agreements

# Environment check
uv run artemis doctor
```

> Do not run `start.bat` for a scripted setup — it is interactive and ends by launching the
> web UI server in the foreground. Use it later when you want the console (http://localhost:8000).

## 4. Connect to OpenCode Go

Append to `C:\tools\artemis\.env`:

```ini
OPENAI_API_KEY=<your OpenCode Go API key>
OPENAI_BASE_URL=https://opencode.ai/zen/go/v1
```

Why `OPENAI_*`: ARTEMIS' `custom` provider rides the OpenAI-compatible path
(`langchain-openai`) and reads exactly these environment variables. `settings.py` loads
`.env` into the process environment at import time, so nothing else needs wiring.

> Do NOT keep any Google/Gemini key in `.env` — this adaptation needs no Google credential.
> (The upstream default object detector used a Gemini ER model; see §6.)

Handy: list the models served by your key:

```powershell
curl.exe -s -H "Authorization: Bearer $env:OPENAI_API_KEY" https://opencode.ai/zen/go/v1/models
```

Model IDs used by the shipped config: `deepseek-v4.1-flash` (primary) and
`deepseek-v4-flash` (fallback / light tasks). Other Go models use different endpoint
shapes (`/responses` for GPT/Grok, `/messages` for Anthropic-style); the OpenAI-compatible
`/chat/completions` models (DeepSeek, GLM, Kimi, LongCat, MiMo, MiniMax) work with this
adaptation as-is.

## 5. Apply this adaptation

Everything lives in this branch (`feat/opencode-go-deepseek`, 2 commits on top of `351ca84`):

- **Use the branch**: `git checkout feat/opencode-go-deepseek`
- **Cherry-pick into your own fork**: `git cherry-pick <code-commit>` then the docs commit
- **Manual port** — the changes are:

| # | File | Change |
| --- | --- | --- |
| 1 | `artemis/services/llm.py` | New `get_lens_llm()`: resolves `"provider/model"` strings through `ModelFactory`; plain names keep Google behavior |
| 2 | `artemis/agents/flash/summarizer.py` | Step-summarizer lens uses `get_lens_llm`; fixed the fallback node lookup (`is_utils` flag) |
| 3 | `artemis/memory/chunking.py` | Chunk-capsule lens uses `get_lens_llm` |
| 4 | `artemis/llm/router.py` | For `custom`/`ollama`/`vllm` base URLs containing `opencode.ai`: sends `User-Agent: artemis-mobile-agent/1.0` + `x-opencode-session` (gateway docs requirement) |
| 5 | `artemis/sdk/agent.py` | Skips the hardcoded Gemini pre-warm when the primary models are not Google (stops quota-burning pings and 429 retry noise) |
| 6 | `artemis/config/llm.py` | Pro-profile lightweight judges (pixel safety net, planner validation) default to DeepSeek instead of `gemini-3.5-flash-lite` |
| 7 | `config/artemis.jsonc` | Working configuration for OpenCode Go (see §6) |

## 6. Configuration reference (`config/artemis.jsonc`)

```jsonc
"default": { "provider": "custom", "model": "deepseek-v4.1-flash",
             "fallback": { "provider": "custom", "model": "deepseek-v4-flash" } },
"nodes": {
  "hopper":          { "provider": "custom", "model": "deepseek-v4-flash" },
  "object_detector": { "provider": "custom", "model": "deepseek-v4.1-flash" }
},
"agent": {
  "flash": { "explorer_mode": "pro",
             "step_summarizer": { "model": "custom/deepseek-v4.1-flash" } },
  "pro":   { "explorer": { "mode": "pro" } },
  "memory": { "chunking": { "model": "custom/deepseek-v4.1-flash" } }
}
```

**Why explorer tiers are `pro`:** the `flash` tier delegates directly to the object
detector, which upstream documents as requiring Gemini ER models
(`gemini-robotics-er-2-preview`) for sub-pixel coordinate grounding. The `pro`/`ultra`
tiers run multi-turn ReAct grounding loops on the configured provider instead.

**Grounding caveat:** with no Gemini ER model, visual coordinate detection runs on
DeepSeek V4.1 Flash. Element-index clicks — the default interaction path driven by the
accessibility tree — are unaffected. Custom/Canvas/Flutter surfaces without a usable UI
tree may see lower grounding precision. If you need the upstream precision there, restore
the `google` object detector and add a Google key.

## 7. Verify

```powershell
uv run artemis doctor     # expect "Status: Ready"; the OpenAI-compatible key is accepted
uv run artemis stop       # restart the daemon so it re-reads .env / artemis.jsonc
uv run artemis run "Open Settings and tell me the current battery level" --profile flash
```

Expected observations:

- `Planner: custom/deepseek-v4.1-flash (fallback: custom/deepseek-v4-flash)`
- All requests to `https://opencode.ai/zen/go/v1/chat/completions` return `200 OK`
- Closing summary line: `LLM usage ... custom:deepseek-v4.1-flash: N calls, cached_ratio=...`
- Session traces + replay video under `C:\tools\artemis\traces\<session-id>\`

## 8. OpenCode (editor) integration — MCP + skill

1. Add to the global `~/.config/opencode/opencode.json` (inside the existing `mcp` object):

```json
"artemis": {
  "type": "local",
  "command": ["C:\\tools\\artemis\\.venv\\Scripts\\python.exe", "-m", "mcp_server"],
  "enabled": true,
  "environment": {
    "ARTEMIS_DESKTOP_NOTIFY": "true",
    "PYTHONPATH": "C:\\tools\\artemis",
    "PYTHONUNBUFFERED": "1"
  }
}
```

2. Install the Mobile Testing Mindset as an OpenCode skill: copy `mcp_server/rules.md` to
   `~/.config/opencode/skills/artemis-mobile-testing/SKILL.md`, prepending this frontmatter:

```yaml
---
name: artemis-mobile-testing
description: Drive real Android devices/emulators through the ARTEMIS MCP tools
  (mobile_run_task, mobile_manage_task, mobile_get_device_state, mobile_inspect_trace,
  mobile_diagnose) to explore app flows, capture screenshots/logcat, reproduce bugs on
  hardware, and author device-verified E2E tests. Use ONLY when working with ARTEMIS or
  real-device Android testing.
license: Apache-2.0
---
```

3. Restart OpenCode (config loads at startup). The `mobile_*` tools should be available;
   ask the agent to run `mobile_diagnose` to verify the connection end to end.

## 9. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `429 RESOURCE_EXHAUSTED ... free_tier` in task logs | A Google-backed call hit the exhausted free tier. This adaptation removes all Google calls; if these appear, check that nodes use the custom provider and that the pre-warm skip (§5 item 5) is applied. |
| `Task failed:` with an empty reason | The CLI does not print the failure reason. Read `traces\<session>\stdout.log` — the real error is there. |
| Grounding feels off on custom UIs | See §6 caveat; try explorer `ultra`, or restore a Gemini key for the object detector only. |
| `start.bat` seems to hang | It launches the UI server in the foreground by design. Use `uv run artemis doctor` / `run` for CLI work. |
| Device not detected | `adb devices -l`; enable USB debugging; `uv run artemis doctor` lists ordered fixes; `mobile_diagnose` (via MCP) can self-heal device locks and ADB keys. |
| First `uv sync` takes very long | It downloads ~2 GB of Python dependencies. One-time cost. |

## 10. Keeping in sync with upstream

This branch is based on `google/artemis@351ca84`. If your clone's `origin` is this fork,
add the upstream remote once and rebase:

```powershell
git remote add upstream https://github.com/google/artemis.git   # once
git fetch upstream main
git rebase upstream/main   # resolve conflicts in the 7 touched files if any
```

The changes are intentionally small and generic (multi-provider lenses, gateway headers,
pre-warm guard). Consider proposing them upstream so the adaptation becomes unnecessary.
