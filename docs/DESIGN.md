# PRRisk: Design

Version: 0.1 (draft)
Related: PRD.md, ARCHITECTURE.md, PHASES.md

This document specifies the scoring rules, default weights, the config schema, and the comment format. Numbers here are starting defaults for calibration in Phase 3, not final values.

## 1. Design principles

1. **Every point has a reason.** A point that cannot be attributed to a finding is a bug.
2. **Few signals.** Four signals. Adding a fifth requires removing or justifying one.
3. **Advisory, not authoritative.** The output helps a reviewer decide where to look. It never decides for them.
4. **Reproducible.** Same commits, same config, same score, every time.
5. **Boring defaults.** Tier tables and integer points, not curves or floats. Easy to read in a YAML file and easy to argue about.

## 2. Scoring model

Total score = min(100, sum of signal points).

| Signal | Cap | Input |
|---|---|---|
| Change size | 25 | diff stats |
| Hotspots | 30 | bug-fix history |
| Test presence | 20 | changed paths, optional coverage |
| Author familiarity | 15 | prior authorship of touched files |

Caps sum to 90, so no combination of signals reaches 100 alone unless one signal is at its cap plus others. The total cap of 100 is a safety clamp; in practice most high scores come from several signals together. This is intentional: a single large-but-tested change should not score High by itself.

Band thresholds (configurable):

| Band | Range |
|---|---|
| Low | 0 to 29 |
| Medium | 30 to 59 |
| High | 60 to 100 |

## 3. Signal specifications

### 3.1 Change size (cap 25)

Two components, summed and capped at 25.

**Lines component (cap 15).** Based on total added plus deleted lines in production files. Deleted lines count at half weight because removals are generally easier to review than additions.

Effective lines = added + floor(deleted / 2), excluding ignored and binary paths.

| Effective lines | Points |
|---|---|
| under 100 | 0 |
| 100 to 299 | 4 |
| 300 to 599 | 8 |
| 600 to 999 | 12 |
| 1000 and above | 15 |

**Files component (cap 10).** Based on the count of production files changed.

| Files changed | Points |
|---|---|
| 1 to 5 | 0 |
| 6 to 10 | 3 |
| 11 to 20 | 6 |
| 21 and above | 10 |

Findings: one finding for the lines component when points are above 0, with the effective line count in the message. One finding for the files component when points are above 0, listing the count.

Example message: `Large change: 640 effective lines across 14 files (+12 lines, +6 files)`.

Note on the overview example: the earlier overview showed +18 for 640 lines and 14 files. Under these tiers the same PR scores +18 (12 lines + 6 files), which matches. The example is consistent with this design.

### 3.2 Hotspots (cap 30)

For each production file in the change set, count commits in the history window whose message matches the bug-fix classifier.

**Bug-fix classifier (v0.1).** A commit matches if its subject line contains one of these words as a whole word, case-insensitive: `fix`, `fixes`, `fixed`, `bug`, `hotfix`, `revert`, `regression`. Also matches conventional-commit prefixes `fix:` and `fix(...):`. Does not match `prefix`, `fixture`, `fixtures`, or `bugfix-free` style substrings.

Matching is done on the first line of the message only. Body text is ignored to avoid matching phrases like "this does not fix anything".

**Per-file points.** Each matching commit adds 2 points to that file, capped at 12 per file.

| Bug-fix commits in window | Points per file |
|---|---|
| 0 | 0 |
| 1 | 2 |
| 2 to 3 | 4 to 6 |
| 4 to 5 | 8 to 10 |
| 6 and above | 12 |

Signal total is the sum across files, capped at 30.

Findings: one finding per file with points above 0, ordered by points descending. Message format: `Hotspot: PaymentService.java has 9 bug-fix commits in the last 90 days (+12)`. Only the top three files get individual findings; the rest are summarized in one finding: `Hotspots: 4 more files with bug-fix history (+6)`.

Window default: 90 days from the head commit time.

### 3.3 Test presence (cap 20)

