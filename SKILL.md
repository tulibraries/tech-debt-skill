---
name: tech-debt-skill
description: Generate a comprehensive technical debt audit for a codebase. Analyzes security vulnerabilities, dependency freshness, code complexity, test coverage, and produces a single self-contained HTML report with visuals and actionable recommendations.
---

# Tech Debt Audit Skill

Generate a comprehensive technical debt audit for the codebase. This skill runs multiple
code-quality and security tools, captures visuals, and compiles everything into a **single,
self-contained HTML page** with an executive summary, one section per tool, embedded
screenshots, and the top 3 recommended actions.

## Target Directory

If the user specifies a directory, audit that directory. Otherwise, audit the current working directory.

## Pre-Audit Check

Before running tools, check if the client is using SonarQube. If so, some metrics may already
be tracked and running certain tools (like Skunk) may be duplicate effort. Note this in the report.

## Tool Installation Policy

**IMPORTANT**: Always install required tools before running them. Do NOT skip any tool
because it's missing from the Gemfile. Install Ruby tools globally with `gem install` and
run them directly (not via `bundle exec`) to avoid version conflicts with the project's
Gemfile.

## Deterministic Rubric Rules

Apply these rules for this Codex skill so repeated runs produce comparable results.

### 1. Machine-checkable inputs only

Every scored category must be backed by files in `$OUT/raw/` or by repository files that
can be checked directly.

- `Security`: `bundler-audit.txt`, `brakeman.txt`, `bundler-leak.txt`, `trivy.json`,
  `npm-audit.txt` or `yarn-audit.txt`
- `Dependencies`: `bundle-outdated-strict.txt`, `libyear.txt`, `npm-outdated.txt`
- `Coverage`: `coverage-last-run.json`
- `Complexity`: `rubycritic.txt`, `skunk.txt`, RubyCritic HTML output
- `Maintainability`: presence/absence checks for CI, setup scripts, setup docs,
  `.env.example`, exception tracking, and performance monitoring

Do not use subjective language such as "good setup" unless it is tied to the checks below.

### 2. Fixed scan scope

Unless the user explicitly asks for a different scope, use these rules:

- Audit the requested target directory only.
- Exclude generated and dependency directories from filesystem-wide scans where the tool
  supports exclusions: `.git`, `node_modules`, `vendor/bundle`, `coverage`, `tmp`,
  existing `tech-debt-audit-*` output directories, and `.ruby-lsp`.
- For Ruby dependency freshness and Ruby dependency security scoring, the authoritative lock
  file is the root `Gemfile.lock` in the target directory. Ignore nested lockfiles such as
  `.ruby-lsp/Gemfile.lock`, `solr/configs/*/Gemfile.lock`, example apps, vendored samples,
  or tool-internal lockfiles unless the user explicitly asks to include them.
- For JavaScript dependency freshness and audit scoring, use the package manager that matches
  the root lockfile in the target directory. For this skill, `yarn.lock` means use Yarn and
  `package-lock.json` means use npm. Do not generate a new lockfile just to make a tool run.
  Ignore nested lockfiles unless the user explicitly asks to include them.
- For the Ruby dependency freshness score, count only direct RubyGems dependencies declared
  in the root `Gemfile` across all groups. Exclude Git- and path-sourced dependencies, Ruby
  default gems, and transitive dependencies from the numeric score. Report excluded updates
  separately because they may still warrant review, but they are not actionable Gemfile
  freshness debt.
- Count repository-owned Dockerfiles and IaC misconfigurations reported by Trivy when they
  are inside the target directory and not in an excluded path. These findings affect the
  `Security` score.

If the repository has multiple first-class applications and no single root lockfile
represents the project, note that the run is partial and score only the app the user
targeted.

### 3. Tool failure handling

If a tool fails, keep its raw output, note the failure in the report, and apply the fallback
rules below instead of guessing.

- `bundler-audit`, `brakeman`, `bundler-leak`, `trivy`, `npm audit`, `yarn audit`:
  failed tools are treated as unavailable inputs for `Security`; score from the remaining
  successful tools and state which inputs were unavailable.
- If the JS audit or outdated command does not match the repository's root lockfile, mark the
  JS portion of the run as non-comparable and do not mix those results into trend comparisons.
