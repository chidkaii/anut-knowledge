**TL;DR:** Register Hermes Agent as a Devin ACP agent via `~/.windsurf/acp/registry.json` — Devin Desktop launches `hermes acp` for editor-aware sessions. Requires Devin Pro/Max/Teams plan.

**Key Points:**
- Hermes Agent CLI (`hermes acp`) is already installed and passing `--check` at v0.21.2
- Devin's ACP registry lives at `~/.windsurf/acp/registry.json` (macOS/Linux)
- Third-party ACP agents require Devin **Pro, Max, or Teams** plan
- Agent binary must already be on PATH; Devin does not download distributions
- Restart Devin Desktop after enabling the agent in Settings → Agents

Tags: {#devin} {#hermes} {#acp} | Updated: 2026-09-27
Related: [devin-openrouter-acp.md](devin-openrouter-acp.md)

# Devin + Hermes Agent via ACP

## Architecture

```
User → Devin Desktop (editor/UI) → ACP → Hermes Agent (hermes acp) → LLM provider
```

| Layer | Component | Role |
|-------|-----------|------|
| Editor/UI | Devin Desktop | Agent Command Center, chat UI, file browser |
| Agent | Hermes Agent (`hermes acp`) | Coding agent with skills, memory, terminal backends |
| LLM Provider | OpenRouter / Nous Portal / OpenAI / etc. | Configured in Hermes `~/.hermes/config.yaml` |

## Why this works

Hermes Agent implements the [Agent Client Protocol](https://agentclientprotocol.com/) via `hermes acp`. Devin Desktop supports any ACP-compatible agent — it launches the agent binary and communicates over stdio. The agent's own model/provider config is used, so there is no paywall bypass involved (unlike the OpenCode+OpenRouter setup in `devin-openrouter-acp.md`).

## Setup

1. **Hermes CLI installed & on PATH**
   ```bash
   which hermes          # /home/nut-ubuntu/.local/bin/hermes
   hermes acp --check    # "Hermes ACP check OK"
   ```
   Current version: v0.21.2. Install via `curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash` if needed.

2. **Add Hermes to Devin's ACP registry**
   Edit `~/.windsurf/acp/registry.json` and add the `hermes` agent entry (already done — see file). The `cmd` is `hermes` with `args: ["acp"]`; the `archive` field is empty since Devin does not download distributions.

3. **Enable the agent in Devin**
   - Open Devin Desktop
   - `Cmd+Shift+P` / `Ctrl+Shift+P` → `Devin User Settings`
   - **Agents** tab → toggle **Hermes Agent** ON
   - Restart Devin Desktop

4. **Start a session**
   Hermes Agent appears in the agent selector (bottom-right) when starting new conversations.

## Hermes Agent for VS Code (alternative)

The community extension [hermes-agent-vs-code](https://github.com/eyan-ai/hermes-agent-vs-code) (v0.2.53) brings the same Hermes Agent into the VS Code Activity Bar with editor-aware context, structured Thinking/Action timeline, skills, session history, and approval modes. It also uses `hermes acp` as the default transport, with a CLI fallback (`hermes chat -q -v`).

Install options:
- **VS Code Marketplace:** Search "Hermes Agent" by EyanLin
- **From source:** `vsce package` then `code --install-extension hermes-agent-vs-code-0.2.53.vsix`

## Caveat

Third-party ACP agents require Devin **Pro, Max, or Teams** plan. Enterprise admins may need to enable third-party agents first.

## References

- [Devin Desktop ACP docs](https://docs.devin.ai/desktop/acp)
- [Agent Client Protocol](https://agentclientprotocol.com/)
- [Hermes Agent](https://github.com/NousResearch/hermes-agent)
- [hermes-agent-vs-code](https://github.com/eyan-ai/hermes-agent-vs-code)