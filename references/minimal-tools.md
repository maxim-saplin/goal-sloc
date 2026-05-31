# Minimal tools — when nothing is installed

Use a real semantic tool if you can (`dart_code_metrics`/DCM, `knip`, `ts-prune`, `vulture`, `madge`, a language server). When you can't install anything, here are paste-ready fallbacks. **Always cross-check any "dead code" hit before deleting — entrypoints (`main`, CLI commands, test mains, framework-invoked callbacks, tear-offs, dynamic/string dispatch) look orphan but aren't.**

## SLOC counting convention
When you must define the metric yourself, default to **non-blank, non-comment-only lines**, and decide doc-comments explicitly (recommend: count them — they're maintained text). State your choice so before/after are comparable. Whatever you pick, **measure both sides the same way** (and format-normalized — see SKILL §2).

Portable line counter (no deps):
```bash
# raw tracked lines per top-level dir (respects git; excludes build/vendor via .gitignore)
git ls-files | awk -F/ '{print $1}' | sort -u | while read d; do
  n=$(git ls-files "$d" | xargs wc -l 2>/dev/null | tail -1 | awk '{print $1}'); printf '%8s  %s\n' "${n:-0}" "$d"; done
# code-only (skip blank + // or # comment-only) for a file set:
grep -vhE '^\s*($|//|#)' $(git ls-files '*.dart' '*.py' '*.ts') | wc -l
```
(`cloc` / `tokei` / `scc` are better if present.)

## Python — orphan modules & unused functions (stdlib `ast`)
```python
import ast, pathlib, collections
root = pathlib.Path("src")
files = list(root.rglob("*.py"))
defs, used = collections.defaultdict(set), set()
for f in files:
    t = ast.parse(f.read_text())
    for n in ast.walk(t):
        if isinstance(n, (ast.FunctionDef, ast.AsyncFunctionDef, ast.ClassDef)):
            defs[f].add(n.name)
        if isinstance(n, ast.Name): used.add(n.id)
        if isinstance(n, ast.Attribute): used.add(n.attr)
        if isinstance(n, (ast.Import, ast.ImportFrom)):
            for a in n.names: used.add((a.asname or a.name).split(".")[0])
ENTRY = {"main"}  # add CLI/command/fixture names
for f, names in defs.items():
    dead = {n for n in names if n not in used and n not in ENTRY}
    if dead: print(f, sorted(dead))
# orphan modules: a module whose name is never imported anywhere
mods = {f.stem for f in files}
imported = set()
for f in files:
    for n in ast.walk(ast.parse(f.read_text())):
        if isinstance(n, ast.ImportFrom) and n.module: imported.add(n.module.split(".")[-1])
        if isinstance(n, ast.Import):
            for a in n.names: imported.add(a.name.split(".")[-1])
print("orphan modules:", sorted(mods - imported - {"__init__","__main__"}))
```
Heuristic only — confirm each hit (entrypoints, dynamic `getattr`, plugin registries).

## JS/TS
Prefer `npx knip` or `npx ts-prune` / `npx madge --orphans src`. Without network: a crude orphan-file scan — list every `*.ts/js`, grep for `from ['"]...<basename>` / `require(.*<basename>)`; files with zero inbound (minus entrypoints in `package.json` `main`/`bin` and test files) are candidates.

## Dart/Flutter
`dart pub global activate dart_code_metrics` → `metrics check-unused-code lib`, `metrics check-unused-files lib`, `metrics analyze --cyclomatic-complexity=N lib`. For an accurate import graph, a small script that resolves both relative *and* `package:<pkg>/` imports to file paths and counts inbound edges (grep alone misses relative imports). See `flutter-sloc-reference.md`.

## General
A dependency/knowledge-graph tool (e.g. GitNexus) is great for fan-in/out, clusters and impact analysis — but treat its **call-graph "uncalled" lists as candidates, not truth** (they miss tear-offs, cross-file and string-dispatched calls). Cross-check with a semantic dead-code tool before deleting.
