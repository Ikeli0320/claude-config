---
name: hims-code-archaeologist-skill
description: Use this skill when you need to perform software archaeology on a .NET/ASP/MSSQL codebase. Systematically uncovers technical debt, architectural drift, dependency rot, and dead code through version history analysis, structural inspection, and dependency mapping. Read-only — never modifies source code.
version: 1.0.0
tags: [legacy, technical-debt, archaeology, analysis, dotnet, mssql, asp]
type: analysis
os: [windows, linux, macos]
---

# HIMS Code Archaeologist

Uncover hidden risks in .NET / ASP.NET / MSSQL systems before they cause production incidents. Performs systematic read-only analysis across 7 phases. All findings are documented in a report — no source files are ever modified.

## Trigger Conditions

Use this skill when asked to:
- Assess a legacy .NET system before modernization
- Find technical debt hotspots in ASP.NET solutions
- Understand undocumented dependencies in a .NET project
- Map actual vs. intended architecture (Controllers → Services → Domain → Infrastructure)
- Identify dead code, orphaned files, or unused NuGet packages
- Analyse version history for instability signals

---

## Phase 0: Dependency Check (Always Run First)

Before executing any phase, detect available tools and set analysis mode accordingly.

```bash
echo "=== Dependency Check ==="

# Core (required)
command -v git    >/dev/null 2>&1 && echo "[OK] git"    || echo "[MISSING] git — cannot proceed"
command -v dotnet >/dev/null 2>&1 && echo "[OK] dotnet" || echo "[MISSING] dotnet — .NET phases will be skipped"

# Enhanced (optional — phases degrade gracefully if missing)
command -v rg     >/dev/null 2>&1 && echo "[OK] ripgrep"   || echo "[WARN] ripgrep missing — using grep fallback"
command -v jq     >/dev/null 2>&1 && echo "[OK] jq"        || echo "[WARN] jq missing — JSON output disabled"
command -v python3>/dev/null 2>&1 && echo "[OK] python3"   || echo "[WARN] python3 missing — script-based phases skipped"
command -v tree   >/dev/null 2>&1 && echo "[OK] tree"      || echo "[WARN] tree missing — directory view disabled"

# Windows install hints (if missing)
# ripgrep : winget install BurntSushi.ripgrep.MSVC
# jq      : winget install jqlang.jq
# python3 : winget install Python.Python.3
```

> If `git` or `dotnet` are missing, stop and inform the user. All other tools are optional — note which phases are degraded.

---

## Phase 1: Historical Excavation

Goal: Identify architectural stress points and instability signals via commit history.

```bash
# Commit frequency + authors (last 2 years)
git log --stat --since="2 years ago" --pretty=format:"%h %an %ad %s" --date=short

# Hotfix frequency (instability signal)
git log --oneline --grep="hotfix\|fix\|urgent\|patch\|rollback" | wc -l

# Files with highest churn (top 20 risk hotspots)
git log --oneline --name-only --since="1 year ago" \
  | grep -v "^[a-f0-9]\{7\}" | sort | uniq -c | sort -nr | head -20

# Revert frequency (architectural instability)
git log --oneline --grep="[Rr]evert" | wc -l

# Large commits (potential architecture changes)
git log --pretty=format:"%h %s" --numstat \
  | awk 'NF==3 {files++; added+=$1; removed+=$2} END {
      print "Commits with file changes:", files;
      print "Avg files changed:", files ? int((added+removed)/files) : 0
    }'

# Identify contributors per module
git log --pretty=format: --name-only -- "*.cs" \
  | sort | uniq -c | sort -rn | head -30
```

---

## Phase 2: Dependency Archaeology (.NET / NuGet)

Goal: Detect declared vs actual dependencies, outdated packages, and cyclic project references.

