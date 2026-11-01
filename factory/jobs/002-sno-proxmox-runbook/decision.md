APPROVE (with conditions)

Job 002-sno-proxmox-runbook, branch `factory/002-sno-proxmox-runbook`.

## Rationale

- Round 2 is all PASS (code, security, ui, ux). Round-1 CHANGES from code and ux
  (F1–F4 and F1–F10) were each verified as resolved in round 2.
- All 12 acceptance criteria were re-run independently by the code reviewer and
  pass. `markdownlint-cli2` exits 0.
- No ESCALATE trigger applies:
  - Rounds were not exhausted. The work converged in round 2.
  - Reviewers do not disagree on the verdict. All four PASS.
  - The change is docs only. It adds no credentials, RBAC, or cluster
    configuration. The security reviewer passed it twice, and the runbook keeps
    kubeadmin and pull-secret material out of the repo and covered by
    `.gitignore`.

## The flagged issue: §5.0 vs §5.2 ordering

I confirmed this is real (`docs/sno-proxmox.md` lines 238–252 vs 266–301).
§5.0 says "check operator status before downloading credentials" and then runs
`oc get clusteroperators`. `oc` needs `KUBECONFIG`, which is first downloaded
and exported in §5.2. A learner reading top to bottom will hit an
authentication or "connection refused" error at §5.0.

Why this is acceptable to APPROVE rather than ESCALATE:

- It is a sequencing defect in one subsection. It does not make the runbook
  wrong, unsafe, or unusable. Every command and value is correct, and the
  learner recovers by continuing to §5.2 and then returning to §5.0.
- It is self-correcting. §5.2 step 4 runs `oc get clusterversion`, and the §8
  troubleshooting row for operators also gives `oc get clusteroperators`.
- The fix is mechanical and low risk, and needs no human judgment. A human
  escalation would add latency without adding information.
- It is not a regression against the spec. No acceptance criterion covers
  section ordering. The ux reviewer flagged it as non-blocking. The ui
  reviewer independently reached the same conclusion and suggested the fix.

It is not a clean approval, because a learner-facing runbook should not send
the reader into a known error. I therefore attach conditions below.

## Conditions — what the orchestrator should do

1. **Do not run another full review round.** Do not re-spawn all four reviewers
   for this. The defect is minor and the remedy is a few lines.
2. **Record a follow-up item** in `factory/backlog.md`, for example
   "sno-proxmox runbook: fix §5.0 ordering (operator check runs before
   KUBECONFIG is available)". Reference this decision file.
3. **Preferred handling:** make the fix as a small follow-up builder pass,
   either before merge if the merge has not yet happened or right after it.
   Either of these is acceptable:
   - **Reorder (preferred):** move the operator-readiness check to after
     §5.2. The order becomes 5.1 retrieve kubeadmin credentials, 5.2 download
     and export the credential file, 5.3 wait for cluster operators, 5.4
     persist `KUBECONFIG`. Drop the "before downloading credentials" wording.
   - **Minimal reword:** keep §5.0 where it is but change its text to "after
     completing §5.2" and note that you can also watch operator status in the
     Assisted Installer UI before downloading anything.
4. **Constraints on that follow-up edit:**
   - Change only `docs/sno-proxmox.md`. Do not alter the security guidance in
     §5.1 or the gitignored `auth/kubeconfig` and `kubeconfig-<cluster-name>`
     path conventions.
   - Re-run acceptance criteria 9, 10, 11 and 12. These are the secret-like
     assignment check, the `kubeconfig` placeholder/`auth/`/`KUBECONFIG` check,
     the `markdownlint-cli2` run and the port-pattern check. Any new
     `kubeconfig` line must contain `<`, `auth/` or `KUBECONFIG`.
   - Update any cross-references to the renumbered subsections if the reorder
     option is used. I found no explicit references to "§5.0" in the prose
     I read, but grep to confirm.
   - A single reviewer pass (ux or code) is sufficient to confirm the fix.
5. **If the follow-up cannot be done promptly,** merge the branch as is. The
   defect does not block the merge. Keep the backlog item open so it is not
   lost.
6. **Escalate to a human only if** the follow-up edit unexpectedly touches
   credential-handling guidance, or if the post-fix re-check regresses any
   acceptance criterion.

## Other known non-blocking items (carried from build.md)

- The SNO docs link points to OCP 4.13 and is labelled as such. Update it once
  4.22 docs are published.
- `mise` tasks are documented but not implemented. This is a separate backlog
  item.
- AC4's regex is fragile if the sizing table is reformatted.

## Orchestrator note

The section 5.0 ordering issue was fixed before merge: the operator check now follows the KUBECONFIG download (new 5.3) and the later subsection is renumbered 5.4. Acceptance criteria 1, 2, 9, 10, 12 and markdownlint re-run clean. No cross-references to the old numbers existed.
