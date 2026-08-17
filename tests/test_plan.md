# AgentTrust test plan

The following cases should be implemented with the GenLayer testing suite before mainnet use.

## Happy path

1. Client funds task.
2. Worker accepts.
3. Worker submits evidence.
4. Client approves.
5. Worker receives exact escrow.
6. Worker reputation increases.

## Cancellation

1. Client funds task.
2. Worker does not accept.
3. Client cancels.
4. Client receives exact escrow.

## Dispute - worker wins

Mock the nondeterministic adjudicator to return WORKER.
Verify:

- status = COMPLETED
- worker receives escrow
- worker jobs_completed increments
- worker disputes_won increments
- worker reputation increases

## Dispute - client wins

Mock the adjudicator to return CLIENT.
Verify:

- status = REFUNDED
- client receives escrow
- worker jobs_failed increments
- worker disputes_lost increments
- worker reputation decreases

## Authorization

Verify that:

- only assigned worker can accept/submit;
- only client can approve/cancel;
- only task parties can dispute.

## Consensus edge cases

Test:

- leader/validator same verdict with different reasons;
- validator disagreement;
- malformed LLM result;
- external web failure;
- empty evidence page.

## Appeal/finality behavior

On a live GenLayer network, verify that settlement transfers occur only at the
safe finalized stage and cannot be duplicated by an appeal/re-execution.
