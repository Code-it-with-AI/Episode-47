# Episode-47: Claude + NInfer

## Claude Code CLI with Local NInfer + Qwen 3.8 Vision on Windows

Carl and Rocky share their experiences running Claude Code against a local LLM

📺 YouTube video: https://youtu.be/w9swPQxZAw4

🏠 Code it with AI Home Page: [https://codeitwithai.com](https://codeitwithai.com/)

## Overview

This documents the tested Windows setup for using **Claude Code CLI** with **Qwen 3.8 27B NVFP4** served by **NInfer** on another machine.

``` text
Windows development PC
        |
        | Claude Code CLI / Anthropic Messages API
        v
http://<YOUR IP>:8080/v1/messages
        |
        v
NInfer Vision -> Qwen 3.8 27B NVFP4 -> RTX 5090
```

The setup was tested in September 2026.

## 1. NInfer / Qwen 3.8 Vision

Model artifact:

``` text
qwen3_8_27b_nvfp4.ninfer
```

The NInfer Qwen 3.8 NVFP4 model card documents support for text, vision, thinking, MTP speculative decoding, FP8 KV cache, and Anthropic-compatible Messages serving.

### Working `start_ninfer_vision.bat`

``` bat
@echo off
echo Starting NInfer on RTX 5090 (Vision + 131k Context + FP8 + MTP5)
ninfer-serve-vision.exe qwen3_8_27b_nvfp4.ninfer ^
  --vision ^
  --host 0.0.0.0 ^
  --port 8080 ^
  --max-context 131072 ^
  --kv-capacity 131072 ^
  --max-concurrency 1 ^
  --kv-dtype fp8 ^
  --prefill-chunk 1024 ^
  --device-state-slots 1 ^
  --host-state-slots 16 ^
  --host-kv-mib 16384 ^
  --spec mtp --draft-tokens 5 ^
  --lm-head-draft ^
  --preserve-thinking ^
  --pending-timeout-ms 600000
pause
```

Key points:

-   `--vision` enables multimodal input.
-   `--host 0.0.0.0` makes the server reachable from another LAN
machine.
-   `--port 8080` is the HTTP port used by Claude Code.
-   `--max-context 131072` and `--kv-capacity 131072` provide the 131K
context configuration.
-   `--max-concurrency 1` is appropriate for the single-user agent
workload.
-   `--kv-dtype fp8` uses FP8 KV storage.
-   `--spec mtp --draft-tokens 5` enables MTP speculative decoding with
five draft positions.
-   `--lm-head-draft` selects the optimized proposal head.
-   `--preserve-thinking` preserves thinking content for
Anthropic-compatible clients.

Allow inbound TCP 8080 through Windows Firewall as appropriate for the LAN.

NInfer sources:

-   https://github.com/Neroued/ninfer
-   https://github.com/Neroued/ninfer/blob/master/model-cards/Qwen3.8-27B-nvfp4-NInfer/README.md
-   https://github.com/Neroued/ninfer/blob/master/docs/cli.md

## 2. Verify NInfer's Anthropic endpoint

From the development PC:

``` powershell
curl.exe http://<YOUR IP>:8080/v1/messages `
  -H "Content-Type: application/json" `
  -d '{ "model": "qwen3_8_27b_nvfp4", "max_tokens": 100, "messages": [{ "role": "user", "content": "Say hello in one sentence." }] }'
```

The tested response contained Anthropic-style `thinking` and `text` content blocks. This verifies networking, the `/v1/messages` endpoint, the model ID, and thinking support before Claude Code is involved.

## 3. Install Claude Code CLI

Anthropic's recommended native Windows installer is:

``` powershell
irm https://claude.ai/install.ps1 | iex
```

WinGet is also supported:

``` powershell
winget install Anthropic.ClaudeCode
```

Git for Windows is recommended on native Windows. Current Claude Code can use PowerShell when Git Bash is unavailable.

Sources:

-   https://code.claude.com/docs/en/quickstart
-   https://code.claude.com/docs/en/setup

## 4. Route Claude Code to NInfer

Claude Code officially supports `ANTHROPIC_BASE_URL` for routing requests through a proxy or gateway.

For a temporary PowerShell session:

``` powershell
$env:ANTHROPIC_BASE_URL="http://<YOUR IP>:8080"
$env:ANTHROPIC_AUTH_TOKEN="local"

$env:ANTHROPIC_MODEL="qwen3_8_27b_nvfp4"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="qwen3_8_27b_nvfp4"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="qwen3_8_27b_nvfp4"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="qwen3_8_27b_nvfp4"

$env:MAX_THINKING_TOKENS="8000"

claude
```

Use:

``` text
http://<YOUR IP>:8080
```

not `http://<YOUR IP>:8080/v1`; Claude Code constructs the Messages path.

`local` is a placeholder bearer token for this unauthenticated NInfer setup. Replace it if NInfer is later configured to require authentication.

Sources:

-   https://code.claude.com/docs/en/env-vars
-   https://docs.anthropic.com/en/docs/claude-code/llm-gateway

## 5. Thinking-budget issue

The first Claude Code request reached NInfer but returned HTTP 400:

``` text
effective output capacity after the thinking budget must fit the complete
control suffix and one post-close model token
```

The fix was:

``` powershell
$env:MAX_THINKING_TOKENS="8000"
```

After restarting Claude Code, NInfer logged successful requests with characteristics such as:

``` text
anthropic_messages
stream
max_tokens=32000
tools=20
thinking=on
thinking_budget=8000
```

For this tested NInfer/Qwen configuration, keep `MAX_THINKING_TOKENS=8000`.

### Claude Code 2.1.273+: set reasoning effort to Medium

After Claude Code updated to **2.1.273**, the previously working configuration began returning:

```text
API Error: 400 reasoning effort 'high' is not supported by the loaded chat template
```

This is separate from the thinking-budget error above. Claude Code was sending a **High** reasoning-effort setting that the loaded NInfer/Qwen 3.8 chat template rejected.

Inside Claude Code, run:

```text
/effort
```

and select **Medium**.

Claude Code reports that this selection is saved as the default for new sessions. After changing from High to Medium, Qwen 3.8 through NInfer immediately worked again.

The tested combination with Claude Code 2.1.273 is:

```text
Claude Code effort:  Medium
MAX_THINKING_TOKENS: 8000
Model:                qwen3_8_27b_nvfp4
NInfer context:       131072
```

If a future Claude Code update resets the effort level to High and this error returns, run `/effort` again and select **Medium**.

## 6. Make the variables persistent

Set Windows **User** environment variables:

``` powershell
[Environment]::SetEnvironmentVariable("ANTHROPIC_BASE_URL","http://<YOUR IP>:8080","User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_AUTH_TOKEN","local","User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_MODEL","qwen3_8_27b_nvfp4","User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_SONNET_MODEL","qwen3_8_27b_nvfp4","User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_OPUS_MODEL","qwen3_8_27b_nvfp4","User")
[Environment]::SetEnvironmentVariable("ANTHROPIC_DEFAULT_HAIKU_MODEL","qwen3_8_27b_nvfp4","User")
[Environment]::SetEnvironmentVariable("MAX_THINKING_TOKENS","8000","User")
```

Administrator privileges are not required for User-scope variables. The `"User"` argument explicitly targets the current user's environment.

Open a new normal PowerShell window and verify:

``` powershell
[Environment]::GetEnvironmentVariable("ANTHROPIC_BASE_URL","User")
[Environment]::GetEnvironmentVariable("ANTHROPIC_MODEL","User")
[Environment]::GetEnvironmentVariable("MAX_THINKING_TOKENS","User")
```

Expected:

``` text
http://<YOUR IP>:8080
qwen3_8_27b_nvfp4
8000
```

## 7. Normal use

Start NInfer on the 5090 machine, then on the development PC:

``` powershell
cd C:\Path\To\Your\Repository
claude
```

A useful first read-only test:

``` text
Explore this project and explain its architecture to me. Do not modify anything.
```

Watch the NInfer console. Server-side `anthropic_messages` traffic is a more authoritative confirmation of the backend than asking the language model to identify itself.

## 8. Pasting images into Claude Code CLI

The Qwen 3.8 NInfer server is started with `--vision`, so the backend can process images. Claude Code CLI on Windows also supports attaching an image directly from the clipboard.

**Do not use `Ctrl+V` for a clipboard image.** In Claude Code CLI, use:

```text
Alt+V
```

A convenient screenshot workflow is:

1. Press `Win+Shift+S` and capture the desired region with Windows Snipping Tool.
2. Return to the Claude Code prompt.
3. Press `Alt+V`.
4. Claude Code attaches the clipboard image to the prompt.
5. Add the question or instruction that should accompany the image and submit it.

This was tested successfully with the local Claude Code -> NInfer -> Qwen 3.8 Vision configuration documented here.

Because the NInfer server is running `ninfer-serve-vision.exe` with `--vision`, the attached image can be forwarded through the Anthropic-compatible request to the vision-capable Qwen backend.

Claude Code keyboard/interactive documentation:

- https://code.claude.com/docs/en/interactive-mode

## 9. Context

The server is configured for 131,072 tokens. Claude Code itself consumes significant context for system instructions, tool definitions, environment data, conversation history, and tool results.

During testing, an early real agent request showed approximately:

``` text
prompt=16670
tools=20
```

The 131K context is therefore useful for sustained coding sessions.

### Tell Claude Code the actual context window

Because `qwen3_8_27b_nvfp4` is a custom model that is not in Claude Code's model catalog, Claude Code may assume a 200K context window. The NInfer server in this configuration is actually limited to 131,072 tokens:

```text
--max-context 131072
--kv-capacity 131072
```

Set the Windows User environment variable: 

```powershell
[Environment]::SetEnvironmentVariable("CLAUDE_CODE_MAX_CONTEXT_TOKENS","131072","User")
```

Open a new PowerShell window after setting this variable. Claude Code will then use 131,072 tokens as the model's context limit for auto-compaction instead of its default 200K assumption for an unknown model.

## 10. Why CLI works

Anthropic documents `ANTHROPIC_BASE_URL` as an endpoint override for Claude Code:

https://code.claude.com/docs/en/env-vars

Anthropic's LLM gateway documentation likewise shows `ANTHROPIC_BASE_URL` pointing to an Anthropic-format gateway:

https://docs.anthropic.com/en/docs/claude-code/llm-gateway

NInfer supplies an Anthropic-compatible Messages API, so the path is direct:

``` text
Claude Code CLI
      |
      | ANTHROPIC_BASE_URL
      v
NInfer /v1/messages
      |
      v
Qwen 3.8
```

No LiteLLM, Ollama, or OpenAI-to-Anthropic translation proxy is required.

## 11. Why this does not work directly in Claude Desktop

We tested Claude Desktop's integrated Code interface.

Windows correctly stored:

``` text
ANTHROPIC_BASE_URL=http://<YOUR IP>:8080
ANTHROPIC_MODEL=qwen3_8_27b_nvfp4
MAX_THINKING_TOKENS=8000
```

But a Desktop Local session reported:

``` text
ANTHROPIC_BASE_URL=https://api.anthropic.com
ANTHROPIC_MODEL=qwen3_8_27b_nvfp4
MAX_THINKING_TOKENS=8000
```

We then used Desktop's **Edit local environment** dialog and tried to set:

``` text
ANTHROPIC_BASE_URL=http://<YOUR IP>:8080
```

Desktop explicitly rejected it:

``` text
"ANTHROPIC_BASE_URL" is managed by Claude Desktop and cannot be overridden.
```

This matches Anthropic's current Desktop documentation. Its **What's not available in Desktop** section says third-party providers are not generally available there: Desktop connects to Anthropic's API by default, while enterprise deployments can configure supported gateway/provider options.

Source:

https://code.claude.com/docs/en/desktop

The same page says the CLI supports third-party providers while Desktop uses Anthropic's API by default (with enterprise exceptions).

Therefore the Desktop failure is **not** caused by Windows permissions, administrator elevation, NInfer networking, or incorrectly stored User variables. Desktop manages the inference endpoint for its integrated Code sessions and blocks the local environment from overriding `ANTHROPIC_BASE_URL`.

For this NInfer/Qwen setup, **use Claude Code CLI**.

## 12. CLI versus Desktop

Desktop adds GUI/workflow features including parallel sessions, automatic Git worktree isolation, integrated terminal/editor panes, visual diff review, app previews, side chats, computer use, scheduled tasks, and PR monitoring.

Anthropic says the underlying Claude Code agentic loop is the same across interfaces; what changes is execution location and interface.

Sources:

-   https://code.claude.com/docs/en/desktop
-   https://code.claude.com/docs/en/how-claude-code-works

For this setup, the CLI's decisive advantage is support for the custom `ANTHROPIC_BASE_URL`.

## 13. Remote Control limitation with a custom NInfer endpoint

Claude Code includes a Remote Control feature that allows a running Claude Code session on the development computer to be accessed from the Claude mobile app or other Claude interfaces.

Current Claude Code exposes Remote Control from the command line:

``` powershell
claude --remote-control
```

The current Claude mobile app also instructs users to pair a computer with:

``` powershell
claude rc
```

### Remote Control requires a Claude.ai login

When `claude rc` was first tested, Claude Code reported:

``` text
Error: You must be logged in to use Remote Control.

Remote Control is only available with claude.ai subscriptions.
Please use `/login` to sign in with your claude.ai account.
```

Running Claude Code and entering:

``` text
/login
```

successfully authenticated the CLI with the same Claude.ai account used by the mobile app.

The Qwen/NInfer configuration remained active after login. Claude Code continued to show the configured model as `qwen3_8_27b_nvfp4`, while the Claude.ai login was stored separately.

### Custom API endpoints are explicitly blocked

After successfully logging in, running:

``` powershell
claude rc
```

produced the definitive error:

``` text
Error: Remote Control is only available when using Claude via api.anthropic.com.
ANTHROPIC_BASE_URL is set and does not point at api.anthropic.com, so this
session is using a custom endpoint — unset it (or run in a shell without it)
to use Remote Control.
```

This means Remote Control cannot be used with the NInfer/Qwen configuration documented here.

The conflict is fundamental to the current Claude Code implementation.

Normal local configuration:

``` text
Claude Code CLI
      |
      | ANTHROPIC_BASE_URL
      v
NInfer
      |
      v
Qwen 3.8
```

Remote Control requires Claude Code to use Anthropic's API endpoint:

``` text
Claude mobile app
      |
      v
Claude Remote Control
      |
      v
Claude Code CLI
      |
      v
api.anthropic.com
```

Claude Code deliberately refuses to enable Remote Control when `ANTHROPIC_BASE_URL` points somewhere other than `api.anthropic.com`.

### Why removing ANTHROPIC_BASE_URL is not a solution

It is possible to launch a shell without the custom `ANTHROPIC_BASE_URL` and use Remote Control normally. However, doing so also removes the setting that routes inference to NInfer.

The resulting session would use Anthropic's hosted models instead of the local Qwen model, defeating the purpose of this configuration.

The desired architecture is:

``` text
Claude mobile app
      |
      v
Remote Control
      |
      v
Claude Code on development PC
      |
      v
NInfer
      |
      v
Qwen 3.8
```

That architecture is not supported by the current Claude Code Remote Control implementation.

### Older Claude Code versions

Older Claude Code releases reportedly allowed Remote Control while using a custom `ANTHROPIC_BASE_URL`. Downgrading was considered but rejected for this setup.

The current Claude Code version is already working correctly with NInfer/Qwen and includes newer agent, tool, compatibility, and context-management behavior. The current mobile app and Remote Control
service have also evolved, so compatibility between an old CLI and the current mobile Remote Control implementation cannot be assumed.

Maintaining an obsolete Claude Code installation solely to bypass the current endpoint restriction would introduce additional version-specific behavior and maintenance risk.

### Conclusion

For this configuration, use the current **Claude Code CLI directly with NInfer/Qwen** and do not enable Claude Remote Control.

The known-good local inference path remains:

``` text
Claude Code CLI
      |
      | ANTHROPIC_BASE_URL=http://<NINFER-IP>:8080
      v
NInfer
      |
      v
Qwen 3.8 27B NVFP4
      |
      v
RTX 5090
```

Remote access to the CLI, if needed in the future, should use a mechanism that does not alter Claude Code's inference endpoint, such as remote terminal access to the development computer.

Claude Code Remote Control documentation:

-   https://code.claude.com/docs/en/remote-control

## 14. Troubleshooting

### Claude Code goes to Anthropic instead of NInfer

``` powershell
$env:ANTHROPIC_BASE_URL
```

Expected:

``` text
http://<YOUR IP>:8080
```

Open a new terminal after changing User variables.

### NInfer receives nothing

Repeat the direct `/v1/messages` curl test. If it fails, troubleshoot NInfer binding, IP address, port 8080, and Windows Firewall before Claude Code.

### `thinking_budget_capacity_insufficient`

``` powershell
$env:MAX_THINKING_TOKENS
```

Expected: `8000`. Restart Claude Code after changing it.

### `reasoning effort 'high' is not supported by the loaded chat template`

This appeared after updating to Claude Code 2.1.273. Inside Claude Code, run:

```text
/effort
```

and select **Medium**. Claude Code saves the selection as the default for new sessions. This is distinct from `MAX_THINKING_TOKENS`; both settings are part of the tested working configuration.

### Wrong model

``` powershell
$env:ANTHROPIC_MODEL
```

Expected: `qwen3_8_27b_nvfp4`.

Use the NInfer server log as the authoritative check.

### Desktop still says Sonnet

Expected for this configuration. Desktop manages its inference endpoint and blocks `ANTHROPIC_BASE_URL` in the Local Environment editor. Use the standalone CLI.

## 15. Reference documentation

### NInfer

-   https://github.com/Neroued/ninfer
-   https://github.com/Neroued/ninfer/blob/master/model-cards/Qwen3.8-27B-nvfp4-NInfer/README.md
-   https://github.com/Neroued/ninfer/blob/master/docs/cli.md

### Claude Code

-   https://code.claude.com/docs
-   https://code.claude.com/docs/en/setup
-   https://code.claude.com/docs/en/quickstart
-   https://code.claude.com/docs/en/env-vars
-   https://code.claude.com/docs/en/model-config
-   https://docs.anthropic.com/en/docs/claude-code/llm-gateway
-   https://code.claude.com/docs/en/how-claude-code-works
-   https://code.claude.com/docs/en/desktop

## Known-good configuration summary

``` text
NInfer host:      <YOUR IP>
Port:             8080
Model:            qwen3_8_27b_nvfp4
Context:          131072
KV cache:         FP8
Vision:           enabled
Spec decoding:    MTP5
Thinking:         preserved
Claude effort:     Medium (Claude Code 2.1.273 tested)

ANTHROPIC_BASE_URL=http://<YOUR IP>:8080
ANTHROPIC_AUTH_TOKEN=local
ANTHROPIC_MODEL=qwen3_8_27b_nvfp4
ANTHROPIC_DEFAULT_SONNET_MODEL=qwen3_8_27b_nvfp4
ANTHROPIC_DEFAULT_OPUS_MODEL=qwen3_8_27b_nvfp4
ANTHROPIC_DEFAULT_HAIKU_MODEL=qwen3_8_27b_nvfp4
MAX_THINKING_TOKENS=8000
CLAUDE_CODE_MAX_CONTEXT_TOKENS=131072
```
