---
name: factory
description: Run the software factory loop (spec -> build -> 4 parallel reviews -> approve) for a feature in this repo. Use for "/factory <feature>", "/factory next", "/factory approve <job-id>", "/factory rework <job-id> <note>".
---

# /factory orchestrator

Whichever Bob session runs this skill is the **orchestrator**. It runs the loop, spawns every agent, and is the **only writer of `factory/board.json`**.

## Loop

```
feature -> spec-writer -> builder -> [security | ux | ui | code] reviews
   any CHANGES (round <= 2) -> resume builder with the round folder
   all PASS -> approver -> APPROVE (merge to main) | ESCALATE (needs-human)
```

## Rules

- Agents get context from **files**, never from the conversation. Point each agent at paths.
- Each agent writes **only its own file**.
- Reviews start with `VERDICT: PASS` or `VERDICT: CHANGES`. `decision.md` starts with `APPROVE` or `ESCALATE`.
- The builder works and commits only on branch `factory/<job-id>` and tests against the spec's acceptance criteria.
- **WAIT FOR ALL**: a round is done only when all four review files exist in the round folder. Never resume the builder mid-round.
- Spawn the builder with a name `builder-<job-id>`. For each new round, resume it (SendMessage if available; otherwise spawn a fresh builder pointed at `spec.md`, `build.md`, and the latest round folder).
- Max 2 rework rounds. After that, the approver must ESCALATE.

## Layout

```
factory/
  board.json            # durable state, drives dashboard.html
  dashboard.html        # visual board (serve with `mise run factory:dashboard`)
  backlog.md            # queue for /factory next ("- [ ] item")
  jobs/NNN-slug/
    spec.md  build.md  decision.md
    round-1/review-security.md review-ux.md review-ui.md review-code.md
```

## Commands

### `/factory <feature>`
1. Allocate next job id `NNN-slug` (zero-padded, from `factory/jobs/`). Create the folder. Add job to `board.json` with `stage: "spec"`.
2. Spawn `spec-writer` -> `jobs/<id>/spec.md`. Set stage `build`.
3. Create branch `factory/<id>`. Spawn `builder-<id>` -> implements, tests, commits, writes `build.md`. Set stage `review`, `round: 1`, all reviews `pending`.
4. Spawn `reviewer-security`, `reviewer-ux`, `reviewer-ui`, `reviewer-code` **in parallel**, each writing `round-N/review-<kind>.md`. Update `board.json` as each verdict lands.
5. When all four exist: if any `CHANGES` and round <= 2, increment round, resume builder with feedback pointer, go to 4. Otherwise (all PASS, or rounds exhausted) spawn `approver` -> `decision.md`.
6. `APPROVE`: merge `factory/<id>` into `main`, stage `done`, tick backlog item. `ESCALATE`: stage `needs-human`.

### `/factory next`
Take the first unchecked `- [ ]` item in `factory/backlog.md` and run it as above.

### `/factory approve <job-id>`
Merge a `needs-human` job into `main`, set stage `done`.

### `/factory rework <job-id> <note>`
Write `<note>` to `jobs/<id>/human-note.md`, bump round, resume the builder pointing at it, continue the loop.

## board.json shape

```json
{ "jobs": [ { "id": "001-guide-shell", "branch": "factory/001-guide-shell",
  "stage": "review", "round": 1,
  "reviews": { "security": "pending", "ux": "pending", "ui": "pending", "code": "pending" } } ] }
```

Stages: `spec`, `build`, `review`, `approve`, `needs-human`, `done`.

## Project context to give agents

OpenShift SNO learning lab. See `OVERVIEW.md`. Guide = static HTML with step cards + diagram; a verifier backend runs read-only `oc`/API checks against the SNO cluster when the learner clicks "I did it". Task runner is `mise`.
