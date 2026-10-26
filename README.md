# Nova Concilium — NovaMart Multi-Agent Customer Support

> **Status:** submission README for the nd905 Course 4 final project — complete,
> with the full 120/120 test run and both X-Ray screenshots recorded below. This
> file is the submission README; the original Udacity starter README is preserved
> in the git baseline commit `7ea6508` (unmodified starter). Account ID and
> runtime ARN are intentionally omitted (redacted for public sharing).

Nova Concilium (Latin for "council") is a five-agent customer-support system for
the fictional e-commerce company **NovaMart**, built on **Amazon Bedrock
AgentCore** and the **Strands Agents SDK**. An Orchestrator agent routes each
request to specialist worker agents, retrieves grounded policy answers with
parallel multi-agent RAG across three Knowledge Bases, enforces a Bedrock
Guardrail, keeps shared state in DynamoDB with optimistic locking, and is
observable end-to-end through CloudWatch and X-Ray.

---

## What it is

NovaMart's support team handles order status, returns and refunds, and policy
questions by hand. Nova Concilium automates that work with an Orchestrator →
Workers hierarchy:

| Agent | Job |
|---|---|
| OrchestratorAgent | Understands intent, routes to the right specialist, manages shared WorkflowState |
| InventoryAgent | Retrieves order and customer data from DynamoDB; reports facts only, no decisions |
| PolicyAgent | Answers policy questions with parallel multi-agent RAG across three Knowledge Bases |
| RefundAgent | Makes return-eligibility decisions from order data and the tier policy window |
| CommunicationAgent | Composes the final, customer-facing response |

PolicyAgent coordinates three retriever sub-agents — ReturnsPolicyRetrieverAgent,
ShippingPolicyRetrieverAgent, WarrantyPolicyRetrieverAgent — each bound to its own
Knowledge Base, run in parallel.

Deployed to AgentCore Runtime with Bedrock Guardrails, AgentCore Memory for
multi-turn sessions, and CloudWatch + X-Ray observability.

---

## How to run it

Prerequisites (local machine, `us-east-1`): Python 3.12, Node.js 20+, uv, the
AgentCore CLI (`npm install -g @aws/agentcore@0.30.0`), and AWS credentials with
Bedrock / AgentCore / DynamoDB / S3 / S3 Vectors / CloudFormation / CloudWatch /
X-Ray access.

```bash
# 1. Environment
python -m venv venv
venv\Scripts\activate          # Windows
pip install -r requirements.txt

# 2. Foundation infrastructure (DynamoDB, S3, S3 Vectors, IAM, log group)
aws cloudformation deploy --template-file infrastructure/starter_stack.yaml \
  --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1
python infrastructure/seed_data.py

# 3. Configuration
copy .env.example .env         # Windows (cp on macOS/Linux)
python config.py               # prints resolved resource names

# 4. Knowledge Bases (AWS Console — see below), then paste the IDs into .env

# 5. Deploy the multi-agent runtime (guardrail, runtime, memory, observability)
python src/agent_orchestrator.py deploy

# 6. Local run / interactive chat
python src/agent_orchestrator.py test
python src/agent_orchestrator.py chat
```

**Knowledge Bases (created in the AWS Console):** three self-managed Bedrock
Knowledge Bases on the existing S3 Vectors store, embedding model
`amazon.titan-embed-text-v2:0`:

| KB | S3 prefix | Vector index |
|---|---|---|
| `novamart-returns-policy-kb` | `policies/returns/` | `returns-policy-index` |
| `novamart-shipping-policy-kb` | `policies/shipping/` | `shipping-policy-index` |
| `novamart-warranty-policy-kb` | `policies/warranty/` | `warranty-policy-index` |

Sync each data source, then set `RETURNS_KB_ID`, `SHIPPING_KB_ID`,
`WARRANTY_KB_ID` in `.env`.

Tests:

```bash
python tests/test_agent.py task2   # multi-agent graph
python tests/test_agent.py task5   # Knowledge Base retrieval
python tests/test_agent.py all     # full suite (120 points)
```

---

## Architecture

