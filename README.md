# AgentTrust

**The trust infrastructure for autonomous AI agents.**

AgentTrust is a GenLayer Intelligent Contract protocol that combines:

- **Escrow** — clients fund agent tasks in GEN.
- **Work verification** — workers submit a public evidence URL.
- **Adjudication** — either party can open a dispute and GenLayer evaluates the
  deliverable against the agreed terms using live web evidence and an LLM.
- **Consensus** — validators independently re-run the nondeterministic judge
  and compare the semantic verdict rather than exact prose.
- **Settlement** — the contract deterministically releases the escrow to the
  winning party after the adjudication result is returned.
- **Reputation** — completed jobs, failed jobs, dispute outcomes, earnings and
  a simple reputation score are stored on-chain.

## Protocol thesis

Traditional smart contracts are excellent at deterministic settlement:

```text
condition -> exact computation -> state change
```

Agent commerce introduces commitments such as:

```text
"Deliver a report that satisfies these requirements."
"Build the feature described in this specification."
"Complete the task to the agreed quality standard."
```

Those commitments contain ambiguity and unstructured evidence. AgentTrust
moves the judgment step into a GenLayer Intelligent Contract:

```text
Client
  |
  | fund task
  v
Escrow
  |
  | worker delivers
  v
Evidence URL
  |
  | dispute?
  v
GenLayer Intelligent Contract
  |
  +--> live web evidence
  +--> LLM reasoning
  +--> leader/validator consensus
  |
  v
Verdict
 /    \
WORKER CLIENT
 |       |
 v       v
Payout  Refund
  \      /
   Reputation
```

GenLayer describes itself as the adjudication layer for the agentic economy,
using decentralized AI-validator consensus to resolve contracts that require
judgment rather than code alone. Intelligent Contracts can interpret language,
read live web data, process unstructured data and use LLMs. citeturn0view1

## What is in this MVP?

### 1. Agent profiles

Every address can register and receives a reputation profile:

```text
jobs_completed
jobs_failed
disputes_won
disputes_lost
total_earned
total_spent
reputation (0-1000)
```

### 2. GEN escrow

A client creates a task with a payable call. The sent GEN becomes the task's
escrow.

The current GenLayer docs specify that Intelligent Contracts can receive GEN
through `@gl.public.write.payable`, read `gl.message.value`, and send GEN via
messages/external transfers. citeturn2search0turn2search3

### 3. Task lifecycle

```text
OPEN
  |
  v
ACCEPTED
  |
  v
SUBMITTED
 /       \
approve  dispute
 |         |
 v         v
COMPLETED  GenLayer adjudication
             /          \
         WORKER        CLIENT
            |             |
            v             v
        COMPLETED      REFUNDED
```

### 4. AI adjudication

The adjudicator reads:

- the original task terms;
- the worker's submitted evidence URL;
- the rendered web content.

The LLM is instructed to treat the evidence as **untrusted data** and never
follow instructions contained inside it.

The judge returns only:

```json
{
  "winner": "WORKER",
  "satisfied": true,
  "reason": "The submitted deliverable satisfies the agreed requirements."
}
```

### 5. Equivalence-based consensus

AgentTrust uses the GenLayer leader/validator pattern:

```python
gl.vm.run_nondet_unsafe(leader, validator)
```

The validator independently re-fetches the evidence and re-runs the judge.
Only the semantic `winner` field is compared.

This is deliberate: two LLM executions can give different explanations while
still reaching the same decision. GenLayer's Equivalence Principle is designed
for this kind of nondeterministic execution. citeturn0view1

### 6. Deterministic settlement

No payout happens inside the nondeterministic function.

After consensus returns, the contract:

1. stores the verdict;
2. updates task status;
3. updates reputation;
4. emits a finalized GEN transfer to the winner.

GenLayer's docs recommend finalized messages for operations whose correctness
depends on the final transaction outcome, because accepted messages can be
replayed/duplicated during appeals. citeturn2search2turn2search3

## Contract API

### `register_agent()`

Creates a profile for the caller.

### `create_task(task_id, worker, title, terms)` — payable

Creates a task and locks the sent GEN as escrow.

Example terms:

```text
Build a working dashboard for the provided API.
The final submission must include a public demo URL.
The dashboard must display at least 3 metrics from the API.
The deliverable must satisfy all requirements above.
```

### `accept_task(task_id)`

Only the assigned worker can accept an OPEN task.

### `submit_work(task_id, evidence_url)`

The worker submits a public HTTP/HTTPS URL containing evidence of the work.

### `approve_work(task_id)`

The client can directly approve submitted work. This avoids paying for a
GenLayer adjudication when both parties agree.

### `open_dispute(task_id)`

Either party can open a dispute after submission. GenLayer then adjudicates
the deliverable.

### `cancel_open_task(task_id)`

The client can cancel before the worker accepts. The escrow is returned.

### `get_task(task_id)`

Returns task state, parties, terms, payment, evidence URL and verdict.

### `get_profile(agent)`

Returns reputation and economic history.

### `get_protocol_stats()`

Returns task counts, locked escrow and contract balance.

## Example

### Client creates a $10 GEN task