For each changed production source file, determine whether the PR also changes a test file.

**Pairing rules (default).**

- Production path matches `src/main/**/*.{java,kt}`.
- Test path matches `src/test/**/<Name>Test.{java,kt}` or `src/test/**/<Name>Tests.{java,kt}` where `<Name>` equals the production class name.
- If the repository has no `src/test` tree, pairing is disabled for that file and the file is treated as untested.

Points: 4 per production file with no paired test change, capped at 20.

If a JaCoCo report is configured, the rule changes. For each production file, compute the line coverage ratio on changed lines only. A file counts as untested if coverage on changed lines is below the configured threshold (default 50%). Points are the same, 4 per untested file.

Files that are added in the PR and have no paired test count as untested. Files that are deleted do not count.

Findings: one finding listing the untested files, up to five by name, with the total count. Example: `Untested: 3 changed source files have no test changes: OrderService.java, RefundService.java, LedgerWriter.java (+12)`.

Note: the overview example shows `Untested: 3 changed source files have no test changes (+20)`. Under these rules, 3 files give +12, not +20. The overview example is illustrative. Either the per-file value is 7, or the example changes. This is an open calibration point. Decision: keep 4 per file for v0.1 and update the overview example to +12 in the next revision, or raise the per-file value to 6 to match the overview. Tracked as O1 below.

### 3.4 Author familiarity (cap 15)

For each production file changed by the PR, check whether the PR author has at least one commit touching that file within the history window, excluding the PR's own commits.

Identity matching uses author email, normalized to lowercase. Bot authors (email ending in `[bot]@users.noreply.github.com`) are treated as familiar with all files to avoid noisy findings on dependency updates.

Points: 3 per production file with no prior author commits, capped at 15.

Findings: one finding listing up to three files with the author's first contact, then a summary count. Message: `Low author familiarity: no prior commits by this author in RefundController.java, LedgerWriter.java (+6)`.

This signal is about code familiarity, not about the person. The finding text must always refer to the code and the author's prior commits, never to skill or experience.

## 4. Config schema

File: `.prrisk.yml` at repository root. All keys optional.

```yaml
# .prrisk.yml
schema: 1

history:
  window_days: 90            # history window from head commit time
  max_commits_per_path: 500  # bound on history walk per file

ignore:
  paths:                     # glob patterns excluded from all signals
    - "**/generated/**"
    - "**/*.lock"
    - "**/*.min.js"
  binary: true               # always excluded

signals:
  change_size:
    enabled: true
    cap: 25
    lines_tiers:             # [threshold, points], ascending
      - [100, 4]
      - [300, 8]
      - [600, 12]
      - [1000, 15]
    files_tiers:
      - [6, 3]
      - [11, 6]
      - [21, 10]
  hotspots:
    enabled: true
    cap: 30
    points_per_commit: 2
    per_file_cap: 12
    bug_fix_words: [fix, fixes, fixed, bug, hotfix, revert, regression]
  test_presence:
    enabled: true
    cap: 20
    points_per_untested_file: 4
    test_patterns:           # how to find the test for a production class
      - "src/test/**/{name}Test.java"
      - "src/test/**/{name}Tests.java"
    coverage:
      jacoco_report: null    # path, optional
      min_line_ratio: 0.5
  author_familiarity:
    enabled: true
    cap: 15
    points_per_file: 3
    skip_bots: true

bands:
  medium_at: 30
  high_at: 60
```

### 4.1 Validation rules

- `schema` must be 1. Unknown schema values fail with a message listing supported versions.
- Caps must be between 0 and 100. Points must be non-negative integers.
- Tier lists must be sorted ascending by threshold.
- `bands.medium_at` must be less than `bands.high_at`, and both between 1 and 100.
- Unknown top-level keys produce a warning, not an error, so future versions can add keys without breaking older runs.
- Glob patterns are validated at load time.

### 4.2 Defaults

The default config is the file above with all values as shown. A repository with no `.prrisk.yml` gets exactly these defaults. The defaults are embedded in the jar.

