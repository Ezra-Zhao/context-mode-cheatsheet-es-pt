# 🧠 Context Mode — Quick Guide (English)

> Commands and figures verified against [mksglu/context-mode](https://github.com/mksglu/context-mode) README and BENCHMARK.md (2026-10-12).
> What it is: an MCP server + hooks that keep raw tool output out of your context window. Upstream calls it "the other half of the context problem."

---

## 0. Why you need it

Every MCP tool call dumps raw data into the context window:

- One Playwright snapshot: **56 KB**
- Twenty GitHub issues: **59 KB**
- One access log: **45 KB**

After ~30 minutes, ~40% of the context is gone — and context is tokens, and tokens are money. When the agent compacts the conversation to free space, it also forgets which files it was editing and what you last asked.

Context Mode's own benchmark, full session: **315 KB of raw output becomes 5.4 KB (98% less)**, and sessions stretch from ~30 minutes to ~3 hours.

---

## 1. Install

**Claude Code** (plugin, fully automatic — requires Claude Code v1.0.33+):

```
/plugin marketplace add mksglu/context-mode
/plugin install context-mode@context-mode
```

Restart Claude Code (or `/reload-plugins`), then verify — every check should show `[x]`:

```
/context-mode:ctx-doctor
```

MCP-only trial (all 11 tools, no automatic routing):

```bash
claude mcp add context-mode -- npx -y context-mode
```

**Other platforms** (Node.js ≥ 22.5 or Bun):

```bash
npm install -g context-mode
```

Then add the MCP server + hooks per platform — each platform has its own install section in the upstream README (Gemini CLI: one block in `~/.gemini/settings.json`; VS Code Copilot: `.vscode/mcp.json` + `.github/hooks/context-mode.json`; GitHub Copilot CLI: `copilot plugin install mksglu/context-mode:configs/copilot-cli`).

---

## 2. First session, 4 steps

1. Install (above) and run `ctx doctor` — or `/context-mode:ctx-doctor` on Claude Code.
2. Work as usual. Hooks route big outputs (Bash, Read, WebFetch, external MCP calls) into the sandbox automatically.
3. Index docs you will query again: `ctx index` a file or directory once, then `ctx search` instead of re-reading.
4. Check the payoff any time: `ctx stats` — per-tool breakdown, tokens consumed, savings ratio.

On Claude Code the same actions exist as slash commands: `/context-mode:ctx-stats`, `ctx-doctor`, `ctx-index`, `ctx-search`, `ctx-upgrade`, `ctx-purge`, `ctx-insight`. On other platforms, type `ctx stats`, `ctx doctor`, etc. in chat.

From your terminal, without an AI session:

```bash
context-mode doctor
context-mode index . --source project:my-app
context-mode search "authentication middleware" --source project:my-app
context-mode upgrade
```

---

## 3. The 11 MCP tools

Six sandbox tools do the work; five meta-tools manage the setup.

| Tool | What it does | Raw → context |
|---|---|---|
| `ctx_batch_execute` | Many commands + many searches in ONE call (optional `concurrency: 1-8`) | 986 KB → 62 KB |
| `ctx_execute` | Run code in 12 languages; only stdout enters context | 56 KB → 299 B |
| `ctx_execute_file` | Process a file inside the sandbox; raw content never leaves | 45 KB → 155 B |
| `ctx_index` | Chunk markdown into FTS5 with BM25 ranking | 60 KB → 40 B |
| `ctx_search` | Query indexed content, several queries per call | on-demand retrieval |
| `ctx_fetch_and_index` | Fetch a URL, chunk and index it (24 h TTL cache) | 60 KB → 40 B |
| `ctx_stats` | Savings, call counts, session statistics | — |
| `ctx_doctor` | Diagnose runtimes, hooks, FTS5, versions | — |
| `ctx_upgrade` | Update, rebuild, reconfigure hooks | — |
| `ctx_purge` | Delete all indexed content | — |
| `ctx_insight` | Hosted analytics dashboard for teams | — |

---

## 4. How the sandbox works

Each `ctx_execute` call spawns an isolated subprocess. Your code runs there, stdout is captured, and **only stdout enters the conversation**. Log files, API responses and snapshots never leave the sandbox, and scripts can't see each other's memory.

- 12 runtimes: JavaScript, TypeScript, Python, Shell, Ruby, Go, Rust, PHP, Perl, R, Elixir, C#. Bun is auto-detected for 3–5× faster JS/TS.
- Authenticated CLIs (`gh`, `aws`, `gcloud`, `kubectl`, `docker`) inherit credentials by passthrough — the conversation never sees them.
- Output over 5 KB with an `intent` set? It gets indexed and filtered: you receive only the sections matching your intent, plus searchable terms for follow-ups.

The paradigm upstream pushes is **Think in Code**: don't read 50 files into context to count functions — write one script that counts and prints the result. Their example: 47 × Read() = 700 KB becomes 1 × `ctx_execute()` = 3.6 KB.

---

## 5. The knowledge base

`ctx_index` chunks markdown by headings (code blocks stay intact) into a **SQLite FTS5** table. Search combines two strategies merged with Reciprocal Rank Fusion:

- **Porter stemming** — "caching" also matches "cached", "caches".
- **Trigram substring** — "useEff" finds "useEffect".

Plus BM25 ranking with headings weighted 5×, proximity reranking for multi-term queries, Levenshtein fuzzy correction for typos ("kuberntes" → "kubernetes"), and smart snippets that extract windows around your terms instead of truncating.

Fetched URLs are cached (default TTL 24 h, override per call; `ttl: 0` or `force: true` bypasses). Content older than 14 days is cleaned up on startup. Databases live locally, e.g. `~/.context-mode/content/`.

---

## 6. Session continuity

Every file edit, git operation, task, error and user decision is tracked in SQLite. When the conversation compacts, context-mode does **not** dump this back into context — it retrieves only the relevant events via BM25 search, and the model picks up where you left off.

Start without `--continue` and the previous session's data is deleted immediately: fresh session, clean slate.

One boundary upstream is explicit about: routing controls **where data goes**, never how the model writes its answer. No brevity enforcement.

---

## 7. Routing: hooks vs instruction files

Hooks intercept tool calls before execution — they can block a command and redirect it to the sandbox. Instruction files only advise the model; they can't block anything. Upstream's own numbers:

| Setup | Context saved |
|---|---|
| Hooks enabled | **~98%** |
| Instruction file only | ~60% |

One unrouted Playwright snapshot (56 KB) can wipe out a session's savings. Enable hooks wherever your platform supports them.

---

## 8. Platforms

17 supported clients plus the OpenClaw gateway integration (upstream count). From the official compatibility table:

Claude Code · Qwen Code · Gemini CLI · VS Code Copilot · JetBrains Copilot · GitHub Copilot CLI · Cursor · OpenCode · KiloCode · OpenClaw · Codex CLI · Kimi Code · Antigravity · Antigravity CLI (`agy`) · Kiro · Zed · Pi · OMP

Hook support varies: Antigravity and Zed have no hooks and rely on a manually copied routing file; Codex CLI hooks need `[features].hooks = true`. Full matrix: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md).

