## MANDATORY (obey always, no exceptions)

0. **English only — always.** Every single response 100% in English. No Chinese, Thai, or any other language unless the user explicitly asks for translation. Never default to non-English prose.

1. **When a user request is vague, ambiguous, or confusing — ask before acting.** Use `ask_user_question` to clarify intent. Do not guess and implement the wrong thing. One focused question beats a wrong diff.
   - If there is **any doubt** about scope, target, or intent: state your interpretation in one sentence, then ask via `ask_user_question` with options like "Yes, proceed with this interpretation" vs "Let me clarify".
   - **Never silently assume** — surface the assumption and confirm it. Skip only when the intent is unambiguously clear from context.

2. **ASK before doing anything destructive or impactful.** Before any operation that could break things, lose data, take a long time, or be annoying to undo — **state the plan, estimate the impact, and ask with a Yes/No via `ask_user_question`**. Safe read-only stuff (info commands, listings) is fine to run directly.

3. BEFORE any grep/find/rg/read on source: check `graphify-out/GRAPH_REPORT.md` and query `graph.json` to locate structural paths. If graph doesn't exist yet, proceed normally. Do NOT scan blind when graph is available. Skip this gate for single-file or conversational requests. **NOTE:** `graphify-out/` is gitignored, so the pi `find` tool won't see it — use `read` or `bash ls` to access it directly.
4. Talk ponytail-full (see ponytail extension). Stop if user says "normal mode" or `/ponytail off`.
5. Use `find-docs` skill + `code_search` for documentation lookups.
6. **Always use the pi `grep` and `find` tools — NEVER run `grep`/`find`/`rg`/`fd` inside a `bash` tool call.** These are powered by `@ff-labs/pi-fff` (override mode via `PI_FFF_MODE=override` in env.nu), so the built-in `grep`/`find` tools are already `ffgrep`/`fffind` — faster, frecency-ranked, git-aware. No exceptions. Violation = stop, redo using the pi tools.

VIOLATION = stop, redo step correctly. If graph absent, proceed normally and create graph after.

---

## Knowledge Graph

Use the graphify skill to query `graph.json` when it exists. Run graphify update after: file added/moved/renamed, new module created, or major refactor.

---

## Documentation lookup

Use `find-docs` skill + `code_search`. Always available.
