# PRRisk: Product Requirements Document

Version: 0.1 (draft)
Owner: Biswash Rath
Status: Planning

## 1. Summary

PRRisk is a GitHub Action that scores each pull request from 0 to 100 for review risk and posts a single comment explaining the score. The comment lists each contributing signal with its points, so a reviewer can see why a PR scored 72 instead of 40 and where to look first.

PRRisk never approves, blocks, or merges pull requests. It only informs reviewers.

## 2. Problem

Reviewers have limited attention. On a busy repository, many PRs look similar in the review queue, yet some carry far more risk than others: large diffs, edits to files that break often, changes with no tests, or changes by someone unfamiliar with the code. Today, reviewers judge this by feel, and risky changes sometimes get the same light pass as a typo fix.

Existing tools either report coverage (which does not reflect review risk), or use LLM judgment (which is non-deterministic and hard to audit). Teams need a signal that is repeatable, cheap to run, and explainable.

## 3. Goals

- G1: Give every PR a repeatable risk score from 0 to 100 and a band (Low, Medium, High).
- G2: Make every point traceable to a named signal with a stated reason.
- G3: Point reviewers to the riskiest files and changes.
- G4: Let teams tune weights and thresholds through a config file without changing code.
- G5: Run inside GitHub Actions with no paid services and no external API calls.
- G6: Keep one comment per PR that updates on each push instead of spamming the thread.

## 4. Non-goals

- Approving, blocking, or merging PRs.
- Replacing human review or static analysis.
- Using an LLM to judge code quality.
- Supporting non-GitHub hosts in v0.1 (GitLab, Bitbucket, and others are future work).
- Measuring developer performance. Author familiarity is a risk signal about code, not a metric about people, and the output must not be framed as a ranking of contributors.

## 5. Users

**Primary: reviewers on a team repository.** They open PR comments and want to know where to focus. They need a score they can read in five seconds and a breakdown they can act on.

**Secondary: maintainers and tech leads.** They install and configure the Action, tune weights for their codebase, and decide thresholds.

**Tertiary: the author.** They see the comment before review and can understand why a PR is flagged, which helps them add tests or split the change.

## 6. User stories

- As a reviewer, I want a risk score on each PR so I can prioritize my review queue.
- As a reviewer, I want each point explained so I can decide whether the score is fair.
- As a maintainer, I want to change signal weights in a YAML file so I can match the model to my codebase.
- As a maintainer, I want the Action to work on a fresh install with no configuration.
- As an author, I want the comment to update when I push fixes, not add a new comment each time.
- As a contributor opening a PR from a fork, I want the workflow to fail gracefully instead of breaking my build.

## 7. Functional requirements

### 7.1 Scoring

- FR-1: The system computes a score as the sum of signal points, capped at 100.
- FR-2: The system reports a band: Low (0 to 29), Medium (30 to 59), High (60 to 100). Thresholds are configurable.
- FR-3: The system computes four signals: change size, hotspots, test presence, and author familiarity. Each is independently enabled or disabled through config.
- FR-4: Each signal produces a list of zero or more findings. Each finding has a human-readable message, a point value, and the file it refers to where applicable.
- FR-5: The same inputs always produce the same score. No randomness, no network calls, no LLM calls.

### 7.2 Signals

- FR-6 (change size): Points come from total lines changed and number of files touched, using tiers defined in config.
- FR-7 (hotspots): For each changed file, count commits within the history window whose messages match the bug-fix classifier. Points are per file, capped, and summed to a signal cap.
- FR-8 (test presence): For each changed production source file, check whether the PR also changes a corresponding test file. If a JaCoCo report is present, use line coverage on changed lines instead when configured.
- FR-9 (author familiarity): For each changed file, check whether the PR author has at least one prior commit touching it within the history window.

### 7.3 Output

- FR-10: The CLI prints the score, band, and findings to stdout in plain text.
- FR-11: The CLI can emit JSON for use by other tools.
- FR-12: The Action posts one PR comment containing the score, band, findings grouped by signal, and the top five riskiest files.
- FR-13: On subsequent pushes, the Action finds the existing PRRisk comment by a hidden marker and updates it.
- FR-14: The comment states the commit SHA it was computed for.

### 7.4 Configuration

- FR-15: The system reads `.prrisk.yml` from the repository root if present.
- FR-16: Missing keys fall back to documented defaults. Invalid values fail with a clear error message naming the key.
- FR-17: Configurable items: signal enabled flags, signal caps, tier boundaries, per-file point values, band thresholds, history window in days, bug-fix patterns, test file patterns, and ignored paths.

### 7.5 Edge cases

- FR-18: If the checkout is shallow, the Action detects it and fails with a message telling the user to set `fetch-depth: 0`. Silent degradation is not acceptable.
- FR-19: For fork PRs where the token is read-only, the Action computes the score, writes it to the job summary, and logs that the comment could not be posted.
- FR-20: Binary files and generated paths listed in config are excluded from line counts and test checks.
- FR-21: A PR with zero changed production files produces a score of 0 and a Low band with an explicit "no production code changed" finding.

## 8. Non-functional requirements

- NFR-1 (performance): On a PR touching up to 200 files in a repository with 100,000 commits, the run completes within 60 seconds on a standard GitHub-hosted runner. History walks are bounded by the touched files and the window.
- NFR-2 (determinism): Byte-identical output for identical inputs.
- NFR-3 (portability): Runs on Linux runners. No dependency on a git binary (JGit only).
- NFR-4 (testability): The core scoring library has no git, network, or filesystem dependencies, and is covered by unit tests for every rule.
- NFR-5 (transparency): Every score in the comment can be reproduced by running the CLI locally on the same commits.
- NFR-6 (cost): No paid services. Uses the built-in `GITHUB_TOKEN`.
- NFR-7 (security): Requires only `contents: read` and `pull-requests: write`. No secrets beyond the default token. Does not execute code from the PR.

## 9. Success metrics

These are for judging the project as a portfolio piece and a real tool.

- M1: Calibration: across at least three test repositories, the High band matches a reviewer's judgment of risky PRs in at least 70% of hand-labeled cases. Labeling is done in Phase 3.
- M2: Explainability: a reviewer unfamiliar with the project can explain any score from the comment alone in a usability check with three people.
- M3: Adoption signal for v0.1: the Action runs successfully on the demo repository and on at least two external repositories.
- M4: Stability: no duplicate comments created across 20 consecutive pushes to a test PR.

## 10. Scope by release

**v0.1 (target):** Four signals, config file, CLI with text and JSON output, composite Action with comment create and update, fork fallback, shallow-clone detection, demo repository, README.

**Later (not committed):** conventional-commit classifier, coverage tools beyond JaCoCo, per-file suppression, suppression comments in the PR, GitLab support.

## 11. Open questions

- Q1: Should the bug-fix classifier default to keyword matching with word boundaries, or conventional-commit prefixes? Decision needed before Phase 2. Leaning toward word-boundary keywords for v0.1, since many repositories do not use conventional commits.
- Q2: Should the Action fail the check run when the band is High? The current answer is no, since the tool must never block. Revisit only if teams ask for it, and make it opt-in.
- Q3: How should generated code be detected beyond path patterns? Path patterns only, for v0.1.

## 12. Risks

See the Risks section of the architecture document for technical risks. Product risks:

- Scores feel arbitrary if there are too many signals. Mitigation: four signals only, and always show points.
- Reviewers ignore the comment. Mitigation: keep the comment short, put the top file first, and avoid noise on low-risk PRs.
- Teams treat the score as a quality judgment. Mitigation: the README and comment wording describe it as review effort guidance, not code quality.
