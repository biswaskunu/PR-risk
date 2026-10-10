# PRRisk: Delivery Phases

Version: 0.1 (draft)
Related: PRD.md, ARCHITECTURE.md, DESIGN.md

Each phase has a goal, concrete tasks, a definition of done, and the exit milestone. A phase is not finished until its exit milestone is demonstrated, not just when the code is written.

## Overview

| Phase | Goal | Key output | Rough effort |
|---|---|---|---|
| 1 | Core engine and CLI, no history | Scored text output from a local diff | 1 to 2 weeks |
| 2 | History signals and config | Full four-signal scoring, `.prrisk.yml` | 2 weeks |
| 3 | Calibration | Tuned defaults and labeled evidence | 1 to 2 weeks |
| 4 | GitHub Action and comment | Single updating PR comment | 1 to 2 weeks |
| 5 | Documentation and release | v0.1 tag, demo repo, README | 1 week |
| 6 | Iteration | v0.2 planning from real feedback | ongoing |

Effort estimates assume part-time work alongside coursework and job search. Adjust as needed; the order and exit criteria matter more than the dates.

## Phase 1: Core engine

### Goal

Score a PR from its diff alone, with change size and test presence, and print the result to the terminal.

### Tasks

1. Create the Maven project with Java 21, JUnit 5, AssertJ, and Picocli. Set up the package layout from ARCHITECTURE.md section 2.
2. Implement domain records: `ChangedFile`, `ChangeKind`, `ChangeSet`, `Finding`, `SignalResult`, `Score`, `Band`.
3. Implement `ChangeSizeSignal` with lines and files tiers from DESIGN.md section 3.1, with tiers hard-coded for now.
4. Implement `TestPresenceSignal` with the filename pairing rule from DESIGN.md section 3.3, without coverage.
5. Implement `ScoringEngine` and band mapping.
6. Implement `GitReader` with JGit: compute `ChangeSet` between two refs, with line counts and binary detection.
7. Implement the shallow repository check and exit code 3.
8. Implement `TextRenderer` for the format in DESIGN.md section 5.1, with the arithmetic check.
9. Implement the Picocli command with `--repo`, `--base`, `--head`, and exit codes from ARCHITECTURE.md section 5.
10. Write unit tests for every tier boundary and the pairing rule.
11. Write integration tests that build a fixture repository in a temp directory with JGit and assert the `ChangeSet`.

### Definition of done

- `mvn verify` passes with no warnings from the compiler.
- Unit test coverage on the `core` package is at least 90% of lines.
- Running the CLI against a local clone of any repository prints a score with per-signal breakdown.
- The per-signal totals sum to the score in every test.

### Exit milestone

You can run `prrisk --repo . --base HEAD~3 --head HEAD` on a real project and get a score whose every point you can explain.

### Risks in this phase

- JGit line counting differs from `git diff --numstat` on edge cases. Mitigation: compare against `git diff --numstat` in one integration test per fixture type.

## Phase 2: History signals and configuration

### Goal

Add hotspots and author familiarity, bound the history walk, and make the model configurable through `.prrisk.yml`.

### Tasks

1. Implement the history walk in `GitReader`: per touched production file, commits within the window, with the window anchored to head commit time.
2. Enforce `max_commits_per_path` and stop the walk when the limit is reached. Log the truncation as a note in the output.
3. Implement the bug-fix classifier from DESIGN.md section 3.2, with whole-word matching on the subject line only, and conventional-commit prefix matching.
4. Implement `HotspotSignal` with per-file points, per-file cap, and signal cap.
5. Implement `AuthorFamiliaritySignal` with email normalization, bot skipping, and exclusion of the PR's own commits.
6. Implement `ConfigLoader` with defaults embedded as a resource, SnakeYAML parsing, validation rules from DESIGN.md section 4.1, and clear error messages naming the failing key.
7. Replace hard-coded tiers and caps with config values throughout.
8. Add the `--config` option and JSON output (`--format json`).
9. Write unit tests for the classifier, including the negative cases: `prefix`, `fixture`, `fixtures`, `bugfix-free`, and a body-only mention of "fix".
10. Write integration tests for history walks, including a file with no history, a file renamed in the PR (tracking the old path is O4 in DESIGN.md; document the chosen behavior), and an author with prior commits.
11. Write config validation tests for each error case.

### Optional task (decide at start of phase)

