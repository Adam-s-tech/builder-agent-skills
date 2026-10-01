---
name: improve-codebase
description: Run fallow's project-wide scans (health, dead-code, dupes), report the top 10 issues across the whole codebase, and walk the user through fixing them one at a time. Use when the user wants a full-codebase quality pass, not a diff-scoped audit. Use when the user asks to improve the whole codebase, run a full audit, or find the worst hotspots project-wide.
disable-model-invocation: false
---

# Improve Codebase

Run fallow's **project-wide scans** — `fallow health`, `fallow dead-code`, and `fallow dupes` — surface the top 10 issues, and fix them interactively, one at a time, asking the user before each fix.

Unlike `/improve-diff` (which runs the diff-scoped `fallow audit` and stops when the diff has no findings), this skill never uses the diff-based approach: it ranks issues across the entire codebase, inherited and new alike.

## When to Use

- The user asks to "improve the codebase", "run a full audit", "find the worst hotspots", or wants a project-wide quality pass.
- The user invokes `/improve-codebase`.
- The user asks for a project-wide audit after `/improve-diff` reported `NO_FINDINGS` (the diff-scoped gate has nothing to work with on a clean tree).

## Step 1 — Run the three project-wide scans

From the repository root. Do NOT use `fallow audit` (it is diff-scoped). Do NOT run bare `fallow health` with no section flags — the full pipeline can be slow or hang on large repos; select sections explicitly.

Run the three commands in parallel bash calls. Each writes JSON to `.agents/tmp/` (files are large — hundreds of KB — so never cat them; summarize with python). Redirect stderr: npx prints install warnings there. Treat exit codes `0` and `1` as success (`1` = findings found); exit `2` is a real error reported as a JSON envelope on stdout — report it and stop. Give each call a generous timeout (~240s).

**Scan A — complexity (hotspots):**

```bash
mkdir -p .agents/tmp && npx fallow health --complexity --format json > .agents/tmp/health-cx.json 2>/dev/null; echo "exit=$?"
python3 - <<'EOF'
import json
d=json.load(open(".agents/tmp/health-cx.json"))
f=d["findings"]
def istest(p): return "/__tests__/" in p or ".test." in p
src=[x for x in f if not istest(x["path"])]
def sevrank(s): return {"critical":0,"high":1,"moderate":2}.get(s,3)
def crap_only(x): return x.get("exceeded")=="crap" and x.get("coverage_tier")=="none"
src.sort(key=lambda x:(sevrank(x["severity"]), -x["cyclomatic"], -x["cognitive"]))
crit=sum(1 for x in src if x["severity"]=="critical")
high=sum(1 for x in src if x["severity"]=="high")
crapn=sum(1 for x in src if crap_only(x))
print("findings=%d src=%d critical=%d high=%d crap_artifacts=%d" % (len(f),len(src),crit,high,crapn))
for x in src[:25]:
    print("%-8s cc=%3d cog=%3d %s:%s %s exceeded=%s" % (x["severity"],x["cyclomatic"],x["cognitive"],x["path"],x["line"],x["name"],x.get("exceeded")))
EOF
```

Key fields per finding: `severity` (critical/high/moderate), `cyclomatic`, `cognitive`, `crap`, `exceeded` (what it tripped: `all`, `both`, `cyclomatic`, `cognitive`, `cyclomatic_crap`, or `crap`), `coverage_tier`, `path`, `line`, `name`. `large_functions` (in the same JSON) holds unit-size findings — a useful secondary signal but not the primary ranking.

If you need to explain _why_ a function scored high before refactoring, re-run with `--complexity-breakdown` to get the per-decision-point `contributions[]`.

**Scan B — dead code:**

```bash
mkdir -p .agents/tmp && npx fallow dead-code --format json > .agents/tmp/dead-code.json 2>/dev/null; echo "exit=$?"
python3 - <<'EOF'
import json
d=json.load(open(".agents/tmp/dead-code.json"))
ue=d["unused_exports"]
byfile={}
for e in ue: byfile.setdefault(e["path"],[]).append(e)
clusters=sorted(byfile.items(), key=lambda kv:-len(kv[1]))
print("unused_exports=%d files_with=%d unused_files=%d" % (len(ue),len(byfile),len(d["unused_files"])))
print("-- clusters (5+ exports, largest first) --")
for p,es in clusters:
    if len(es)<5: break
    print("%3d %s" % (len(es), p))
print("-- remaining exports (top 15 files) --")
for p,es in clusters[:15]:
    if len(es)>=5: continue
    print("%3d %s %s" % (len(es), p, ",".join(x["export_name"] for x in es)[:100]))
print("-- unused deps --", d["unused_dependencies"], "dev:", d["unused_dev_dependencies"])
EOF
```

`unused_exports` entries carry `path`, `export_name`, `line`, `is_type_only`, `is_re_export`, and `actions` (often `remove-export`). `unused_files` may be empty if config excludes those paths; `fallow health`'s `vital_signs.dead_files` count can still report dead files — cross-check if the counts disagree.

