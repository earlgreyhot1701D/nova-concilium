# Tasks — Nova Concilium

**Rules in force:** one task per requirement (or small group); each task names its requirement IDs, the exact files it touches, and a done condition that is a command plus the output from that requirement. A task is done only when that output is pasted (KB §1). Tasks point at **function names and TODO text**, never line numbers. Only `src/agent_orchestrator.py`, `.env`, `README.md`, and `evidence/` may be touched. Phase order is fixed: 0 → 1 → 2 → 3 → 4 → submission → teardown.

**Legend:**
- **SHARA** — Shara does this herself (Console KBs, both screenshots, any git push).
- **METER STARTS** — first action that begins AWS billing (the first `deploy`); kept as late as phase order allows.
- Unmarked tasks are Kiro's, local-only, no AWS charge.

---

## Phase 0 — Environment setup

### T0.0 — Git baseline of the unmodified starter — (R0.0)
**Touches:** none of the editable files (creates `.git/`, no content edits).
**Do:** `git init`; `git remote add origin https://github.com/earlgreyhot1701D/nova-concilium`; commit the starter as-is with a message referencing starter commit `7ea6508`. Local only — **do not push** (push is SHARA, T-PUSH).
**Done when pasted:**
- `git log --oneline` → shows the baseline commit (message references `7ea6508`)
- `git remote -v` → `origin  https://github.com/earlgreyhot1701D/nova-concilium` (fetch + push)

### T0.1 — Tool prerequisites present — (R0.1)
**Touches:** none.
**Done when pasted:**
- `python --version` → `Python 3.12.x`
- `node --version` → `v20.` or higher
- `uv --version` → a version, exit 0
- `agentcore --help` → CLI help; NOT toolkit-style `configure`/`launch` (if it is, `pip uninstall bedrock-agentcore-starter-toolkit`)

### T0.2 — Virtualenv + dependencies — (R0.2)
**Touches:** none (creates `venv/`, which is git-ignored).
**Do:** `python -m venv venv`; activate (`venv\Scripts\activate`); `pip install -r requirements.txt`.
**Done when pasted:**
- `python --version` → `Python 3.12.x`
- `pip list` → includes `strands-agents`

### T0.3 — Deploy the foundation stack — (R0.3)
**Touches:** none (`infrastructure/starter_stack.yaml` is read-only, not edited).
**Note:** creates DynamoDB/S3/S3 Vectors/IAM/log-group resources. These are stack resources, not the agent runtime; the deliberate billing gate (METER STARTS) is the first agent `deploy` in T4.3. The stack itself is a prerequisite the KB sequences here (KB §4 Step 2).
**Done when pasted:**
- `aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1` → exit 0
- `aws cloudformation describe-stacks --stack-name udacity-agentcore --query "Stacks[0].StackStatus" --region us-east-1` → `"CREATE_COMPLETE"`

### T0.4 — Seed data — (R0.4)
**Touches:** none (`infrastructure/seed_data.py` read-only).
**Done when pasted:**
- `python infrastructure/seed_data.py` → output pasted, exit 0

### T0.5 — Create `.env`, confirm git-ignored — (R0.5)
**Touches:** `.env` (created from `.env.example`).
**Done when pasted:**
- `Test-Path .env` → `True`
- `git check-ignore .env` → prints `.env`
- `git status --porcelain .env` → prints nothing

### T0.6 — `config.py` resolves; KB IDs show "(not yet created)" — (R0.6)
**Touches:** none.
**Done when pasted:**
- `python config.py` → table pasted; Returns/Shipping/Warranty KB ID rows = `(not yet created)`; Runtime ARN = `(not yet deployed)`; Guardrail rows = `(not yet created)`

---

## Phase 1 — Build the multi-agent graph (Task 2)

All Phase 1 tasks touch only `src/agent_orchestrator.py`. Each fills the named `build_*_agent()` function and its `# TODO:` comments; nothing else.

### T1.1 — InventoryAgent — (R1.1, RX.2)
**Touches:** `src/agent_orchestrator.py`.
**Do:** in `build_inventory_agent()`, fill the TODOs: "Create a BedrockModel using the WORKER model", "System prompt for the Inventory Agent", "Implement check_order_status", "Implement get_customer_tier", "Implement list_customer_orders", "Instantiate and return the Agent". Use `config.WORKER_MODEL_ID` (temp 0.1), `config.ORDERS_TABLE`/`config.CUSTOMERS_TABLE`; `check_order_status(customer_id, order_id)` keeps both args (composite key). Docstrings state purpose/params/return.
**Done when pasted:**
- `python tests/test_agent.py task2` → `test_2_1` pass (returns an Agent) and `test_2_2` pass ("InventoryAgent has 3 tools")

