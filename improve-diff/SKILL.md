---
name: improve-diff
description: Run the fallow diff-scoped code-quality audit (changes vs the merge-base), report the top 10 issues, and walk the user through fixing them one at a time. Stops immediately when the diff has no findings. Use when the user asks to audit or improve recent changes, the current diff, or a branch vs main. For a whole-codebase quality pass, use /improve-codebase instead.
disable-model-invocation: false
---

# Improve Diff

Run the diff-scoped `fallow` audit (changed files vs the merge-base), surface the top 10 issues, and fix them interactively — one at a time, asking the user before each fix. If the diff has no findings, stop.

This skill only looks at **the diff**, never the whole codebase. For project-wide scans (`fallow health`, `fallow dead-code`, `fallow dupes`) use `/improve-codebase`.

## When to Use

- The user asks to audit or improve the current diff, recent changes, or this branch's changes.
- The user invokes `/improve-diff`.
- The whole-codebase pass (`/improve-codebase`) is explicitly not requested.

## Step 1 — Run the audit

From the repository root, run a single command that emits a compact, ranked summary. **You must pass `--pretty`** — without it the JSON arrays are compressed (`_compressed: "array"`) and you'd need extra steps to expand them. The `--pretty` flag expands them inline.

The summarizer is defensive: when the diff is clean (e.g. `changed_files_count: 0`), the audit output omits the `dead_code`/`complexity` sections entirely, and the summarizer prints a `NO_FINDINGS` marker instead of crashing.

```bash
npx fallow audit --format json --pretty --quiet 2>/dev/null | python3 -c '
import json,sys
d=json.load(sys.stdin)
a=d.get("attribution") or {}
s=d.get("summary") or {}
print("verdict=%s dead_intro=%s cx_intro=%s dup_intro=%s" % (d.get("verdict"), a.get("dead_code_introduced"), a.get("complexity_introduced"), a.get("duplication_introduced")))
total=(s.get("dead_code_issues") or 0)+(s.get("complexity_findings") or 0)+(s.get("duplication_clone_groups") or 0)
if total==0:
    print("NO_FINDINGS changed_files=%s" % d.get("changed_files_count"))
    sys.exit(0)
dc=d.get("dead_code") or {}
print("-- introduced dead code --")
for e in dc.get("unused_exports") or []:
    if e.get("introduced"): print("export %s:%s %s" % (e["path"], e["line"], e["export_name"]))
for e in dc.get("unused_files") or []:
    if e.get("introduced"): print("file %s" % e["path"])
print("-- complexity (introduced first, by severity then cc desc) --")
cx=d.get("complexity") or {}
def key(f):
    sev={"critical":0,"high":1,"moderate":2}[f["severity"]]
    return (0 if f.get("introduced") else 1, sev, -f["cyclomatic"])
for f in sorted(cx.get("findings") or [], key=key):
    m="INTRO" if f.get("introduced") else "inh"
    print("[%s] %-8s cc=%3d cog=%3d %s:%s %s" % (m, f["severity"], f["cyclomatic"], f["cognitive"], f["path"], f["line"], f["name"]))
'
```

Exit codes: `0` and `1` both mean the run succeeded (`1` = findings found). `2` is a real error, reported as a JSON envelope on stdout. If you get exit `2`, report the error and stop.

**Stop here if there are no findings.** If the output prints `NO_FINDINGS` (zero dead-code issues, complexity findings, and duplication clone groups in the changed set — including the clean-working-tree case where `changed_files_count: 0`), tell the user there is nothing to improve in the current diff and stop. Do not build a top-10 list, do not fall back to project-wide scans, do not fix anything.

The summarizer prints everything you need to rank the top 10 in one shot:

- `verdict` and the `*_introduced` counts (the gate is `new-only`, so `introduced` findings are the blockers).
- **Introduced dead code** — unused exports _and_ unused files with `introduced: true` (the gate blocker); each has `path`, `export_name` (exports only), `line`, and `actions` (often `remove-export` / `delete-file`).
- **Complexity findings** sorted introduced-first, then by severity (`critical`/`high`/`moderate`) and `cyclomatic` descending, with `cog` (cognitive) and `line` for tie-breaking and reporting.

If you need the raw detail for a specific finding (e.g. `exceeded` reason, `actions`, or duplication clone groups), re-run with `--pretty` and inspect the relevant section directly.

## Step 2 — Rank the top 10

Build the top 10 list by priority:

1. **Introduced dead code first** — any unused export/type/file with `introduced: true`. This is what flips the gate to `fail` and is the highest-value fix.
2. **Critical complexity hotspots** — `complexity.findings` with `severity: "critical"`, sorted by `cyclomatic` descending (tie-break by `cognitive`).
3. **High complexity hotspots** — `severity: "high"`, sorted by `cyclomatic` descending.
4. **Dead-code clusters** — large groups of unused exports in one file (e.g. many in `app/lib/api.ts`) can be batched as a single item.
5. **Duplication** — the largest clone groups.

Cap the list at 10 items. For each item report: rank, file path, symbol/name, line, and the metric that made it rank (e.g. `cc=51`, `introduced`, `unused export`). Keep each line concise.

## Step 3 — Report and ask

Present the ranked top 10 to the user as a numbered list. Then ask whether they want to fix **issue #1** (use `ask_user`). Do not fix anything yet.

## Step 4 — Interactive fix loop

For each issue the user agrees to fix:

1. Fix it. Prefer the auto-fixable action when available (e.g. `remove-export` for unused exports). For complexity hotspots, refactor by extracting named sub-functions/helpers — do not just suppress.
2. After fixing, re-run the audit to confirm the issue is gone and nothing regressed. Use `--pretty` and a targeted check for the symbol you changed:

   ```bash
   npx fallow audit --format json --pretty --quiet 2>/dev/null | python3 -c '
   import json,sys
   d=json.load(sys.stdin)
   a=d.get("attribution") or {}
   print("verdict:", d.get("verdict"), "dead_intro:", a.get("dead_code_introduced"), "cx_intro:", a.get("complexity_introduced"), "dup_intro:", a.get("duplication_introduced"))
   name="SYMBOL_NAME"
   print("still flagged:", any(f["name"]==name for f in (d.get("complexity") or {}).get("findings") or []))
   '
   ```

   If the refactor introduced a new small helper that is now flagged (common with CRAP — no test coverage), either simplify it below the CRAP threshold or add a focused test for it; don't leave a new introduced finding behind.

3. If the change was significant, run the `build-test` skill to verify the pipeline.
4. Report the result briefly, then ask about the **next** issue (issue #2, then #3, and so on) via `ask_user`.

Stop when the user declines an issue or the list is exhausted. If the user declines, move on to the next issue rather than stopping entirely, unless they say to stop.

## Notes

- Do not suppress findings with `fallow-ignore` comments unless the user explicitly asks — prefer real fixes.
- For unused exports, verify the symbol is genuinely unused before removing (check it is not part of an intentional public API / re-export). If it's used internally but not imported elsewhere, drop the `export` keyword instead of deleting it.
- Introduced dead **files** (flagged via `dead_code.unused_files`) are auto-fixable only as `delete-file` — check for dynamic imports and side-effect-only entrypoints before deleting.
- A finding with `exceeded: "crap"` (and `coverage_tier: "none"`) is a **test-coverage artifact**, not raw complexity: CRAP = CC² + CC with zero coverage. If the function is already below the cyclomatic/cognitive thresholds, the fix is to add a focused test (export the function if needed) rather than refactor further. For a pure helper, a small test file clears it.
- Keep the loop interactive: one issue at a time, always ask before fixing.