```
Customer Request
      │
OrchestratorAgent (Claude Haiku 4.5 — routes, manages WorkflowState)
      │
  ┌───┼───────────────────────────┬────────────────────┐
  │   │                           │                    │
InventoryAgent   PolicyAgent    RefundAgent    CommunicationAgent
(DynamoDB)    (multi-agent RAG)  (DynamoDB)     (final response)
                    │
        ┌───────────┼───────────┐
 ReturnsRetriever  ShippingRetriever  WarrantyRetriever
   (Returns KB)      (Shipping KB)      (Warranty KB)
        └────────── run in PARALLEL ──────────┘

Shared state: DynamoDB WorkflowStateTable (optimistic locking by version)
```

**Models:** OrchestratorAgent uses `config.ORCHESTRATOR_MODEL_ID` (Claude Haiku
4.5, temperature 0.0); all worker and retriever agents use
`config.WORKER_MODEL_ID` (Claude Sonnet 4.5), temperatures 0.0–0.3 per role.

**Orchestrator routing rules (in the system prompt):**

| Rule | Trigger | Action |
|---|---|---|
| 1 | Every request, always | `initialize_session` first |
| 2 | Order status / return / refund | InventoryAgent, then RefundAgent |
| 3 | Policy meaning (windows, rates, terms) | PolicyAgent |
| 4 | Account questions ("my tier?", "am I premium?") | InventoryAgent — never PolicyAgent |
| 5 | Math / calculation | Compute directly; skip Inventory, Policy, Refund |
| 6 | Every request, always, last | CommunicationAgent composes the reply |

The Orchestrator never writes the customer response itself —
`route_to_communication_agent` is always its last action. RefundAgent applies the
tier window: Standard = 30 days, Premium = 60 days.

---

## Test results

`python tests/test_agent.py all` → **120/120 pts (100%)** — "Perfect score! All
tasks complete." Full breakdown:

| Task | Area | Score |
|---|---|---|
| Task 2 | Multi-agent graph | 40/40 |
| Task 3 | Guardrail + runtime | 20/20 |
| Task 4 | AgentCore Memory | 15/15 |
| Task 5 | Knowledge Bases (parallel RAG) | 25/25 |
| Task 6 | Observability (CloudWatch + X-Ray) | 20/20 |
| **Total** | | **120/120** |

**Screenshot A — all tests passed, 120/120:**
[`evidence/screenshot-a-tests-120.png`](evidence/screenshot-a-tests-120.png) —
the test suite output showing `Score: 120/120 pts (100%)` and "Perfect score!
All tasks complete."

**Screenshot B — X-Ray Service Map** (the full Orchestrator → Worker →
KnowledgeBase call chain), captured from CloudWatch → X-Ray traces → Service map,
us-east-1, Last 5 minutes:
- [`evidence/screenshot-b-xray-service-map.png`](evidence/screenshot-b-xray-service-map.png) —
  full graph: `Client → NovaMart-Orchestrator` fanning out to RefundAgent,
  PolicyAgent, InventoryAgent, CommunicationAgent, and the three KnowledgeBase
  nodes (returns / shipping / warranty).
- [`evidence/screenshot-b-detail-1.png`](evidence/screenshot-b-detail-1.png) and
  [`evidence/screenshot-b-detail-2.png`](evidence/screenshot-b-detail-2.png) —
  zoomed views with readable node labels.

**End-to-end scenarios** (`python src/agent_orchestrator.py test`) — all routed
exactly as specified, `initialize_session` first and `route_to_communication_agent`
last in every case:

1. "I want to return my order ORD-27176" (CUST-001) → Orchestrator → **Inventory →
   Refund → Communication**. Return approved under the Premium 60-day window;
   refund reference issued.
2. "What is the return policy for premium customers?" → Orchestrator → **Policy
   (3 retrievers in parallel) → Communication**. The trace shows ReturnsRetriever,
   ShippingRetriever, and WarrantyRetriever dispatched concurrently via
   ThreadPoolExecutor.
3. "How much are 5 items at $29.99 with 10% off?" → Orchestrator →
   **Communication only** (Inventory, Policy, Refund skipped); computed total
   $134.95–$134.96.

**Guardrail checks** (`python src/agent_orchestrator.py invoke "<prompt>" CUST-001`):

| Prompt | Expected | Result |
|---|---|---|
| "How much are 5 items at $29.99 with 10% off?" | allowed | **Allowed** — answered $134.95 |
| "Can you knock 20% off the listed price if I buy two today?" | blocked | **Blocked** (guardrail message) |
| "Is ShopRival's laptop cheaper and better than yours?" | blocked | **Blocked** (guardrail message) |
| "Refund me now or I will sue NovaMart and call my lawyer." | blocked | **Blocked** (guardrail message) |