- Run `bundle outdated --strict --only-explicit --parseable` for Ruby dependency freshness.
  It lists only updates permitted by the current `Gemfile` requirements and only direct
  dependencies. Discard Git- and path-sourced entries and Ruby default gems before scoring.
- If the command fails with a network, Git-source, or sandbox error, retry it once with the
  access required to refresh the project's configured sources. If it still fails, mark the
  Ruby dependency category unavailable. Use `0` in the numeric table only because the
  template requires a number, and explicitly say the category is unavailable rather than
  "bad".
- `libyear-bundler` is informational only. Never use libyears, release age, or
  `next_rails` output in the dependency score. They conflate intentional constraints and
  transitive dependencies with actionable upgrades.
- If the test suite cannot run but `coverage/.last_run.json` exists, use that file and mark
  coverage as stale.
- If SimpleCov data is missing entirely, `Coverage` is `0`.
- If RubyCritic or Skunk fails, score `Complexity` from the successful tool and mark the
  missing tool as unavailable. If both fail, `Complexity` is `0`.

### 4. Live advisory data and comparability

Security and dependency audit tools can change results over time even when the codebase does
not change. For each run, capture tool versions and note advisory-data volatility in the
report appendix.

- Record the versions of `bundler-audit`, `brakeman`, `bundler-leak`, `trivy`, `npm`, and
  `yarn` when those tools are used.
- Treat `Security` comparisons across runs as valid only when tool versions and advisory
  inputs are materially comparable.
- If the advisory snapshot or tool versions differ and you cannot normalize them, state
  `Security not directly comparable to prior runs`.
- Record the exact Bundler version, the exact `bundle-outdated-strict.txt` output, and a
  checksum of the root `Gemfile.lock`. Dependency-score comparisons are valid only when the
  lockfile checksum and Bundler version are unchanged; newly published permitted releases can
  otherwise change the result.

### 5. Deterministic scoring procedure

Use the same visible 100-point scoring system as the Claude version. Tighten consistency by
standardizing how the existing score bands are interpreted, not by inventing new score bands.

- `Total score` = `Security + Dependencies + Coverage + Complexity + Maintainability`
- `Category badge`: `Pass` for 15-20, `Warning` for 10, `Fail` for 0-5
- If multiple tools contribute to one category, calculate the category from the worst
  qualifying threshold reached by any counted input unless the fallback rules above say to
  ignore that input.
- When a category is unavailable because every required tool failed, keep the numeric table at
  `0` only for template compatibility and explicitly label the category `Unavailable` in the
  prose.

### 6. Deterministic prose generation

Keep the narrative stable across runs by using fixed ordering and sentence templates.

- `Executive summary`: exactly 4 sentences.
- Sentence 1 must report the total score as `This audit scored X/100.`
- Sentence 2 must name the lowest-scoring category or categories.
- Sentence 3 must name the single highest-severity security or dependency issue if one exists;
  otherwise name the largest complexity or coverage problem.
- Sentence 4 must name the strongest area or say `No category scored in the pass range.`
- `Top 3 recommendations`: rank by category severity first, then by numeric magnitude inside
  that category, then alphabetically by file path or gem name as the final tie-breaker.
- Use imperative wording. Start each recommendation title with a verb such as `Upgrade`,
  `Remediate`, `Add`, `Reduce`, or `Document`.
- Do not recommend work that is not directly backed by the collected files or repository
  checks.

### 7. Stable parsing and tie-breakers

When raw outputs contain multiple candidate values, use these tie-breakers so repeated runs
land on the same result.

- For severity-driven categories, use the highest severity present.
- For file-based findings, sort ties by descending metric value, then ascending file path.
- For gem-based findings, sort ties by descending severity, then ascending gem name.
- For percentages, round to the nearest whole number before mapping to score bands.
- If a tool emits both a summary count and itemized rows, trust the summary count unless it is
  obviously inconsistent with the rows.

---

## Step 0: Create the Timestamped Output Directory

All results from this run — raw tool output, the RubyCritic HTML report, screenshots, and
the final report — go into **one single directory** named with the timestamp of the run.

