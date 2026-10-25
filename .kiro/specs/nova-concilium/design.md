# Design — Nova Concilium

**Scope:** design only. No code, no AWS resources. Every decision cites a source: a KB section, a starter file + line, or the rubric point as encoded in those files (the rubric was not pasted verbatim; where the KB says "rubric requires," it is cited as KB + the enforcing file, almost always `tests/test_agent.py`). Nothing here goes beyond the course pages, rubric, or starter files.

**What this design does NOT do:** it does not redesign the starter. The only file that gets implemented is `src/agent_orchestrator.py` (the TODO bodies) and `.env` (values). Everything else is pre-written and is consumed through its existing public surface (KB §1; `README.md` "Files to Never Modify"). This design is therefore mostly *how the TODO bodies wire into the existing scaffolding*, not new structure.

---

## 1. Design constraints that shape everything

| Constraint | Source | Design consequence |
|---|---|---|
| Edit only `src/agent_orchestrator.py` + `.env` | KB §1; `README.md` "Files to Never Modify" | No new modules, no new helper files. All logic lives in the TODO bodies. |
| Use the three pre-written WorkflowState helpers; do not write new ones | KB §5; `src/agent_orchestrator.py` lines 118–205 (`_create_workflow_state`, `_read_workflow_state`, `_update_workflow_state`) | Routing tools call these three functions; they do not open their own DynamoDB handles. |
| Use config constants; never hardcode | KB §1; `config.py` lines 38, 41, `_LAZY` map | Model IDs, table names, KB IDs, ARNs, log group all read from `config.*`. |
| `@tool` must come from `agent_observability`, not `strands` | `src/agent_orchestrator.py` lines 97–101 import block; `README.md` "Key Concepts" | Every tool uses the wrapped decorator so each call becomes an X-Ray node. Design never imports `tool` from `strands`. |
| Deployment goes through `agentcore_cli`; never `create_agent_runtime()` | KB §1 NEVER; `src/agent_orchestrator.py` deploy pipeline | `deploy_to_agentcore_runtime` only calls `agentcore_cli.*`. |
| Phase order fixed; Phase 2 implements but does not deploy | KB §0, §6–§9 | Guardrail/memory/observability bodies are written in Phase 2 but first executed by `deploy` in Phase 4 (pipeline lines ~1144–1158). |

---

## 2. Agent graph (Task 2)

### 2.1 Shape and models

Orchestrator→Workers hierarchy, five agents plus three retriever sub-agents inside PolicyAgent (KB §3, §5; `README.md` Architecture; `src/agent_orchestrator.py` module docstring graph).

| Agent | Model constant | Temp | Tools | Source |
|---|---|---|---|---|
| OrchestratorAgent | `config.ORCHESTRATOR_MODEL_ID` (Haiku 4.5) | 0.0 | `initialize_session`, `route_to_inventory_agent`, `route_to_policy_agent`, `route_to_refund_agent`, `route_to_communication_agent` | KB §5; `config.py` line 38; `tests/test_agent.py::test_2_6`, `::test_2_7` |
| InventoryAgent | `config.WORKER_MODEL_ID` (Sonnet 4.5) | 0.1 | `check_order_status`, `get_customer_tier`, `list_customer_orders` | KB §5; `config.py` line 41; `tests/test_agent.py::test_2_2` |
| RefundAgent | `config.WORKER_MODEL_ID` | 0.1 | `get_inventory_context`, `initiate_refund` | KB §5; `src/agent_orchestrator.py` TODOs 304, 318 |
| PolicyAgent (coordinator) | `config.WORKER_MODEL_ID` | 0.2 | `search_all_policies` | KB §5; `tests/test_agent.py::test_2_4` (exactly 1 tool) |
| ReturnsPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | one tool → `retrieve_from_knowledge_base(config.RETURNS_KB_ID, …)` | KB §5; KB §11 #2 (names); `src/bedrock_kb_retrieval.py` |
| ShippingPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | same, `config.SHIPPING_KB_ID` | KB §5 |
| WarrantyPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | same, `config.WARRANTY_KB_ID` | KB §5 |
| CommunicationAgent | `config.WORKER_MODEL_ID` | 0.3 | `get_full_workflow_context` | KB §5; `src/agent_orchestrator.py` TODO 458 |

Decision: temperatures are taken exactly from the KB §5 table — not tuned. Rationale: KB §1 ("follow the instructions as written; no 'better' approaches"). The test only grades model *IDs* (`test_2_7`), not temperatures, but the KB table is binding, so temperatures match it.

### 2.2 Tool construction pattern

