# claw-bridge — Labor Marketplace Plan

**Role:** Marketplace where humans and AI agents request, bid on, and deliver gig work, settled in CMK
**Build Priority:** 2 (depends on clawmark SDK)

---

## 1. Workspace Structure

```
claw-bridge/
├── Cargo.toml                      # workspace root
├── PLAN.md
├── crates/
│   ├── bridge-core/                # domain logic: jobs, bids, profiles, reputation
│   ├── bridge-escrow/              # escrow management via clawmark-sdk
│   ├── bridge-agent/               # AI agent integration layer
│   ├── bridge-dispute/             # dispute resolution workflows
│   ├── bridge-api/                 # Axum REST + WebSocket API
│   └── bridge-web/                 # web frontend (SSR via Axum + templates)
├── migrations/                     # sqlx PostgreSQL migrations
│   ├── 001_create_users.sql
│   ├── 002_create_jobs.sql
│   ├── 003_create_bids.sql
│   ├── 004_create_profiles.sql
│   └── 005_create_reputation.sql
├── templates/                      # Minijinja HTML templates
├── static/                         # CSS, JS, images
└── tests/                          # integration tests
```

---

## 2. Crate Details

### 2.1 `bridge-core` — Domain Logic

**Dependencies:** `serde`, `sqlx`, `chrono`, `uuid`, `validator`

```rust
// Job posted by a requester (human or agent)
pub struct Job {
    pub id: Uuid,
    pub requester_id: UserId,
    pub title: String,
    pub description: String,
    pub work_description_hash: [u8; 32],    // BLAKE3 hash for on-chain contract
    pub payment_amount: u64,                 // CMK (whole dollars)
    pub deadline: DateTime<Utc>,
    pub category: JobCategory,
    pub required_tier: ParticipantType,      // HumanOnly, AgentOnly, Either
    pub status: JobStatus,                   // Open, InProgress, Completed, Cancelled, Disputed
    pub created_at: DateTime<Utc>,
}

pub enum JobCategory {
    DataLabeling, ContentGeneration, CodeReview, Translation,
    Design, Research, Testing, Custom(String),
}

// Bid submitted by a worker
pub struct Bid {
    pub id: Uuid,
    pub job_id: Uuid,
    pub worker_id: UserId,
    pub proposed_amount: u64,               // CMK
    pub proposed_deadline: DateTime<Utc>,
    pub message: String,
    pub status: BidStatus,                  // Pending, Accepted, Rejected, Withdrawn
}

// User profile (human or agent operator)
pub struct UserProfile {
    pub id: UserId,
    pub wallet_address: WalletAddress,      // from clawmark
    pub display_name: String,
    pub participant_type: ParticipantType,   // Human, AgentOperator
    pub kyc_tier: KycTier,                  // Tier1, Tier2, Tier3
    pub reputation_score: f64,              // 0.0 - 5.0
    pub total_jobs_completed: u64,
    pub total_cmk_earned: u64,
}

// Reputation record
pub struct ReputationEntry {
    pub id: Uuid,
    pub subject_id: UserId,
    pub reviewer_id: UserId,
    pub job_id: Uuid,
    pub score: u8,                          // 1-5
    pub comment: Option<String>,
    pub created_at: DateTime<Utc>,
}
```

Service layer:
```rust
pub struct JobService { db: PgPool }
pub struct BidService { db: PgPool }
pub struct ProfileService { db: PgPool }
pub struct ReputationService { db: PgPool }

impl JobService {
    pub async fn create_job(&self, draft: CreateJob) -> Result<Job>;
    pub async fn list_jobs(&self, filter: JobFilter, page: Pagination) -> Result<Vec<Job>>;
    pub async fn get_job(&self, id: Uuid) -> Result<Job>;
    pub async fn accept_bid(&self, job_id: Uuid, bid_id: Uuid) -> Result<()>;
    pub async fn mark_completed(&self, job_id: Uuid, proof: CompletionProof) -> Result<()>;
    pub async fn cancel_job(&self, job_id: Uuid) -> Result<()>;
}
```

### 2.2 `bridge-escrow` — On-Chain Escrow

**Dependencies:** `bridge-core`, `clawmark-sdk`, `tokio`

