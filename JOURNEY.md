# Journey: LOAN-12 on the Gemini + ADK (hand-built) path

A live run of one ticket, from issue to merged PR, with `SDLC_MODE=adk`. It ran on
2026-09-28 and 2026-09-29 (UTC). In stage-3 the fix is made by an **ADK agent we wrote ourselves** (`google-adk`). The model is Gemini, and it gets four narrow hand-written tools: `list_files`, `read_file`, `write_file` (confined to `services/`, and it must read a file before overwriting it) and `run_tests`. It has no shell. This is the "more manual" path.

Identities on GitHub:
- **Developer**, [@robertoTerralogiq](https://github.com/robertoTerralogiq): a person. Files the
  ticket, makes the judgment calls, and owns the merge.
- **Developer assistant**, [@developmentAssistant](https://github.com/developmentAssistant): the
  automated actor. It writes the first draft, posts the CI review and pushes the AI fixes. It has
  write access, no admin rights and no `workflow` scope.

The gate (branch protection on `main`): `stage-4: merge-gate` must pass, and every conversation
must be resolved.

## Stages

| # | Stage | Who | What happened | Evidence |
| --- | --- | --- | --- | --- |
| 1 | Ticket | Developer | LOAN-12 filed | [#1](../../issues/1) |
| 2 | Implement | Assistant | First draft pushed and PR opened with auto-merge | [`ca82c48`](../../pull/2/commits/ca82c48), [#2](../../pull/2) |
| 3 | Review, run 1 | CI stage-2 | **7 findings, including blockers** ⇒ gate failed | [run](../../actions/runs/36408881272) |
| ✗ | ai-fix, run 1 | CI stage-3 | The ADK agent fixed everything, but **the push was refused** by the path guard, because of a bug in the orchestrator (not the agent): stripping `git status` output turned the first path into `ervices/…`. Pytest was also missing from the fix job | [run](../../actions/runs/36408881272) |
| 0.1 | Pipeline fix | Developer | Path parsing fixed and pytest installed. Branch updated | [`d4a3c7a`](../../pull/2/commits/d4a3c7a) |
| 3 | Review, run 2 | CI stage-2 | 7 findings ⇒ gate failed | [run](../../actions/runs/36409506951) |
| 4 | **ai-fix round 1** | CI stage-3, **ADK** | **Fixed 7/7** in 140 s, 29 model calls. Tests: 5 → 15 passed. Pushed as the assistant | [`e1605bc`](../../pull/2/commits/e1605bc), [run](../../actions/runs/36409506951) |
| 5 | Review, run 3 | CI stage-2 | 3 new findings: 1 major (HTTP error handling) and 2 minors, no blockers ⇒ **gate passed**, but the open conversations still blocked the merge | [run](../../actions/runs/36409984512) |
| 6 | ai-fix round 2 | CI stage-3, **ADK** | **Changed nothing on purpose.** The agent judged the remaining major a false positive: *`HTTPError` is meant to propagate, and both the rejection path and the non-2xx path are covered by tests* (130 s, 28 model calls) | [run](../../actions/runs/36409984512) |
| 0.2–0.3 | Pipeline fix | Developer | (a) The CI review now runs as the assistant, because `GITHUB_TOKEN` is refused `resolveReviewThread` and stale threads stayed open. (b) A thread a person resolves now counts as *accepted*: the gate and ai-fix skip it | `main` history |
| 7 | Decision | Developer | Agreed with the agent and resolved that thread with a reason. It filed the 2 minors as follow-up [#3](../../issues/3). Branch updated | [#2](../../pull/2) |
| 8 | Review, run 4 | CI stage-2, as the assistant | **Resolved 8 stale round-1 threads by itself**, excluded 3 accepted finding(s), found 1 new low-severity point ⇒ **gate passed** | [run](../../actions/runs/36412111689) |
| 9 | Decision | Developer | Recorded the last point as a follow-up and resolved it | [#3](../../issues/3) |
| 10 | Merge | GitHub auto-merge | Squash-merged, branch deleted, #1 closed | [`a3b4ebf`](../../commit/a3b4ebf) |

Totals: 4 review runs, 2 ai-fix rounds (1 push, 1 deliberate no-change), 12 threads (8 resolved by the re-review, 4 by the developer), 1 follow-up issue. Tests went from 5 to 15. The merge happened the next morning (2026-09-29 01:51 UTC). The pipeline was done the evening before; the wait was for the developer's last decision.

## What this path shows

- **Most control, most code.** The engine is about 250 lines, and every tool is ours, so we
  decide exactly what the agent can touch. There is no shell at all.
- **Guardrails we had to learn the hard way** (while building it, before this run): with a raw
  dict schema the model's structured output was not constrained, and without a read-before-write
  rule it once rewrote files blind and changed the business constant `PENALTY_RATE`. The tests
  still passed, so only the guard caught it.
- **Portable.** Plain ADK with Gemini also runs on Vertex AI / Agent Engine, with no
  Antigravity runtime involved.
- **Its judgment was good here too.** It fixed all 7 findings, then declined an incorrect major,
  with the reason tied to its own tests.

## What was live, and what the stand-ins were

- **Live:** all GitHub activity, every CI run, every Gemini review (`gemini-2.5-pro`), and every
  ADK fix (`gemini-3.8-flash`). The fix code in this PR was written by the agent in CI;
  nobody typed it.
- **Stand-in:** the stage-2 first draft is the committed `demo/mr-fixture/`, with the defects
  planted on purpose.
- **Three pipeline bugs were found by running it live.** They are fixed on `main` as stages
  0.1–0.3 and described above. None of them let unreviewed code through: each one made the
  pipeline refuse or wait.
