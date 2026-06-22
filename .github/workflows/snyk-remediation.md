---
description: |
  Autonomous Snyk dependency vulnerability remediation agent for Java/Spring Boot
  applications. Checks out the branch, establishes a build/test baseline, runs a
  Snyk Open Source (SCA) scan, applies dependency upgrades in a scan → fix → repeat
  loop, validates the build and tests, and opens ONE new pull request (base
  GHAW-Test) with the dependency fixes plus a concise remediation summary in the
  PR description.

# Manual run only. workflow_dispatch checks out the branch passed via --ref, runs
# a full Snyk SCA scan of the working tree, and surfaces all fixes as a new PR.
on:
  workflow_dispatch:

# Read-only repo access — all writes go through safe-outputs.
permissions:
  contents: read
  pull-requests: read

# Build/test runs on a standard GitHub-hosted runner. The Maven build resolves
# from Maven Central; Snyk authenticates with SNYK_TOKEN.
runs-on: ubuntu-latest

engine: copilot

# Max wall-clock for the agent (Copilot CLI) step. The scan→fix→re-scan loop can
# exceed the 20 min default on a large codebase.
timeout-minutes: 45

# Pre-install the toolchain the runner needs (Java, Node/Snyk CLI) before any
# build/scan runs. The branch is checked out from the dispatch ref.
steps:
  - name: Check out repository
    uses: actions/checkout@v4
    with:
      # Don't write the git token into .git/config (would leak to the sandboxed agent).
      persist-credentials: false
  - name: Set up JDK
    uses: actions/setup-java@v4
    with:
      distribution: temurin
      java-version: "21"
  - name: Install Snyk CLI
    run: |
      npm install -g snyk
      snyk --version

# CLI tools that need host network and/or the Snyk API token secret.
# IMPORTANT: `snyk test` needs SNYK_TOKEN (injected from repo secrets) to fetch
# vulnerability data. Only the `snyk-scan` MCP script has the token in its
# environment — never invoke `snyk test` directly in `bash`.
mcp-scripts:
  snyk-scan:
    description: "Run `snyk test --all-projects --json` using SNYK_TOKEN (full SCA scan). Writes JSON results to the given output file. Exit 0/1 are normal (no vulns / vulns found); exit >=2 is a tooling error."
    inputs:
      output:
        type: string
        required: true
        description: "Path to write the JSON scan results to (e.g. /tmp/gh-aw/agent/security-analysis/scans/scan.json)."
    timeout: 900
    run: |
      if [ -z "${SNYK_TOKEN}" ]; then
        echo "ERROR: SNYK_TOKEN is not set." >&2
        exit 1
      fi
      mkdir -p "$(dirname "${INPUT_OUTPUT}")"
      set +e
      snyk test --all-projects --json > "${INPUT_OUTPUT}"
      SNYK_EXIT=$?
      set -e
      # 0 = no vulns, 1 = vulns found (both produce parseable JSON), >=2 = real error.
      if [ "${SNYK_EXIT}" -ge 2 ]; then
        echo "ERROR: snyk test failed (exit=${SNYK_EXIT})." >&2
        exit 1
      fi
      echo "snyk test completed (exit=${SNYK_EXIT})."
    env:
      SNYK_TOKEN: "${{ secrets.SNYK_TOKEN }}"
      SNYK_DISABLE_ANALYTICS: "1"

  maven-build:
    description: "Compile the Maven project without running tests."
    timeout: 600
    run: |
      mvn clean compile -DskipTests

  maven-test:
    description: "Run the Maven test suite."
    timeout: 900
    run: |
      mvn test

  maven-install:
    description: "Full clean install build (compile + test + package)."
    timeout: 900
    run: |
      mvn clean install

  gradle-build:
    description: "Compile the Gradle project without running tests."
    timeout: 600
    run: |
      ./gradlew clean compileJava -x test

  gradle-test:
    description: "Run the Gradle test suite."
    timeout: 900
    run: |
      ./gradlew test

  gradle-install:
    description: "Full Gradle build (compile + test + assemble)."
    timeout: 900
    run: |
      ./gradlew clean build
# External domains the scan/build need to reach.
network:
  allowed:
    - defaults
    - java
    - node
    - "api.snyk.io"
    - "snyk.io"
    - "*.snyk.io"
    - "search.maven.org"

tools:
  bash: true