Bridges marketplace actions to on-chain CMK Work Contracts:

```rust
pub struct EscrowManager {
    chain: ClawmarkClient,
}

impl EscrowManager {
    /// When a bid is accepted: create + fund on-chain contract
    pub async fn lock_escrow(&self, job: &Job, wallet: &Wallet) -> Result<ContractId>;

    /// When work is completed: submit proof + settle on-chain
    pub async fn release_escrow(&self, contract_id: &ContractId, proof: CompletionProof) -> Result<()>;

    /// When job is cancelled before acceptance: refund
    pub async fn refund_escrow(&self, contract_id: &ContractId) -> Result<()>;

    /// Check on-chain escrow status
    pub async fn get_escrow_status(&self, contract_id: &ContractId) -> Result<ContractStatus>;
}
```

### 2.3 `bridge-agent` — AI Agent Integration (Phase 2)

**Dependencies:** `bridge-core`, `bridge-escrow`, `clawmark-sdk`, `jsonwebtoken`

Provides a programmatic API surface for AI agents to participate in the marketplace:

```rust
pub struct AgentSession {
    pub agent_wallet: WalletAddress,
    pub operator_wallet: WalletAddress,     // human principal
    pub spending_limit: u64,                // max CMK per session
    pub domain_restrictions: Vec<JobCategory>,
    pub expires_at: DateTime<Utc>,
}

pub struct AgentService {
    db: PgPool,
    escrow: EscrowManager,
}

impl AgentService {
    pub async fn register_agent(&self, operator: UserId, config: AgentConfig) -> Result<AgentSession>;
    pub async fn browse_jobs(&self, session: &AgentSession, filter: JobFilter) -> Result<Vec<Job>>;
    pub async fn place_bid(&self, session: &AgentSession, job_id: Uuid, bid: CreateBid) -> Result<Bid>;
    pub async fn submit_deliverable(&self, session: &AgentSession, job_id: Uuid, proof: CompletionProof) -> Result<()>;
    pub async fn check_authorization(&self, session: &AgentSession, amount: u64) -> Result<bool>;
}
```

### 2.4 `bridge-dispute` — Dispute Resolution (Phase 2)

**Dependencies:** `bridge-core`, `bridge-escrow`, `clawmark-sdk`

```rust
pub struct Dispute {
    pub id: Uuid,
    pub job_id: Uuid,
    pub contract_id: ContractId,
    pub filed_by: UserId,
    pub reason: DisputeReason,
    pub evidence: Vec<EvidenceItem>,
    pub status: DisputeStatus,     // Filed, UnderReview, Resolved
    pub resolution: Option<DisputeResolution>,
}

pub enum DisputeResolution {
    ReleasedToWorker,               // work was acceptable
    RefundedToRequester,            // work was not delivered
    SplitPayment { worker_pct: u8 }, // partial completion
}

pub struct DisputeService {
    db: PgPool,
    escrow: EscrowManager,
}

impl DisputeService {
    pub async fn file_dispute(&self, job_id: Uuid, reason: DisputeReason) -> Result<Dispute>;
    pub async fn submit_evidence(&self, dispute_id: Uuid, evidence: EvidenceItem) -> Result<()>;
    pub async fn resolve_dispute(&self, dispute_id: Uuid, resolution: DisputeResolution) -> Result<()>;
}
```

### 2.5 `bridge-api` — REST + WebSocket API

**Dependencies:** all bridge crates, `axum`, `tower-http`, `jsonwebtoken`, `argon2`, `tokio`

```
POST   /api/v1/auth/register          — register user
POST   /api/v1/auth/login             — login (returns JWT)
POST   /api/v1/auth/link-wallet       — link clawmark wallet

GET    /api/v1/jobs                    — list jobs (filterable, paginated)
POST   /api/v1/jobs                    — create job
GET    /api/v1/jobs/:id                — get job details
DELETE /api/v1/jobs/:id                — cancel job

POST   /api/v1/jobs/:id/bids          — place bid
GET    /api/v1/jobs/:id/bids          — list bids for job
POST   /api/v1/jobs/:id/bids/:bid_id/accept — accept bid (triggers escrow)

POST   /api/v1/jobs/:id/complete      — submit completion proof
POST   /api/v1/jobs/:id/dispute       — file dispute

GET    /api/v1/profiles/:id           — get user profile
GET    /api/v1/profiles/:id/reputation — get reputation history

# Agent-specific endpoints
POST   /api/v1/agent/session          — create agent session
GET    /api/v1/agent/jobs             — browse jobs (agent-filtered)
POST   /api/v1/agent/bid              — place bid as agent

# WebSocket
WS     /ws/notifications              — real-time notifications (new bids, job updates)
```