JaCoCo parsing for test presence. Only do this if Phases 1 and 2 finish with time left. It improves accuracy but adds a file format dependency. The default is to skip it and list it under Phase 6.

### Definition of done

- All four signals run and each has its own unit tests.
- Invalid `.prrisk.yml` values produce an error message that names the key and the allowed range.
- Removing `.prrisk.yml` produces the same output as an explicit file containing only `schema: 1`.
- A run on a repository with 100,000 commits and a 200-file PR completes in under 60 seconds locally. Record the measured time.

### Exit milestone

Scores reflect both the change and its history, and changing `points_per_commit` in a YAML file changes the output without recompiling.

### Risks in this phase

- History walks are slow on large repositories. Mitigation: per-path commit cap, touched-file restriction, and a timing test that fails the build if the limit is exceeded on the fixture.
- The bug-fix classifier has false positives and negatives. Mitigation: a table of example subjects in the test suite that documents expected classification, so changes to the classifier are visible in diffs.

## Phase 3: Calibration

### Goal

Check whether the scores match judgment on real repositories, and adjust defaults where they do not, with a written record of each change.

### Tasks

1. Choose at least three repositories:
   - One of your own projects with a real history.
   - Two well-known open-source Java projects with long histories and active bug-fix commits. Pick ones with a mix of tested and untested code.
2. Select 30 to 50 merged PRs across those repositories. Include a spread: small, large, tested, untested, from frequent and infrequent contributors.
3. Before running the tool, write down your own Low, Medium, or High label for each PR based on the diff and its context. Record the labels in a CSV file, separate from tool output, so the labels are not influenced by the score.
4. Run PRRisk on each PR and record score, band, and per-signal points in the same CSV.
5. Compare labels with bands. Compute agreement overall and per signal. Note the PRs where they disagree.
6. For each disagreement, classify the cause: tier too coarse, classifier false positive, classifier false negative, pairing rule miss, or label disagreement (your judgment was wrong).
7. Adjust weights, tiers, or classifier words where the cause is in the tool. Do not change defaults to fit a single PR.
8. Re-run the labeled set after each change and record the agreement numbers.
9. Write a calibration log in `docs/CALIBRATION.md`: each change, the reason, and the agreement before and after.
10. Measure runtime on the largest repository in the set and record it.
11. Resolve open issues O1 and O3 from DESIGN.md based on the evidence.

### Definition of done

- A labeled dataset of at least 30 PRs exists in the repository under `calibration/`.
- The calibration log records every change to a default value.
- The M1 success metric from the PRD is measured and reported, whether or not it meets the 70% target.
- Open issue O1 is closed with a decision.

### Exit milestone

The defaults in DESIGN.md are either confirmed by evidence or changed with a documented reason. You can explain each weight in an interview without guessing.

### Risks in this phase

- Labels are biased by the tool's output. Mitigation: label before running the tool, as the task list requires.
- Overfitting to a small set. Mitigation: keep a holdout of 10 PRs not used for tuning, and report agreement on it separately.
- Open-source repositories have unusual commit conventions. Record this as a limitation in the calibration log.

## Phase 4: GitHub Action and comment posting

### Goal

Run PRRisk automatically on pull requests and maintain one comment that updates on each push.

### Tasks

1. Write `action.yml` as a composite action with inputs `config-path`, `history-days`, and `version`.
2. Implement the setup step that downloads the release jar for the pinned version.
3. Implement the CLI call with `--format json` and write output to a file in the runner temp directory.
4. Implement the poster entry point in the `action` package: read the JSON, read the event payload, find the existing comment by marker, and create or update.
5. Implement the comment body from DESIGN.md section 5.3, including the commit SHA and advisory footer.
6. Implement the fork check using `github.event.pull_request.head.repo.fork` and write the comment body to `$GITHUB_STEP_SUMMARY` for forks.
7. Implement shallow detection surfacing: exit code 3 fails the step with the `fetch-depth` message.
8. Implement one retry for 5xx responses. Permission errors fall back to the job summary.
9. Write poster tests against a fake HTTP server: create, update, find-by-marker when multiple comments exist, fork path, and 5xx retry.
10. Create a test repository with a workflow file that uses the action from a local path during development, then switch to a pinned release.
11. Verify the action on a test PR: open, push a fix commit, confirm the same comment is updated and not duplicated.
12. Verify the fork path by opening a PR from a fork of the test repository and confirming the job summary is written and the step succeeds.

