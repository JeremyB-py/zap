## Description

Fixes BYOP Agent Mode with local Ollama providers (shell tools did not execute, multi-turn context lost) and OpenAI `gpt-5.x` routing (400 on Agent requests with tools + reasoning).

**Problem (reproduced on `origin/main` @ `5c145ccd`):**

Local models (e.g. `qwen2.5-coder`, `llama3.1`, `gemma4`) respond to agent prompts but rarely emit Ollama native `message.tool_calls` stream events. Logs show `tool_chunks=0` and `captured_tools=0` on every run. Models instead emit tool-shaped JSON (or simulated prose) in assistant `content`, which Zap displayed but did not execute. Because no real `ToolCall` / `ToolCallResult` pairs were persisted, follow-up turns sent prior messages upstream but with empty `tool_calls` / `tool_responses` in history, so the model could not recall actual command output.

A separate routing bug sent Ollama requests to `http://localhost:11434/v1/api/chat` (404) when the native API expects `http://localhost:11434/api/chat`.

OpenAI `gpt-5.4` (and similar `gpt-5*` / codex / pro models) with BYOP `api_type=OpenAi` (default) hit `/v1/chat/completions` and return HTTP 400: *"Function tools with reasoning_effort are not supported for gpt-5.4 … Please use /v1/responses instead."* Agent mode always sends tools + reasoning effort, so these models require the Responses API (`/v1/responses`), not a base URL change.

Fixes #___ <!-- replace with issue number -->

### Changes (commits `d2e025b3` .. `0e5d3c1a`)

**OpenAI routing (`fix/openai-routing`, `0e5d3c1a`)**
- Add `effective_adapter_kind_for` in [`chat_stream.rs`](app/src/ai/agent_providers/chat_stream.rs) to resolve the genai adapter from `api_type` + model id
- When provider is configured as `OpenAi` (default) and the model is `gpt-5*` / codex / pro, auto-upgrade to `OpenAIResp` so requests use `/v1/responses` instead of `/v1/chat/completions`
- Log auto-upgrade in `build_client`; update request diagnostics to show the effective adapter
- Unit tests in `adapter_routing_tests` (`gpt-5.4` upgrades, `gpt-4o` unchanged, explicit `OpenAiResp` unchanged)

**Ollama routing (`fix/Ollama_routing`)**
- Correct `AgentProviderApiType::Ollama` default `base_url` from `http://localhost:11434/v1/` to `http://localhost:11434/`
- Strip erroneous `/v1/` prefix in `normalize_endpoint_url` when `api_type` is Ollama, avoiding `/v1/api/chat` 404s

**Linux settings paste (`fix/linux_settings_paste_issue`)**
- Add Ctrl+V paste keybinding on Linux and Cut/Copy/Paste entries in the settings editor context menu (needed to configure BYOP API keys on Linux during manual testing)

**BYOP tool execution (`fix/agent`)**
- New [`content_tool_calls.rs`](app/src/ai/agent_providers/content_tool_calls.rs): parse tool-shaped JSON from assistant text when native `tool_calls` stream events are absent
- Wire extraction into `generate_byop_output` in [`chat_stream.rs`](app/src/ai/agent_providers/chat_stream.rs) after stream end when `tool_bufs` is empty; accumulate streamed chunk text (Ollama often leaves `End.captured_content` empty)
- Remap common local-model mistakes to `run_shell_command` (`echo`, `run_code_command`, wrong `write_to_long_running_shell_command` with bare `command`, unknown names with a `command` field, etc.)
- Prefer `run_shell_command` when multiple pseudo-tools appear in one response
- Support OpenAI-style `tool_calls[]` wrappers and nested `function` objects

**Multi-turn history retention**
- Ollama outbound history: skip tool-shaped JSON `AgentOutput` blobs only when structured `ToolCall` messages already exist in the conversation (avoids stripping history when extraction failed on the prior turn)
- After successful extraction and execution, history carries real `ToolCall` + `ToolCallResult` so follow-up turns can reason about command output

**Local model prompt**
- New [`prompts/system/local.j2`](app/src/ai/agent_providers/prompts/system/local.j2): short Ollama-specific system template (~500B vs ~9k `default.j2`) via `pick_template` in [`prompt_renderer.rs`](app/src/ai/agent_providers/prompt_renderer.rs)

**Tests**
- [`content_tool_calls_tests.rs`](app/src/ai/agent_providers/content_tool_calls_tests.rs): 13 unit tests for Qwen/Llama/Gemma-style JSON-in-text patterns
- [`mod_test.rs`](app/src/ai/agent_providers/mod_test.rs): BYOP provider lookup / model exposure smoke tests
- [`byop_readiness/mod_test.rs`](app/src/ai/byop_readiness/mod_test.rs): blocked readiness user-facing message smoke test
- Chat request serialization smoke tests in `chat_stream.rs`

**Note:** Commit `582600aa` also added [`.cursor/AGENTS.md`](.cursor/AGENTS.md). Consider dropping that file from this PR if it is local dev context only and not intended for upstream.

## Testing

**Automated:**
```bash
cargo test -p warp content_tool_calls
cargo test -p warp adapter_routing_tests
cargo test -p warp smoke_
cargo test -p warp byop_readiness serializer_readiness
./script/presubmit
```

**Manual (Ollama BYOP):**
1. Configure provider: `api_type=Ollama`, `base_url=http://localhost:11434/`, model e.g. `qwen2.5-coder`
2. Agent prompt: `Run echo hello and tell me the output`
   - Expect: shell command executes (approval flow if configured), not JSON-only display
   - Logs: `[byop] content_tool_extract: found=1 names=["run_shell_command"]`, `captured_tools=1` (`native_tool_chunks=0` is still expected for local models)
3. Follow-up: `What did the previous command output?`
   - Expect: model answers from prior tool results; turn-2 `message_flow` includes non-empty `tool_calls` / `tool_responses` when turn 1 ran a shell command

**Manual (OpenAI BYOP, `gpt-5.4`):**
1. Configure provider: `api_type=OpenAi` (default), `base_url=https://api.openai.com/v1/`, model `gpt-5.4`
2. Agent prompt with tool use (e.g. run a shell command)
   - Expect: no HTTP 400 about `reasoning_effort` on `/v1/chat/completions`
   - Logs: `[byop] auto-upgrade OpenAi → OpenAIResp for model=gpt-5.4`

Tested on Linux with Ollama. Reliability varies by model size; small local models may still hallucinate tool names occasionally.

## Server API dependencies
- [ ] Is this change necessary to make the client compatible with a desired server API breaking change?
- [ ] Does this change rely on a new server API?
- [ ] Is this change enabling the use of a server API on client channels that rely on the production server?

N/A. BYOP local provider path only; no Warp cloud server changes.

## Agent Mode
- [x] Zap Agent Mode - This PR was created via Zap's AI Agent Mode

## Changelog Entries for Stable

CHANGELOG-BUG-FIX: BYOP Agent Mode with Ollama and other local models now executes shell tools when models emit tool JSON in assistant text instead of native tool_calls, and preserves structured tool history for multi-turn follow-ups.
CHANGELOG-BUG-FIX: OpenAI `gpt-5.x` / codex BYOP models configured with default `OpenAi` api_type now auto-route to `/v1/responses`, fixing 400 errors when Agent mode sends function tools with reasoning effort.
CHANGELOG-BUG-FIX: Ollama BYOP default endpoint no longer appends `/v1/`, fixing 404 errors against the native `/api/chat` API.
CHANGELOG-IMPROVEMENT: Linux settings editor now supports Ctrl+V paste and context-menu Cut/Copy/Paste for configuring agent providers.