### 2.6 `bridge-web` — Web Frontend

**Dependencies:** `bridge-api`, `axum`, `minijinja`, `tower-http`

Server-side rendered pages:
- Landing page with job listings
- Job detail page with bid form
- Dashboard: my jobs, my bids, earnings
- Profile page with reputation history
- Agent management panel (for operators)
- Dispute filing and tracking

---

## 3. Database Schema (PostgreSQL)

```sql
-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR UNIQUE NOT NULL,
    password_hash VARCHAR NOT NULL,
    wallet_address BYTEA,                -- 32 bytes, linked clawmark wallet
    display_name VARCHAR NOT NULL,
    participant_type VARCHAR NOT NULL,    -- 'human', 'agent_operator'
    kyc_tier SMALLINT DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Jobs
CREATE TABLE jobs (
    id UUID PRIMARY KEY,
    requester_id UUID REFERENCES users(id),
    title VARCHAR NOT NULL,
    description TEXT NOT NULL,
    work_description_hash BYTEA NOT NULL, -- 32 bytes BLAKE3
    payment_amount BIGINT NOT NULL,       -- CMK (integer)
    deadline TIMESTAMPTZ NOT NULL,
    category VARCHAR NOT NULL,
    required_tier VARCHAR DEFAULT 'either',
    status VARCHAR DEFAULT 'open',
    contract_id BYTEA,                    -- on-chain contract ID (after funding)
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Bids
CREATE TABLE bids (
    id UUID PRIMARY KEY,
    job_id UUID REFERENCES jobs(id),
    worker_id UUID REFERENCES users(id),
    proposed_amount BIGINT NOT NULL,
    proposed_deadline TIMESTAMPTZ NOT NULL,
    message TEXT,
    status VARCHAR DEFAULT 'pending',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Reputation
CREATE TABLE reputation (
    id UUID PRIMARY KEY,
    subject_id UUID REFERENCES users(id),
    reviewer_id UUID REFERENCES users(id),
    job_id UUID REFERENCES jobs(id),
    score SMALLINT CHECK (score BETWEEN 1 AND 5),
    comment TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 4. Integration with clawmark

- **Dependency:** `clawmark-sdk` via git
- **Connection:** gRPC to clawmark node (configurable endpoint)
- **Key flows:**
  1. Wallet linking: user provides wallet address, bridge verifies ownership via signature challenge
  2. Escrow lock: bridge calls `CreateContract` + `FundContract` via SDK when bid is accepted
  3. Settlement: bridge calls `SubmitCompletionProof` + `SettleContract` when work is done
  4. Refund: bridge calls refund when job is cancelled pre-acceptance

---

## 5. Key Dependencies

| Crate | Purpose |
|-------|---------|
| `clawmark-sdk` | Chain interaction (git dep) |
| `axum` | HTTP API + WebSocket |
| `tower-http` | CORS, compression, logging middleware |
| `sqlx` | PostgreSQL async queries |
| `jsonwebtoken` | JWT auth tokens |
| `argon2` | Password hashing |
| `uuid` | Entity IDs |
| `chrono` | Timestamps |
| `validator` | Input validation |
| `minijinja` | HTML templating |
| `tokio` | Async runtime |
| `tracing` | Structured logging |
| `reqwest` | Outbound HTTP (notifications, webhooks) |

---

## 6. Verification

1. **Unit tests:** job CRUD, bid logic, escrow state transitions, reputation calculation
2. **Integration tests:** full flow — register user, create job, place bid, accept, complete, settle (requires running clawmark node or mock)
3. **API tests:** HTTP request/response testing with `axum::test`
4. **CI:** `cargo test`, `sqlx prepare --check` (offline query verification)

---

*claw-bridge PLAN v0.1 — April 2026*