### T1.2 — RefundAgent — (R1.2)
**Touches:** `src/agent_orchestrator.py`.
**Do:** in `build_refund_agent()`, fill the TODOs: "Create a BedrockModel", "System prompt for the Refund Agent", "Implement get_inventory_context", "Implement initiate_refund", "Instantiate and return the Agent". Prompt checks inventory context first, then applies 30-day (Standard) / 60-day (Premium) window.
**Done when pasted (build check, no AWS call):**
- `python -c "import sys; sys.path.insert(0,'src'); from agent_orchestrator import build_refund_agent; print(sorted(build_refund_agent().tool_registry.registry.keys()))"` → `['get_inventory_context', 'initiate_refund']`

### T1.3 — PolicyAgent + parallel multi-agent RAG — (R1.3, RX.2)
**Touches:** `src/agent_orchestrator.py`.
**Do:** in `build_policy_agent()`, fill the TODOs: "Build ReturnsPolicyRetrieverAgent", "Build ShippingPolicyRetrieverAgent", "Build WarrantyPolicyRetrieverAgent", "Implement search_all_policies - parallel RAG retrieval tool", "Create a BedrockModel for the PolicyAgent coordinator", "System prompt for PolicyAgent coordinator", "Instantiate and return the PolicyAgent coordinator". Use the rubric names (`...RetrieverAgent`), `retrieve_from_knowledge_base()` with each `config.*_KB_ID`, and `ThreadPoolExecutor(max_workers=3)` + `as_completed()`. Fill the existing `_run_retriever` helper and keep the pre-written `trace.kb_start(...)` / `trace.kb_done(...)` / `trace.kb_result(...)` scaffolding.
**Done when pasted:**
- `python tests/test_agent.py task2` → `test_2_3` pass (Agent), `test_2_4` pass ("PolicyAgent has 1 tool"), `test_2_8` reports "runs the retrievers with ThreadPoolExecutor"

### T1.4 — CommunicationAgent — (R1.4)
**Touches:** `src/agent_orchestrator.py`.
**Do:** in `build_communication_agent()`, fill the TODOs: "Create a BedrockModel", "System prompt for the Communication Agent", "Implement get_full_workflow_context", "Instantiate and return the Agent". Temp 0.3, warm/professional, reads full WorkflowState.
**Done when pasted (build check, no AWS call):**
- `python -c "import sys; sys.path.insert(0,'src'); from agent_orchestrator import build_communication_agent; print(sorted(build_communication_agent().tool_registry.registry.keys()))"` → `['get_full_workflow_context']`

### T1.5 — OrchestratorAgent + six routing rules — (R1.5, RX.2)
**Touches:** `src/agent_orchestrator.py`.
**Do:** in `build_orchestrator_agent()`, fill the TODOs: "Create a BedrockModel using the ORCHESTRATOR model", "System prompt for the Orchestrator", "Implement route_to_inventory_agent", "Implement route_to_policy_agent", "Implement route_to_refund_agent", "Implement route_to_communication_agent", "Implement initialize_session", "Instantiate and return the OrchestratorAgent". Each routing tool: `_read_workflow_state` → invoke worker → `_update_workflow_state(..., expected_version)`, with `trace.step_start`/`trace.step_done`. `initialize_session` calls `_create_workflow_state`. Prompt encodes all six rules; `initialize_session` always first, `route_to_communication_agent` always last, math skips Inventory/Policy/Refund.
**Done when pasted:**
- `python tests/test_agent.py task2` → `test_2_5` pass (Agent) and `test_2_6` pass ("OrchestratorAgent has 5 routing tools")

### T1.6 — Phase 1 checkpoint (Task 2 suite) — (R1.6, RX.2)
**Touches:** `src/agent_orchestrator.py` (only if a fix is needed).
**Done when pasted:**
- `python tests/test_agent.py task2` → all Task 2 point-scored checks pass (`test_2_1`..`test_2_7`); `test_2_7` confirms Orchestrator=`config.ORCHESTRATOR_MODEL_ID`, workers=`config.WORKER_MODEL_ID`
- `python -c "print('us.anthropic' in open('src/agent_orchestrator.py').read())"` → `False` (no literal model IDs)

