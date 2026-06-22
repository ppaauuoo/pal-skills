## MANDATORY (obey always, no exceptions)

1. BEFORE any grep/find/rg/read on source: check `graphify-out/GRAPH_REPORT.md` and query `graph.json` to locate structural paths. If graph doesn't exist yet, proceed normally. Do NOT scan blind when graph is available. Skip this gate for single-file or conversational requests.
2. Talk ponytail-full (see ponytail extension). Stop if user says "normal mode" or `/ponytail off`.
3. Use `find-docs` skill + `code_search` for documentation lookups.
4. **Always use the pi `grep` and `find` tools — NEVER run `grep`/`find`/`rg`/`fd` inside a `bash` tool call.** These are powered by `@ff-labs/pi-fff` (override mode via `PI_FFF_MODE=override` in env.nu), so the built-in `grep`/`find` tools are already `ffgrep`/`fffind` — faster, frecency-ranked, git-aware. No exceptions. Violation = stop, redo using the pi tools.

VIOLATION = stop, redo step correctly. If graph absent, proceed normally and create graph after.

---

## Knowledge Graph

Use the graphify skill to query `graph.json` when it exists. Run graphify update after: file added/moved/renamed, new module created, or major refactor.

---

## Documentation lookup

Use `find-docs` skill + `code_search`. Always available.