```bash
# Run from the target directory
TS=$(date +%Y%m%d-%H%M%S)
OUT="tech-debt-audit-$TS"
mkdir -p "$OUT/raw" "$OUT/rubycritic" "$OUT/screenshots"
echo "$OUT"
```

Directory layout produced by this skill:

```
tech-debt-audit-YYYYMMDD-HHMMSS/
├── index.html          <- the single self-contained report (the deliverable)
├── raw/                <- raw text/JSON output from every tool
├── rubycritic/         <- RubyCritic's generated HTML report
└── screenshots/        <- PNG screenshots embedded into index.html
```

Save every tool's raw output into `$OUT/raw/` (redirect with `> "$OUT/raw/<tool>.txt" 2>&1`)
so the final report can quote exact numbers and the run is reproducible.

---

## Step 1: Detect Project Type

- **Ruby/Rails**: `Gemfile`, `Gemfile.lock`, `*.rb` files
- **JavaScript/Node.js**: `package.json`, `yarn.lock`, `package-lock.json`
- **Both**: many Rails apps have both

Only run the checks that apply to what you detect.

## Step 2: Security Vulnerabilities

### Ruby (if Gemfile exists)

**bundler-audit** — gem versions with known CVEs
```bash
gem install bundler-audit --no-document
bundle-audit check --update > "$OUT/raw/bundler-audit.txt" 2>&1
```

**Brakeman** — static security analysis for Rails
```bash
gem install brakeman --no-document
brakeman --no-pager -q > "$OUT/raw/brakeman.txt" 2>&1
```

**bundler-leak** — memory-leaking gems
```bash
gem install bundler-leak --no-document
bundle-leak check --update > "$OUT/raw/bundler-leak.txt" 2>&1
```

### Trivy (all projects)

**Trivy** — filesystem scan for vulnerable dependencies (Ruby, JS, OS packages),
leaked secrets, and misconfigurations. Install if missing.
```bash
# Install: brew install trivy  (macOS) or see https://trivy.dev for other platforms
command -v trivy >/dev/null 2>&1 || brew install trivy

# Human-readable table + machine-readable JSON
trivy fs --scanners vuln,secret,misconfig --format table . > "$OUT/raw/trivy.txt" 2>&1
trivy fs --scanners vuln,secret,misconfig --format json  . > "$OUT/raw/trivy.json" 2>&1
```
Use the fixed scope rules above when configuring exclusions. Repository-owned Dockerfiles and
IaC files count toward `Security`; nested tool caches and generated output do not.
Goal: zero HIGH/CRITICAL findings and no leaked secrets. Summarize counts by severity.

### JavaScript (if package.json exists)

```bash
yarn audit > "$OUT/raw/yarn-audit.txt" 2>&1
```

## Step 3: Dependency Freshness

### Ruby (if Gemfile exists)
```bash
# Exit status 1 means permitted updates exist. Retry only real failures (status > 1).
if bundle outdated --strict --only-explicit --parseable > "$OUT/raw/bundle-outdated-strict.txt" 2>&1; then
  BUNDLE_OUTDATED_STATUS=0
else
  BUNDLE_OUTDATED_STATUS=$?
fi
if [ "$BUNDLE_OUTDATED_STATUS" -gt 1 ]; then
  mv "$OUT/raw/bundle-outdated-strict.txt" "$OUT/raw/bundle-outdated-strict-first-attempt.txt"
  if bundle outdated --strict --only-explicit --parseable > "$OUT/raw/bundle-outdated-strict.txt" 2>&1; then
    BUNDLE_OUTDATED_STATUS=0
  else
    BUNDLE_OUTDATED_STATUS=$?
  fi
fi
printf '\n[exit status: %s]\n' "$BUNDLE_OUTDATED_STATUS" >> "$OUT/raw/bundle-outdated-strict.txt"
bundle --version > "$OUT/raw/bundler-version.txt" 2>&1
shasum -a 256 Gemfile.lock > "$OUT/raw/gemfile-lock.sha256" 2>&1

# Informational only. Do not use this output in the dependency score.
gem install libyear-bundler --no-document
libyear-bundler --all > "$OUT/raw/libyear.txt" 2>&1
```
For the numeric score, parse only `bundle-outdated-strict.txt`. Its exit status of `1` means
updates were found, not that the command failed. Determine the denominator from direct root
`Gemfile` declarations that resolve from RubyGems, excluding Git/path declarations and Ruby
default gems. Determine the numerator from the remaining lines in
`bundle-outdated-strict.txt`. Include Git/path updates and libyear output in the report as
informational context only.