```bash
# List all NuGet package references across solution
dotnet list package 2>/dev/null || find . -name "*.csproj" -exec dotnet list {} package \;

# Find outdated packages
dotnet list package --outdated 2>/dev/null

# Find deprecated packages
dotnet list package --deprecated 2>/dev/null

# Actual using statements (what's really used in code)
# ripgrep version
rg "^using " --type cs --no-filename | sort | uniq -c | sort -nr | head -50
# fallback (no ripgrep)
grep -r "^using " --include="*.cs" | sed 's/.*using //' | sort | uniq -c | sort -nr | head -50

# Find cyclic ProjectReferences between .csproj files
python3 - <<'PYEOF'
import os, re, sys

projects = {}
for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d not in ('bin','obj','node_modules','.git')]
    for f in files:
        if f.endswith('.csproj'):
            path = os.path.join(root, f)
            with open(path, encoding='utf-8', errors='ignore') as fh:
                content = fh.read()
            refs = re.findall(r'<ProjectReference Include="([^"]+)"', content)
            projects[f] = [os.path.basename(r) for r in refs]

for proj, deps in projects.items():
    for dep in deps:
        if proj in projects.get(dep, []):
            print(f"CYCLE: {proj} <-> {dep}")
PYEOF

# Find all .csproj files (solution structure overview)
find . -name "*.csproj" | sort
```

> **If python3 missing**: Skip cyclic dependency detection, note it in report.

---

## Phase 3: Structural Decay Analysis

Goal: Identify large classes, long methods, and duplicated code in C# files.

```bash
# Find large C# files (> 300 lines — potential god classes)
find . -name "*.cs" -not -path "*/bin/*" -not -path "*/obj/*" \
  | xargs wc -l 2>/dev/null | awk '$1 > 300 {print $0}' | sort -nr | head -20

# Find long methods (> 50 lines between method declarations) via python3
python3 - <<'PYEOF'
import os, re

def analyze_file(path):
    with open(path, encoding='utf-8', errors='ignore') as f:
        lines = f.readlines()

    method_pattern = re.compile(
        r'^\s+(public|private|protected|internal|static)\s+.*\s+\w+\s*\(.*\)\s*$'
    )
    results = []
    start = None
    name = None
    depth = 0

    for i, line in enumerate(lines):
        if method_pattern.match(line):
            start = i
            name = line.strip()
        if start is not None:
            depth += line.count('{') - line.count('}')
            if depth <= 0 and start != i:
                length = i - start
                if length > 50:
                    results.append((length, start + 1, name[:80]))
                start = None
                depth = 0
    return results

for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d not in ('bin','obj','.git')]
    for f in files:
        if f.endswith('.cs'):
            path = os.path.join(root, f)
            hits = analyze_file(path)
            for length, line, name in sorted(hits, reverse=True):
                print(f"{path}:{line} ({length} lines) {name}")
PYEOF

# Code duplication (jscpd supports C#)
if command -v jscpd >/dev/null 2>&1; then
  jscpd --languages csharp --min-tokens 80 --reporters consoleFull \
    --ignore "**/bin/**,**/obj/**" .
else
  echo "[WARN] jscpd not installed — skipping duplication check"
  echo "Install: npm install -g jscpd"
fi
```

---

## Phase 4: Architecture Drift Detection

Goal: Detect layer violations in standard ASP.NET layered architecture.

Expected layer structure:
```
Controllers / API  →  (calls)
Application / Services  →  (calls)
Domain / Core  →  (calls)
Infrastructure / Data / Repositories
```

```bash
# Detect downward layer violations (Domain should NOT import Controllers/Application namespaces)
echo "=== Layer Violation Check ==="

# Adjust folder names to match actual solution structure
DOMAIN_DIRS="Domain Core"
INFRA_DIRS="Infrastructure Data Repositories"
APP_DIRS="Application Services"
API_DIRS="Controllers API"

for domain_dir in $DOMAIN_DIRS; do
  if [ -d "$domain_dir" ]; then
    echo "--- Checking $domain_dir for upward imports ---"
    for api_dir in $API_DIRS $APP_DIRS; do
      rg "using.*\.$api_dir\." "$domain_dir/" --type cs 2>/dev/null \
        || grep -r "using.*\.$api_dir\." "$domain_dir/" --include="*.cs" 2>/dev/null
    done
  fi
done

# Find direct DB access from Controllers (should go through Services)
echo "--- Direct DB calls in Controllers ---"
rg "(SqlConnection|DbContext|IDbConnection|SqlCommand)" \
  --type cs -g "*Controller*.cs" 2>/dev/null \
  || grep -r "SqlConnection\|DbContext" --include="*Controller*.cs" . 2>/dev/null

# Find hardcoded connection strings (security + architecture smell)
echo "--- Hardcoded connection strings ---"
rg "(Server=|Data Source=|Initial Catalog=|User ID=|Password=)" \
  --type cs --type xml 2>/dev/null \
  || grep -r "Server=\|Data Source=" --include="*.cs" --include="*.config" . 2>/dev/null

# Namespace usage heatmap (shows coupling patterns)
echo "--- Top namespace dependencies ---"
rg "^using " --type cs --no-filename 2>/dev/null \
  | grep -v "^using System" | sort | uniq -c | sort -nr | head -30
```

