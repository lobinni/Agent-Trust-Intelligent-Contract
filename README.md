# AgentTrust

**A public, trustless task marketplace powered by GenLayer Intelligent Contracts.**

Clients fund tasks with native GEN tokens into an escrow contract. Workers accept tasks and submit public web evidence. GenLayer immediately reviews the submitted work using independent web fetching, AI evaluation, and validator consensus before the creator receives the final result. Escrow settlements and reputation updates are executed entirely on-chain.

[![GenLayer](https://img.shields.io/badge/GenLayer-Studionet-8cffbd)](https://genlayer.com)
[![Contract](https://img.shields.io/badge/Contract-0xFf7c...612c-7dd3fc)](https://explorer-studio.genlayer.com/address/0xFf7cCC740271Ee6664398503D8564380578b612c)
[![Next.js](https://img.shields.io/badge/Next.js-16-black)](https://nextjs.org)
[![License](https://img.shields.io/badge/License-MIT-blue)](#license)

---

## 📖 Table of Contents

- [Latest Update](#latest-update)
- [Deployed Contract](#deployed-contract)
- [How It Works](#how-it-works)
- [Workflow Diagram](#workflow-diagram)
- [Evidence Review & AI Verdict](#evidence-review--ai-verdict)
- [Contract States & Methods](#contract-states--methods)
- [Reputation System](#reputation-system)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [MetaMask Setup](#metamask-setup)
- [Tests](#tests)
- [Deploy to Vercel](#deploy-to-vercel)
- [Push to GitHub](#push-to-github)
- [Security Notes](#security-notes)
- [License](#license)

---

## Latest Update

### Contract `0xFf7cCC740271Ee6664398503D8564380578b612c`

The latest Intelligent Contract is live on GenLayer Studionet with:

1. **Immediate Atomic AI Review:**
   - When a worker calls `submit_work(task_id, evidence_url, evidence_note)`, the contract triggers GenLayer AI review in the **same transaction**.
   - Validators independently fetch the evidence URL, evaluate it against the task terms, and reach consensus.
   - The contract atomically saves `ai_verdict` (`APPROVED` / `REJECTED`), `ai_score` (0–100), `ai_reason`, and `reviewed_at`.
   - The creator receives the result only **after** GenLayer AI has evaluated the work.

2. **Web Response Compatibility Fix:**
   - Uses `int(page.status)` conforming to `py-genlayer` `Response(status, headers, body)`.
   - Robust body decoding with UTF-8 replacement and empty-body checks.

3. **Nondeterministic Safety:**
   - All contract/storage values are resolved to primitives before entering nondeterministic evaluation.
   - Evaluator and validator closures capture clean primitives without referencing contract storage.

4. **Transaction Result Verification:**
   - The DApp frontend verifies the transaction consensus result (`MAJORITY_AGREE` vs `MAJORITY_DISAGREE`) after finality.
   - If consensus fails, the full GenVM error message and transaction hash are shown with a link to the explorer.

---

## Deployed Contract

| Parameter | Value |
|---|---|
| **Network** | GenLayer Studionet |
| **Contract Address** | [`0xFf7cCC740271Ee6664398503D8564380578b612c`](https://explorer-studio.genlayer.com/address/0xFf7cCC740271Ee6664398503D8564380578b612c) |
| **Chain ID** | `61999` (`0xf22f`) |
| **RPC Endpoint** | `https://studio.genlayer.com/api` |
| **Block Explorer** | https://explorer-studio.genlayer.com |
| **Native Token** | GEN (18 decimals) |
| **Contract Owner** | `0x5dB05F47cfFe01272Bc7139095Cd15981879284D` |
| **Review Window** | 24 hours (`86400` seconds) |
| **Protocol Status** | Active (not paused) |

---

## How It Works

```
1. Client creates a task and deposits native GEN into escrow.
2. Worker accepts the task and begins work.
3. Worker submits evidence URL + notes via submit_work.
4. GenLayer immediately fetches the URL, runs AI judgment, and reaches validator consensus.
5. Contract records AI Verdict (APPROVED or REJECTED) + score + reason.
6. Task moves to SUBMITTED — client sees the AI evaluation.
7. Client approves (worker paid) OR disputes with a GEN bond (triggers full court).
8. Escrow settles automatically and reputation scores update on-chain.
```

---

## Workflow Diagram

```text
                         CLIENT
                            │
                            │ create_task + GEN (escrow deposit)
                            ▼
                          OPEN ──── cancel_task ────► CANCELLED (client refunded)
                            │
                            │ accept_task (any worker wallet)
                            ▼
                        ACCEPTED
                            │
                            │ submit_work + evidence URL
                            ▼
               ┌────────────────────────┐
               │ GENLAYER AI REVIEW     │
               │                        │
               │ 1. Validators fetch URL│
               │ 2. AI evaluates terms  │
               │ 3. Consensus on score  │
               └───────────┬────────────┘
                           │
                 APPROVED / REJECTED
                 score (0-100) + reason
                           │
                           ▼
                       SUBMITTED (client reviews AI verdict)
                      /         \
                     /           \
            approve_task     open_dispute + GEN bond
                  │                  │
                  ▼                  ▼
             COMPLETED           DISPUTED
            worker paid              │
                                     │ adjudicate()
                                     ▼
                           ┌──────────────────┐
                           │  GENLAYER COURT  │
                           │  fresh AI review │
                           │  + consensus     │
                           └────────┬─────────┘
                               ┌────┴────┐
                               ▼         ▼
                            WORKER     CLIENT
                               │         │
                         reward+bond reward+bond
                               │         │
                               ▼         ▼
                          COMPLETED   REFUNDED

  Recovery paths:
  OPEN/ACCEPTED + deadline passed ──► claim_expired ──► REFUNDED (worker −10 rep if accepted)
  SUBMITTED + 24h review passed   ──► auto_release  ──► COMPLETED (worker paid)
```

---

## Evidence Review & AI Verdict

When evidence is submitted, the contract and DApp provide complete transparency:

- **Evidence URL:** Clickable public link proving task delivery.
- **Worker's Note:** Contextual details provided by the worker.
- **AI Verdict Badge:** `✅ AI APPROVED` (score ≥ 50) or `❌ AI REJECTED` (score < 50).
- **AI Score Bar:** Visual progress indicator (0 to 100).
- **AI Reason:** Factual explanation generated by GenLayer validator consensus.
- **Timeline:** Timestamped event log tracking the task from creation to settlement.
- **Settlement Summary:** Clear breakdown of escrow reward, dispute bond, recipient, and resolution time.

---

## Contract States & Methods

### Task States

| State | Description |
|---|---|
| `OPEN` | Task is active in marketplace, waiting for a worker to claim. |
| `ACCEPTED` | Worker is assigned, working towards the deadline. |
| `SUBMITTED` | Worker submitted evidence, GenLayer AI review completed, waiting for client decision. |
| `DISPUTED` | Client opened dispute with a bond, waiting for court adjudication. |
| `COMPLETED` | Task approved or won by worker; escrow released to worker. |
| `REFUNDED` | Task cancelled, expired, or won by client; funds refunded to client. |
| `CANCELLED` | Client cancelled an unassigned `OPEN` task. |
| `REVIEWING` | Transitional state during AI review execution. |

### Write Methods (MetaMask signed)

| Method | Access | Payable | Description |
|---|---|---:|---|
| `create_task(title, terms, deadline)` | Public | GEN reward | Create task and lock reward in escrow |
| `accept_task(task_id)` | Worker | No | Claim an open task |
| `submit_work(task_id, url, note)` | Worker | No | Submit evidence and run atomic AI review |
| `review_work(task_id)` | Public | No | Recovery AI review trigger |
| `approve_task(task_id)` | Client | No | Approve work and release escrow to worker |
| `open_dispute(task_id, bond)` | Client | GEN bond | Dispute submission and post bond |
| `adjudicate(task_id)` | Public | No | Run GenLayer Court on disputed task |
| `auto_release(task_id)` | Public | No | Release funds to worker after 24h review window |
| `claim_expired(task_id)` | Public | No | Refund expired OPEN/ACCEPTED task |
| `cancel_task(task_id)` | Client | No | Cancel unassigned OPEN task and refund client |

### Read Methods (Public, no gas)

| Method | Return Type | Description |
|---|---|---|
| `get_task(task_id)` | `dict` | Full task record |
| `get_task_state(task_id)` | `dict` | Task state and available action flags |
| `get_task_ids(offset, limit)` | `list[str]` | Paginated task IDs |
| `get_open_tasks(offset, limit)` | `list[dict]` | Paginated open tasks |
| `get_tasks_by_status(status, offset, limit)` | `list[dict]` | Filtered task list |
| `get_profile(address)` | `dict` | Wallet reputation profile |
| `get_my_profile()` | `dict` | Caller's reputation profile |
| `get_leaderboard(offset, limit)` | `list[dict]` | Reputation ranking |
| `get_stats()` | `dict` | Protocol task/profile counts and pause state |
| `get_config()` | `dict` | Owner, limits, review period |

---

## Reputation System

Reputation is stored and updated directly on-chain:

| Action | Score Change |
|---|---:|
| Client approves task | Worker **+5** |
| Worker wins dispute in court | Worker **+8**, Client **−2** |
| Client wins dispute in court | Client **+2**, Worker **−8** |
| Worker misses deadline (after accepting) | Worker **−10** |
| **Initial Default Score** | **500** |
| **Score Range** | **0 to 1000** |

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Intelligent Contract** | Python with GenLayer SDK (`py-genlayer`) |
| **Network** | GenLayer Studionet (Chain ID: `61999` / `0xf22f`) |
| **Frontend Framework** | Next.js 16 (App Router), React 19, TypeScript |
| **Web3 Client** | `genlayer-js` with MetaMask (EIP-1193) provider |
| **Styling & Fonts** | Tailwind CSS v4, Space Grotesk, JetBrains Mono |
| **Optional Mirror** | PostgreSQL with Drizzle ORM |
| **Deployment** | Vercel (Frontend), GenLayer CLI / Studio (Contract) |

---

## Repository Structure

```text
├── contracts/
│   └── agenttrust.py          # GenLayer Intelligent Contract (Python)
├── tests/
│   ├── frontend_tests.sh      # Next.js API & deployed contract integration tests
│   └── contract_tests.sh      # GenLayer CLI on-chain test script
├── src/
│   ├── app/
│   │   ├── page.tsx           # DApp shell (Marketplace, Create, My Tasks, Reputation)
│   │   ├── layout.tsx         # Root layout with Web3 fonts and metadata
│   │   ├── globals.css        # Cyberpunk Web3 theme, buttons, pills, tabs
│   │   └── api/
│   │       ├── health/        # Health check endpoint
│   │       └── contract/sync/ # Optional PostgreSQL mirror sync endpoint
│   ├── components/
│   │   ├── WalletContext.tsx  # MetaMask connection and chain auto-switching
│   │   ├── Header.tsx         # Top bar, network status, wallet button, tabs
│   │   ├── NetworkStats.tsx   # On-chain stats and contract configuration
│   │   ├── TaskMarketplace.tsx# Live task explorer with status filters
│   │   ├── CreateTask.tsx     # Escrow creation form
│   │   ├── TaskDetail.tsx     # Task view: evidence review, AI verdict, actions
│   │   ├── MyTasks.tsx        # Personal client/worker task views
│   │   └── Reputation.tsx     # Profile stats and on-chain leaderboard
│   ├── lib/
│   │   ├── genlayer-client.ts # Browser SDK: MetaMask reads, writes, error parser
│   │   ├── genlayer.ts        # Server SDK: contract reads
│   │   ├── types.ts           # TypeScript interfaces matching contract
│   │   └── utils.ts           # Formatters (addresses, wei/GEN, timestamps)
│   └── db/
│       ├── index.ts           # Lazy PostgreSQL pool client
│       └── schema.ts          # Database schema for optional mirror
├── deploy.sh                  # Automated push script
├── .env.example               # Template environment variables
├── vercel.json                # Vercel deployment configuration
└── package.json
```

---

## Getting Started

### Prerequisites

- Node.js 20+
- npm
- MetaMask browser extension
- GEN testnet tokens on GenLayer Studionet

### Installation

```bash
# 1. Clone the repository
git clone https://github.com/lobinni/AgentTrust.git
cd AgentTrust

# 2. Install dependencies
npm install

# 3. Setup environment variables
cp .env.example .env.local

# 4. Start local development server
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser.

---

## MetaMask Setup

The DApp handles network switching automatically:

1. Click **Connect Wallet** in the header.
2. The DApp checks if **GenLayer Studionet** (`0xf22f` / `61999`) is present in MetaMask.
3. If missing, it prompts MetaMask to add the network with:
   - **Network Name:** `GenLayer Studionet`
   - **RPC URL:** `https://studio.genlayer.com/api`
   - **Chain ID:** `61999` (`0xf22f`)
   - **Currency Symbol:** `GEN`
   - **Block Explorer:** `https://explorer-studio.genlayer.com`
4. If on another chain, it prompts to switch.
5. If the chain is wrong, a **Switch Network** banner appears with a single-click fix.

---

## Tests

### 1. Build & Type Validation

```bash
npx next typegen
npm exec tsc -- --noEmit --pretty false
npm run build
```

### 2. Live Contract & API Integration Tests

```bash
# Start server in background
npm run dev &

# Run test suite against http://localhost:3000
./tests/frontend_tests.sh
```

Verifies:
- `/api/health`
- `/api/contract/sync` reads contract stats and configuration
- Contract address matches `0xFf7cCC740271Ee6664398503D8564380578b612c`
- Review period is 86400s
- DApp pages render cleanly

### 3. Contract CLI Tests (using GenLayer CLI)

```bash
pip install genlayer-cli
genlayer login
NETWORK=studionet ./tests/contract_tests.sh
```

---

## Deploy to Vercel

1. Import `lobinni/AgentTrust` into [Vercel](https://vercel.com).
2. Framework Preset: **Next.js** (auto-detected).
3. Set Environment Variables (Production, Preview, Development):

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_GENLAYER_NETWORK` | `studionet` |
| `NEXT_PUBLIC_CONTRACT_ADDRESS` | `0xFf7cCC740271Ee6664398503D8564380578b612c` |
| `DATABASE_URL` | *(Optional)* PostgreSQL connection string |

4. Click **Deploy**.

> **Zero-Database Dependency:** The public DApp reads directly from GenLayer RPC in the browser. The database is only used for the optional mirror cache. Builds never fail if `DATABASE_URL` is omitted.

---

## Push to GitHub

### Option A: Using the deploy script

```bash
chmod +x deploy.sh
./deploy.sh "feat: connect verified AgentTrust contract 0xFf7cCC740271Ee6664398503D8564380578b612c"
```

### Option B: Manual git commands

```bash
# Stage all changes
git add .

# Commit
git commit -m "feat: connect verified AgentTrust contract 0xFf7cCC740271Ee6664398503D8564380578b612c"

# Push to main
git push origin main
```

If initializing a fresh repository:

```bash
git init
git add .
git commit -m "feat: AgentTrust public task marketplace on GenLayer"
git branch -M main
git remote add origin https://github.com/lobinni/AgentTrust.git
git push -u origin main --force-with-lease
```

---

## Security Notes

- Evidence URLs and notes are treated as untrusted user input.
- Contract prompt instructions prevent the LLM from executing prompt injection attacks found inside evidence content.
- Nondeterministic closures capture only clean primitive variables.
- Token transfers and storage mutations execute strictly after consensus is finalized.
- This contract is deployed for testing and community evaluation on GenLayer Studionet.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

<p align="center">
  <strong>AgentTrust</strong> — Trustless. Autonomous. Verifiable.<br />
  Built on <a href="https://genlayer.com">GenLayer</a> Intelligent Contracts
</p>