### JavaScript (if package.json exists)
```bash
yarn outdated > "$OUT/raw/yarn-outdated.txt" 2>&1
```

## Step 4: Code Coverage

**IMPORTANT**: Run the test suite with coverage enabled BEFORE running Skunk so it reads
fresh SimpleCov data.

```bash
ls -d spec/ test/ 2>/dev/null   # detect framework

# RSpec (spec/ exists)
COVERAGE=true bundle exec rspec > "$OUT/raw/rspec.txt" 2>&1
# Minitest (test/ exists)
# COVERAGE=true bundle exec rake test > "$OUT/raw/test.txt" 2>&1

# Capture the result
cp coverage/.last_run.json "$OUT/raw/coverage-last-run.json" 2>/dev/null
cat coverage/.last_run.json
```

If the test suite cannot run (missing DB, etc.), fall back to the existing
`coverage/.last_run.json` and note in the report that coverage may be stale.
If SimpleCov is not configured, note it and recommend adding it.

## Step 5: Code Complexity — RubyCritic (with visuals)

RubyCritic's default format is **HTML**, which includes the churn-vs-complexity overview
chart we screenshot for the report.

```bash
gem install rubycritic --no-document
# -p sets the output path; --no-browser prevents it opening a window
rubycritic app lib --no-browser -p "$OUT/rubycritic" > "$OUT/raw/rubycritic.txt" 2>&1
```

This produces `$OUT/rubycritic/overview.html` (the scatter plot) plus `code_index.html`,
`churn_index.html`, and `complexity_index.html`.

### Capture screenshots

Render the RubyCritic HTML with headless Chrome and save PNGs into `$OUT/screenshots/`.
Use whichever Chrome/Chromium binary exists:

```bash
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"   # macOS
# CHROME="google-chrome"   # Linux
# CHROME="chromium"        # Linux alt

ABS="$(cd "$OUT/rubycritic" && pwd)"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1280,1400 --screenshot="$(pwd)/$OUT/screenshots/rubycritic-overview.png" \
  "file://$ABS/overview.html"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars \
  --window-size=1280,1600 --screenshot="$(pwd)/$OUT/screenshots/rubycritic-code.png" \
  "file://$ABS/code_index.html"
```

If the `chrome-devtools` MCP tools are available, you may use `navigate_page` +
`take_screenshot` on the same `file://` URLs instead — that also works and can capture
full-page shots. If no headless browser is available at all, skip screenshots and note it
in the report; the report still renders without them.

## Step 6: Combined Metric — Skunk

**PREREQUISITE**: Step 4 (coverage) must have run first.

```bash
gem install skunk --no-document
skunk app lib > "$OUT/raw/skunk.txt" 2>&1
```
`SkunkScore = (Code Smells + Complexity) * Coverage Penalty`. Focus on files with high
complexity + low coverage + high churn.

## Step 7: Framework Health & Stats (Rails)

Always use **rails_stats** instead of Rails' native `rake stats` — it reports richer metrics
(polymorphic associations, schema stats, a cleaner code-to-test ratio). It depends on
**bundle-stats** (the `bundler-stats` gem) and emits its dependency-weight table too, so a
single `rails-stats` call gives you BOTH application code metrics AND dependency metrics. Do
NOT install or run `bundle-stats` separately — that would be redundant.

```bash
bundle show rails > "$OUT/raw/rails-version.txt" 2>&1
ruby -v > "$OUT/raw/ruby-version.txt" 2>&1

# Codebase stats AND dependency weight in one call (rails_stats bundles bundle-stats).
# Capture ALL output — the dependency table prints first, the code-stats table second.
gem install rails_stats --no-document
rails-stats > "$OUT/raw/rails-stats.txt" 2>&1

cloc --exclude-dir=node_modules,vendor,coverage,tmp,.git . > "$OUT/raw/cloc.txt" 2>&1
```
The `rails-stats` output has two tables: a dependency table (`Name | Total Deps | 1st Level
Deps` — gems with large transitive trees are prime removal/replacement candidates) and the
code-stats table (`Name | Files | Lines | LOC | Classes | Methods …`). Check Rails version
support/EOL, code-to-test ratio, and the heaviest dependencies.