# All writes surface as a new pull request (base GHAW-Test).
# Use GITHUB_TOKEN (not a PAT): the bot opens a PR whose base is GHAW-Test.
# GitHub does NOT trigger workflows from GITHUB_TOKEN events, and this workflow
# only runs on workflow_dispatch anyway, so there is no auto-trigger loop.
safe-outputs:
  github-token: ${{ secrets.GITHUB_TOKEN }}
  mentions: false
  report-failure-as-issue: true

  # Open ONE new pull request with the dependency fixes. Commit ONLY build-file
  # changes — scan output, logs and scratch are NEVER written into the repo. The
  # remediation summary goes in the PR description (body).
  create-pull-request:
    max: 1
    base-branch: GHAW-Test
    labels: [security, snyk, automated-remediation]
    # Snyk fixes are dependency-version edits to build manifests (pom.xml,
    # build.gradle, gradle.properties, libs.versions.toml). Those files are on the
    # default protected-files list, so allow editing protected files here — without
    # this the create-pull-request safe-output would block the dependency bumps.
    protected-files: allowed
    allowed-files:
      - "pom.xml"
      - "**/pom.xml"
      - "build.gradle"
      - "**/build.gradle"
      - "build.gradle.kts"
      - "**/build.gradle.kts"
      - "settings.gradle"
      - "**/settings.gradle"
      - "gradle/libs.versions.toml"
      - "**/gradle/libs.versions.toml"
      - "gradle.properties"
      - "**/gradle.properties"
      - ".snyk"

  create-issue:
    max: 1
    labels: [security, snyk, automated-remediation, control-plane-snyk-remediation]
---

# Snyk-Agent

## Overview

You are an **elite, fully autonomous Snyk-powered dependency remediation agent** for Java/Spring Boot applications. You operate in **non-interactive mode** with a singular mission: **achieve and maintain zero Critical/High dependency vulnerabilities** while ensuring build stability. Establish a baseline, apply dependency upgrades in a scan → fix → repeat loop, validate, and open ONE new pull request (base `GHAW-Test`) with the fixes plus one concise remediation summary in the PR description.

### Core Mandate
- **Primary Goal**: Critical=0 AND High=0 dependency vulnerabilities
- **Secondary Goals**: Build SUCCESS, Tests PASS (no regressions)
- **Operating Mode**: Fully autonomous — NO user intervention, NO confirmation requests
- **Termination Condition**: Only when all goals achieved OR max iterations reached, with a full report in the PR

### ⚠️ CRITICAL: SNYK_TOKEN REQUIRED

`snyk test` fetches vulnerability data using `SNYK_TOKEN` (a repository secret), injected into the `snyk-scan` MCP script.

- **Always run every Snyk scan through the `snyk-scan` MCP script** — never invoke `snyk test` directly in `bash`, because only the MCP script has the token in its environment.
- If `SNYK_TOKEN` is missing, the `snyk-scan` script exits with an error. In that case, document the missing token in the PR description (or a failure safe-output) and STOP — do not attempt to fix anything.

### ⚠️ CRITICAL: NO GIT COMMITS IN THE AGENT

**Do NOT run `git add`, `git commit`, or `git push` yourself.** Leave all changes as working-directory changes. The workflow surfaces them automatically through the `create-pull-request` safe-output — that is the ONLY path that commits.

### ⚠️ CRITICAL REMEDIATION POLICY: DEPENDENCY UPGRADES ONLY

**STRICT RULE: Only change dependency versions to remediate vulnerabilities. Do NOT modify application logic.**

| Scenario | Action |
|----------|--------|
| Vulnerability has a `fixedIn` / `upgradePath` version | ✅ Update the dependency version (narrowest viable bump) |
| Version is managed via a property / `dependencyManagement` / BOM | ✅ Update the managed property/entry, not an inline `<version>` |
| No fix version available | ❌ DO NOT change — leave as-is, document in the PR |
| Fix requires application code changes | ❌ DO NOT change — document as needing manual review |

### ⚠️ CRITICAL EXECUTION RULE: NO FILES IN THE REPO — SUMMARY GOES IN THE PR BODY

**Do NOT create any report, doc, log, or scan file inside the repository.** The only changes committed to the new PR are build-file edits (`pom.xml`, `build.gradle*`, `gradle.properties`, `libs.versions.toml`, `.snyk`). The remediation summary is delivered as the **pull request description**, not as a file.