### Definition of done

- The workflow runs on `pull_request` events for `opened`, `synchronize`, and `reopened`.
- Across 20 consecutive pushes to a test PR, exactly one PRRisk comment exists (metric M4).
- The fork path succeeds and writes the summary.
- Shallow checkout fails with the documented message.

### Exit milestone

Opening or updating a PR in the test repository produces a single, correct, updating comment, and the fork case degrades without failing.

### Risks in this phase

- Marker-based lookup can match a comment from a different bot if the marker is copied. Mitigation: match on both the marker and the comment author being the Actions bot (`github-actions[bot]`).
- Pagination of PR comments: a PR with many comments may hide the PRRisk comment on page one. Mitigation: paginate the comment list fully before deciding to create.
- The `pull_request` event for forks runs with restricted permissions. Documented, not worked around.

## Phase 5: Documentation and release

### Goal

Make the project understandable and installable by someone who has never seen it.

### Tasks

1. Create the demo repository with a sample PR designed to trigger all four signals: a large diff, a file with bug-fix history, an untested change, and a change by an author without prior commits in the file. Commit the history needed to make the bug-fix signal fire.
2. Run the action on the demo PR and capture a real comment.
3. Write the README with these sections: what it does, what it does not do, quick start, how scoring works (link to DESIGN.md), configuration reference, the `fetch-depth: 0` requirement, the fork limitation, and the advisory note.
4. Include a screenshot of the real comment from the demo PR.
5. Add `CHANGELOG.md` starting with v0.1.0.
6. Add `LICENSE` (already present, MIT, 2026, Biswash Rath).
7. Add `.prrisk.yml.example` with all defaults and comments.
8. Add a `CONTRIBUTING.md` with build and test commands. Keep it short.
9. Tag v0.1.0. Attach the jar to the GitHub release.
10. Ask two people unfamiliar with the project to install it from the README only, and note every place they got stuck. Fix the README for each.

### Definition of done

- A new user can install the Action from the README alone (metric M2 and the usability check).
- The demo PR shows a real, current comment.
- The release includes the jar and matches the changelog.

### Exit milestone

A stranger can install and understand the project from the README alone.

### Risks in this phase

- README drifts from the actual behavior. Mitigation: the example config in the README is generated from `.prrisk.yml.example` by a test, so they cannot disagree silently.

## Phase 6: Iteration

### Goal

Use the tool on real work, collect feedback, and plan v0.2 from evidence.

### Tasks

1. Run PRRisk on your own PRs for at least four weeks. Keep a short log of each PR's band and whether you agreed with it.
2. Track false positives (flagged but not risky) and missed risks (risky but scored Low or Medium).
3. Triage each issue into one of: classifier fix, weight change, new signal candidate, or won't fix.
4. Evaluate candidate extensions from the PRD section 10: conventional-commit classifier improvements, JaCoCo parsing, per-file or per-PR suppression.
5. Maintain `CHANGELOG.md` for every change.
6. Decide on v0.2 scope and write it as a short PRD addendum.

### Possible extensions, in rough priority

1. Per-file or per-PR suppression through a PR label or a comment command. Addresses the most likely source of frustration.
2. Richer bug-fix classifier that uses conventional-commit types and issue references.
3. JaCoCo parsing if not done in Phase 2.
4. Support for additional build tools in coverage parsing (Gradle, Kotlin tooling).
5. A small static HTML report generated from JSON output for a repository-wide view. Only if users ask for it.

### Definition of done for each release

- Changelog entry written.
- Calibration log updated if any default changed.
- Demo repository still produces a valid comment.

## Cross-phase checklist

- [ ] Every PR to the project passes `mvn verify`.
- [ ] Every default value change is recorded in `docs/CALIBRATION.md`.
- [ ] No signal produces a finding that names a person in a judging tone.
- [ ] Documentation is updated in the same change as the behavior it describes.
- [ ] The advisory footer appears in every comment.

## Dependencies between phases

- Phase 2 depends on Phase 1's domain model and `GitReader`.
- Phase 3 depends on Phase 2 being complete, since calibration needs all four signals.
- Phase 4 can begin the poster work during Phase 3, since the poster depends only on the JSON contract, which is fixed by Phase 2.
- Phase 5 depends on Phase 4 for a real comment to screenshot.
- Phase 6 depends on Phase 5 being released.