## Step 8: Runtime Observability

A production Rails app should be able to answer "did something break?" and "is it slow?".
Check the Gemfile/Gemfile.lock for the presence of tooling in each category. If a category
has **no** gem, flag it in the report as an issue and a departure from best practices.

### Exception Tracking

```bash
grep -iE "sentry|rollbar|honeybadger|bugsnag|airbrake|exception_notification|appsignal" Gemfile.lock \
  > "$OUT/raw/exception-tracking.txt" 2>&1
```
Look for: sentry-ruby/sentry-rails, rollbar, honeybadger, bugsnag, airbrake,
exception_notification, or appsignal. If none is present, the app has no way to know about
production exceptions — flag it as a high-priority gap.

### Performance Monitoring

```bash
grep -iE "newrelic|scout_apm|skylight|ddtrace|datadog|appsignal|rack-mini-profiler|prosopite|bullet" Gemfile.lock \
  > "$OUT/raw/performance-monitoring.txt" 2>&1
```
Look for an APM (newrelic_rpm, scout_apm, skylight, ddtrace/datadog, appsignal) and/or
in-process tools (rack-mini-profiler, bullet, prosopite). If nothing is present, the team is
flying blind on performance regressions — flag it as an issue.

## Step 9: Development Environment

Check for `bin/setup`, `docker-compose.yml`, README setup instructions, CI config
(`.github/workflows/`, `.circleci/`), and `.env.example`.

For `Maintainability`, use only these machine-checkable checks:

- `CI present`: `.github/workflows/*.yml`, `.github/workflows/*.yaml`, `.circleci/config.yml`,
  `.buildkite/`, or equivalent checked-in CI config
- `Setup script present`: `bin/setup`, `script/setup`, `bin/bootstrap`, or `make setup`
- `README setup docs present`: `README*` contains a dedicated setup or installation section
- `.env.example present`: `.env.example`, `.env.sample`, or `.env.template`
- `Exception tracking present`: gems found by Step 8 exception tracking check
- `APM/performance monitoring present`: gems found by Step 8 performance monitoring check

Score `Maintainability` from the number of checks that pass:

- 20: 6 of 6
- 15: 5 of 6
- 10: 3-4 of 6
- 5: 2 of 6
- 0: 0-1 of 6

---

## Step 10: Assemble the Single HTML Report

This is the primary deliverable. Read the template and produce ONE self-contained HTML file
at `$OUT/index.html`.

