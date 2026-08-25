# Operation log

Append-only chronological record. Format: `## [<ISO-date>] <op> | <subject> | <one-line>`.
Operations: bootstrap · sync · code-truth-update.

---

## [2026-08-25T07:20:00Z] bootstrap | beyla | base-sha=b4da5978747b

Pages: 20 (8 top-level, 12 concepts). Topics: 4. Raw files at init: 0.

Invoked by Context Maintainer in `reconcile` mode targeting `docs/wiki/.citation-index.json`.
No wiki existed at `docs/wiki/SCHEMA.md`, so the run escalated to full bootstrap per the
Phase 1 `no-wiki-schema` rule (`ctxm.reconcile.escalate reason=no-wiki-schema`).

## [2026-08-25T07:58:47Z] sync | beyla | base-sha=b4da5978747b

0-raws-picked-up · 0-externals-picked-up · 197-citations-verified · 20-pages-touched · 0-citations-promoted · 0-allowlist-docs-synthesized · 0-queue-entries-drained · 0-reconcile-hints-folded

Invoked by Context Maintainer in `reconcile` mode. The invocation carried a single
`update_docs_index` work item targeting `docs/index.md` and **no `update_wiki` targets**, so
`$CTXM_TARGET_KEYS` was empty and the run escalated per the Phase 1 rule
(`ctxm.reconcile.escalate reason=empty-target-keys`).

Code-truth: 197 `path:Symbol` citations verified against HEAD — 0 unresolved, 0 missing files,
0 drifted pages. 5 `path:LINE` citations remain unpromotable (all anchor `var`/`const` block
members, not `func`/`type` declarations) — verified accurate by inspection.

Fixed: stripped a leaked template authoring instruction ("Cite by `path:FunctionName`…") that
bootstrap had emitted verbatim into all 20 pages; corrected the `api-dependency-counter` index
blurb that had picked it up; replaced three non-existent skill references
(`wiki-ingest`/`wiki-query`/`wiki-lint`) in index.md's "Operating this wiki" table.

qmd re-index SKIPPED — `better-sqlite3` native module ABI mismatch (built 131, Node needs 147).
