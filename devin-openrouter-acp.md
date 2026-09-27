**TL;DR:** Bypass Devin's paywalled model catalog by routing LLM calls through Opencode + OpenRouter via ACP protocol — architecture, setup, and plan requirements.

**Key Points:**
- LLM routing เกิดขึ้นที่ layer ของ Opencode (มี OpenRouter key ของตัวเอง) ไม่ใช่ Devin
- Devin เห็นแค่ agent protocol ไม่เห็น model catalog → หลบ paywall
- ต้องใช้ Devin Pro, Max, หรือ Teams plan สำหรับ third-party ACP agents
- ตรวจ key ที่ `~/.local/share/opencode/auth.json` และ registry ที่ `~/.windsurf/acp/registry.json`

Tags: {#devin} {#openrouter} {#acp} | Updated: 2026-09-28
Related: [devin-hermes-acp.md](devin-hermes-acp.md)
# Devin + OpenRouter via Opencode ACP

## Architecture

```
User → Devin Desktop (editor/UI) → ACP → Opencode (agent) → OpenRouter (LLM)
```

| Layer | Component | Role |
|-------|-----------|------|
| Editor/UI | Devin Desktop | Agent Command Center, chat UI, file browser |
| Agent | Opencode (`opencode acp`) | Coding agent, calls OpenRouter API |
| LLM Provider | OpenRouter | Unified multi-model API (BYO key) |

## Why this works

Devin's native model catalog is paywalled because model routing is server-side on Cognition's cloud. By making Opencode the agent (which owns its own OpenRouter API key), **LLM routing happens at Opencode's layer, not Devin's** — bypassing the paywall entirely. Devin only sees the agent protocol, not the model catalog.

## Setup

1. Opencode already connected to OpenRouter — `~/.local/share/opencode/auth.json` contains the OpenRouter API key.
2. ACP registry at `~/.windsurf/acp/registry.json` registers Opencode as a Devin-compatible agent.
3. Enable OpenCode agent in `Devin Settings → Agents`.
4. Opencode runs as ACP agent via `opencode acp`.

## Caveat

Third-party ACP agents require Devin **Pro, Max, or Teams** plan. Devin-native features like the **Generate Commit** button are also paywalled to paid plans only — the ACP setup only bypasses the model catalog, not Devin's UI feature restrictions. For commit messages without upgrading, ask OpenCode via chat (`/commit` or prompt it to generate a commit message).

## References

- [OpenCode + OpenRouter integration](https://openrouter.ai/docs/cookbook/coding-agents/opencode-integration)
- [Devin Desktop ACP docs](https://docs.devin.ai/desktop/acp)
- [Agent Client Protocol](https://agentclientprotocol.com/)