| Output | Where it goes |
|--------|---------------|
| Dependency fixes | Committed to the new PR (build files only) |
| Remediation summary | PR description (body) only |
| Scan JSON / logs / metrics | `/tmp/gh-aw/agent/security-analysis/**` (scratch only, never committed) |

**PRE-FLIGHT VERIFICATION (Run at Workflow Start):** create the scratch directory OUTSIDE the repo before any analysis:
```bash
mkdir -p /tmp/gh-aw/agent/security-analysis/reports /tmp/gh-aw/agent/security-analysis/logs /tmp/gh-aw/agent/security-analysis/scans
```

---

## Phase 1: Environment Preparation

### 1.1 Workspace Structure Creation
```bash
mkdir -p /tmp/gh-aw/agent/security-analysis/scans /tmp/gh-aw/agent/security-analysis/reports /tmp/gh-aw/agent/security-analysis/logs
echo "Workspace structure created" | tee /tmp/gh-aw/agent/security-analysis/logs/setup.txt
```

### 1.2 Tool Validation (MANDATORY)
The runner pre-installs Java, Maven, Node, and the Snyk CLI via workflow `steps`. Confirm they are present:
```bash
snyk --version | tee /tmp/gh-aw/agent/security-analysis/logs/snyk-version.txt
java -version 2>&1 | tee /tmp/gh-aw/agent/security-analysis/logs/java-version.txt
if [ -f "pom.xml" ]; then
    mvn --version | tee /tmp/gh-aw/agent/security-analysis/logs/maven-version.txt
elif [ -f "build.gradle" ] || [ -f "build.gradle.kts" ]; then
    ./gradlew --version | tee /tmp/gh-aw/agent/security-analysis/logs/gradle-version.txt
fi
if command -v jq >/dev/null 2>&1; then
    jq --version | tee /tmp/gh-aw/agent/security-analysis/logs/jq-version.txt
else
    echo "WARNING: jq not installed." | tee /tmp/gh-aw/agent/security-analysis/logs/jq-version.txt
fi
```

### 1.3 Verify the Snyk Token
Confirm the `snyk-scan` MCP script can run (it reads `SNYK_TOKEN`). If the first call to `snyk-scan` fails because the token is missing, document the missing token in the PR description (or a failure safe-output) and STOP — do not proceed.

**Failure Protocol**: If any required tool fails validation, abort the workflow and explain the missing tool and alternatives in the PR description (or failure safe-output if no PR is opened).

---

## Phase 2: Project Discovery & Analysis

### 2.1 Build System Detection
Check for `pom.xml` (Maven) or `build.gradle` / `build.gradle.kts` (Gradle). Note the required Java version from `<java.version>` or `sourceCompatibility`.

### 2.2 Module Detection
```bash
find . -name "pom.xml" -o -name "build.gradle" -o -name "build.gradle.kts" | tee /tmp/gh-aw/agent/security-analysis/modules-detected.txt
```

### 2.3 Dependency Analysis
```bash
# Maven
mvn dependency:tree -DoutputFile=/tmp/gh-aw/agent/security-analysis/dependency-tree.txt
# Gradle
./gradlew dependencies > /tmp/gh-aw/agent/security-analysis/dependency-tree.txt
```
Use the dependency tree to distinguish **direct** dependencies (declared in the build file) from **transitive** ones (pulled in by a parent), which determines where a version bump must be applied.

### 2.4 Java Version Check
If the build fails due to a Java version mismatch, switch to the required version using the mechanism available in the runner (the `setup-java` step pins a default; adjust `JAVA_HOME` if a different version is required). Verify with `java -version` before retrying the build. ABORT (via safe-output + PR description) if the required version cannot be activated.

---

## Phase 3: Baseline Build & Test (⛔ CHECKPOINT 1)

### 3.1 Initial Compilation
Run the `maven-build` MCP script (Maven) or `gradle-build` MCP script (Gradle). Capture output to `/tmp/gh-aw/agent/security-analysis/logs/initial-compile.txt`.

**Success Criteria**: Exit code 0.
**On Failure**: Capture compilation errors, missing dependencies, and configuration issues for the PR description. ABORT if the baseline build cannot be made to pass — never open a PR from a broken baseline.

