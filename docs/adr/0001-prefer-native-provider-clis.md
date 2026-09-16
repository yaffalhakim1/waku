# ADR 0001: Prefer native (Rust) provider CLIs to cut resident memory

- **Status:** Accepted
- **Date:** 2026-09-15
- **Deciders:** yaffalhakim1
- **Context:** Kerenzikov desktop app, provider layer (`crates/waku-core/src/driver/`)

## Context

Kerenzikov's own footprint is small (reported ~50 MB). The memory problem comes
from the **provider CLIs it drives**, not from the app. Each provider is a
long-lived child process spanning the whole session (`docs/providers.md`,
"Runtime lifetime in the app"), so its cost is resident for as long as the task
is open.

Measured on the author's Windows machine:

| Process | On disk | Observed RSS |
|---|---|---|
| `opencode.exe` | 171 MB | **583 MB + 8 MB** (two processes) |
| `claude.exe` | 235 MB | not measured |
| `codex.exe` | 284 MB | not measured (native, small heap expected) |

Both `opencode` and `claude` are JavaScript runtimes with the app embedded
(Bun and Node respectively). A JS runtime baseline plus JIT code, heap, and the
LSP/helper processes a coding agent spawns is what produces the observed
figures. This is inherent to the architecture, not a leak.

The user's requirement: keep Kerenzikov's provider integrations **at least as
capable as OpenCode**, but stop paying a JavaScript runtime per session.

## Decision

**Prefer providers whose CLI is a compiled native binary when memory matters,
and make Codex CLI the default local agent.**

Codex CLI is Rust (openai/codex is ~63 MB Rust vs ~100 KB TypeScript). It is also
the *most complete* driver Kerenzikov has:

| Capability | Codex | OpenCode |
|---|---|---|
| Transport | JSON-RPC over stdio | HTTP + SSE (extra local server) |
| Interactive approvals | yes (only real approval channel) | yes |
| Rewind + branch | yes, native `thread/rollback` / `thread/fork` | yes, via fork |
| Mid-turn steering | yes, with server-side turn check | yes |
| Model discovery | yes (`model/list`, paged) | yes |
| Computer Use | yes | yes |
| Extra resident server | no | yes |

So this is a capability **win**, not a trade.

## Consequences

### Positive

- Removes a JavaScript runtime and its spawned helpers from the resident set.
- Codex is the best-integrated driver, not a compromise.
- One fewer moving part: no local HTTP server per session.

### Negative / risks

- **Different model family.** OpenCode is model-agnostic and can point at any
  provider; Codex is OpenAI-wire-native. This fork routes it at the Kenari
  gateway, which serves many models over the OpenAI Responses wire, so the
  lock-in is mostly the *wire*, not the model.
- **Not a clean benchmark.** Memory and output quality change together, so
  comparing "OpenCode" to "Codex" is not a controlled experiment. Compare
  memory alone, and judge quality separately.
- **Rust binary is not automatically small.** `codex.exe` is 284 MB on disk;
  what matters is *heap*, which should be far below a JS runtime's. Measure
  before claiming a win.

### Neutral

- Providers whose CLI is TypeScript (Pi / Oh My Pi) or Node (Claude, Copilot,
  Amp) are unaffected and can still be used; this ADR only sets the default.
- Crush (Go) is a native alternative but has no Kerenzikov driver, so adopting
  it would cost the app integration.

## Language survey (reference)

| CLI | Language | Native | Kerenzikov driver |
|---|---|---|---|
| Codex | Rust | yes | yes (best) |
| Crush | Go | yes | **no** |
| OpenCode | TS / Bun | no | yes |
| Claude Code | TS / Node | no | yes |
| Copilot CLI | Node | no | yes |
| Amp | TS | no | yes |
| Pi / Oh My Pi | TypeScript | no | yes |

## Local configuration applied (2026-09-15)

Codex CLI 0.154.0 installed via Scoop (native `rust-v0.154.0` package).

- `~/.codex/config.toml` — added `[model_providers.kenari]` (base
  `https://kenari.id/v1`, `wire_api = "responses"`, `env_key = KENARI_API_KEY`),
  `[agents]` defaults, and made kenari the default provider.
- `KENARI_API_KEY` set at **User** environment scope (not written to disk).
- Profiles migrated to the post-0.134 separate-file form:
  `kenari.config.toml`, `kenari-kimi.config.toml`, `kenari-deepseek.config.toml`,
  `mimo.config.toml`, `mimo-pro.config.toml`.
  The legacy `[profiles.*]` tables were removed because 0.134+ rejects them and
  refused to load the entire config.
- Master rules copied to `~/.codex/AGENTS.md` (global scope).
- Custom agents `~/.codex/agents/techlead.toml` and `backend.toml`.

### Pre-existing breakage fixed

The `xiaomi` provider used `wire_api = "chat"`, removed in Codex 0.134. It made
the **entire** config unloadable, so no profile worked. Switched to
`"responses"`; the provider still needs `XIAOMI_MIMO_API_KEY` set to function.

## Verification

- `codex exec` returns `KENARI OK` on default profile (glm-5-3-flash).
- `codex exec --profile kenari-kimi` returns correctly (kimi-k2-7-code).
- `~/.codex/AGENTS.md` loads: agent correctly recited the Tech Lead / Backend
  roles and the WORK-vs-PERSONAL routing rule.
- All seven config files parse as valid TOML.

**Not yet verified:** custom agent files are valid and present, but `codex exec`
does not expose the subagent spawn tool, so end-to-end subagent invocation is
unproven. Verify in an interactive TUI session with `/agent`.

## Open question for the repo

Kerenzikov's agent *profiles* are provider-specific. `supports_agent_presets`
(`crates/waku-protocol/src/model.rs`) returns true only for
`DeepSeek | OpenCode | OpenCode2`; Codex returns false. The mobile client mirrors
this in `apps/mobile/src/app/new-task.tsx`. So if the fork wants Codex to offer
agent compositions, that is a **separate code change**, not a config change.
Codex's own subagent roles (`~/.codex/agents/*.toml`) are a CLI-level concept
and are not surfaced through Kerenzikov's profile picker.
