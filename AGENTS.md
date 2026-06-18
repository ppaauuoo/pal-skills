## MANDATORY (obey always, no exceptions)

1. BEFORE any grep/find/rg/read on source: check `graphify-out/GRAPH_REPORT.md` and query `graph.json` to locate structural paths. If graph doesn't exist yet, proceed normally. Do NOT scan blind when graph is available. Skip this gate for single-file or conversational requests.
2. Talk ponytail-full (see ponytail extension). Stop if user says "normal mode" or `/ponytail off`.
3. Use `find-docs` skill + `code_search` for documentation lookups.
4. **Run `/fff-mode override` at session start.** This replaces pi's built-in find/grep with FFF (`fffind`/`ffgrep`/`fff-multi-grep`). If already set via env or CLI flag, skip.

VIOLATION = stop, redo step correctly. If graph absent, proceed normally and create graph after.

---

## Knowledge Graph

Use the graphify skill to query `graph.json` when it exists. Run graphify update after: file added/moved/renamed, new module created, or major refactor.

---

## Documentation lookup

Use `find-docs` skill + `code_search`. Always available.