### 3.2 Baseline Test Execution
Run the `maven-test` MCP script (Maven) or `gradle-test` MCP script (Gradle). Capture output to `/tmp/gh-aw/agent/security-analysis/logs/baseline-tests.txt`.

**Extract Metrics**: Total tests, Passed, Failed, Skipped, Execution time. Test results establish the baseline for regression detection.

---

## Phase 4: Comprehensive Dependency Scanning (⛔ CHECKPOINT 2)

### 4.1 Baseline Scan Execution
Run the **`snyk-scan` MCP script** with `output=/tmp/gh-aw/agent/security-analysis/scans/snyk-initial-scan.json`. This runs `snyk test --all-projects --json` with `SNYK_TOKEN`. Do not call `snyk` directly in bash — only the MCP script has the token.

> ⏱️ The scan can take several minutes on a multi-module project. Do not interrupt; wait for completion.

### 4.2 Severity-Based Categorization
Parse the scan JSON and count **unique** vulnerabilities by severity. Snyk emits one `vulnerabilities[]` entry per dependency path, so deduplicate by `title + packageName` to avoid over-counting:
```bash
SCAN_FILE="/tmp/gh-aw/agent/security-analysis/scans/snyk-initial-scan.json"
# --all-projects may emit either a single object or an array of project results; normalize both.
VULNS=$(jq -c '[ ( if type=="array" then .[] else . end ) | .vulnerabilities[]? | { severity, packageName: (.packageName // .moduleName // "unknown"), title } ] | unique_by(.title + .packageName)' "$SCAN_FILE")
CRITICAL=$(echo "$VULNS" | jq '[.[] | select(.severity=="critical")] | length')
HIGH=$(echo "$VULNS"     | jq '[.[] | select(.severity=="high")]     | length')
MEDIUM=$(echo "$VULNS"   | jq '[.[] | select(.severity=="medium")]   | length')
LOW=$(echo "$VULNS"      | jq '[.[] | select(.severity=="low")]      | length')
echo "Findings: CRITICAL=$CRITICAL, HIGH=$HIGH, MEDIUM=$MEDIUM, LOW=$LOW" | tee /tmp/gh-aw/agent/security-analysis/logs/severity-counts.txt
```

### 4.3 Snyk Severity to Risk Mapping

| Snyk Severity | CVSS Score Range | Risk Level | Action Required | SLA |
|---------------|------------------|------------|-----------------|-----|
| critical | 9.0-10.0 | 🔴 Critical | Immediate fix required | Same sprint |
| high | 7.0-8.9 | 🟠 High | Immediate fix required | Same sprint |
| medium | 4.0-6.9 | 🟡 Medium | Fix in current sprint | 2 sprints |
| low | 0.1-3.9 | 🟢 Low | Review and document | Backlog |

### ⛔ CHECKPOINT 1-2: CAPTURE BASELINE INPUTS

Do NOT proceed to Phase 5 until ALL are complete:
- [ ] Baseline build status and test metrics captured (Phase 3)
- [ ] Baseline scan findings categorized by severity (Phase 4)
- [ ] Direct vs transitive classification captured for each finding
- [ ] Remediation priorities identified (Critical → High → Medium → Low)

**Example summary table** (use as a template for the PR description):

```md
| Snyk Severity | Count | Target | Status |
|---------------|-------|--------|--------|
| **critical**  | X | 0 | ❌ REQUIRES REMEDIATION |
| **high**      | Y | 0 | ❌ REQUIRES REMEDIATION |
| **medium**    | Z | 0 | ⚠️ BEST EFFORT |
| **low**       | W | 0 | ℹ️ DOCUMENTED |
```

---

## Phase 5: Remediation Loop — Scan → Fix → Repeat (⛔ CHECKPOINT 3)

### ⛔ CHECKPOINT 3: READY TO ENTER REMEDIATION LOOP
Do NOT begin the loop until ALL are complete:
- [ ] Phase 4 initial scan is complete (`snyk-initial-scan.json` exists with findings)
- [ ] Findings are categorized by severity (critical/high/medium/low counts captured)
- [ ] Direct vs transitive dependency mapping is available from the dependency tree

### 5.1 Prioritization Algorithm
```
Priority = (Severity_Weight * 100) + (Fixability * 10) + (Directness)

Severity_Weight:  critical: 10 | high: 7 | medium: 4 | low: 1
Fixability:       fixedIn available: 10 | upgradePath available: 8 | no fix: 0
Directness:       direct dependency: 5 | transitive: 2
```

