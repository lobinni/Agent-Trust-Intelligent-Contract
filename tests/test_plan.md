# AgentTrust v3 — End-to-End Test Matrix

## 1. Constructor / schema
- Deploy with zero constructor inputs.
- Confirm no required constructor arguments.

## 2. Create
- Zero value -> revert.
- Below minimum reward -> revert.
- Past deadline -> revert.
- Valid task -> OPEN and reward escrowed.

## 3. Accept
- Client accepts own task -> revert.
- Second user accepts OPEN task -> ACCEPTED.
- Second user tries again -> revert.
- Accept after deadline -> revert.

## 4. Submit
- Wrong worker -> revert.
- Invalid/empty evidence URL -> revert.
- Submit after deadline -> revert.
- Valid evidence -> SUBMITTED.

## 5. Approval
- Non-client approves -> revert.
- Client approves -> worker paid; COMPLETED; reputation updated.

## 6. Auto-release
- Before review deadline -> revert.
- After review deadline -> worker paid; COMPLETED.

## 7. Dispute
- Non-client -> revert.
- After review deadline -> revert.
- Bond below minimum -> revert.
- Attached value != bond -> revert.
- Valid dispute -> DISPUTED.

## 8. Court
- Adjudicate non-disputed -> revert.
- Worker-winning evidence -> worker gets reward + bond.
- Client-winning evidence -> client gets reward + bond.
- Leader/validator disagreement -> no settlement; retry with another leader.
- Malicious webpage instructions -> ignored as untrusted data.
- HTTP error -> no settlement.

## 9. Deadline
- OPEN after deadline -> client refund.
- ACCEPTED after deadline -> client refund + worker failure penalty.
- SUBMITTED after deadline -> not refundable by `claim_expired`; use review/dispute flow.

## 10. Cancel
- Client can cancel only OPEN tasks.
- Worker cannot cancel.
- ACCEPTED task cannot be cancelled.

## 11. Reputation
- Completion increments jobs_completed and earned.
- Missed deadline increments jobs_failed and reduces reputation.
- Worker court win increments disputes_won.
- Worker court loss increments disputes_lost/jobs_failed.
- Client dispute outcome updates dispute counters.

## 12. Frontend integration
Map:
- Marketplace -> `get_open_tasks`
- Task page -> `get_task`, `get_task_state`
- Create -> `create_task`
- Accept -> `accept_task`
- Evidence -> `submit_work`
- Review -> `approve_task`, `open_dispute`
- Court -> `adjudicate`
- Reputation -> `get_profile`, `get_leaderboard`