**Scan C — duplication:**

```bash
mkdir -p .agents/tmp && npx fallow dupes --format json > .agents/tmp/dupes.json 2>/dev/null; echo "exit=$?"
python3 - <<'EOF'
import json
d=json.load(open(".agents/tmp/dupes.json"))
cg=d["clone_groups"]
cg.sort(key=lambda g:-(g.get("line_count") or 0))
st=d.get("stats") or {}
print("clone_groups=%d stats=%s" % (len(cg), json.dumps(st)[:200]))
for g in cg[:12]:
    inst=g.get("instances") or []
    files=sorted(set(str(i.get("path") or i.get("file")) for i in inst))
    print("lines=%3d n=%d %s" % (g.get("line_count",0), len(inst), " | ".join(files)[:140]))
EOF
```

Clone groups have `instances` (each with `file`, `start_line`, `end_line`), `token_count`, `line_count`, `suggested_name`, and `actions`.

## Step 2 — Rank the top 10

No "introduced" attribution exists here — rank by real-world value across the whole codebase:

1. **Dead-code clusters first** — files with 5+ unused exports, largest cluster first. Batch a whole file as one item.
2. **Critical complexity hotspots** — non-test files, `severity: "critical"`, sorted by `cyclomatic` descending (tie-break by `cognitive`). Prefer findings whose `exceeded` includes cyclomatic/cognitive (`all`, `both`); findings that are pure `crap` with `coverage_tier: "none"` are test-coverage artifacts — see Notes.
3. **High complexity hotspots** — same ordering within `severity: "high"`.
4. **Dead files** — from `unused_files`.
5. **Duplication** — largest clone groups by `line_count`.
6. **Unused dependencies** — batch all as one item.

Exclude test files (`*.test.ts(x)`, `__tests__/`) from the complexity ranking — refactoring them is low value. Deprioritize one-off scripts (e.g. `scripts/`, `ai-data/scripts/`) below app/CLI source. Cap at 10. For each item report: rank, file path, symbol/name, line, and the metric that made it rank (e.g. `cc=31 cog=54`, `24 unused exports`, `230 dup lines`). Keep each line concise.

## Step 3 — Report and ask

Present the ranked top 10 as a numbered table/list. Then ask whether to fix **issue #1** (use `ask_user`, recommended option first). Do not fix anything yet.

## Step 4 — Interactive fix loop

For each issue the user agrees to fix:

1. Fix it. Prefer the auto-fixable action when available (e.g. `remove-export`). For complexity hotspots, refactor by extracting named sub-functions/helpers — do not just suppress. For duplication, extract the shared fragment into one named helper (use the clone group's `suggested_name`).
2. Re-run **only the scan covering that issue** (A, B, or C) and do a targeted check for the symbol:

   ```bash
    npx fallow health --complexity --format json > .agents/tmp/health-cx.json 2>/dev/null; python3 - <<'EOF'
   import json
    d=json.load(open(".agents/tmp/health-cx.json"))
   name="SYMBOL_NAME"
   print("still flagged:", any(f["name"]==name for f in d["findings"]))
   print("critical total:", sum(1 for f in d["findings"] if f["severity"]=="critical"))
   EOF
   ```

   For dead code: confirm the export name no longer appears in `unused_exports`. For duplication: confirm the clone group's files no longer appear together.

   If the refactor introduced a new small helper that is now flagged (common with CRAP — no test coverage), either simplify it below the CRAP threshold or add a focused test for it; don't leave a new introduced finding behind. (If the diff gate matters for the change, `fallow audit --gate new-only` will catch regressions on commit.)

3. If the change was significant, run the `build-test` skill to verify the pipeline.
4. Report the result briefly, then ask about the **next** issue (issue #2, then #3, and so on) via `ask_user`.

Stop when the user declines an issue and says to stop, or the list is exhausted. If the user declines one issue, move on to the next rather than stopping entirely, unless they say to stop.

## Notes

- Do not suppress findings with `fallow-ignore` comments unless the user explicitly asks — prefer real fixes.
- For unused exports, verify the symbol is genuinely unused before removing (check it is not part of an intentional public API / re-export). If it's used internally but not imported elsewhere, drop the `export` keyword instead of deleting it.
- Dead files: check for dynamic imports and side-effect-only entrypoints before deleting; fallow marks some `delete-file` actions as not auto-fixable for a reason.
- A finding with `exceeded: "crap"` (and `coverage_tier: "none"`) is a **test-coverage artifact**, not raw complexity: CRAP = CC² + CC with zero coverage. If the function is already below the cyclomatic/cognitive thresholds, the fix is to add a focused test (export the function if needed) rather than refactor further. For a pure helper, a small test file clears it. `exceeded: "cyclomatic_crap"` means both — tests help but won't clear the cyclomatic overage; refactor too.
- Keep the loop interactive: one issue at a time, always ask before fixing.