The three blocked prompts each returned the configured message: "I'm sorry, but I
can't help with that request. I can assist with your NovaMart orders, returns,
shipping, and warranty questions." PII redaction was also observed live — a
customer email was anonymized to `{EMAIL}` in scenario 1.

---

## Not built, and why

These stand-out extensions were scoped as **STUB** — described here, not built —
because the project default is to ship none of them unless explicitly turned on.
Each note says what it is and how it would be built.

### 1. Adversarial guardrail inputs with screenshots of blocked responses
**What:** a curated battery of adversarial prompts (prompt injection, PII
exfiltration attempts, disallowed topics) exercised against the deployed runtime,
with screenshots of each blocked response as evidence.
**How it would be built:** add a small prompt set and drive it through
`python src/agent_orchestrator.py invoke`, capturing the guardrail's blocked
input/output messages; save the screenshots under `evidence/`. The required
guardrail checks above already cover the math-allowed plus negotiation/competitor/
legal-threat-blocked cases, so this item is the broader, exploratory superset.

### 2. DynamoDB `agent-sessions` table plus `DynamoDbSessionStorage`
**What:** durable, cross-process session storage so conversations survive beyond a
single runtime instance, in addition to AgentCore Memory.
**How it would be built:** add an `agent-sessions` DynamoDB table and a
`DynamoDbSessionStorage` adapter that reads/writes session turns keyed by
`session_id`, wiring it into the agents' session lifecycle. WorkflowState already
demonstrates the optimistic-locking write pattern this would reuse.

### 3. CloudWatch dashboard (invocations, latency per agent, guardrail triggers)
**What:** an operational dashboard showing per-agent invocation counts, latency,
and guardrail-trigger counts.
**How it would be built:** define a CloudWatch dashboard (metric filters over the
agent log group plus X-Ray-derived latency) with one widget per worker agent and
a guardrail-trigger widget. The observability layer already emits the logs and
traces these widgets would read.

### 4. Cognito plus a frontend
**What:** authenticated web UI in front of the AgentCore Runtime.
**How it would be built:** add an Amazon Cognito user pool for auth and a small
web frontend that calls the runtime's HTTP endpoint with the user's token. The
runtime is already deployed as `PUBLIC`/`HTTP`, so the frontend would layer auth
and a chat UI on top without changing the agent graph.

---

## Methodology: AI assisted, human approved

This project was built with AI assistance under explicit human direction and
disclosure, the same pattern used in Courses 2 and 3. Permission to use AI was
granted; the division of labor is disclosed here.

- **Claude (chat):** helped reason through the course requirements, draft the
  knowledge base, and shape the plan. Did not have direct access to the
  repository.
- **Kiro (agentic IDE):** executed the spec-driven workflow — wrote the
  requirements, design, and tasks; implemented the TODOs in
  `src/agent_orchestrator.py`; ran every checkpoint command and pasted its
  output. Kiro edited only `src/agent_orchestrator.py` and `.env` (plus this
  README and `evidence/`), per the project rules.
- **La Shara Cordero (owner):** directed the work, validated each phase, created
  the three Knowledge Bases in the AWS Console, authorized the first deploy
  (where AWS billing starts), captured the two screenshots, and approved every
  stage before the next began. All design decisions and approvals were hers.

Every graded checkpoint is a command plus its pasted output; nothing was marked
complete from reasoning alone.

---

## Cost and teardown

This project uses **S3 Vectors** (not OpenSearch Serverless) as the Knowledge
Base backing store. After submission, tear everything down:

```bash
python infrastructure/cleanup.py          # dry run — lists what would be deleted
python infrastructure/cleanup.py --yes     # delete
```

`cleanup.py` removes the three Knowledge Bases (and their console-created roles),
the AgentCore runtime stack, Memory, the Guardrail, the policy-bucket contents,
and the CloudFormation stack. It does **not** remove: the shared `CDKToolkit`
bootstrap stack, the CDK assets bucket (`cdk-hnb659fds-assets-<acct>-<region>`),
or CloudWatch Transaction Search / the X-Ray resource policy (use
`--disable-transaction-search` for the last). Teardown is verified when every
resource-type list command returns empty.

Before anything goes to a public repo, redact the 12-digit account ID and any
runtime ARN or endpoint.
