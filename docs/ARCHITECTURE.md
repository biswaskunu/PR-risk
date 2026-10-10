# PRRisk: Architecture

Version: 0.1 (draft)
Related: PRD.md, DESIGN.md, PHASES.md

## 1. Overview

PRRisk is organized in three layers with strict dependency direction:

```
 +---------------------------+
 |  Action wrapper (YAML)    |   GitHub Actions composite action
 +-------------+-------------+
               |  invokes CLI, passes output, posts comment
 +-------------v-------------+
 |  CLI (Picocli)            |   argument parsing, output formatting
 +-------------+-------------+
               |  builds inputs, calls engine
 +-------------v-------------+        +--------------------------+
 |  Git adapter (JGit)       |------->|  Core engine (pure Java) |
 |  diff, history, authors   |  data  |  signals, scoring, config|
 +---------------------------+        +--------------------------+
```

Dependencies point downward only. The core engine depends on nothing in this repository except itself and on a small set of libraries for YAML parsing. It never imports JGit, Picocli, or the GitHub API.

## 2. Module layout

Maven multi-module project, or a single module with packages. A single module is preferred for v0.1 to reduce overhead, with package boundaries enforced by convention and an ArchUnit test.

```
prrisk/
  pom.xml
  src/main/java/dev/prrisk/
    core/                  pure logic, no I/O
      model/               ChangedFile, ChangeSet, Commit, Finding, Score, Band
      signal/              Signal interface and four implementations
      scoring/             ScoringEngine, weight application, cap, band mapping
      config/              PrRiskConfig, ConfigLoader (takes InputStream or Map)
    git/                   JGit adapter, produces core model objects
      GitReader            diff between refs, file history, author lookup
      HistoryWindow        time-window arithmetic
    cli/                   Picocli command, renderers
      PrRiskCommand, TextRenderer, JsonRenderer
    action/                GitHub API client for comment create and update
      CommentPoster, CommentMarker
  src/test/java/dev/prrisk/
    core/                  unit tests, hand-built inputs
    git/                   integration tests against fixture repositories
    cli/                   end-to-end tests with golden output files
  action.yml               composite action definition
  .prrisk.yml.example      documented default config
  demo/                    demo repository fixture
```

## 3. Core engine

### 3.1 Domain model

All model types are immutable records in Java 21.

- `ChangedFile(String path, ChangeKind kind, int addedLines, int deletedLines, boolean binary)`. `ChangeKind` is ADDED, MODIFIED, DELETED, RENAMED.
- `ChangeSet(String baseSha, String headSha, String authorEmail, List<ChangedFile> files)`.
- `Commit(String sha, Instant when, String authorEmail, String message, Set<String> pathsTouched)`. Used for history.
- `History(Map<String, List<Commit>> commitsByPath)`. Provided to signals already filtered to the window and to touched paths.
- `Finding(String signalId, String message, int points, String path)`. `path` may be null for PR-level findings.
- `SignalResult(String signalId, int points, int cap, List<Finding> findings)`.
- `Score(int total, Band band, List<SignalResult> signals)`.
- `Band` enum: LOW, MEDIUM, HIGH.

### 3.2 Signal interface

```java
public interface Signal {
    String id();
    SignalResult evaluate(ChangeSet changes, History history, SignalConfig config);
}
```

Signals are stateless. Each receives everything it needs as arguments. This keeps them trivially testable.

Four implementations:

- `ChangeSizeSignal` uses only `ChangeSet`. No history.
- `HotspotSignal` uses `ChangeSet` and `History`.
- `TestPresenceSignal` uses `ChangeSet`, plus optional coverage data supplied through `ChangeSet` as a map from path to line coverage ratio.
- `AuthorFamiliaritySignal` uses `ChangeSet` and `History`.

### 3.3 Scoring engine

```java
public final class ScoringEngine {
    public Score score(ChangeSet changes, History history, PrRiskConfig config, List<Signal> signals);
}
```

Algorithm:

1. Run each enabled signal. Each returns points already clamped to its own cap.
2. Sum points across signals.
3. Clamp total to 0 through 100.
4. Map total to band using config thresholds.
5. Return `Score` with the full list of signal results, so the output can show the breakdown.

The engine does not reorder or round points after signals return them. All rounding is done inside signals with integer arithmetic.

### 3.4 Config

`PrRiskConfig` is a typed object built by `ConfigLoader` from a YAML map. The loader applies defaults for missing keys and throws `ConfigException` with the key path for invalid values. The default config is embedded as a resource so it works without a file in the repository.

## 4. Git adapter

The adapter converts repository state into core model objects. It is the only layer that touches the filesystem and the git object database.

### 4.1 Responsibilities

- Open a repository from a path. Detect shallow clones and report them.
- Compute the diff between `baseSha` and `headSha`, producing `ChangedFile` entries with line counts.
- For each production path touched, walk commit history within the window, restricted to commits that touch that path, and produce `Commit` entries.
- For author familiarity, the same walk supplies the set of authors per path.

### 4.2 Implementation notes

- Use JGit `DiffFormatter` with `RawTextComparator` for line counts. Binary detection uses the JGit `RawText.isBinary` check.
- History walks use `Git.log().addPath(path).setRevFilter(CommitTimeRevFilter.after(...))`. One walk per touched file, with the window computed from the head commit time, not the wall clock, so results are reproducible for old PRs.
- Walks are bounded: a configurable maximum number of commits per path (default 500) prevents runaway work on very old files.
- Merge commits are excluded from bug-fix counts and author counts by default.