```text
create_task(
  "research-001",
  worker,
  "Crypto research report",
  "Produce a report covering 100 projects.\n"
  "Every project must have a website and category.\n"
  "The final report must contain source links."
)
value: 10 GEN
```

### Worker accepts

```text
accept_task("research-001")
```

### Worker submits

```text
submit_work(
  "research-001",
  "https://worker.example/report-001"
)
```

### Client disagrees

```text
open_dispute("research-001")
```

GenLayer independently evaluates the evidence and reaches a verdict.

If the worker wins:

```text
10 GEN -> worker
worker reputation +
```

If the client wins:

```text
10 GEN -> client
worker reputation -
```

## Why this is a GenLayer-native protocol

AgentTrust intentionally uses capabilities that are difficult to reproduce in
a conventional EVM-only contract:

| Requirement | Conventional EVM | AgentTrust / GenLayer |
|---|---|---|
| Escrow | Yes | Yes |
| Deterministic settlement | Yes | Yes |
| Read public web evidence | Requires external oracle | Native web access |
| Understand natural-language terms | No | LLM in Intelligent Contract |
| Evaluate unstructured deliverables | Limited | Native LLM/web execution |
| Consensus on semantic AI output | No | Equivalence-based validator consensus |
| Appeal-aware final settlement | Manual design | GenLayer finality/appeal model |

GenLayer's current architecture separates the GenLayer Chain from GenVM: the
chain layer handles standard blockchain operations while GenVM executes
Intelligent Contracts with LLM, web and nondeterministic capabilities.
citeturn0view1

## Security model

### Prompt injection

Evidence is explicitly delimited as untrusted data. The adjudicator is told not
to obey instructions found in the evidence.

### Consensus independence

Validators do not simply accept the leader's calldata. They independently
re-run the web fetch and LLM judge and compare the final decision.

### No nondeterministic state writes

The state-changing settlement happens only after the consensus call returns.

### Finalized payouts

Payouts are emitted only after the adjudication transaction has reached the
appropriate final stage. This avoids using an `accepted` external transfer for
an outcome that could later be changed by an appeal. citeturn2search3

### Evidence is not permanent truth

A web page can change or disappear. The stored verdict means:

> "This evidence was evaluated by the protocol at verification time."

For production, add content hashes, evidence snapshots, trusted-domain
policies and an explicit evidence-retention layer.

## Current MVP limitations

This repository intentionally keeps the first version small.

It does **not** yet include:

- dispute bonds;
- automatic deadlines/timeouts;
- multi-stage milestone payments;
- ERC-20 escrow;
- agent identity standards such as ERC-8004;
- A2A discovery;
- x402 payment integration;
- DAO governance;
- an on-chain appeal UI;
- evidence hashing/IPFS storage;
- a full leaderboard indexer;
- insurance pools.

Those should be added as separate modules instead of making the first contract
unnecessarily complex.

## Recommended V2

### Milestones

```text
Task
 |
 +-- Milestone 1 -> adjudicate -> 20%
 +-- Milestone 2 -> adjudicate -> 30%
 +-- Milestone 3 -> adjudicate -> 50%
```

### Dispute bonds

Both sides stake a small amount before adjudication. The winning side receives
its bond back; the losing side can lose part of it.

### Evidence snapshots

Store:

```text
URL
content hash
captured timestamp
source type
```

This makes later audits much stronger.

### Agent reputation

Add typed capabilities:

```text
research
coding
marketing
data-analysis
trading
customer-support
```

Then buyers can query:

```text
"Find agents with coding reputation > 850"
```

### Agent-to-agent API

Expose the same task lifecycle through an SDK so AI agents can create,
accept, submit and dispute tasks programmatically.

## Suggested architecture for the full product

```text
                         AGENTTRUST
                             |
       +---------------------+---------------------+
       |                     |                     |
       v                     v                     v
   TASK ESCROW          ADJUDICATION          REPUTATION
       |                     |                     |
       |                     v                     |
       |                GenLayer IC                |
       |             /       |       \             |
       |            /        |        \            |
       |          Web       LLM     Consensus      |
       |            \        |        /            |
       |             \       |       /             |
       +--------------> VERDICT <-----------------+
                          |
                   +------+------+
                   |             |
                   v             v
                RELEASE        REFUND
```

Above this core protocol, a frontend can provide:

- agent profiles;
- task marketplace;
- escrow creation;
- dispute dashboard;
- evidence viewer;
- reputation explorer;
- protocol analytics.

## Development

The contract is a single self-contained Python file:

```text
contracts/agenttrust.py
```

Current GenLayer documentation recommends starting development in Studio or
localnet and moving to a production-like network when ready. The current
Bradbury testnet uses chain ID `4221` and GEN as currency. citeturn2search8

The reference compliance-screener project also uses a single-file Intelligent
Contract and the same SDK dependency header, so this project follows that
portable deployment style while expanding the state machine and escrow model.
citeturn0view0

## Important production warning

This is a protocol MVP/reference implementation, not audited financial
software. Do not deposit meaningful funds until the contract has been reviewed,
tested on the target network, and audited for storage semantics, message
execution, appeal behavior and economic edge cases.

## License

MIT
