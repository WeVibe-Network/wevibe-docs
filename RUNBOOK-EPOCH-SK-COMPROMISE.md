# Runbook: Epoch SK Compromise

## Purpose

Define incident response for suspected or confirmed epoch secret key (SK) compromise, including containment, status transitions, and deferred background re-encryption planning.

## Trigger Conditions

- Evidence that 2-of-3 shareholders may have reconstructed an epoch SK outside approved procedure.
- Evidence of endpoint compromise during kfrag generation window.
- Unexplained retrieval behavior consistent with unauthorized decrypt capability.

## Severity and Ownership

- Severity: SEV-1 (cryptographic incident).
- Incident commander: org leader or delegated security lead.
- Required participants: leader, hub operator, recovery holder, audit scribe.

## Immediate Incident Response

1. Declare incident and open timeline with incident ID.
2. Freeze non-essential operational changes for affected org.
3. Mark affected epoch as `compromised` on-chain as soon as governance rules allow.
4. Notify moderators and members that compromise protocol is active.
5. Preserve logs and evidence (hub logs, signer approvals, transaction hashes).

## Chain and Hub Behavior During Compromised Window

- Chain epoch status must be explicit (`active` -> `compromised` -> `rotated`).
- Hub must treat `compromised` as a hard security signal for incident mode handling.
- No hidden fallback behavior is allowed; all incident handling is explicit and auditable.

## Recovery and Rotation Procedure

1. Execute PRE recovery ceremony per `RUNBOOK-PRE-RECOVERY.md`.
2. Authorize and perform epoch rotation to establish fresh cryptographic material.
3. Re-seal authorized member access material for the new epoch.
4. Resume normal operations only after post-rotation validation is complete.

## Deferred Background Re-Encryption Notes

Background re-encryption at full corpus scale is operationally deferred and not implemented as an instant response path.

- This runbook does not assume immediate re-encryption of all historical memories.
- Treat the compromised epoch as burned risk per D-10.1 acceptance.
- Plan any background re-encryption as a separate tracked operation with explicit throughput, gas budget, and completion monitoring.
- Do not describe or implement this as endpoint erasure semantics.

## Monitoring and Completion Criteria

- Incident timeline contains all approvals, chain TX hashes, and role assignments.
- Epoch status is `rotated` and visible to all required operators.
- Retrieval behavior matches post-rotation authorization boundaries.
- Deferred re-encryption decision is documented with owner, schedule, and risk acceptance.

## Communication Requirements

- Publish an internal incident notice at declaration, containment, and closure.
- Include: affected org, affected epoch, user impact, actions taken, and remaining risk.
- Archive the final incident report with links to on-chain evidence.