### 4.3 Shallow clone detection

JGit exposes `Repository.getRepositoryState()` and the shallow file at `.git/shallow`. If that file exists, the adapter throws `ShallowRepositoryException`. The CLI maps this to exit code 3 with a message pointing to `fetch-depth: 0`. The Action surfaces the same message as a failed step.

## 5. CLI

Built with Picocli. A single command with options:

```
prrisk --repo <path> --base <ref> --head <ref> [--config <file>] [--format text|json]
```

Exit codes:

- 0: scored successfully, regardless of band.
- 2: invalid arguments or config.
- 3: shallow repository detected.
- 4: git error (missing ref, corrupt repository).
- 1: unexpected error.

The exit code never reflects the risk band. The tool does not gate pipelines.

The CLI wires dependencies manually: build the git reader, load config, select enabled signals, call the engine, render output. No dependency injection framework.

## 6. Action wrapper

### 6.1 Composite action

`action.yml` defines inputs (`config-path`, `history-days`, `fail-on-shallow`), runs a setup step that downloads the CLI jar from the release, runs the CLI with JSON output, and runs a poster step.

Poster logic lives in a small Java entry point in the `action` package, not in shell, so the comment logic is testable.

### 6.2 Event data

The PR base and head SHAs come from `github.event.pull_request.base.sha` and `github.event.pull_request.head.sha`. The PR number comes from `github.event.pull_request.number`. The poster reads these from environment variables set by the workflow.

### 6.3 Comment posting

- The comment body begins with a hidden HTML marker: `<!-- prrisk:v1 -->`.
- The poster lists PR comments and finds the first one containing the marker from the authenticated bot. If found, it issues a PATCH. If not, it issues a POST.
- The body includes the head SHA so readers can tell which push the score belongs to.

### 6.4 Fork pull requests

For PRs from forks, `GITHUB_TOKEN` is read-only. The poster catches the 403 response, writes the full comment body to `$GITHUB_STEP_SUMMARY`, and logs a notice. The step succeeds.

Detection: the poster checks `github.event.pull_request.head.repo.fork` and skips the comment call up front when true, instead of relying on the error path.

## 7. Data flow for one run

1. Workflow checks out the repository with `fetch-depth: 0`.
2. Action runs the CLI with `--base` and `--head` from the event payload.
3. CLI opens the repository and detects shallow state. If shallow, exits with code 3.
4. Git adapter computes the `ChangeSet` from the diff.
5. Git adapter walks history for each production path in the changeset within the window.
6. Core engine runs enabled signals and produces a `Score`.
7. CLI renders JSON to stdout and to a file.
8. Poster reads the JSON, builds the comment, finds or creates the comment, and posts or updates it.

## 8. Error handling

- Config errors fail fast before any git work.
- Git errors include the ref name in the message.
- The poster never fails the workflow on a comment-API error except for unexpected 5xx responses after one retry. Permission errors fall back to the job summary.
- Unexpected exceptions print a stack trace only when `--debug` is passed. Default output is a single line.

## 9. Testing strategy

- Core: unit tests for each signal with hand-built `ChangeSet` and `History` objects. Table-driven tests for tier boundaries and cap behavior. Property test that the total never exceeds 100 and is never negative.
- Git adapter: integration tests create small fixture repositories in a temp directory using JGit, commit known changes, and assert the produced model. Cover renames, binary files, merge commits, shallow detection, and empty diffs.
- CLI: end-to-end tests run against a fixture repository and compare text and JSON output to golden files.
- Action poster: tests use a fake HTTP server to verify create, update, and fork fallback paths.
- Determinism: a test runs the engine twice on the same inputs and asserts equal output.

## 10. Build and distribution

- Java 21, Maven, produces a single fat jar with the `prrisk` main class.
- Release workflow builds the jar and attaches it to a GitHub release. The composite action downloads the jar by version tag.
- The action pins a version by default. Users can override with the `version` input.
- No Docker image in v0.1. A Docker-based action is a later option if startup time becomes a problem.

## 11. Dependencies

| Library | Purpose | Layer |
|---|---|---|
| JGit | git access | git adapter |
| Picocli | CLI parsing | CLI |
| SnakeYAML | config parsing | core config |
| Jackson | JSON output and GitHub API payloads | CLI, action |
| JUnit 5 | tests | all |
| AssertJ | readable assertions | all tests |
| ArchUnit | enforce layer rules | tests |

The core package depends only on SnakeYAML among these. Everything else is kept out of `core`.

## 12. Technical risks

- **Shallow clones.** Handled by detection and failure, not silent degradation. Documented in the README.
- **Slow history walks on large repositories.** Mitigated by window limits, per-path commit caps, and touched-file restriction. Measured in Phase 3 against a large open-source repository.
- **Clock-dependent results.** Window is computed from head commit time, so re-running on the same commits gives the same answer.
- **Token permissions changing.** The poster tests both write and read-only paths, and the fork check is explicit rather than error-driven.
- **Config drift across teams.** Defaults are embedded and versioned with the jar. The `.prrisk.yml` file should include a schema version key so future changes can migrate cleanly.

## 13. Decisions

- D1: Use JGit, not a git subprocess. Portable, testable, no dependency on the runner's git version.
- D2: Keep the core pure. Costs some plumbing, buys fast and deterministic tests.
- D3: Single Maven module for v0.1. Revisit if the action or CLI grows independently.
- D4: Composite action, not Docker. Faster startup, simpler to read.
- D5: Never gate the pipeline on the band. Keeps the tool advisory.