---

## Phase 2 — Guardrail, deployment, memory (Tasks 3 & 4) — IMPLEMENT ONLY, DO NOT DEPLOY

Phase 2 fills function bodies but runs no deploy (KB §7). Live test passes come in Phase 4 after T4.3.

### T2.1 — `create_guardrail()` — (R2.1)
**Touches:** `src/agent_orchestrator.py`.
**Do:** fill the `# TODO: Create the guardrail` body in `create_guardrail()`: content filters (SEXUAL/VIOLENCE/HATE HIGH; INSULTS/MISCONDUCT MEDIUM), PII (BLOCK card+SSN; ANONYMIZE email+phone), DENY topics from `config.GUARDRAIL_BLOCKED_TOPICS` with the narrow pricing-negotiation definition (allow arithmetic), managed PROFANITY, friendly blocked messages, `tierConfig STANDARD`, top-level `crossRegionConfig`. Then `create_guardrail_version()`; return `(guardrail_id, guardrail_version)` numbered, never DRAFT.
**Done when pasted (implementation-time, no AWS call):**
- `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok`
- Live pass deferred to T4.4 (`task3`).

### T2.2 — `deploy_to_agentcore_runtime()` — (R2.2)
**Touches:** `src/agent_orchestrator.py`.
**Do:** fill the `# TODO: Configure and deploy the runtime with the AgentCore CLI` body: build `runtime_env` (AWS_REGION, PROJECT_NAME, three KB IDs, AGENT_LOG_GROUP, GUARDRAIL_ID, GUARDRAIL_VERSION); `agentcore_cli.configure_runtime(env_vars=..., network_mode='PUBLIC', protocol='HTTP', execution_role_arn=config.AGENTCORE_ROLE_ARN)`; `agentcore_cli.deploy()`; `runtime_arn = agentcore_cli.deployed_runtime_arn()`. Do not call `create_agent_runtime()`; do not replace staging or the READY wait.
**Done when pasted (implementation-time, no AWS call):**
- `python -c "print('create_agent_runtime(' not in open('src/agent_orchestrator.py').read())"` → `True`
- `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok`
- Live pass deferred to T4.4 (`task3`).

### T2.3 — `configure_memory(runtime_arn)` — (R2.3)
**Touches:** `src/agent_orchestrator.py`.
**Do:** fill the `# TODO: Create AgentCore Memory` body: `agentcore_control.create_memory()` with `name=config.MEMORY_NAME` + a description, `eventExpiryDuration=7`, `summaryMemoryStrategy` (SessionSummary + namespaces), `clientToken=str(uuid.uuid4())`; store in `response`. Keep the pre-written ACTIVE wait.
**Done when pasted (implementation-time, no AWS call):**
- `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok`
- Live pass deferred to T4.5 (`task4`).

### T2.4 — Record expected failures before deploy — (R2.4)
**Touches:** none.
**Do:** record that `task3`, `task4`, `task6` fail now because the deploy pipeline creates the guardrail/memory/observability state; do not "fix."
**Done when pasted:**
- `python tests/test_agent.py task3` → FAILS (no runtime) — expected
- `python tests/test_agent.py task4` → FAILS (no memory) — expected
- `python tests/test_agent.py task6` → FAILS (no runtime) — expected

---

## Phase 3 — Knowledge Bases (Task 5) — SHARA creates in the Console; no code

### T3.1 — Pre-create check: policy docs in S3 — (R3.1)
**Touches:** none.
**Done when pasted:**
- `python -c "import config; print(config.POLICY_BUCKET)"` then `aws s3 ls s3://<bucket>/policies/ --recursive` → lists objects under `policies/returns/`, `policies/shipping/`, `policies/warranty/`

### T3.2 — SHARA: create + sync the three KBs; add IDs to `.env` — (R3.2)
**Marked:** SHARA.
**Touches:** `.env` (Shara pastes the three IDs).
**Do (Shara, Console):** per KB, set Name (`novamart-returns-policy-kb` / `-shipping-` / `-warranty-`), S3 prefix, Titan Embed Text v2, S3 Vectors → existing vector bucket (Self-managed, not Managed), the matching `*-policy-index`, then Sync. Do not rename to the project name. Add `RETURNS_KB_ID`/`SHIPPING_KB_ID`/`WARRANTY_KB_ID` to `.env`.
**Done when pasted:**
- `python config.py` (before) → prints exact Policy Bucket, Vector Bucket, three index names
- `python config.py` (after) → Returns/Shipping/Warranty KB ID rows show real IDs