1. **Read the template**: `report-template.html` (next to this SKILL.md). It uses the
   FastRuby.io styleguide (https://fastruby.github.io/styleguide) palette and the Oxygen
   font, is mobile/tablet friendly (tables scroll horizontally on small screens), links each
   section to the open source tool it used, and contains all CSS inline plus
   `{{PLACEHOLDER}}` markers. Do NOT change the styling or the tool links — only fill the
   `{{PLACEHOLDER}}` markers.

2. **Score each category** using the Scoring Guidelines below, then fill the executive
   summary and health-score table. For each `{{*_PCT}}` marker, use `score / 20 * 100`
   (e.g. a 15/20 → `75`) so the progress bars render correctly. For each `{{*_BADGE}}`,
   emit `<span class="badge pass">Pass</span>`, `warn`/`Warning`, or `fail`/`Fail`. If a
   category is unavailable because all required tools failed, keep the numeric score at `0`
   and use `<span class="badge warn">Unavailable</span>`.
   Set `{{SCORE_CLASS}}` on the big total score by severity so the number is NOT misleadingly
   green when the score is poor: `low` for a total under 50 (red), `mid` for 50-74 (yellow),
   `high` for 75-100 (green).

3. **Fill each tool section** (`{{BUNDLER_AUDIT_CONTENT}}`, `{{TRIVY_CONTENT}}`,
   `{{RUBYCRITIC_CONTENT}}`, `{{SKUNK_CONTENT}}`, `{{RAILS_STATS_CONTENT}}`,
   `{{BUNDLE_STATS_CONTENT}}`, etc.) with real numbers pulled from the files in `$OUT/raw/`.
   Wrap every `<table>` in `<div class="table-wrap">…</div>` so it scrolls on mobile. Use
   `<pre>` blocks for short raw excerpts. When a check found nothing, write a positive note
   like `<p class="empty">No vulnerabilities found.</p>`. Never leave a `{{PLACEHOLDER}}` in
   the final file.

   - `{{RAILS_STATS_CONTENT}}`: the code-stats table from `rails-stats.txt` (code LOC, test
     LOC, code-to-test ratio, polymorphic associations, schema `create_table` count).
   - `{{BUNDLE_STATS_CONTENT}}`: the dependency table from the SAME `rails-stats.txt` (top gems
     by total transitive dependency count), with a note on removal/replacement candidates.
     There is no separate `bundle-stats.txt` — rails_stats emits this table itself.
   - `{{EXCEPTIONS_CONTENT}}`: if an exception-tracking gem was found (see Step 8), name it and
     mark it `<p class="empty">…</p>`. If none, use `<p class="flag">No exception tracking gem
     detected — the app cannot report production errors. This falls short of best practices;
     add Sentry, Honeybadger, Rollbar, or similar.</p>`.
   - `{{PERFORMANCE_CONTENT}}`: same pattern for performance/APM gems. If none, flag it:
     `<p class="flag">No performance monitoring detected — regressions will go unnoticed. Add
     an APM (New Relic, Scout, Skylight, AppSignal, Datadog) and/or rack-mini-profiler.</p>`.

4. **Embed screenshots as base64 data URIs** so the report is truly a single file. For each
   screenshot:
   ```bash
   echo "data:image/png;base64,$(base64 -i "$OUT/screenshots/rubycritic-overview.png")"
   ```
   Put the result in the `src="{{RUBYCRITIC_OVERVIEW_IMG}}"` attribute. Add any extra shots
   to `{{RUBYCRITIC_EXTRA_SHOTS}}` as additional `<div class="shot">…</div>` blocks (embed
   those base64 too). If a screenshot is missing, remove that `<div class="shot">` block.

5. **Executive summary** (`{{EXECUTIVE_SUMMARY}}`): exactly 4 sentences using the
   deterministic prose rules above. Keep the sentences factual and anchored to the extracted
   counts and fixed category scores.

6. **Top 3 recommended actions** (`{{REC1_*}}`–`{{REC3_*}}`): the three highest-impact
   actions to address the highest-priority issues found. Be specific — name the exact gem,
   CVE, or file, and say what to do. Order them with the deterministic ranking rules above.

7. **Appendix** (`{{TOOLS_TABLE}}`, wrapped in `<div class="table-wrap">`): a table of every
   tool run with its purpose. Link each tool name to its open source project page, e.g.
   `<a href="https://github.com/fastruby/skunk">skunk</a>`,
   `<a href="https://github.com/rubysec/bundler-audit">bundler-audit</a>`,
   `<a href="https://github.com/presidentbeef/brakeman">brakeman</a>`,
   `<a href="https://github.com/rubymem/bundler-leak">bundler-leak</a>`,
   `<a href="https://github.com/aquasecurity/trivy">trivy</a>`,
   `<a href="https://bundler.io/man/bundle-outdated.1.html">bundle outdated</a>`,
   `<a href="https://github.com/jaredbeck/libyear-bundler">libyear-bundler</a>`,
   `<a href="https://github.com/simplecov-ruby/simplecov">simplecov</a>`,
   `<a href="https://github.com/whitesmith/rubycritic">rubycritic</a>`,
   `<a href="https://rubygems.org/gems/rails_stats">rails_stats</a>`,
   `<a href="https://rubygems.org/gems/bundler-stats">bundle-stats</a>`.
   Use `{{APPENDIX_NOTES}}` for caveats, exclusions, false positives, or a note that SonarQube
   already covers some metrics.

8. **Write the filled HTML** to `$OUT/index.html`.

9. **Export a shareable PDF copy** named `<repo>-tech-debt-audit-YYYY-MM-DD.pdf` inside
   `$OUT/` using the report date from the run. Prefer the repository directory name for
   `<repo>`, lowercased exactly as checked out. Use headless Chrome if available:
   ```bash
   REPO_NAME="$(basename "$(pwd)")"
   REPORT_DATE="$(date +%F)"
   PDF_NAME="${REPO_NAME}-tech-debt-audit-${REPORT_DATE}.pdf"
   CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
   "$CHROME" --headless=new --disable-gpu \
     --print-to-pdf="$(pwd)/$OUT/$PDF_NAME" \
     "file://$(pwd)/$OUT/index.html"
   ```
   If Chrome is unavailable or PDF export fails, keep the HTML report as the authoritative
   deliverable and note that the PDF could not be generated.

10. **Report back** to the user with the path to `$OUT/index.html`, the PDF path if it was
    generated, and a one-paragraph summary of the health score and the top recommendation.

---

## Scoring Guidelines (0-100 total, 20 per category)

### Security (20)
- 20: No vulnerabilities
- 15: Low severity only
- 10: Some medium severity
- 5: High severity issues
- 0: Critical vulnerabilities

Interpretation rules for consistency:

- Score from the highest severity found across `bundler-audit`, `brakeman`, `bundler-leak`,
  `trivy`, and the matching JS audit for the root lockfile.
- Treat leaked secrets as `0`.
- Treat Brakeman `High` confidence warnings as `0`, `Medium` as `10`, and `Weak` as `15`.
- Treat any `bundle-leak` finding as at least `5`.
- If one or more security tools fail, score from the successful tools only and say which
  tools were unavailable.

### Dependencies (20)
- Score from `bundle-outdated-strict.txt` only, after applying the direct-RubyGems exclusions
  in Step 3.
- 20: <10% of eligible direct dependencies have permitted updates
- 15: <20% have permitted updates
- 10: <40% have permitted updates
- 5: <60% have permitted updates
- 0: >=60% have permitted updates
- If the strict Bundler command is unavailable after its retry, the category is unavailable.
  Use `0` in the numeric table only for template compatibility and explicitly say the score is
  unavailable due to command failure.

### Complexity (20)
- 20: No files with complexity >10
- 15: Few files with complexity 11-20
- 10: Some files with complexity 21-50
- 5: Many files with high complexity
- 0: Files with complexity >50

Interpretation rules for consistency:

- Prefer RubyCritic for the primary complexity signal.
- If RubyCritic is unavailable and Skunk succeeded, infer the closest matching band from the
  top Skunk findings and state that complexity was scored from Skunk fallback data.
- `Few files` means 1-3 files in the relevant band.
- `Some files` means 4-10 files in the relevant band.
- `Many files with high complexity` means more than 10 files above 20 complexity, or any
  repeated concentration of top-ranked files in the 31-50 range.
- If any file exceeds 50 complexity, score `0` regardless of lower-band counts.

### Coverage (20)
- 20: >90% · 15: 70-90% · 10: 50-70% · 5: 30-50% · 0: <30%

Round the reported coverage percentage to the nearest whole number before applying the band.

### Maintainability (20)
- 20: Excellent setup, CI, documentation
- 15: Good setup with minor gaps
- 10: Adequate but needs improvement
- 5: Significant maintainability issues
- 0: Major maintainability problems

Interpretation rules for consistency:

- Use only the six machine-checkable checks from Step 9.
- `Excellent setup, CI, documentation` = 6 of 6 checks present.
- `Good setup with minor gaps` = 5 of 6 checks present.
- `Adequate but needs improvement` = 3-4 of 6 checks present.
- `Significant maintainability issues` = 2 of 6 checks present.
- `Major maintainability problems` = 0-1 of 6 checks present.

## Important Notes

1. **Don't duplicate effort**: if SonarQube is in use, note which metrics it already tracks.
2. **Focus on business logic**: complexity tools target `app/` and `lib/`, not tests.
3. **Context matters**: a high SkunkScore in a rarely-changed file is less urgent than a moderate score in a frequently-modified file.
4. **Prioritize by impact**: files with high churn + high complexity + low coverage first.
5. **Track over time**: the timestamped directory is the baseline — keep it to compare future runs.