> **Note**: Update `DOMAIN_DIRS`, `INFRA_DIRS`, `APP_DIRS`, `API_DIRS` to match actual folder names in the target solution before running.

---

## Phase 5: Test Coverage Archaeology

Goal: Find untested critical paths in .NET projects.

```bash
# Run tests with coverage (requires coverlet or built-in collector)
dotnet test --collect:"XPlat Code Coverage" --results-directory ./coverage-results 2>/dev/null \
  || echo "[WARN] Test coverage collection failed — check coverlet is installed"

# Find .cs files without corresponding test files
echo "=== Files without tests ==="
find . -name "*.cs" \
  -not -path "*/bin/*" -not -path "*/obj/*" \
  -not -name "*Test*.cs" -not -name "*Spec*.cs" \
  | while read f; do
      base=$(basename "$f" .cs)
      # Look for test file anywhere in tree
      if ! find . -name "*${base}Test*.cs" -o -name "*${base}Spec*.cs" 2>/dev/null | grep -q .; then
        echo "NO_TEST: $f"
      fi
    done

# Find test smells
echo "=== Test Smells ==="
# Thread.Sleep in tests (flaky)
rg "Thread\.Sleep|Task\.Delay" --type cs -g "*Test*.cs" 2>/dev/null
# Too many mocks (over-mocking smell)
rg "\.Setup\(|\.Returns\(|Mock<" --type cs -g "*Test*.cs" 2>/dev/null | wc -l | \
  xargs -I{} echo "Mock setup calls in tests: {}"
# Empty catch in tests
rg "catch\s*\(\s*\)\s*\{\s*\}" --type cs -g "*Test*.cs" 2>/dev/null
```

---

## Phase 6: Documentation Decay Check

Goal: Find stale TODOs, outdated config files, and mismatched documentation.

```bash
# Find stale TODOs with last-touch date
echo "=== Stale TODOs (sorted by age) ==="
rg "TODO|FIXME|HACK|XXX|UNDONE" --type cs --with-filename 2>/dev/null \
  | while IFS=: read file line content; do
      last=$(git log -1 --format="%ad" --date=short -- "$file" 2>/dev/null || echo "unknown")
      echo "$last  $file:$line  $content"
    done | sort | head -30

# Find outdated Web.config / appsettings.json keys (not referenced in code)
echo "=== Potentially unused config keys ==="
if [ -f "appsettings.json" ]; then
  if command -v jq >/dev/null 2>&1; then
    jq -r 'paths(scalars) | join(".")' appsettings.json | while read key; do
      simple_key=$(echo "$key" | tr '.' '\n' | tail -1)
      if ! rg "$simple_key" --type cs -q 2>/dev/null; then
        echo "UNUSED_CONFIG: $key"
      fi
    done
  else
    echo "[WARN] jq not available — skipping config key analysis"
  fi
fi

# Architecture docs freshness
echo "=== Doc freshness ==="
find docs/ -name "*.md" -o -name "*.puml" -o -name "*.drawio" 2>/dev/null \
  | while read f; do
      last=$(git log -1 --format="%ad" --date=short -- "$f" 2>/dev/null || echo "never committed")
      echo "$last  $f"
    done | sort
```

---

## Phase 7: Dead Code Detection

Goal: Find unused code in .NET / C# projects.