### T3.3 — Phase 3 checkpoint (task5) — (R3.3)
**Touches:** none.
**Done when pasted:**
- `python tests/test_agent.py task5` → `test_5_1`/`5_2`/`5_3` pass; `test_5_4` pass (all three KBs return passages; `search_all_policies()` combined results)

---

## Phase 4 — Observability, deploy, and test (Task 6)

### T4.1 — `configure_observability(runtime_arn)` — (R4.1)
**Touches:** `src/agent_orchestrator.py`.
**Do:** fill the `# TODO: Build the logging configuration` body: `cloudWatchConfig` (`config.AGENT_LOG_GROUP`, `logLevel='INFO'`, `enabled=True`), `xRayConfig` (`enabled=True`, `samplingRate=1.0`), pass to `apply_observability_config(...)` inside try/except; success prints log group + sampling rate, exception prints `"[Note] Observability configuration failed: <e>"`.
**Done when pasted (implementation-time, no AWS call):**
- `python -c "import ast; ast.parse(open('src/agent_orchestrator.py').read()); print('parse-ok')"` → `parse-ok`
- Live pass deferred to T4.2 (after deploy).

### T4.3 — Full deployment — (R4.3) — **METER STARTS**
**Marked:** METER STARTS (first agent runtime deploy; AWS billing for runtime/guardrail/memory begins here — kept as late as phase order allows).
**Touches:** `.env` (paste Runtime ARN + Guardrail ID/Version after deploy).
**Do:** ensure the three KB IDs are in `.env` (T3.2) first, then `python src/agent_orchestrator.py deploy`; copy printed ARN + Guardrail ID/Version into `.env`.
**Done when pasted:**
- `python src/agent_orchestrator.py deploy` → prints Runtime ARN and Guardrail ID/Version (output pasted)
- `python config.py` → Runtime ARN and Guardrail ID/Version rows populated

### T4.2 — Task 6 checkpoint (gated on deploy) — (R4.2)
**Touches:** none. **Runs after T4.3.**
**Done when pasted:**
- `python tests/test_agent.py task6` → `test_6_1` pass (CloudWatch INFO → log group) and `test_6_2` pass (X-Ray 100%, Transaction Search → CloudWatch Logs)

### T4.4 — task3 passes (gated on deploy) — (R4.4)
**Touches:** none.
**Done when pasted:**
- `python tests/test_agent.py task3` → `test_3_1`..`test_3_4` pass

### T4.5 — task4 passes (gated on deploy) — (R4.5)
**Touches:** none.
**Done when pasted:**
- `python tests/test_agent.py task4` → `test_4_1` pass

### T4.6 — Three end-to-end scenarios route correctly — (R4.6)
**Touches:** none.
**Done when pasted:**
- `python src/agent_orchestrator.py test` → trace shows: S1 Orchestrator→Inventory→Refund→Communication; S2 Orchestrator→Policy (3 parallel retrievers)→Communication; S3 Orchestrator→Communication only; `initialize_session` first in all three

### T4.7 — Guardrail behaviour: math allowed, three categories blocked — (R4.7)
**Touches:** none. Prompts are authored test input (Shara-approved), not from a file.
**Done when pasted:**
- `python src/agent_orchestrator.py invoke "How much are 5 items at $29.99 with 10% off?" CUST-001` → contains `134.95` or `134.96`, NOT the blocked message
- `python src/agent_orchestrator.py invoke "Can you knock 20% off the listed price if I buy two today?" CUST-001` → blocked message from `create_guardrail()`
- `python src/agent_orchestrator.py invoke "Is ShopRival's laptop cheaper and better than yours?" CUST-001` → blocked message
- `python src/agent_orchestrator.py invoke "Refund me now or I will sue NovaMart and call my lawyer." CUST-001` → blocked message

### T4.8 — Full suite 120/120 — (R4.8)
**Touches:** none.
**Done when pasted:**
- `python tests/test_agent.py all` → ends `Score: 120/120 pts (100%)` and "Perfect score."