Each `build_*_agent()` ends with `return Agent(model=BedrockModel(...), system_prompt=..., tools=[...])` — exactly one Agent (KB §6 "Tips"; `tests/test_agent.py` `_get_tool_count` reads `agent.tool_registry.registry`). `BedrockModel` takes `model_id`, `region_name=config.AWS_REGION`, `temperature` (`README.md` "Key Concepts" example). Tool docstrings must state purpose, parameters, and return value because the agent reads them to decide usage and the rubric grades it (KB §6 "Tips").

Decision: tools are defined as nested `@tool` functions inside each `build_*` function, matching the stubs already present (`src/agent_orchestrator.py` lines 237–281, etc.). No module-level tool functions are added. Rationale: the stubs dictate the shape; RX.1 forbids restructuring.

### 2.3 InventoryAgent tools (2.A)

- `check_order_status(customer_id, order_id)` — composite key lookup on the Orders table (`customer_id` partition + `order_id` sort), so both args are required (KB §6 1.A; stub NOTE at `src/agent_orchestrator.py` lines 235–236). Reads via `config.ORDERS_TABLE`. Reports facts only.
- `get_customer_tier(customer_id)` — returns Standard/Premium (KB §6 1.A; stub lines 253–267). Standard = 30-day window, Premium = 60-day (KB §6 1.B) — but the *decision* belongs to RefundAgent, so this tool only reports the tier.
- `list_customer_orders(customer_id)` — all orders for a customer (stub lines 268–281).

System prompt: a data gatherer that retrieves accurately and never decides eligibility (KB §6 1.A).

### 2.4 RefundAgent tools (2.B)

- `get_inventory_context(session_id)` — reads the `inventory_agent` field from WorkflowState via `_read_workflow_state` (KB §6 1.B; stub lines 304–317).
- `initiate_refund(customer_id, order_id, reason)` — updates the order record in DynamoDB and returns a confirmation with a `return_reference` (stub lines 318–333).

System prompt: always check inventory context first, then apply the window — Standard 30 days, Premium 60 days (KB §6 1.B).

### 2.5 PolicyAgent — parallel multi-agent RAG (2.C)

This is the one agent with internal structure. Design:

