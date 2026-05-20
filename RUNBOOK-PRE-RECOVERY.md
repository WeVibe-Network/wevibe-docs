# PRE Recovery Runbook

## Purpose

Define the emergency-only procedure for PRE recovery authorization and execution, aligned with decision D-10.2.

## Scope

- Organization-level PRE recovery actions when normal epoch operations cannot proceed.
- Governance-gated recovery path only; this is not for routine key rotation.

## Preconditions

- Incident commander has declared PRE recovery required.
- Two of three Shamir shareholders are available and authenticated.
- On-chain multi-sig authorization path is available.
- Org leadership has prepared an incident record and communication plan.

## Roles and Responsibilities

- Leader shareholder: initiates incident declaration and co-signs recovery authorization.
- Hub shareholder: validates operational state and co-signs recovery authorization.
- Recovery shareholder: independent co-signer for separation of duties.
- Scribe: records timeline, signer identities, and transaction IDs.

## Recovery Ceremony

1. Open incident and assign an incident ID.
2. Verify trigger conditions and capture supporting evidence.
3. Obtain 2-of-3 shareholder approvals in writing.
4. Submit on-chain multi-sig recovery authorization transaction.
5. Execute PRE recovery flow using the dedicated recovery path.
6. Record all recovery actions and resulting transaction IDs in the incident log.
7. Confirm org epoch state and member access behavior match expected post-recovery state.

## Required On-Chain Evidence

- Multi-sig authorization transaction hash.
- Any epoch status transition transaction hash.
- Audit log entry mapping incident ID to chain transactions.

## Post-Recovery Validation

- Authorized members can retrieve approved memories in the active epoch.
- Removed members cannot obtain new re-encryption material.
- Hub and client logs show no fallback path usage.
- Incident record includes full signer/audit trace.

## D-10.2 Caveat (Current Implementation State)

D-10.2 requires the recovery path to run outside the hub binary through a separate `echo-recover` CLI and to remain operationally inconvenient. That dedicated `echo-recover` CLI is not implemented yet. Until it exists, treat this runbook as the required ceremony and controls baseline, and do not fold recovery execution into routine hub operations.

## Prohibited Uses

- Do not use PRE recovery for convenience, operator turnover, or normal key rotation.
- Do not bypass multi-sig or 2-of-3 shareholder requirements.
- Do not execute recovery without incident logging.