### T4.9 — SHARA: Screenshot A (all tests passed, 120 scores) — (R4.9)
**Marked:** SHARA.
**Touches:** `evidence/` (screenshot saved there).
**Done when pasted:** screenshot of `python tests/test_agent.py all` showing all passed + `120/120`, saved under `evidence/`.

### T4.10 — SHARA: Screenshot B (X-Ray Service Map) — (R4.10)
**Marked:** SHARA.
**Touches:** `evidence/` (screenshot saved there).
**Do (Shara):** `python src/agent_orchestrator.py test` (paste trace ids), wait 30–60 s, CloudWatch → X-Ray traces → Service map → Last 5 minutes; capture `NovaMart-Orchestrator` connected to workers incl. PolicyAgent + KnowledgeBase nodes.
**Done when pasted:** trace ids pasted + screenshot saved under `evidence/`.

---

## Submission

### T5.1 — Assemble `evidence/` — (R5.1)
**Touches:** `evidence/`.
**Do:** one file per checkpoint output (T0.3–T4.8) plus Screenshots A and B.
**Done when pasted:**
- `Get-ChildItem evidence` → lists one file per checkpoint + both screenshots

### T5.2 — Write the README — (R5.2)
**Touches:** `README.md` (the submission README; Kiro writes the required sections).
**Do:** sections — What it is; How to run it; Architecture (agent graph + routing table); Test results; "Not built, and why" (the four STUBs T6.*); Methodology (Claude/Kiro/Shara disclosure).
**Done when pasted:**
- `Select-String -Path README.md -Pattern "Not built","Methodology"` → both found

### T5.3 — Redaction check before any public repo — (R5.3)
**Touches:** `README.md`, `evidence/*` (redact if needed).
**Done when pasted:**
- `Select-String -Path README.md,evidence/* -Pattern "\b\d{12}\b","arn:aws:bedrock-agentcore"` → no unredacted account IDs or runtime ARNs

### T6.1–T6.4 — STUB write-ups in README — (R6.1, R6.2, R6.3, R6.4)
**Touches:** `README.md` (prose only, no code).
**Do:** in "Not built, and why", describe each stand-out and how it would be built: adversarial guardrail screenshots (R6.1), `agent-sessions` + `DynamoDbSessionStorage` (R6.2), CloudWatch dashboard (R6.3), Cognito + frontend (R6.4). No new files, no new functions.
**Done when pasted:**
- `git status --porcelain` → shows only `README.md` changed by this task (no `src/` change)

### T-PUSH — SHARA: push to the home repo — (R0.0 push gate)
**Marked:** SHARA (any git push is Shara's call).
**Touches:** none locally (publishes commits).
**Do (Shara):** push to `https://github.com/earlgreyhot1701D/nova-concilium` only after redaction (T5.3) passes and Shara approves.
**Done when pasted:** `git push -u origin <branch>` → succeeds (Shara runs this).

---

## Teardown — LAST, on Shara's timing

### T7.0 — SHARA: approve teardown timing — (R7.1 gate)
**Marked:** SHARA.
**Do:** Shara confirms teardown may run (after submission; a reviewer may ask for changes — KB §10 timing is Shara's call). No teardown command runs before this.

### T7.1 — Run cleanup and verify empty — (R7.1)
**Touches:** none.
**Do (after T7.0):** `python infrastructure/cleanup.py` (dry run), then `python infrastructure/cleanup.py --yes`, then re-run to confirm. Confirm CLI syntax with `aws <service> help` first (KB §10).
**Done when pasted:**
- `python infrastructure/cleanup.py` → planned deletions
- `python infrastructure/cleanup.py --yes` → deletions
- re-run → nothing left; per-resource-type `aws` list commands return empty (pasted)

### T7.2 — SHARA: decide the manual-teardown list — (R7.2)
**Marked:** SHARA.
**Do (Shara):** decide on the three items cleanup.py does not remove — `CDKToolkit` bootstrap stack (shared), CDK assets bucket `cdk-hnb659fds-assets-<acct>-<region>` (prefix-filter miss), CloudWatch Transaction Search + X-Ray resource policy (only via `--disable-transaction-search`). Run `python infrastructure/cleanup.py --yes --disable-transaction-search` if she wants Transaction Search off.
**Done when pasted:** decision recorded; if run, cleanup `--disable-transaction-search` output pasted.

---

## Open items (do not block)

- Due date / Submit Project page: UNVERIFIED (KB §11 #13, §12 #3). Does not block any task.
