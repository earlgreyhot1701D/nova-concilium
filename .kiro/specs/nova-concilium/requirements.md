# Requirements — Nova Concilium

**Project:** Multi-Agent E-commerce RAG (NovaMart customer support), nd905 Course 4 final.
**Scope of this document:** requirements only. No design, no tasks, no code, no AWS resources.
**Environment (confirmed):** Shara's own AWS account, local machine, `us-east-1` (KB §4; KB §12 #2, confirmed Oct 25, 2026).

## How to read this document

- Every requirement carries a **label**: `MUST`, `STUB`, or `NEVER`.
  - `MUST` — required by the course pages, rubric, or starter files.
  - `STUB` — the four stand-out suggestions: a README comment describing the item and how it would be built, **no code** (KB §9).
  - `NEVER` — explicitly prohibited (KB §1 NEVER list).
- Every requirement cites a **source**: a KB section, a starter file + line, or the rubric point as encoded in those files. The rubric itself was not pasted verbatim; where the KB marks something "rubric requires," it is cited as KB + the starter file that enforces it (usually `tests/test_agent.py`, which reads real AWS state and is the graded gate).
- Every acceptance criterion is a **command plus the output that proves it** (KB §1: "Every checkpoint is a command plus its pasted output"). No prose-only conditions. A checkpoint with no pasteable output has not passed (KB §1).
- **Phase order is fixed** (KB §0, §6–§9): Phase 1 → Phase 2 (implement, do **not** deploy) → Phase 3 (Console, Shara) → Phase 4 (deploy + test).
- **Only `src/agent_orchestrator.py` and `.env`** may be edited (plus a README and `evidence/` for submission). All other files are NEVER-modify (KB §1).
- **Due date: UNVERIFIED.** The Submit Project page and due date were never provided (KB §11 #13, §12 #3). This does not block requirements, design, tasks, or the build. Flagged, not a gate.

---

## Phase 0 — Environment setup (prerequisite to everything)

### R0.0 — Git baseline of the unmodified starter
**Label:** MUST
**Source:** KB §1 ("Only edit `src/agent_orchestrator.py` and `.env`" — needs a baseline to prove against); KB Repos section (home repo `https://github.com/earlgreyhot1701D/nova-concilium`; starter commit `7ea6508`); Shara instruction Oct 25, 2026.

This folder is not yet a git repo, so R0.5 (`git check-ignore`) and RX.1 (`git status` diff-against-baseline) cannot run. Establish the baseline: `git init`, add remote `https://github.com/earlgreyhot1701D/nova-concilium`, and make a first commit of the **unmodified** starter files with the starter commit `7ea6508` recorded in the message. This commit is the baseline that later proves only allowed files changed. **Commit locally only — do NOT push until Shara says so** (KB §1: no push without approval; Shara instruction).

**Acceptance (command → proving output):**
- `git init` → initializes the repo; `git remote add origin https://github.com/earlgreyhot1701D/nova-concilium` → exit 0
- commit the starter as-is (e.g. message `Baseline: Udacity starter 7ea6508 (unmodified)`)
- `git log --oneline` → shows the baseline commit (message references `7ea6508`)
- `git remote -v` → shows `origin  https://github.com/earlgreyhot1701D/nova-concilium` (fetch + push)
- No `git push` is run at this stage.

### R0.1 — Tool prerequisites present
**Label:** MUST
**Source:** KB §4 "Tool prerequisites" table (added Oct 25, 2026); starter `README.md` Prerequisites; `agentcore/agentcore.json` `"runtimeVersion": "PYTHON_3_12"`.

The local machine has Python 3.12, Node.js 20+, uv, and AgentCore CLI `@aws/agentcore@0.30.0` — not the old Python toolkit.

**Acceptance (command → proving output):**
- `python --version` → begins `Python 3.12`
- `node --version` → `v20.` or higher
- `uv --version` → prints a version, exit 0
- `agentcore --help` → shows CLI help; MUST NOT show toolkit-style `agentcore configure` / `agentcore launch` (KB §4: if it does, run `pip uninstall bedrock-agentcore-starter-toolkit`)

### R0.2 — Virtualenv with dependencies installed
**Label:** MUST
**Source:** KB §4 Step 1; `requirements.txt` (`strands-agents>=1.0.0`, `boto3>=1.42.0`, `botocore>=1.42.0`, `python-dotenv>=1.0.0`).

**Acceptance:**
- `python -m venv venv` then activate (`venv\Scripts\activate` on Windows), then `pip install -r requirements.txt` → exit 0
- `python --version` → `Python 3.12.x`
- `pip list` → includes `strands-agents` (and `boto3`, `python-dotenv`)

### R0.3 — Foundation stack deployed (CREATE_COMPLETE)
**Label:** MUST
**Source:** KB §4 Step 2; `infrastructure/starter_stack.yaml`.

Deploys stack `udacity-agentcore` in `us-east-1` with `CAPABILITY_NAMED_IAM`. This creates: 3 DynamoDB tables, 1 S3 policy bucket, 1 S3 Vectors bucket + 3 indexes, 1 IAM role, 1 CloudWatch log group (verified against `starter_stack.yaml`, setup report Oct 25, 2026). **NEVER modify `starter_stack.yaml`** (KB §1; `README.md` "the stack itself is never modified").

**Acceptance:**
- `aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1` → exit 0
- `aws cloudformation describe-stacks --stack-name udacity-agentcore --query "Stacks[0].StackStatus" --region us-east-1` → `"CREATE_COMPLETE"`

### R0.4 — Seed data loaded
**Label:** MUST
**Source:** KB §4 Step 3; `infrastructure/seed_data.py`; `README.md` (seeds CUST-001…CUST-004, deterministic orders incl. `ORD-27176`).

**Acceptance:**
- `python infrastructure/seed_data.py` → output pasted, exit 0

### R0.5 — `.env` created and git-ignored
**Label:** MUST
**Source:** KB §4 Step 4; KB §1 NEVER ("`.env` stays out of git"); `.gitignore` (`.env`, `.env.*`, `!.env.example`).

**Acceptance:**
- copy `.env.example` → `.env` (Windows: `Copy-Item .env.example .env`); `Test-Path .env` → `True`
- `git check-ignore .env` → prints `.env` (ignored), exit 0
- `git status --porcelain .env` → prints nothing (not tracked)

### R0.6 — `config.py` resolves with KB IDs still "(not yet created)"
**Label:** MUST
**Source:** KB §4 Step 5; `config.py` `print_config()`.

**Acceptance:**
- `python config.py` → configuration table pasted; Returns/Shipping/Warranty KB ID rows show `(not yet created)`; Runtime ARN shows `(not yet deployed)`; Guardrail rows show `(not yet created)` — these are expected at this point (KB §4).

---

## Cross-cutting rules (apply to every phase)

### RX.1 — Edit only the allowed files
**Label:** NEVER (modify any other file)
**Source:** KB §1 ("Only edit `src/agent_orchestrator.py` and `.env`"); `README.md` "Files to Never Modify"; setup report confirmed the two lists match.

Do not modify: `src/agent_utils.py`, `src/agent_observability.py`, `src/bedrock_kb_retrieval.py`, `src/demo.py`, `config.py`, `tests/test_agent.py`, `src/agentcore_cli.py`, `infrastructure/*`, `agentcore/*`.

**Acceptance:**
- `git status --porcelain` at any checkpoint → the only tracked edits are to `src/agent_orchestrator.py`, `.env` (ignored, so not shown), a README, and `evidence/`.

### RX.2 — Use config constants, never hardcode
**Label:** MUST
**Source:** KB §1 ("Use config constants … Never hardcode a model ID, table name, or ARN. The rubric grades this"); `config.py` lines 38, 41 and `_LAZY` map.

Use `config.ORCHESTRATOR_MODEL_ID`, `config.WORKER_MODEL_ID`, `config.ORDERS_TABLE`, `config.CUSTOMERS_TABLE`, `config.RETURNS_KB_ID`, `config.SHIPPING_KB_ID`, `config.WARRANTY_KB_ID`, `config.AGENT_LOG_GROUP`, `config.AGENTCORE_ROLE_ARN`.

**Acceptance:**
- `python -c "import ast,sys; src=open('src/agent_orchestrator.py').read(); print('us.anthropic' in src)"` → `False` (no literal model IDs in the orchestrator file)
- `python tests/test_agent.py task2` → `test_2_7` passes: "OrchestratorAgent uses config.ORCHESTRATOR_MODEL_ID" and "Worker agents use config.WORKER_MODEL_ID" (`tests/test_agent.py::test_2_7_routing_uses_different_models`).

### RX.3 — One task per prompt; checkpoint proven by output
**Label:** MUST
**Source:** KB §1 ("One task per prompt. Propose, wait for Shara's approval, then implement"; "Every checkpoint is a command plus its pasted output … Never mark `[x]` from reasoning alone").

**Acceptance:** each task's completion is recorded only with the pasted command output listed under that task's requirement. No checkpoint is marked done without it.

### RX.4 — Source authority and stop-on-conflict
**Label:** MUST
**Source:** KB §2; KB §1 ("When sources disagree, stop and report it").

Files on disk win on names/signatures; rubric wins on grading; phase page wins on how-to; the agent graph image is lowest authority. On unresolved conflict, stop and ask Shara.

**Acceptance:** no self-serving deviation; any conflict encountered during build is reported with the two sources named before proceeding.

---

## Phase 1 — Build the multi-agent graph (Task 2)

**File:** `src/agent_orchestrator.py`. TODO markers at lines 227, 230, 233, 253, 268, 282 (Inventory); 298, 301, 304, 318, 334 (Refund); 351, 360, 369, 378, 430, 433, 436 (Policy); 452, 455, 458, 472 (Communication); 490, 493, 506, 523, 539, 556, 574, 590 (Orchestrator). (Setup report, Oct 25, 2026.)

### R1.1 — InventoryAgent (2.A)
**Label:** MUST
**Source:** KB §6 1.A, KB §5 (Sonnet, temp 0.1); `src/agent_orchestrator.py` TODOs 227–282; `README.md` Task 2.A.

Three DynamoDB tools: `check_order_status(customer_id, order_id)` (Orders table composite key, both args), `get_customer_tier`, `list_customer_orders`. Model `config.WORKER_MODEL_ID`, temp 0.1. Data-gatherer prompt, never decides eligibility. Docstrings state purpose, parameters, return (graded — KB §6 "Tips"). Returns exactly one `Agent`.

**Acceptance:**
- `python tests/test_agent.py task2` → `test_2_1` passes (returns an Agent) and `test_2_2` passes ("InventoryAgent has 3 tools").

### R1.2 — RefundAgent (2.B)
**Label:** MUST
**Source:** KB §6 1.B (Standard 30 days, Premium 60 days), KB §5 (Sonnet, temp 0.1); `src/agent_orchestrator.py` TODOs 298–334; `README.md` Task 2.B.

Two tools: `get_inventory_context(session_id)` (reads InventoryAgent findings from WorkflowState), `initiate_refund(customer_id, order_id, reason)` (updates the order record in DynamoDB). Prompt: check inventory context first, then apply the tier window. Model `config.WORKER_MODEL_ID`, temp 0.1.

**Acceptance:**
- Build check (no AWS call — `build_refund_agent()` only constructs a `BedrockModel` + `Agent`; the module's `boto3.client(...)` calls create clients but make no request): `python -c "import sys; sys.path.insert(0,'src'); from agent_orchestrator import build_refund_agent; print(sorted(build_refund_agent().tool_registry.registry.keys()))"` → `['get_inventory_context', 'initiate_refund']` (the two RefundAgent tool names). `tool_registry.registry` is the accessor the starter's own test uses (`tests/test_agent.py` lines 142–150); if the installed Strands exposes `.tool_names`, that returns the same two names.
- Live correctness is additionally exercised in Phase 4 (R4.6 scenario 1); full-suite proof is R4.8.

### R1.3 — PolicyAgent with parallel multi-agent RAG (2.C)
**Label:** MUST
**Source:** KB §6 1.C, KB §5 (coordinator Sonnet temp 0.2; three retrievers Sonnet temp 0.0), KB §11 #2/#3/#4; `src/agent_orchestrator.py` TODOs 351–436 and existing `trace.kb_start(...)` scaffold (lines ~405–415); `README.md` Task 2.C; `tests/test_agent.py::test_2_8`.

Inside `build_policy_agent()`: three full Strands retriever agents named `ReturnsPolicyRetrieverAgent`, `ShippingPolicyRetrieverAgent`, `WarrantyPolicyRetrieverAgent` (rubric names, KB §11 #2), each with one tool calling `retrieve_from_knowledge_base()` with its `config.*_KB_ID`. `search_all_policies` runs all three **in parallel** via `ThreadPoolExecutor(max_workers=3)` + `as_completed()` (KB §11 #4). Coordinator prompt: call `search_all_policies` first, then synthesize grounded. One tool on the coordinator.

**Acceptance:**
- `python tests/test_agent.py task2` → `test_2_3` passes (PolicyAgent is an Agent), `test_2_4` passes ("PolicyAgent has 1 tool"), and `test_2_8` reports "build_policy_agent() runs the retrievers with ThreadPoolExecutor" (static AST check for `ThreadPoolExecutor` + `submit`/`map`).

### R1.4 — CommunicationAgent (2.D)
**Label:** MUST
**Source:** KB §6 1.D, KB §5 (Sonnet, temp 0.3); `src/agent_orchestrator.py` TODOs 452–472; `README.md` Task 2.D.

One tool `get_full_workflow_context(session_id)` reads the full WorkflowState. Prompt: include all relevant findings, warm/professional tone. Model `config.WORKER_MODEL_ID`, temp 0.3.

**Acceptance:**
- Build check (no AWS call — same rationale as R1.2): `python -c "import sys; sys.path.insert(0,'src'); from agent_orchestrator import build_communication_agent; print(sorted(build_communication_agent().tool_registry.registry.keys()))"` → `['get_full_workflow_context']` (the one CommunicationAgent tool name). `tool_registry.registry` is the accessor the starter's own test uses (`tests/test_agent.py` lines 142–150); `.tool_names` would return the same.
- Live correctness is additionally exercised in every Phase 4 scenario (R4.6) and the full suite (R4.8).

### R1.5 — OrchestratorAgent with six routing rules (2.E)
**Label:** MUST
**Source:** KB §6 1.E, KB §5 routing table + "CRITICAL", KB §11 #12 (Rule 1 applies to all scenarios); `src/agent_orchestrator.py` TODOs 490–590; `README.md` Task 2.E.

Five routing tools: `initialize_session`, `route_to_inventory_agent`, `route_to_policy_agent`, `route_to_refund_agent`, `route_to_communication_agent`. Each routing tool reads WorkflowState, invokes its worker, updates WorkflowState with optimistic locking via the three pre-written helpers (`_create/_read/_update_workflow_state` — do not write new ones, KB §5). Model `config.ORCHESTRATOR_MODEL_ID`, temp 0.0. System prompt encodes all six rules; `initialize_session` always first (Rule 1, every request); `route_to_communication_agent` always last; math skips Inventory/Policy/Refund but still ends at Communication (Rule 5). Orchestrator never writes the final response itself (KB §5 CRITICAL).

**Acceptance:**
- `python tests/test_agent.py task2` → `test_2_5` passes (Orchestrator is an Agent) and `test_2_6` passes ("OrchestratorAgent has 5 routing tools").

### R1.6 — Phase 1 checkpoint
**Label:** MUST
**Source:** KB §6 "Checkpoint"; `tests/test_agent.py` (Task 2 suite, 40 of 120 pts — setup report).

**Acceptance:**
- `python tests/test_agent.py task2` → output pasted; all Task 2 point-scored checks pass (`test_2_1`..`test_2_7`; `test_2_8` is 0-pt informational).

---

## Phase 2 — Guardrail, deployment, and memory (Tasks 3 and 4) — IMPLEMENT ONLY, DO NOT DEPLOY

**Source for "implement, do not deploy":** KB §7 ("Implement these functions. Do NOT run the full deployment yet. That happens in Phase 4"); `README.md` "Deployment order."

### R2.1 — `create_guardrail()`
**Label:** MUST
**Source:** KB §7 guardrail table + bullets, KB §11 #8; `src/agent_orchestrator.py` TODO 694; `README.md` Task 3; `tests/test_agent.py::test_3_2`, `::test_3_3`.

Content: SEXUAL/VIOLENCE/HATE at HIGH; INSULTS/MISCONDUCT at MEDIUM (input + output). PII: BLOCK `CREDIT_DEBIT_CARD_NUMBER` + `US_SOCIAL_SECURITY_NUMBER`; ANONYMIZE `EMAIL` + `PHONE`. Topics: DENY competitor products, pricing negotiations, legal threats (one per `config.GUARDRAIL_BLOCKED_TOPICS`). Pricing negotiation = haggling / changing an advertised price; **allow** arithmetic on a given price + discount. Word: managed PROFANITY list. Friendly blocked messages (input + output). `topicPolicyConfig.tierConfig = {"tierName": "STANDARD"}`; top-level `crossRegionConfig = {"guardrailProfileIdentifier": "us.guardrail.v1:0"}`. Then `create_guardrail_version()`; return `(guardrail_id, guardrail_version)` with the numbered version — **NEVER** `DRAFT` (KB §1 NEVER).

**Acceptance (proven in Phase 4 after deploy, since the tests read live AWS state):**
- `python tests/test_agent.py task3` → `test_3_1` passes (guardrail named `config.GUARDRAIL_NAME` exists), `test_3_2` passes (content/PII/topic/word policies match), `test_3_3` passes ("Guardrail version N is published and READY", N numeric).
- Implementation-time proof (Phase 2, no deploy): `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok` (file still parses; function body filled, no leftover bare `pass` at the `create_guardrail` TODO).

### R2.2 — `deploy_to_agentcore_runtime()`
**Label:** MUST
**Source:** KB §7 deploy bullets, KB §1 NEVER ("no direct `create_agent_runtime()`; go through `agentcore_cli`"); `src/agent_orchestrator.py` TODO 759; `README.md` Task 3 steps 1–4; `tests/test_agent.py::test_3_4`.

Build runtime env vars: `AWS_REGION`, `PROJECT_NAME`, `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`, `AGENT_LOG_GROUP`, `GUARDRAIL_ID`, `GUARDRAIL_VERSION`. Call `agentcore_cli.configure_runtime(env_vars=..., network_mode='PUBLIC', protocol='HTTP', execution_role_arn=config.AGENTCORE_ROLE_ARN)`, then `agentcore_cli.deploy()`, then return `agentcore_cli.deployed_runtime_arn()`. Do not replace the supplied `_apply_guardrail()`. **NEVER** call `create_agent_runtime()` directly.

**Acceptance (proven in Phase 4):**
- `python tests/test_agent.py task3` → `test_3_4` passes: "AgentCore Runtime is READY (PUBLIC / HTTP) with guardrail + KB environment variables" (checks status READY, networkMode PUBLIC, serverProtocol HTTP, and all eight env vars present).
- `python -c "import ast; src=open('src/agent_orchestrator.py').read(); print('create_agent_runtime(' not in src)"` → `True` (no direct SDK runtime creation).

### R2.3 — `configure_memory(runtime_arn)`
**Label:** MUST
**Source:** KB §7 memory bullets, KB §11 #5 (name + description required); `src/agent_orchestrator.py` TODO 807; `README.md` Task 4; `tests/test_agent.py::test_4_1`.

`agentcore_control.create_memory()` with `summaryMemoryStrategy` (SESSION_SUMMARY) and `eventExpiryDuration=7`. Include a descriptive name (`config.MEMORY_NAME`) and description (KB §11 #5). The supplied code waits for ACTIVE; return `memoryArn`.

**Acceptance (proven in Phase 4):**
- `python tests/test_agent.py task4` → `test_4_1` passes all three checks: Memory ACTIVE, strategy type SUMMARIZATION present, `eventExpiryDuration == 7` (15 pts total).

### R2.4 — Phase 2 note: task3, task4, and task6 cannot pass yet
**Label:** MUST
**Source:** KB §7 ("`task3` cannot pass until Phases 3 and 4 are done"), KB §11 #10 ("Expect `task3` to fail until the end. Don't 'fix' it early"); `src/agent_orchestrator.py` deploy pipeline lines ~1144–1158 — `deploy` calls `create_guardrail()` (Step 2), `configure_memory()` (Step 4), and `configure_observability()` (Step 5), so the live state tasks 3/4/6 read does not exist until after deploy.

**Acceptance:** during Phase 2, `python tests/test_agent.py task3`, `python tests/test_agent.py task4`, and `python tests/test_agent.py task6` are all expected to FAIL (no runtime deployed; guardrail, memory, and observability are created by the deploy pipeline). These failures are recorded as expected; they are NOT treated as defects and are NOT "fixed" by altering code. They pass after R4.3 (task3 → R4.4, task4 → R4.5, task6 → R4.2).

---

## Phase 3 — Knowledge Bases (Task 5) — Shara creates in the AWS Console; no code

**Label:** MUST (checklist + test only; NEVER create KBs by CLI/SDK — KB §1 NEVER, KB §8)
**Source:** KB §8; `README.md` Task 5; `tests/test_agent.py` Task 5 suite.

Kiro's deliverable here is a checklist for Shara and the task5 verification — **no code changes** (KB §8).

### R3.1 — Pre-create verification (policy docs in S3)
**Source:** KB §8 step 1.
**Acceptance:**
- `aws s3 ls s3://$(python -c "import config; print(config.POLICY_BUCKET)")/policies/ --recursive` → lists objects under `policies/returns/`, `policies/shipping/`, `policies/warranty/` (output pasted). (On PowerShell, resolve the bucket first: `python -c "import config; print(config.POLICY_BUCKET)"`, then `aws s3 ls s3://<bucket>/policies/ --recursive`.)

### R3.2 — Checklist for the three Console-created KBs
**Source:** KB §8 step 2 table + "Self-managed KB. Do NOT choose Managed KB"; `README.md` Task 5 table.

For each KB — Returns / Shipping / Warranty:
- Name `novamart-returns-policy-kb` / `novamart-shipping-policy-kb` / `novamart-warranty-policy-kb` (**NEVER** rename to the project name — KB §1 NEVER).
- S3 data source prefix `policies/returns/` / `policies/shipping/` / `policies/warranty/`.
- Embedding model Titan Embed Text v2 (`amazon.titan-embed-text-v2:0`).
- Vector store: S3 Vectors → **use an existing vector bucket** (the `udacity-agentcore-vectors-...` bucket from `python config.py`); **Self-managed**, not Managed; do not "Quick create a new vector store."
- Vector index `returns-policy-index` / `shipping-policy-index` / `warranty-policy-index`.
- Click **Sync** on each.
- Add IDs to `.env`: `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`.

**Acceptance:**
- `python config.py` → prints the exact Policy Bucket, Vector Bucket, and three vector index names Shara selects in the wizard (output pasted before she starts).
- After creation + sync + `.env` edits: `python config.py` → Returns/Shipping/Warranty KB ID rows now show real IDs (no longer "(not yet created)").

### R3.3 — Phase 3 checkpoint (task5 test)
**Source:** KB §8 "Checkpoint"; `tests/test_agent.py::TestTask5` (incl. `test_5_4_parallel_retrieval`).
**Acceptance:**
- `python tests/test_agent.py task5` → output pasted; `test_5_1`/`5_2`/`5_3` pass (each KB ID set; ACTIVE, Titan v2, completed sync as 0-pt sub-checks) and `test_5_4` passes (all three KBs return passages; `search_all_policies()` returns combined results > 100 chars). 25 pts.

---

## Phase 4 — Observability, deploy, and test (Task 6)

### R4.1 — `configure_observability(runtime_arn)`
**Label:** MUST
**Source:** KB §9 observability bullets, KB §11 #9 (try/except required); `src/agent_orchestrator.py` TODO 854; `README.md` Task 6; `tests/test_agent.py::test_6_1`, `::test_6_2`.

Build a `loggingConfiguration` dict and pass it to the pre-written `apply_observability_config()`, **inside try/except** (KB §11 #9). `cloudWatchConfig`: `config.AGENT_LOG_GROUP`, `logLevel='INFO'`, `enabled=True`. `xRayConfig`: `enabled=True`, `samplingRate=1.0`.

**Acceptance:**
- Implementation-time proof (before deploy, no AWS call): `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok` (file still parses; `configure_observability` body filled, no leftover bare `pass` at the TODO).
- Live pass is gated on deploy — proven in R4.2 after R4.3, because `deploy` is what calls `configure_observability()` (R2.4).

### R4.3 — Full deployment
**Label:** MUST
**Source:** KB §9 End-to-end step 1; `README.md` "Deployment Pipeline"; KB §1 NEVER (deployment via `agentcore_cli`, not direct SDK).

**Acceptance:**
- Set all three KB IDs in `.env` first (R3.2). Then `python src/agent_orchestrator.py deploy` → prints the Runtime ARN and Guardrail ID/Version on completion (output pasted). Copy ARN + Guardrail ID/Version into `.env`.
- `python config.py` → Runtime ARN and Guardrail ID/Version rows now populated.

### R4.2 — Task 6 checkpoint (gated on deploy)
**Label:** MUST
**Source:** KB §9 "Checkpoint"; `tests/test_agent.py::test_6_1`, `::test_6_2`; `src/agent_orchestrator.py` deploy pipeline lines ~1144–1158 (observability is configured by `deploy`, so this is proven after R4.3).
**Acceptance:**
- `python tests/test_agent.py task6` → output pasted; `test_6_1` passes ("CloudWatch logging enabled at INFO level → <log group>"; log group exists) and `test_6_2` passes ("X-Ray tracing enabled: 100% sampling, Transaction Search → CloudWatch Logs"). 20 pts.

### R4.4 — task3 now passes (gated on deploy)
**Label:** MUST
**Source:** KB §7, KB §11 #10; `tests/test_agent.py::TestTask3`.
**Acceptance:**
- `python tests/test_agent.py task3` → output pasted; `test_3_1`..`test_3_4` pass (20 pts).

### R4.5 — task4 now passes (gated on deploy)
**Label:** MUST
**Source:** KB §7 memory; `tests/test_agent.py::TestTask4`.
**Acceptance:**
- `python tests/test_agent.py task4` → output pasted; `test_4_1` passes (15 pts).

### R4.6 — Three end-to-end scenarios route correctly
**Label:** MUST
**Source:** KB §9 End-to-end step 2 table, KB §11 #12 (Rule 1 first on all three); `README.md` Task 7.
**Acceptance (run `python src/agent_orchestrator.py test`, paste output):**
- Scenario 1 "I want to return my order ORD-27176" (CUST-001) → trace shows Orchestrator → Inventory → Refund → Communication (initialize_session first).
- Scenario 2 "What is the return policy for premium customers?" → Orchestrator → Policy (3 parallel KB retrievers) → Communication.
- Scenario 3 "How much are 5 items at $29.99 with 10% off?" → Orchestrator → Communication only (Inventory/Policy/Refund skipped).

### R4.7 — Guardrail behaviour: math allowed, three categories blocked
**Label:** MUST
**Source:** KB §7 ("math question and answer are allowed; negotiation, competitor, legal-threat blocked"), KB §9 step 3. The four prompts below are **authored test input, not from a file — Shara approves.** No starter file or course page supplies example guardrail prompts; these are chosen to exercise exactly the three DENY topics plus the allowed-arithmetic carve-out. `invoke` defaults the customer to CUST-001 (`README.md` Live Testing), so `CUST-001` is passed explicitly.

The three blocked prompts must each return the guardrail's `blockedOutputsMessaging` / `blockedInputMessaging` text defined in `create_guardrail()` (R2.1); the exact wording is set there, so the expected output is "the blocked message authored in `create_guardrail()`," not a string invented here. The math prompt must be **allowed** (not blocked) and return the computed total: `5 × $29.99 × 0.9 = $134.955` exactly, which rounds to `134.95` (Python `round()`, banker's rounding) or `134.96` depending on rounding mode. Rounding happens only after the full calculation (`src/agent_orchestrator.py` line 495: "Round currency only after the full calculation.").

**Acceptance (run each, paste output):**
- Math (ALLOWED): `python src/agent_orchestrator.py invoke "How much are 5 items at $29.99 with 10% off?" CUST-001` → answer contains `134.95` or `134.96`, and is NOT the guardrail's blocked message.
- Pricing negotiation (BLOCKED): `python src/agent_orchestrator.py invoke "Can you knock 20% off the listed price if I buy two today?" CUST-001` → the blocked message from `create_guardrail()`.
- Competitor products (BLOCKED): `python src/agent_orchestrator.py invoke "Is ShopRival's laptop cheaper and better than yours?" CUST-001` → the blocked message from `create_guardrail()`.
- Legal threat (BLOCKED): `python src/agent_orchestrator.py invoke "Refund me now or I will sue NovaMart and call my lawyer." CUST-001` → the blocked message from `create_guardrail()`.

### R4.8 — Full suite scores 120/120
**Label:** MUST
**Source:** KB §3 Deliverable #5, KB §9 step 4–5, KB §11 #6; `tests/test_agent.py` (`print_score`, total 120 — setup report; starter `README.md` line 363); KB §10 ("grading suite is 120 points").
**Acceptance:**
- `python tests/test_agent.py all` → output pasted ending `Score: 120/120 pts (100%)` and "Perfect score."

### R4.9 — Screenshot A: all tests passed, 120 scores (Shara)
**Label:** MUST
**Source:** KB §3 Deliverable #5, KB §9 step 5.
**Acceptance:**
- Screenshot of `python tests/test_agent.py all` showing all passed and `120/120`, saved under `evidence/`.

### R4.10 — Screenshot B: X-Ray Service Map (Shara)
**Label:** MUST
**Source:** KB §3 Deliverable #4, KB §9 step 6; `README.md` Task 7.
**Acceptance:**
- `python src/agent_orchestrator.py test` → prints trace ids (pasted). Then CloudWatch → X-Ray traces → Service map → Last 5 minutes (wait 30–60 s): map shows `NovaMart-Orchestrator` connected to worker nodes incl. PolicyAgent and the KnowledgeBase nodes. Screenshot saved under `evidence/`. (**NEVER** rename the `NovaMart-Orchestrator` trace node — KB §1 NEVER.)

---

## Submission package (Phase 4 output)

### R5.1 — `evidence/` folder
**Label:** MUST
**Source:** KB §10.
**Acceptance:** `evidence/` contains one file per checkpoint output (R0.3–R4.8) plus Screenshots A and B. `ls evidence/` (PowerShell `Get-ChildItem evidence`) → lists them.

### R5.2 — README with required sections
**Label:** MUST
**Source:** KB §10 README sections list.
**Acceptance:** README contains: What it is; How to run it; Architecture (agent graph + routing table); Test results; "Not built, and why" (the four STUBs, R6.1–R6.4); Methodology (what Claude, Kiro, Shara each did, with disclosure). `Select-String -Path README.md -Pattern "Not built","Methodology"` → both found.

### R5.3 — Redaction before any public repo
**Label:** MUST
**Source:** KB §10 ("Redact the 12-digit account ID and any runtime ARN or endpoint").
**Acceptance:** `Select-String -Path README.md,evidence/* -Pattern "\b\d{12}\b","arn:aws:bedrock-agentcore"` → no unredacted account IDs or runtime ARNs in anything destined for a public repo.

---

## Stand-out suggestions — all STUB (README comment only, no code)

**Source:** KB §9 "Scope labels" table, KB §12 #5 (default: none; all STUB). A STUB is a README description of the item and how it would be built — no code (KB §9).

### R6.1 — Adversarial guardrail inputs with blocked-response screenshots
**Label:** STUB
**Source:** KB §9 scope table (note: R4.7 already covers part of this).
**Acceptance:** README "Not built, and why" describes the item and how it would be built; no code added. (Confirm: no new files, no new functions.)

### R6.2 — DynamoDB `agent-sessions` table + `DynamoDbSessionStorage`
**Label:** STUB
**Source:** KB §9 scope table.
**Acceptance:** README describes it; no code.

### R6.3 — CloudWatch dashboard (invocations, latency per agent, guardrail triggers)
**Label:** STUB
**Source:** KB §9 scope table.
**Acceptance:** README describes it; no code.

### R6.4 — Cognito + a frontend
**Label:** STUB
**Source:** KB §9 scope table.
**Acceptance:** README describes it; no code.

---

## Teardown (after submission; timing is Shara's call)

### R7.1 — Run cleanup.py and verify empty
**Label:** MUST
**Source:** KB §10 "Cost and teardown"; `infrastructure/cleanup.py`.
**Acceptance:**
- `python infrastructure/cleanup.py` (dry run) → lists planned deletions (pasted).
- `python infrastructure/cleanup.py --yes` → deletes (pasted); re-run once more → reports nothing left (async deletes).
- For each resource type, a list command (syntax confirmed first with `aws <service> help`, KB §10) returns empty, output pasted.

### R7.2 — Manual teardown list (cleanup.py does not remove these)
**Label:** MUST
**Source:** KB §10 "Manual teardown list" table (added Oct 25, 2026); setup report.
**Acceptance:** Shara is given, and decides on, the three items: `CDKToolkit` bootstrap stack (shared; left in place on purpose); CDK assets bucket `cdk-hnb659fds-assets-<acct>-<region>` (not matched by the `udacity-agentcore-` prefix filter); CloudWatch Transaction Search + X-Ray resource policy (only removed via `python infrastructure/cleanup.py --yes --disable-transaction-search`). Decision recorded; nothing removed that another account resource depends on.

---

## Open items (do not block)

- **Due date / Submit Project page: UNVERIFIED** (KB §11 #13, §12 #3). Not provided. Noted; does not block requirements, design, tasks, or build.