```bash
# Unused using statements via Roslyn (requires dotnet-format or build analyzers)
dotnet build /p:RunAnalyzers=true /p:TreatWarningsAsErrors=false 2>&1 \
  | grep -i "unused\|IDE0005\|CS8019" | head -30

# Find classes/interfaces never referenced outside their own file
echo "=== Potentially orphaned types ==="
python3 - <<'PYEOF'
import os, re

types = {}
for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d not in ('bin','obj','.git')]
    for f in files:
        if not f.endswith('.cs'):
            continue
        path = os.path.join(root, f)
        with open(path, encoding='utf-8', errors='ignore') as fh:
            content = fh.read()
        for match in re.finditer(r'\b(class|interface|enum)\s+(\w+)', content):
            name = match.group(2)
            types.setdefault(name, {'defined_in': path, 'ref_count': 0})

# Count references across all files
all_content = {}
for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d not in ('bin','obj','.git')]
    for f in files:
        if f.endswith('.cs'):
            path = os.path.join(root, f)
            with open(path, encoding='utf-8', errors='ignore') as fh:
                all_content[path] = fh.read()

for name, info in types.items():
    for path, content in all_content.items():
        if path == info['defined_in']:
            continue
        if re.search(r'\b' + re.escape(name) + r'\b', content):
            info['ref_count'] += 1

for name, info in sorted(types.items(), key=lambda x: x[1]['ref_count']):
    if info['ref_count'] == 0:
        print(f"ORPHAN: {name}  ({info['defined_in']})")
PYEOF

# Find commented-out code blocks (dead code smell)
echo "=== Commented-out code blocks ==="
rg "^\s*//\s*(public|private|protected|if|for|var|return|new |void )" \
  --type cs --count 2>/dev/null | sort -t: -k2 -nr | head -20

# Unreferenced MSSQL stored procedures / views (compare .sql files vs C# calls)
echo "=== Potentially unused SQL objects ==="
find . -name "*.sql" | while read f; do
  proc=$(basename "$f" .sql)
  if ! rg "$proc" --type cs -q 2>/dev/null; then
    echo "SQL_ORPHAN: $f"
  fi
done
```

---

## Golden Rules

1. **Read-only**: Never modify source code. All findings go into the report only.
2. **Quantify, don't qualify**: Report counts, percentages, trends. Never "messy" or "bad".
3. **Trace to source**: Every finding must include file path, line number, commit reference.
4. **Prioritize by impact**: Rank findings by churn + complexity + coupling score.
5. **Validate before asserting**: Don't assume — run checks and report what you find.
6. **Respect exclusions**: Always skip `bin/`, `obj/`, `node_modules/`, `dist/`, `.git/`.
7. **No assumptions on intent**: If a pattern is unclear, mark as "needs human review".
8. **Report good and bad**: Include well-structured areas to provide contrast.
9. **Degrade gracefully**: If a tool is missing, skip that sub-check, log the gap, continue.
10. **Generate missing scripts at runtime**: If a required Python script does not exist, generate it inline before executing (do not abort the phase).

---

## Environment Variables (Optional Tuning)

```bash
export ARCHAEOLOGIST_MAX_FILES=1000        # Stop after N files
export ARCHAEOLOGIST_MIN_COMPLEXITY=10    # Report complexity >= N only
export ARCHAEOLOGIST_CHURN_DAYS=365       # Analyse last N days of churn
export ARCHAEOLOGIST_OUTPUT_FORMAT=markdown  # markdown | json
export ARCHAEOLOGIST_EXCLUDE_TEST=false   # Include test files in analysis
```

---

## Install Dependencies (Windows)

```powershell
# ripgrep
winget install BurntSushi.ripgrep.MSVC

# jq
winget install jqlang.jq

# python3
winget install Python.Python.3

# jscpd (duplication)
npm install -g jscpd

# coverlet (test coverage)
dotnet tool install --global coverlet.console
```

---

## Troubleshooting

| Issue | Diagnosis | Fix |
|-------|-----------|-----|
| `rg: command not found` | ripgrep not installed | `winget install BurntSushi.ripgrep.MSVC` |
| `dotnet: command not found` | .NET SDK missing | Install from https://dotnet.microsoft.com |
| `jq: command not found` | jq not installed | `winget install jqlang.jq` |
| Python script errors | Missing python3 | `winget install Python.Python.3` |
| Cyclic dep check hangs | Too many `.csproj` | Run from solution root, not drive root |
| Phase 4 no violations found | Folder names don't match | Update layer dir variables at top of Phase 4 |
| Coverage collection fails | coverlet not installed | `dotnet tool install --global coverlet.console` |
| False positives (dead code) | Reflection / dynamic loading | Mark as "needs human review" in report |

---

## Rollback & Cleanup

This skill is **read-only**. No source code, database, or config is modified.

Clean up analysis artifacts:
```bash
rm -f actual_deps.txt duplication_report.json
rm -rf ./coverage-results
```

---

## Next Steps After Analysis

1. Human review of top 10 hotspots (score > 7.0).
2. Prioritize by: `churn × complexity × coupling`.
3. Create work items for top 3 findings.
4. Write Architecture Decision Records (ADRs) for recurring layer violations.
5. Schedule recurring archaeology (monthly) to track decay trends.
