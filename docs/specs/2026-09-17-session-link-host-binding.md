# Bind qualified session links to their host

## Traceability

- Spec ID: `2026-09-17-session-link-host-binding`
- Status: Implemented

## Intent

Keep a qualified `Harness-Session` trailer from proving an explicit link to a
same-named session collected from another coding-agent host. A trailer such as
`claude/shared-id` must not make a Codex `shared-id` session explicit merely
because both ids share the final path segment.

## Acceptance Scenarios

- **AC-1** A non-empty, slash-free bare `Harness-Session: <id>` remains an
  explicit match for a session whose `sessionId` is `<id>`. Bare values that
  contain `/` are not legacy ids: they must satisfy AC-2 instead.
- **AC-2** A qualified `Harness-Session: <platform>/<id>` is an explicit match
  only when both the session platform and session id match exactly.
- **AC-3** A qualified trailer for another platform, or a value with an
  arbitrary nested prefix, supplies no explicit evidence. With no independent
  time, file, cwd, or observed-commit evidence, correlation returns `null`;
  with such evidence it retains the existing heuristic confidence.
- **AC-4** `Entire-Checkpoint` links keep their existing explicit matching
  semantics.
- **AC-5** The real `correlate` CLI path, which collects sessions for its
  selected `--platform`, does not report an explicit match from a trailer
  qualified for a different host. A mixed-platform pure report with equal
  session ids applies the same rule.

## Non-goals

- Adding a platform-alias registry, trailer format, API, or schema field.
- Changing session discovery, platform selection, ranking, checkpoint
  collection, or existing heuristic confidence rules.
- Rewriting or normalizing existing Git trailers.

## Plan and Tasks

1. Add focused red tests in `test/sessions/commit-session-link.test.mjs` for
   mixed-platform matching and the subprocess CLI regression fixture.
2. Replace suffix matching in `scripts/commit-session-link/correlate.mjs` with
   the two existing supported representations: a slash-free bare id and one
   non-empty `platform/id` segment pair. Compare the qualified platform to the
   session's existing `platform` field without introducing alias expansion.
3. Keep `entire-checkpoint` handling and all heuristic ranking unchanged.
4. Run the focused test red then green, regenerate the Better Harness document
   link graph, run its integrity test, and perform a local-diff traceability
   readiness review.

## Test and Review Evidence

- AC-1--AC-4: `npx.cmd vitest run test/sessions/commit-session-link.test.mjs`
  passed 49 tests with 1 Windows-only skip. It covers slash-free bare,
  qualified, mismatched-host, missing/unknown host, alias, nested-prefix,
  slash-bearing id, heuristic fallback, mixed-platform report, and Entire
  checkpoint links.
- AC-5: a fixture Git repository invokes `cli.mjs correlate` with a real host
  session artifact and asserts that the other host's qualified trailer produces
  no explicit match.
- Documentation: `node scripts/doc-link-graph/cli.mjs skills/better-harness`
  regenerated the graph; `npx.cmd vitest run
  test/skills-docs/doc-link-graph.test.mjs` passed 8 tests.
- Red evidence: with the temporary historical suffix matcher, the named real
  CLI regression received `explicit` where it expects `medium`; the structured
  matcher was restored before final verification.
- Repository suite: the main worktree's `npm.cmd test -- --maxWorkers=4
  --reporter=dot` reported 1769 passed, 1 failed, and 8 skipped. The sole
  failure is the unchanged local baseline DSH assertion at
  `test/agents/agent-customize-dsh.test.mjs:397` (`expected other`, received
  `user`); it also fails in an isolated baseline worktree. Main CI's four jobs
  are green, so this is recorded as local baseline evidence rather than a
  regression from this change.
- Packaging: the main worktree pack verification passed 735 of 996 checks. The
  preview health check was not run because the Canvas SDK prerequisite is absent.
- Risk: rejecting malformed nested prefixes can reduce false-positive explicit
  provenance. The intentionally unchanged heuristic path preserves independent
  observed evidence rather than turning absence of a valid explicit link into a
  blanket exclusion.

## Decisions and Risks

- **Decision**: Treat `platform/id` as a two-segment identifier, not a generic
  pathname. The previous suffix rule accepted unrelated prefixes, which is the
  source of the cross-host collision.
- **Decision**: Do not resolve aliases here. No existing trailer alias contract
  is consumed by this capability, and expanding identity semantics would change
  its public behavior beyond this correction.
- **Risk**: Slash-free legacy bare ids remain host-agnostic and can still
  explicitly match same-named sessions. A legacy opaque id containing `/` no
  longer bypasses qualified-link structure; users must emit the supported
  `platform/id` form when host identity is encoded in the trailer.