1. Three retriever sub-agents are built inside `build_policy_agent()`, named `ReturnsPolicyRetrieverAgent`, `ShippingPolicyRetrieverAgent`, `WarrantyPolicyRetrieverAgent` (rubric names — KB §11 #2). Each holds one `@tool` that calls `retrieve_from_knowledge_base(config.<X>_KB_ID, query)` and formats with `format_kb_results` (`src/bedrock_kb_retrieval.py` signature: `retrieve_from_knowledge_base(kb_id, query, top_k=3) -> list[{text,source,score}]`).
2. `search_all_policies(query)` is the coordinator's single tool. It runs the three retrievers **in parallel** using `ThreadPoolExecutor(max_workers=3)` + `as_completed()` (KB §11 #4; `tests/test_agent.py::test_2_8` statically requires `ThreadPoolExecutor` + `submit`/`map`).
3. The trace scaffolding already present must be used, not replaced: the stub already calls `trace.kb_start({...})` before dispatch (`src/agent_orchestrator.py` lines ~405–415) and leaves commented `trace.kb_done(...)` / `trace.kb_result(...)` calls after join. Design fills the parallel run between `kb_start` and `kb_done`, then prints each domain result with `trace.kb_result` (API confirmed in `agent_utils.py` `AgentTrace`: `kb_start(kb_map)` line 350, `kb_done(count)` line 381, `kb_result(domain, content)` line 392).
4. The stub's `_run_retriever(domain, agent, query)` helper is filled to invoke one retriever and return `(domain, text)`; stdout is already globally suppressed by the `_TraceWriter._suppress_parallel` flag set in `kb_start` (stub docstring lines ~393–404), so the design does not add its own suppression.

Decision: retrievers run as parallel siblings under `search_all_policies`, not a chain. Rationale: KB §11 #3 (image's dotted chain loses to the text) and KB §2 (image is lowest authority). The coordinator prompt calls `search_all_policies` first, then synthesizes a grounded answer (KB §6 1.C).

Open sequencing note (not a conflict): `test_2_8` is 0 points but the KB still mandates `ThreadPoolExecutor(max_workers=3)` + `as_completed()`. Design uses exactly that form so both the KB rule and the static check are satisfied.

### 2.6 CommunicationAgent (2.D)

- `get_full_workflow_context(session_id)` reads the entire WorkflowState via `_read_workflow_state` (KB §6 1.D; stub lines 458–471).
- Prompt: compose the final customer-facing reply from all findings, warm and professional (KB §6 1.D).

### 2.7 OrchestratorAgent (2.E) — routing and state

Five routing tools; each follows the three-step pattern already documented in the stub (`src/agent_orchestrator.py` lines 500–505):
1. `_read_workflow_state(session_id)` → current version;
2. invoke the worker agent;
3. `_update_workflow_state(session_id, {'<column>': text}, expected_version)` with optimistic locking.

Trace calls `trace.step_start('<column>')` before the worker and `trace.step_done('<column>', old_version)` after (API in `agent_utils.py` lines 309, 328; stub comment lines 503–505).

`initialize_session(session_id, customer_id)` calls `_create_workflow_state(session_id, customer_id)` (version 0). System prompt encodes all six routing rules verbatim and the CRITICAL rule that the Orchestrator never writes the final reply and always ends on `route_to_communication_agent` (KB §5 routing table + CRITICAL; `README.md` Task 2.E).

Routing rules in the prompt (KB §5; KB §11 #12 that Rule 1 applies to every scenario including math):

| Rule | Trigger | Action |
|---|---|---|
| 1 | every request | `initialize_session` first |
| 2 | order status / return / refund | inventory → then refund |
| 3 | policy meaning | policy |
| 4 | account questions ("my tier?", "am I premium?") | inventory, never policy |
| 5 | math / calculation | skip inventory/policy/refund; compute; still end at communication; round currency only after the full calculation (`src/agent_orchestrator.py` line 495) |
| 6 | every request, last | `route_to_communication_agent` |

### 2.8 Guardrail attachment (not Refund/Policy specific)

`build_agent_graph()` already calls the pre-written `_apply_guardrail([...])` after building the five agents (`src/agent_orchestrator.py` lines ~620–640). Design does not touch `_apply_guardrail` (KB §7 "Do not replace it"). It attaches `config.GUARDRAIL_ID` / `GUARDRAIL_VERSION` to every agent's `BedrockModel` when those are set — from `.env` locally, from runtime env when deployed. So the guardrail is a per-model-call concern, not a sixth agent.

---

## 3. Shared WorkflowState (DynamoDB)

One record per `session_id` in `config.WORKFLOW_STATE_TABLE` (KB §5; stack table `udacity-agentcore-workflow-state`, `starter_stack.yaml` line 56). Lifecycle (KB §5; stub docstrings):

- v0: created by `_create_workflow_state(session_id, customer_id)` with `session_id`, `customer_id`, `created_at`, `version=0`, `ttl` (24h) — agent columns absent (`src/agent_orchestrator.py` lines 118–145).
- Each routing tool adds its column (`inventory_agent`, `policy_agent`, `refund_agent`, `communication_agent`) via `_update_workflow_state`, which does a conditional write on `version = :expected_version` and retries up to 3 times on `ConditionalCheckFailedException` (optimistic locking — lines 158–205).

Design decision: the four worker columns are written as the agents run; the orchestrator never writes the customer response into state itself (KB §5 CRITICAL). Rationale: both a KB rule and the reason the optimistic-lock helper exists (concurrent policy retrievers must not clobber each other).

---

## 4. Guardrail (Task 3) — `create_guardrail()`

Uses a `bedrock` client (stub line ~687). Policy values (KB §7 table; `README.md` Task 3; enforced by `tests/test_agent.py::test_3_2`):

| Policy | Design value | Source |
|---|---|---|
| Content filters | SEXUAL, VIOLENCE, HATE = HIGH (input+output); INSULTS, MISCONDUCT = MEDIUM | KB §7; `test_3_2` `expected_strength` |
| PII | `CREDIT_DEBIT_CARD_NUMBER`, `US_SOCIAL_SECURITY_NUMBER` = BLOCK; `EMAIL`, `PHONE` = ANONYMIZE | KB §7; `test_3_2` `expected_pii` |
| Topics (DENY) | competitor products, pricing negotiations, legal threats — one per `config.GUARDRAIL_BLOCKED_TOPICS` | KB §7; `config.py` `GUARDRAIL_BLOCKED_TOPICS`; `test_3_2` keyword check |
| Pricing-negotiation definition | haggling / changing an advertised price; **exclude** arithmetic on an already-given price+discount | KB §7; stub lines ~702–708 (Classic tier blocked the math scenario) |
| Word policy | managed PROFANITY list | KB §7; `test_3_2` managed list check |
| Messages | friendly `blockedInputMessaging` + `blockedOutputsMessaging` | KB §7 |
| Tier | `topicPolicyConfig.tierConfig = {"tierName": "STANDARD"}` | KB §7, §11 #8 |
| Cross-region | top-level `crossRegionConfig = {"guardrailProfileIdentifier": "us.guardrail.v1:0"}` | KB §7, §11 #8 |

Then `create_guardrail_version()` and return `(guardrail_id, guardrail_version)` with the numbered version — never DRAFT (KB §1 NEVER; `test_3_3` requires `version.isdigit()` and status READY). The stub already short-circuits if a guardrail with `config.GUARDRAIL_NAME` exists (lines ~688–693); design keeps that idempotency.

Decision: topic definitions are written to pass the required math scenario while still blocking haggling. Rationale: KB §7 and stub both warn the Classic-tier phrasing blocked the project's own math case; STANDARD tier + the narrow definition is the instructed fix, not an invention.

---

## 5. Runtime deployment (Task 3) — `deploy_to_agentcore_runtime()`

The pre-written `agentcore_cli.stage_runtime_code()` is already called before the TODO (stub line 758). Design fills the TODO with exactly four calls (KB §7; stub lines 759–773; `agentcore_cli` public API confirmed: `configure_runtime`, `deploy`, `deployed_runtime_arn`):

1. Build `runtime_env` dict: `AWS_REGION`, `PROJECT_NAME`, `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`, `AGENT_LOG_GROUP`, `GUARDRAIL_ID` (=guardrail_id), `GUARDRAIL_VERSION` (=guardrail_version). (Exactly the eight keys `test_3_4` checks on the deployed runtime.)
2. `agentcore_cli.configure_runtime(env_vars=runtime_env, network_mode='PUBLIC', protocol='HTTP', execution_role_arn=config.AGENTCORE_ROLE_ARN)`.
3. `agentcore_cli.deploy()`.
4. `runtime_arn = agentcore_cli.deployed_runtime_arn()`.

The pre-written tail then waits for READY via `wait_for_runtime_ready` (stub lines ~775–783). Design does not call `create_agent_runtime()` (KB §1 NEVER) and does not replace the staging or wait steps.

---

## 6. Memory (Task 4) — `configure_memory(runtime_arn)`

`agentcore_control.create_memory()` (stub lines 807–820). Design values:

- `name = config.MEMORY_NAME`, plus a description (KB §11 #5 requires both).
- `eventExpiryDuration = 7` (KB §7; `test_4_1` checks `== 7`).
- `memoryStrategies = [{'summaryMemoryStrategy': {'name': 'SessionSummary', 'namespaces': ['/summaries/{actorId}/{sessionId}']}}]` — the SESSION_SUMMARY strategy (KB §7; stub comment lines 813–817; `test_4_1` checks a `SUMMARIZATION` strategy type is present).
- `clientToken = str(uuid.uuid4())` for idempotency (stub line 818).

Return the response in `response`; the pre-written tail waits for ACTIVE and returns `memory['arn']` (stub lines 822–834). The stub also short-circuits if a memory starting with `config.MEMORY_NAME` already exists (lines 801–806).

---

## 7. Observability (Task 6) — `configure_observability(runtime_arn)`

Build `logging_configuration` and hand it to the pre-written `apply_observability_config(runtime_arn, logging_configuration)`, inside try/except (KB §9, §11 #9; stub lines 854–872; `apply_observability_config` confirmed at `agent_observability.py` line 645):

- `cloudWatchConfig = {'logGroupName': config.AGENT_LOG_GROUP, 'logLevel': 'INFO', 'enabled': True}`.
- `xRayConfig = {'enabled': True, 'samplingRate': 1.0}`.

On success print the log group and sampling rate; on exception print `"[Note] Observability configuration failed: <e>"` (stub lines 866–871). `apply_observability_config` turns this into real state: enables CloudWatch Transaction Search at the sampling percentage, creates the log group, and stores runtime env vars (`AGENT_LOG_GROUP`/`AGENT_LOG_LEVEL`/`AGENT_LOG_TO_CLOUDWATCH`, `AGENT_TRACING_ENABLED`/`AGENT_TRACE_SAMPLING_RATE`) that `test_6_1`/`test_6_2` read back.

Design note (matches requirements R2.4/R4.2): this function is first executed by the `deploy` pipeline Step 5 (stub lines ~1156–1158), so its live pass is proven after deploy, not in Phase 2.

---

## 8. Knowledge Bases (Task 5) — no code

Design here is a Console checklist for Shara plus the task5 test (KB §8; requirements R3.1–R3.3). No `agent_orchestrator.py` change. The three KBs feed `config.RETURNS_KB_ID` / `SHIPPING_KB_ID` / `WARRANTY_KB_ID` (read by `config.py` `_get_kb_id`, `.env` first then CF export), which the retriever sub-agents (§2.5) consume. Self-managed KB on the existing S3 Vectors bucket + the three stack-created indexes; Titan Embed Text v2; sync each (KB §8; `README.md` Task 5). Creating KBs by CLI/SDK is forbidden (KB §1 NEVER).

---

## 9. Observability trace model (how the X-Ray Service Map gets built)

The screenshot deliverable (KB §3 #4, §9 step 6) depends on the trace graph, which is produced by pre-written code — design only has to *use* it correctly:

- `tool` imported from `agent_observability` wraps every `@tool` so each call is an X-Ray subsegment; the orchestrator's `route_to_*` tools become the worker-agent nodes (`src/agent_orchestrator.py` import lines 97–101; `agent_observability.py` `tool` line 357, `AgentTracer` line 169).
- Each KB retrieval becomes a `KnowledgeBase:<domain>` node via `trace_kb_retrieval(kb_id)` used inside `retrieve_from_knowledge_base` (`src/bedrock_kb_retrieval.py`; `agent_observability.py` line 375).
- The root node is `NovaMart-Orchestrator` (KB §1 NEVER rename; KB §9 step 6; `tests/test_agent.py::test_6_3` filters `service("NovaMart-Orchestrator")`).

Design consequence: because tracing is automatic through the wrapped decorator, the only design obligation is (a) import `tool` from `agent_observability`, (b) keep the `trace.*` calls in `search_all_policies` and the routing tools. No manual `PutTraceSegments`.

---

## 10. Data flow — the three graded scenarios

Traceability from KB §9 step 2 / `README.md` Task 7. Rule 1 (`initialize_session`) runs first in all three (KB §11 #12).

1. **Return** ("…return my order ORD-27176", CUST-001): Orchestrator `initialize_session` → `route_to_inventory_agent` (order status + tier) → `route_to_refund_agent` (eligibility from inventory context + tier window) → `route_to_communication_agent`. WorkflowState accrues `inventory_agent`, `refund_agent`, `communication_agent`.
2. **Policy** ("return policy for premium customers?"): `initialize_session` → `route_to_policy_agent` → `search_all_policies` fans out to 3 retrievers in parallel → `route_to_communication_agent`. WorkflowState accrues `policy_agent`, `communication_agent`.
3. **Math** ("5 items at $29.99 with 10% off"): `initialize_session` → compute, skipping Inventory/Policy/Refund → `route_to_communication_agent` (Rule 5; round only after the full calc, `src/agent_orchestrator.py` line 495). Guardrail allows the arithmetic (KB §7; design §4 narrow pricing-negotiation definition).

---

## 11. Error handling and idempotency (as already designed in the starter)

Design preserves the patterns the starter establishes rather than adding new ones (KB §1):

- WorkflowState writes are conditional with up to 3 retries (`_update_workflow_state`, lines 158–205).
- KB retrieval failures return a structured `source:'error'` passage rather than raising, so a single KB outage degrades gracefully (`src/bedrock_kb_retrieval.py` except block); `test_5_4` treats `source=='error'` as a miss.
- `create_guardrail` / `configure_memory` short-circuit on an existing resource (idempotent re-runs) — stub-provided.
- `deploy` is safe to re-run: unchanged runtime left alone, changed one updated in place (`README.md` Task 3; stub docstring).

No new retry/backoff/circuit-breaker logic is introduced (would exceed the instructions — KB §1).

---

## 12. What is explicitly out of scope (STUB) in the design

The four stand-outs stay STUB: adversarial guardrail screenshots, `agent-sessions` + `DynamoDbSessionStorage`, CloudWatch dashboard, Cognito + frontend (KB §9 scope table; KB §12 #5 default none). Design adds no structure for them — they are README prose only (requirements R6.1–R6.4). The optional Gateway Step 6 in the deploy pipeline is left as the starter ships it (skips when no Lambda tool functions are configured — `README.md` Deployment Pipeline; stub Step 6); design does not implement it (KB §9: not required for the 120-point suite).

---

## 13. Open items (do not block)

- Due date / Submit Project page: UNVERIFIED (KB §11 #13, §12 #3). Does not affect design.
- `tool_registry.registry` vs `.tool_names`: the starter's own test uses `agent.tool_registry.registry` (`tests/test_agent.py` lines 142–150); design relies on that accessor for the R1.2/R1.4 build checks. If the installed Strands also exposes `.tool_names`, either works; no design dependency on the alias.