## 5. Output format

### 5.1 Text (CLI)

```
PRRisk 0.1.0
Base: 3f2a91c  Head: 8bd04e7
Score: 72/100  Band: High

Change size (18/25)
  Large change: 640 effective lines across 14 files (+12 lines, +6 files)

Hotspots (22/30)
  Hotspot: PaymentService.java has 9 bug-fix commits in the last 90 days (+12)
  Hotspots: 2 more files with bug-fix history (+4)

Test presence (12/20)
  Untested: 3 changed source files have no test changes: OrderService.java, RefundService.java, LedgerWriter.java (+12)

Author familiarity (6/15)
  Low author familiarity: no prior commits by this author in RefundController.java, LedgerWriter.java (+6)

Top files: PaymentService.java, RefundController.java, OrderService.java
```

Per-signal totals are shown in parentheses after each signal name, so the arithmetic is visible. The sum of the signal totals equals the score, which the test suite checks.

Note: the example above shows Hotspots as 22 but the findings sum to 16 (12 + 4). Corrected example values: Hotspots (16/30). Score recalculated: 18 + 16 + 12 + 6 = 52, Band Medium. The overview example (72, High) is a different illustrative PR. Keep the overview example as it was and use this block only as a format sample; the numbers here must be internally consistent, which they now are at 52.

### 5.2 JSON

```json
{
  "tool": {"name": "prrisk", "version": "0.1.0"},
  "base": "3f2a91c...",
  "head": "8bd04e7...",
  "score": 52,
  "band": "MEDIUM",
  "signals": [
    {
      "id": "change_size",
      "points": 18,
      "cap": 25,
      "findings": [
        {"message": "...", "points": 12, "path": null}
      ]
    }
  ],
  "top_files": ["PaymentService.java", "RefundController.java"]
}
```

Field names are stable across patch versions. Breaking changes require a schema bump in JSON output as well.

### 5.3 PR comment

```
<!-- prrisk:v1 -->
### PRRisk: 52/100 (Medium)

Scored at commit `8bd04e7`.

**Top files to review:** `PaymentService.java`, `RefundController.java`, `OrderService.java`

<details><summary>Why this score</summary>

- **Change size (18/25):** 640 effective lines across 14 files (+12, +6)
- **Hotspots (16/30):** PaymentService.java has 9 bug-fix commits in the last 90 days (+12); 2 more files (+4)
- **Test presence (12/20):** 3 changed source files have no test changes (+12)
- **Author familiarity (6/15):** no prior commits by this author in RefundController.java, LedgerWriter.java (+6)

</details>

<sub>PRRisk is advisory and does not block merges. Review effort guidance, not a code quality judgment.</sub>
```

Design choices:

- The summary line and top files are visible. The breakdown is collapsed to keep the thread short but available in one click.
- The footer states the advisory nature every time.
- The commit SHA is shown so stale comments are obvious after new pushes.

## 6. Message wording rules

- Findings describe code and history, never people. Write "no prior commits by this author in X", not "the author is unfamiliar with X".
- Use plain numbers and file names. No adjectives like "dangerous" or "fragile".
- Each finding ends with its points in parentheses.
- Keep each finding under 160 characters where possible so the text output stays readable.

## 7. Open design issues

- O1: Test presence per-file points. Spec says 4 per file, which gives +12 for 3 files. The earlier overview example shows +20 for the same count. Decide during Phase 3 calibration. Options: keep 4 and update the overview, or set 6 and cap at 20 (which reaches the cap at 4 files).
- O2: Should bot-authored PRs (for example, Dependabot) skip scoring entirely? Current answer: score normally, but hotspot and size signals usually dominate. Revisit after the demo run.
- O3: Should the hotspot classifier count reverts of bug-fix commits twice? Current answer: no, each commit counts once.
- O4: Renamed files: count history under the old path as well. Requires following renames in JGit, which adds work. Defer to Phase 2 if it is cheap, otherwise note as a known limitation.