---

## 9. Benchmarks (upstream, real tool outputs)

| Scenario | Raw | Context | Saved |
|---|---|---|---|
| Playwright snapshot | 56.2 KB | 299 B | 99% |
| GitHub issues (20) | 58.9 KB | 1.1 KB | 98% |
| Access log (500 requests) | 45.1 KB | 155 B | 100% |
| Analytics CSV (500 rows) | 85.5 KB | 222 B | 100% |
| Git log (153 commits) | 11.6 KB | 107 B | 99% |
| Repo research (subagent) | 986 KB | 62 KB | 94% |

Full set: 21 scenarios in [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md) — 376 KB raw → 16.5 KB context (96% overall).

---

## 10. Same problem, other approach: billion-context

[ranxianglei/billion-context](https://github.com/ranxianglei/billion-context) (1.1k ⭐) is also a context-compression plugin, per its own repo description. Different bets:

| | Context Mode | billion-context |
|---|---|---|
| Core idea | Sandbox tool output + FTS5 knowledge base; retrieve on demand | Compress so a small window (100K, per its description) is enough |
| Headline claim | 98% less context on tool output | 5× fewer tokens (its description) |
| Session angle | Continuity across compactions via SQLite event tracking | Month-long single sessions, billions of tokens (its description) |
| Reach | 17 platforms via MCP + hooks | Plugin for small-window setups |

---

## 11. Good to know

- **Local only.** No telemetry, no cloud sync, no account. Code, prompts and session data stay on your machine.
- **License:** upstream is Elastic License 2.0 (source-available) — free to use, fork and modify; you can't offer it as a hosted/managed service.
- **Storage location** can move: set `CONTEXT_MODE_DIR` to an absolute path (adapter default otherwise, e.g. `~/.claude/context-mode`).

---

*Source: https://github.com/mksglu/context-mode · Docs: [docs/platform-support.md](https://github.com/mksglu/context-mode/blob/main/docs/platform-support.md) · Benchmarks: [BENCHMARK.md](https://github.com/mksglu/context-mode/blob/main/BENCHMARK.md)*
