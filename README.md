# AgentTrust v3 — Complete Core Contract

This package replaces the previous "schema fixed" reduction. The goal is not
to make the contract short; it is to keep the **full AgentTrust v3 workflow**
while using a storage schema that GenLayer Studio can parse reliably.

## Full workflow

```text
CLIENT
  |
  | create_task + GEN
  v
OPEN
  |
  | accept_task
  v
ACCEPTED
  |
  | submit_work(evidence_url)
  v
SUBMITTED
  |                  |
  | approve         | open_dispute + GEN bond
  v                  v
COMPLETED          DISPUTED
                       |
                       | adjudicate
                       v
                  GENLAYER COURT
                    /       \
                   /         \
              WORKER        CLIENT
                 |             |
        reward + bond       reward + bond
                 |             |
                 v             v
             COMPLETED      REFUNDED

OPEN / ACCEPTED + deadline expired
              |
              v
       claim_expired
              |
              v
          REFUNDED

OPEN + client
      |
      v
cancel_task
      |
      v
CANCELLED
```

## Contract surface

### Marketplace
- `create_task(title, terms, deadline)` payable
- `accept_task(task_id)`
- `submit_work(task_id, evidence_url, evidence_note)`
- `cancel_task(task_id)`
- `claim_expired(task_id)`

### Escrow/review
- `approve_task(task_id)`
- `auto_release(task_id)`

### Court
- `open_dispute(task_id, bond)` payable
- `adjudicate(task_id)`

### Reputation
- `get_profile(address)`
- `get_my_profile()`
- `get_leaderboard(offset, limit)`

### Discovery/read APIs
- `get_task(task_id)`
- `get_task_ids(offset, limit)`
- `get_open_tasks(offset, limit)`
- `get_tasks_by_status(status, offset, limit)`
- `get_task_state(task_id)`
- `get_stats()`
- `get_config()`

### Protocol admin
- `set_paused(paused)`
- `set_min_reward(amount)`
- `set_min_dispute_bond(amount)`
- `set_review_period(seconds)`

## Why task/profile data is JSON

GenLayer persistent storage supports `TreeMap` and `DynArray`, and all
generic types must be fully specified. The contract therefore stores task and
profile records as JSON strings in `TreeMap[str, str]`. This is intentionally
less elegant than a storage dataclass but much safer for the Studio schema
generator while keeping the complete workflow.

## GenLayer Court design

The Court's non-deterministic section:
1. Fetches the public evidence URL.
2. Sends task terms + evidence to the LLM.
3. Requests structured JSON.
4. A validator independently fetches/evaluates the evidence.
5. Validators must agree on the settlement-critical `winner`.
6. Score is informational and allows a small tolerance.
7. Only after consensus does deterministic code transfer GEN and update state.

No storage write or GEN transfer is performed inside the non-deterministic
block.

## Studio deployment

Upload:

`agenttrust_v3_complete.py`

The constructor takes **zero inputs**.

If Studio shows:

`Could not load contract schema`

then the problem is schema parsing rather than constructor values. This
version deliberately avoids nested persistent Python dictionaries.

## Important production caveats

This is a protocol prototype, not an audited financial contract.

Before mainnet:
- run `genvm-lint check`;
- test every state transition;
- test native GEN transfers;
- test failed external transfers;
- test evidence pages that disappear/change;
- test prompt injection;
- test validator disagreement;
- test deadline edge cases;
- test large task/profile datasets;
- add a migration/version strategy before upgrading storage.

GenLayer's current docs state that non-deterministic web/LLM operations must
run in nondeterministic blocks, while storage writes and message emissions
must happen after consensus. Native GEN transfers to EOAs are external
messages and execute on finalization.