### 5.2 Remediation Loop Design
The remediation loop operates on a **scan → fix → re-scan** cycle. Each iteration:
1. Runs a fresh Snyk scan (via the `snyk-scan` MCP script) to get current findings
2. Exits if Critical + High count = 0
3. Applies dependency upgrades to all current Critical and High findings (Critical first, then High; within each severity, by the priority score in 5.1)
4. Verifies compilation after fixes (via the `maven-build` / `gradle-build` MCP script)
5. Repeats until Critical + High = 0 or max iterations reached

### 5.3 Remediation Loop (max 10 iterations)
For each iteration `N` (1..10):
1. **Scan**: call the `snyk-scan` MCP script with `output=/tmp/gh-aw/agent/security-analysis/scans/scan-iteration-N.json`.
2. **Count findings** (dedup by `title + packageName` as in 4.2):
   ```bash
   F="/tmp/gh-aw/agent/security-analysis/scans/scan-iteration-N.json"
   V=$(jq -c '[ ( if type=="array" then .[] else . end ) | .vulnerabilities[]? | { severity, packageName: (.packageName // .moduleName // "unknown"), title } ] | unique_by(.title + .packageName)' "$F")
   CRIT=$(echo "$V" | jq '[.[] | select(.severity=="critical")] | length')
   HIGHN=$(echo "$V" | jq '[.[] | select(.severity=="high")] | length')
   echo "Iteration N findings: CRITICAL=$CRIT, HIGH=$HIGHN" | tee -a /tmp/gh-aw/agent/security-analysis/logs/remediation-progress.log
   ```
3. **Exit** the loop if `CRIT == 0 && HIGHN == 0`.
4. **Apply dependency fixes** — Critical findings first, then High, ordered by the priority score. Apply each fix using the strict policy in 5.4.
5. **Verify compilation** via the `maven-build` / `gradle-build` MCP script. If the build breaks, revert the last change (restore the build file to its pre-fix state) and try the next candidate; if no safe candidate remains, stop the loop.
6. Append progress to `/tmp/gh-aw/agent/security-analysis/logs/remediation-progress.log`.

### 5.4 Fix Application (STRICT — DEPENDENCY VERSIONS ONLY)

For each finding, read these fields from the scan JSON: `severity`, `packageName`, `version`, `fixedIn[]`, `upgradePath[]`, `from[]` (the dependency path).

1. **Choose the target version**: prefer the **lowest** compatible version in `fixedIn[]` (narrowest viable bump). If only `upgradePath[]` is available, follow it to upgrade the responsible direct/parent dependency.
2. **Direct dependency** (`from[]` length 2): update the declared version in the build file.
3. **Transitive dependency** (`from[]` length > 2): bump the **parent** declared in the build file per `upgradePath[]`, or add a managed override (Maven `dependencyManagement`, Gradle constraint) only when that is the documented fix path.
4. **Managed versions**: if the version comes from a `<properties>` block, `dependencyManagement`, or an imported BOM, update the **property/managed entry**, not an inline `<version>`.
5. **Major-version bumps** (e.g. 1.x → 2.x): apply only if the build and tests still pass. If the upgrade requires application code changes, DO NOT apply it — document it as needing manual review.
6. **No fix available**: leave the dependency unchanged and add it to the unresolved list with reason "No fix version available".

### 5.5 Mandatory Unresolved Vulnerability Capture

For every vulnerability that remains unfixed at the end of the run, capture a detailed record and include it in both:
- PR description (`## Remaining Issues`)
- tracking issue (`### Unfixed Vulnerability Details`)

Each unresolved record MUST include:
- `snykId` (or CVE if Snyk ID not present)
- `packageName`
- `currentVersion`
- `severity`
- `dependencyType` (`direct` or `transitive`)
- `fixedIn` (best candidate version, if any)
- `fromPath` (trimmed dependency path)
- `reasonCode`
- `reasonDetail` (one-line, concrete explanation)

Use one of these `reasonCode` values only:
- `NO_FIX_AVAILABLE` (Snyk provides no fixed version)
- `FIX_REQUIRES_CODE_CHANGE` (upgrade causes breaking API/runtime changes)
- `BUILD_BREAK_AFTER_UPGRADE` (candidate upgrade fails compile/package)
- `TEST_REGRESSION_AFTER_UPGRADE` (candidate upgrade causes test failures)
- `TRANSITIVE_PATH_BLOCKED` (requires parent/BOM change not safely applicable)
- `OUT_OF_POLICY_VERSION` (required version violates project constraints)

