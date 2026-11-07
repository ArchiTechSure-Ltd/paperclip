# Self-hosted lifecycle reliability rollout

Date: 2026-11-05

## Outcome

Qualify and roll out a bounded Paperclip candidate that restores native dispatch,
continuation, recovery, message delivery, and authenticated human review for a
self-hosted authenticated deployment. This plan does not authorize a production
deployment or a merge.

## Candidate

- Base: `canary/v2026.1105.0-canary.18` (`377e98f1a2ccbe041d5b893e435dc8dbd71df2ef`).
- Add upstream fix #19307 so a bootstrap retry that lost its task lock settles,
  exposes reconciliation, and releases its workspace lease.
- Add upstream fix #19318 so a queued answer cannot deadlock recovery of saved
  work after an interrupted copy-back.
- Retain the existing authenticated human review decision UI and its end-to-end
  coverage.
- Build and deploy by immutable image digest. Do not deploy a mutable tag.

The base already contains the relevant upstream fixes for recovered-owner
follow-ups (#19071), successor issue locks (#19156), messages saved during native
finalization (#19183), durable task-reply publication (#19200), preserved results
after export failure (#19034), cancelled workspace retries (#19276), and
pre-turn interrupt recovery (#19292).

Upstream PR #18267 remains open. Its additional clear-in-place path for sibling
settled no-replay holds on an already terminal task is not part of this candidate.
Use the native recovery details action only when the recorded source status and
evidence match. Escalate a terminal-status mismatch instead of repeatedly
reconciling or changing task state.

## Required configuration

The scheduler must be enabled in every long-running application container:

```yaml
environment:
  HEARTBEAT_SCHEDULER_ENABLED: "true"
```

This activates the native periodic backstop for durable continuation,
stranded-assignment dispatch, queued-message cleanup, retry admission, workspace
finalization, and recovery reconciliation. Keep the default 30-second interval
unless a separately measured load test justifies changing it.

Any deployment generator or control script that renders the final compose file
must emit the same value. Verify the rendered compose configuration before
activation; changing only a source template is insufficient.

## Pre-deployment gate

1. Record the current immutable application image digest and build commit.
2. Take a normal application backup and verify the backup command succeeds.
3. Build the candidate once and record its immutable digest and source commit.
4. Run the focused recovery, queue-delivery, and authenticated-review tests.
5. Start an isolated candidate stack with production-shaped configuration and
   verify `/api/health`, login, agent visibility, and scheduler startup logs.
6. Obtain explicit owner approval for the production write.

## Acceptance journeys

### Uninterrupted delivery

Use one small, reversible documentation-only repository change. Exercise the
native chain from Coordinator assignment through Technical Lead delegation,
Hermes Engineer draft pull request, independent review, and an authenticated
owner decision. Do not use Wake, manual reassignment, message relay, process
restart, or direct database repair. Preserve task history, run events, tool audit,
commit, draft pull request, review decision, and owner decision as evidence.

### Controlled interruption

Use a separate reversible documentation-only change. Interrupt one active
provider execution through the supported runtime termination mechanism after the
run is recorded but before completion. Do not terminate the Paperclip database.
Verify that within the bounded retry and reconciliation policy:

- the original run becomes terminal or exposes one actionable recovery decision;
- no workspace lease or execution hold remains indefinitely;
- any accepted queued instruction is delivered exactly once or visibly fails;
- provider evidence prevents replay of an uncertain external mutation;
- the task continues through Technical Lead review after the supported recovery
  action; and
- no owner Wake, reassignment, restart, or database repair is required.

### Repeatability

After both journeys, assign one additional no-op documentation review through
the normal Coordinator-to-Technical-Lead handoff. It must start and reach an
actionable review state without manual intervention. A single successful run is
not sufficient evidence of operational reliability.

## Evidence to retain

- candidate source commit and image digest;
- rendered scheduler configuration and scheduler startup event;
- task identifiers and assignment histories;
- run events, tool audit, queue receipts, provider termination evidence, and
  recovery decision;
- workspace or artifact work products for durable outputs;
- Git commit, draft pull request, independent review, and owner decision; and
- elapsed times for dispatch, handoff, recovery, and review publication.

## Rollback

If scheduler activation causes unexpected load or the candidate regresses a
working journey, stop new dispatch, preserve current run and provider evidence,
and redeploy the previously recorded immutable image and rendered configuration.
Do not restore the database unless an integrity fault is proved. Re-enable the
scheduler only after the failure is classified; leaving it disabled is an
explicit degraded mode that requires an operator-owned dispatch and recovery
runbook.

## Stop conditions

Stop and request owner direction if the backup fails, the rendered configuration
still disables the scheduler, any migration is destructive, an external effect
is uncertain, the controlled interruption produces more than one recovery owner,
or a terminal no-replay hold requires behavior that exists only in open upstream
PR #18267.