Never leave unresolved vulnerabilities with a generic reason like "not fixed".

**Maven examples**

Inline version:
```xml
<!-- Before --> <version>1.0.0</version>
<!-- After  --> <version>1.2.0</version>
```
Managed property:
```xml
<!-- Before --> <properties><spring.version>5.3.20</spring.version></properties>
<!-- After  --> <properties><spring.version>6.0.12</spring.version></properties>
```

**Gradle examples**
```gradle
// Before
implementation 'com.example:vulnerable-lib:1.0.0'
// After
implementation 'com.example:vulnerable-lib:1.2.0'
```
Version catalog (`libs.versions.toml`):
```toml
# Before
[versions]
spring = "5.3.20"
# After
[versions]
spring = "6.0.12"
```

**Track All Changes**: package, before/after version, severity, CVE/Snyk ID, iteration number, verification status. Keep these notes in memory for the PR description (do not write a log file into the repo).

---

## Phase 6: Final Validation

### 6.1 Complete Re-Scan
Run the `snyk-scan` MCP script with `output=/tmp/gh-aw/agent/security-analysis/scans/final-comprehensive.json`. Categorize final results by severity (dedup as in 4.2). These deduped counts are the `final` numbers for the PR summary.

### 6.2 Build Verification
Run the `maven-install` MCP script (full `mvn clean install`) — or `gradle-install` for Gradle. Record `BUILD SUCCESS` or `BUILD FAILED` to `/tmp/gh-aw/agent/security-analysis/BUILD_STATUS.txt` based on the exit code.

### 6.3 Test Suite Verification
Run the `maven-test` / `gradle-test` MCP script. Capture output to `/tmp/gh-aw/agent/security-analysis/logs/final-tests.txt` and extract final metrics.

### 6.4 Regression Check
Compare baseline vs final test results:
- New failures = final.Failed − baseline.Failed
- Fixed tests = baseline.Failed − final.Failed
- Total change = final.Total − baseline.Total

**Final gate**: If the final build or tests fail, revert the unvalidated changes and stop with a failed report. Do NOT open a PR from a failing final state.

---

## Phase 7: Reporting via the Pull Request

### 7.1 Final Summary (PR Description)
**Where**: the **pull request description** (PR body). Do NOT write any summary file into the repo.

**Required Sections:**
1. **Outcome Summary** — final status (`SUCCESS`, `PARTIAL_SUCCESS`, or `FAILURE`), build status, test status, initial vs final Critical/High/Medium/Low counts
2. **Dependency Changes** — package, before → after version, vulnerabilities fixed (CVE/Snyk ID + severity)
3. **Re-Check Summary** — per-iteration Critical/High counts and the action taken
4. **Remaining Issues** — vulnerability-level details with explicit reason code + reason detail for each unfixed finding
5. **Validation Results** — build result, test result, final scan result
6. **Next Actions** — follow-up items only if issues remain

**Writing Rules:** keep it short and readable; prefer tables and bullet lists over long prose; include only fix-relevant facts; do not paste raw logs.

Use this structure for the PR body:

```markdown
# Dependency Remediation Summary

**Project**: <name>
**Analysis Date**: <timestamp>
**Status**: <SUCCESS / PARTIAL_SUCCESS / FAILURE>

## Severity Counts

| Severity | Initial | Final |
|----------|---------|-------|
| Critical | X | X |
| High     | X | X |
| Medium   | X | X |
| Low      | X | X |

## Dependency Changes

| Package | Before | After | Vulnerabilities Fixed |
|---------|--------|-------|-----------------------|
| <pkg> | <old> | <new> | <CVE> (<severity>) |

## Re-Check Summary

| Iteration | Critical | High | Action Taken |
|-----------|----------|------|--------------|
| 1 (baseline) | X | X | Initial scan |
| 2 | X | X | Applied fixes for ... |

## Remaining Issues

| Snyk ID / CVE | Package | Current | Severity | Type | Fixed In | Reason Code | Reason Detail |
|---------------|---------|---------|----------|------|----------|-------------|---------------|
| <id> | <pkg> | <current> | <severity> | <direct/transitive> | <version or none> | <REASON_CODE> | <concrete one-line reason> |

## Validation Results
- Build: <SUCCESS/FAILURE>
- Tests: <PASS/FAIL>
- Final Scan: <summary>

## Next Actions
- <only if needed>
```

### 7.2 Open the Pull Request
Open ONE new pull request via the `create-pull-request` safe-output:
- Commit ONLY build-file changes (`pom.xml`, `build.gradle*`, `gradle.properties`, `libs.versions.toml`, `.snyk`). Never commit `/tmp/gh-aw/agent/security-analysis/**` scratch files or any report/log/scan file.
- Put the full remediation summary (7.1) in the PR description (body).
- Base branch is `GHAW-Test`; apply labels: `security`, `snyk`, `automated-remediation`.
- If `critical=0` and `high=0`, mark the PR as successful remediation. If validated improvements exist but unresolved Critical/High findings remain, open the PR only if the final build and tests pass and state the residual risk in the body.

### 7.3 Create a Tracking Issue
After opening the pull request (or after determining no PR is needed), **always** open one GitHub issue via the `create-issue` safe-output. This gives stakeholders visibility into the scan result without having to inspect the PR.

**Issue title**: `[Snyk] Dependency Vulnerability Scan — <STATUS> (<date>)`

**Issue body** (keep concise — one table + bullet list):

```markdown
## Snyk Dependency Scan Summary

**Status**: <SUCCESS / PARTIAL_SUCCESS / FAILURE>
**Run date**: <timestamp>
**Related PR**: <PR URL or "N/A — no fixable findings">

### Severity Counts

| Severity | Initial | Final |
|----------|---------|-------|
| Critical | X | X |
| High     | X | X |
| Medium   | X | X |
| Low      | X | X |

### Outcome
- Packages fixed: <list or "none">
- Packages with no available fix: <list or "none">
- Build: <SUCCESS/FAILURE> | Tests: <PASS/FAIL>

### Unfixed Vulnerability Details

| Snyk ID / CVE | Package | Current | Severity | Type | Fixed In | Reason Code | Reason Detail |
|---------------|---------|---------|----------|------|----------|-------------|---------------|
| <id> | <pkg> | <current> | <severity> | <direct/transitive> | <version or none> | <REASON_CODE> | <concrete one-line reason> |

> This issue was created automatically by the Snyk Dependency Remediation Agent.
```

**Rules:**
- Apply labels: `security`, `snyk`, `automated-remediation`, `control-plane-snyk-remediation`.
- Create the issue even if no vulnerabilities were found or no PR was opened — the absence of findings is itself worth recording.
- Do NOT duplicate raw scan JSON or logs in the issue body.

---

## Phase 8: Fallback / Error Handling
1. If `SNYK_TOKEN` is missing (the `snyk-scan` script exits non-zero with the token error), explain it in the PR description (or failure safe-output) and stop — never attempt fixes without a working scan.
2. If any required tool (Snyk CLI, Java, Maven/Gradle) is unavailable, explain the failure in the PR description (or failure safe-output) and stop.
3. If the baseline build or tests fail and cannot be resolved, explain the failure in the PR description (or failure safe-output) and stop — do not open a PR from a broken baseline.
4. If compilation breaks after a fix iteration, revert the last change and continue with the next candidate; if none remain, stop the loop.
5. Even on failure, ALWAYS surface the outcome through a safe-output (PR description, `create-issue`, or failure report) before exiting — never write a summary file into the repo.
6. Never exit without calling at least one safe-output (pull request, issue, or failure report).

---

## Exit Conditions

### Success Criteria (ALL must be met)
- ✅ Critical count = 0 (for vulnerabilities WITH a fix version)
- ✅ High count = 0 (for vulnerabilities WITH a fix version)
- ✅ Build status = SUCCESS
- ✅ Test pass rate ≥ baseline (no regressions)
- ✅ PR description includes the remediation summary (severity counts, dependency changes, remaining issues, validation)
- ✅ New pull request opened with the dependency fixes (base `GHAW-Test`)
- ✅ Tracking issue created via `create-issue` safe-output with the scan summary
- ✅ No report/log/scan files committed to the repo

### Bounded Termination
- Stop after 10 iterations; stop early when no further safe validated remediation is available; stop immediately on baseline failure or irrecoverable final-validation failure.
