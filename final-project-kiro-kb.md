# nd905 Final Project: Kiro Knowledge Base

**Course:** Future AWS Agent Engineer (nd905), Course 4: Multi-Agent Systems and Protocol Integration on AWS
**Project:** Multi-Agent E-commerce RAG (NovaMart customer support)
**Project name:** Nova Concilium (Latin for "council": specialists who each weigh in, with an orchestrator)
**Owner:** La Shara Cordero. Shara directs, validates, and decides. Kiro builds.
**Captured:** Oct 25, 2026, from the eight course pages Shara pasted plus the agent graph image.

## Repos

| Repo | URL | Role |
|---|---|---|
| Udacity starter (source) | https://github.com/udacity/cd14764-aws-agentic-c3-classroom/tree/main/project | Read-only source of the starter files. Starter code is in `project/starter/` |
| Shara's project repo (home) | https://github.com/earlgreyhot1701D/nova-concilium | Where the project lives and gets submitted. Empty as of Oct 25, 2026 |

- Local folder: `C:\Users\corde\projects\nova concilium`
- Starter commit seen Oct 25, 2026: `7ea6508` (Sep 29, 2026). Kiro records the commit it copies from.
- Copy the contents of `project/starter/` into the project repo. Do not fork or push to the Udacity repo.
- Files confirmed in `project/starter/` on Oct 25, 2026: `config.py`, `requirements.txt`, `.env.example`, `.gitignore`, `README.md`, `src/` (`agent_orchestrator.py`, `agent_observability.py`, `bedrock_kb_retrieval.py`, `demo.py`, `agent_utils.py`, `agentcore_cli.py`), `infrastructure/` (`cleanup.py`, `starter_stack.yaml`, `seed_data.py`), `tests/test_agent.py`, `agentcore/` (`agentcore.json`, `aws-targets.json`, `.gitignore`), `diagrams/` (three PNGs).
- Not covered by the course pages: `project/starter/README.md`, `project/README.md`, the `agentcore/` folder, and `diagrams/`. Kiro reads them and reports any instruction that disagrees with this document.

---

## 0. How Kiro uses this document

1. Read the whole document before writing anything.
2. Start spec-driven development: requirements, then design, then tasks. Each stage goes to Shara for approval before the next.
3. Sections 1 through 12 are the build. Section 13 is for the dev.to article and is **not** build input. Ignore it for the spec.

---

## 1. Standing rules (apply to every task)

### MUST
- **Follow the course instructions as written.** No redesigns, no "better" approaches, no substitutions. On a past project we deviated from the instructions and Shara had to redo part of the work. Any deviation needs Shara's approval in writing first.
- **Only edit the files the instructions allow:** `src/agent_orchestrator.py` and `.env`. (Plus a README and `evidence/` folder for the submission, see Section 10.)
- **Use config constants.** `config.ORCHESTRATOR_MODEL_ID`, `config.WORKER_MODEL_ID`, `config.ORDERS_TABLE`, `config.CUSTOMERS_TABLE`, `config.RETURNS_KB_ID`, `config.SHIPPING_KB_ID`, `config.WARRANTY_KB_ID`, `config.AGENT_LOG_GROUP`, `config.AGENTCORE_ROLE_ARN`. Never hardcode a model ID, table name, or ARN. The rubric grades this.
- **Every checkpoint is a command plus its pasted output.** If the output cannot be pasted, the checkpoint has not passed. Never mark `[x]` from reasoning alone.
- **One task per prompt.** Propose, wait for Shara's approval, then implement. DO NOT refactor code outside the current task.
- **Name the source file for every fact you use.** If no file supplies a value, mark it `UNVERIFIED` and stop to ask. Do not invent any value that should come from a file.
- **When sources disagree, stop and report it.** Use Section 2 to decide which source wins. If Section 2 does not settle it, ask Shara.

### NEVER
- Modify these files (the instructions say "Do not modify"): `src/agent_utils.py`, `src/agent_observability.py`, `src/bedrock_kb_retrieval.py`, `src/demo.py`, `config.py`, `tests/test_agent.py`, `src/agentcore_cli.py`.
- Create the Knowledge Bases by CLI or SDK. Phase 3 says AWS Console, and the rubric says "created in the AWS Console." Shara does this step. Kiro gives her the checklist.
- Make a direct `create_agent_runtime()` SDK call. Deployment goes through `agentcore_cli`.
- Use the `DRAFT` guardrail version. Publish and use a numbered version.
- Rename AWS resources to match the project name. Stack name `udacity-agentcore`, KB names `novamart-*-policy-kb`, and the `NovaMart-Orchestrator` trace node stay exactly as the course gives them.
- Put API keys or credentials in code. `.env` stays out of git.
- Build any stand-out item unless Shara turns it on (Section 9).

---

## 2. Source authority

| Question | Source that wins |
|---|---|
| File names, function names, signatures, what a helper actually does | The starter repo files on disk |
| What gets graded | The Rubric page. Where the rubric is stricter than a phase page, do the rubric's version |
| How to do a step | The Phase page for that step |
| Anything only in the agent graph image | Lowest authority. The image disagrees with the text on models (see Section 11) |

Lesson from Courses 2 and 3: when Udacity's pages disagree, the checked-in files are the stronger evidence. Read the file before trusting the page.

**Before writing requirements:** list the repo and paste the output. Confirm every file named in this document exists. Report any that are missing.

---

## 3. What we are building

NovaMart is a fictional e-commerce company. Its support team handles order status, returns and refunds, and policy questions by hand. We build the AI layer that automates this.

A five-agent system on **Amazon Bedrock AgentCore** and the **Strands Agents SDK**, in an Orchestrator to Workers hierarchy:

| Agent | Job |
|---|---|
| OrchestratorAgent | Understands intent, routes to the right specialist, manages WorkflowState |
| InventoryAgent | Retrieves order and customer data from DynamoDB. Reports facts only, no decisions |
| PolicyAgent | Answers policy questions with parallel multi-agent RAG across three Knowledge Bases |
| RefundAgent | Makes return eligibility decisions from order data and policy rules |
| CommunicationAgent | Composes the final customer-facing response |

Deployed to AgentCore Runtime with Bedrock Guardrails, AgentCore Memory for multi-turn sessions, and CloudWatch plus X-Ray observability.

### Deliverables

| # | Deliverable | Source page |
|---|---|---|
| 1 | Completed `src/agent_orchestrator.py`, all TODOs (Tasks 2, 3, 4, 6) | Project Overview |
| 2 | Three Bedrock Knowledge Bases, created in the Console, synced (Task 5) | Project Overview |
| 3 | Populated `.env`: three KB IDs, AgentCore Runtime ARN, Guardrail config | Project Overview |
| 4 | Screenshot: X-Ray Service Map showing the full Orchestrator to Worker trace | Project Overview, Phase 4 |
| 5 | Screenshot: all tests passed, showing "120 scores" | Phase 4 only (see Section 11, item 6) |

---

## 4. Environment setup

**Where it runs:** Shara's local machine and her own AWS account (confirmed by Shara, Oct 25, 2026). The Environment Setup page allows local development: Python 3.12+, AWS CLI configured for `us-east-1`.

**Tool prerequisites** (from starter `README.md` Prerequisites, confirmed Oct 25, 2026):

| Tool | Version | Checkpoint |
|---|---|---|
| Python | 3.12 (matches the deployed runtime; `agentcore.json` sets `PYTHON_3_12`) | `python --version` |
| Node.js | 20+ | `node --version` |
| uv | any current | `uv --version` |
| AgentCore CLI | `npm install -g @aws/agentcore@0.30.0` | `agentcore --help`. If it shows toolkit-style help (`agentcore configure`, `agentcore launch`), that is the old Python toolkit: `pip uninstall bedrock-agentcore-starter-toolkit` |

The runtime builds for arm64 and Python 3.12. `agentcore deploy` downloads matching wheels with `uv`, so it needs internet access.

| Step | Command or action | Checkpoint |
|---|---|---|
| 1 | `python -m venv venv`, activate (`venv\Scripts\activate` on Windows), `pip install -r requirements.txt` | Paste `python --version` and `pip list` showing `strands-agents` |
| 2 | Deploy the stack: `aws cloudformation deploy --template-file infrastructure/starter_stack.yaml --stack-name udacity-agentcore --capabilities CAPABILITY_NAMED_IAM --region us-east-1` | `aws cloudformation describe-stacks --stack-name udacity-agentcore --query "Stacks[0].StackStatus" --region us-east-1` returns `CREATE_COMPLETE` |
| 3 | `python infrastructure/seed_data.py` | Paste the output |
| 4 | `cp .env.example .env` | File exists, and `.env` is listed in `.gitignore` |
| 5 | `python config.py` | Paste the configuration table. "(not yet created)" fields are expected at this point |

**What the stack creates** (from the Environment Setup page; Kiro confirms against `starter_stack.yaml`):
- DynamoDB: `udacity-agentcore-orders`, `udacity-agentcore-customers`, `udacity-agentcore-workflow-state`
- S3 bucket for policy documents
- S3 Vectors bucket with three indexes: `returns-policy-index`, `shipping-policy-index`, `warranty-policy-index`
- IAM execution role for AgentCore Runtime
- CloudWatch log group
- On the first runtime deploy, the AgentCore CLI uses the CDK bootstrap assets bucket

**Windows note from Course 3:** the teardown script never ran on Windows because WSL had no `aws` or `uv` on PATH. Before Step 2, confirm which shell runs every command (PowerShell or WSL) and that `aws` works there.

---

## 5. Architecture

### Agent roles (System Architecture page, matches the rubric)

| Agent | Model constant | Temperature | Tools |
|---|---|---|---|
| OrchestratorAgent | `config.ORCHESTRATOR_MODEL_ID` (Claude Haiku 4.5) | 0.0 | `initialize_session`, `route_to_inventory_agent`, `route_to_policy_agent`, `route_to_refund_agent`, `route_to_communication_agent` |
| InventoryAgent | `config.WORKER_MODEL_ID` (Claude Sonnet 4.5) | 0.1 | `check_order_status`, `get_customer_tier`, `list_customer_orders` |
| RefundAgent | `config.WORKER_MODEL_ID` | 0.1 | `get_inventory_context`, `initiate_refund` |
| PolicyAgent (coordinator) | `config.WORKER_MODEL_ID` | 0.2 | `search_all_policies` |
| ReturnsPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | one tool calling `retrieve_from_knowledge_base()` with `config.RETURNS_KB_ID` |
| ShippingPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | same, `config.SHIPPING_KB_ID` |
| WarrantyPolicyRetrieverAgent | `config.WORKER_MODEL_ID` | 0.0 | same, `config.WARRANTY_KB_ID` |
| CommunicationAgent | `config.WORKER_MODEL_ID` | 0.3 | `get_full_workflow_context` |

### Orchestrator routing rules (all six go in the system prompt)

| Rule | Trigger | Action |
|---|---|---|
| 1 | Every request, always | Call `initialize_session` first |
| 2 | Order status, return, refund | Inventory agent first, then refund agent |
| 3 | Policy meaning (return windows, shipping rates, warranty terms) | Policy agent |
| 4 | Account questions ("what is my tier?", "am I premium?") | Inventory agent. Never the policy agent, which only knows policy text |
| 5 | Math or calculation | Straight to CommunicationAgent. Skip Inventory, Policy, Refund |
| 6 | Every request, always, last step | Communication agent composes the final reply |

**CRITICAL:** the Orchestrator never writes the customer response itself. `route_to_communication_agent` is always its last tool call. No exceptions.

### Shared WorkflowState

- One DynamoDB record per `session_id` in the workflow-state table. Each worker's output accumulates in it.
- **Optimistic locking:** every update passes `expected_version`, which must match the current version. This stops concurrent agents from overwriting each other.
- Use the three pre-written helpers. Do not write new ones:
  - `_create_workflow_state(session_id, customer_id)`
  - `_read_workflow_state(session_id)`
  - `_update_workflow_state(session_id, updates, expected_version)`
- Lifecycle: version 0 is created with `session_id` and `customer_id`, then the Inventory, Refund, Policy, and Communication columns get filled in. At version N, all agent columns are populated.

### Request flow for a return

Customer → Orchestrator (`initialize_session`) → Inventory (order status, tier) → Refund (eligibility) → Communication (final response) → Customer.

---

## 6. Phase 1: Build the multi-agent graph (Task 2)

**File:** `src/agent_orchestrator.py`

| Part | What to build | Notes |
|---|---|---|
| 1.A InventoryAgent | Three DynamoDB lookup tools. System prompt: a data gatherer that retrieves accurately and never decides eligibility | The Orders table has a composite key (`customer_id` + `order_id`), so `check_order_status` takes both |
| 1.B RefundAgent | One tool reads inventory findings from WorkflowState. One tool processes the return by updating the order record in DynamoDB. Prompt: always check inventory context first, then apply the window | Standard = **30 days**, Premium = **60 days** |
| 1.C PolicyAgent | Inside `build_policy_agent()`, create three full Strands Agent retrievers. `search_all_policies` runs all three at once. Coordinator prompt: always call `search_all_policies` first, then synthesize a grounded answer | Rubric requires `ThreadPoolExecutor(max_workers=3)` and `as_completed()`. Follow the starter's trace calls for where the parallel run goes |
| 1.D CommunicationAgent | One tool reads the full WorkflowState. Prompt: include all relevant findings, warm and professional tone | |
| 1.E OrchestratorAgent | Five routing tools. Each one reads WorkflowState, invokes its worker, then updates WorkflowState with the result | The six rules above |

**Tips from the page:** each `@tool` docstring is what the agent reads to decide when to use it, so docstrings must state purpose, parameters, and return value (graded). Every `build_*_agent()` ends with `return Agent(...)` and returns exactly one Agent.

**Checkpoint:** `python tests/test_agent.py task2`, output pasted.
**Live tests (once KBs exist):** `python src/demo.py`, `python src/agent_orchestrator.py test`, `python src/agent_orchestrator.py chat`.

---

## 7. Phase 2: Guardrail, deployment, and memory (Tasks 3 and 4)

**Implement these functions. Do NOT run the full deployment yet.** That happens in Phase 4.

### `create_guardrail()` (uses the `bedrock` client)

| Policy | Setting |
|---|---|
| Content | SEXUAL, VIOLENCE, HATE at **HIGH**. INSULTS, MISCONDUCT at **MEDIUM** |
| PII | **BLOCK** credit card numbers and SSNs. **ANONYMIZE** emails and phone numbers |
| Topic | **DENY** competitor products, pricing negotiations, legal threats |
| Word | Managed profanity list enabled |
| Messages | Friendly blocked messages for both inputs and outputs |
| Topic tier | `topicPolicyConfig.tierConfig = {"tierName": "STANDARD"}` |
| Cross region | Top-level `crossRegionConfig = {"guardrailProfileIdentifier": "us.guardrail.v1:0"}` |

- Define pricing negotiations as haggling or asking to change an advertised price. **Allow** calculations using a given price and discount. (The Classic tier blocked the project's own math scenario.)
- Call `create_guardrail_version()`. Return `(guardrail_id, guardrail_version)`, using the numbered version.
- Final testing must show: the math question and answer are allowed; negotiation, competitor, and legal-threat examples are blocked.

### `deploy_to_agentcore_runtime()`

- Build runtime env vars: `AWS_REGION`, `PROJECT_NAME`, `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`, `AGENT_LOG_GROUP`, `GUARDRAIL_ID`, `GUARDRAIL_VERSION`.
- Pass them to `agentcore_cli.configure_runtime()` with `network_mode='PUBLIC'`, `protocol='HTTP'`, `execution_role_arn=config.AGENTCORE_ROLE_ARN`.
- Call `agentcore_cli.deploy()`.
- Return `agentcore_cli.deployed_runtime_arn()`.
- The supplied `_apply_guardrail()` attaches the guardrail to each agent's model. Do not replace it.

### `configure_memory(runtime_arn)`

- `agentcore_control.create_memory()` with `summaryMemoryStrategy` (SESSION_SUMMARY) and `eventExpiryDuration=7`.
- Include a descriptive **name and description** (rubric requirement, not on the Phase 2 page).
- The supplied code waits for ACTIVE. Return the `memoryArn`.

**Checkpoint:** `python tests/test_agent.py task4`, output pasted. `task3` cannot pass until Phases 3 and 4 are done (the rubric says "after all deployment prerequisites are complete").

---

## 8. Phase 3: Knowledge Bases (Task 5). Shara does this in the Console.

**No code changes.** Kiro's job here: produce this checklist for Shara, then verify with the test afterward.

1. **Verify the policy docs** are in S3 under `policies/returns/`, `policies/shipping/`, `policies/warranty/`. Command: `aws s3 ls s3://$(python -c "import config; print(config.POLICY_BUCKET)")/policies/ --recursive`
2. **Create three KBs:** Console → Amazon Bedrock → Knowledge Bases → Create.

| Setting | Returns | Shipping | Warranty |
|---|---|---|---|
| Name | `novamart-returns-policy-kb` | `novamart-shipping-policy-kb` | `novamart-warranty-policy-kb` |
| S3 data source prefix | `policies/returns/` | `policies/shipping/` | `policies/warranty/` |
| Embedding model | Titan Embed Text v2 | same | same |
| Vector store | S3 Vectors → use an existing vector bucket | same | same |
| Vector bucket | The Vector Bucket line from `python config.py` (`udacity-agentcore-vectors-...`) | same | same |
| Vector index | `returns-policy-index` | `shipping-policy-index` | `warranty-policy-index` |

   - **Self-managed KB. Do NOT choose Managed KB.** Managed creates a different vector bucket than the rubric expects, and you can't change the backing store afterward.
3. **Click Sync** on each KB. Queries return nothing until it's synced.
4. **Add IDs to `.env`:** `RETURNS_KB_ID`, `SHIPPING_KB_ID`, `WARRANTY_KB_ID`.

**Checkpoint:** `python tests/test_agent.py task5`, output pasted, including `test_5_parallel_retrieval`. It also reports which vector store each KB uses.

---

## 9. Phase 4: Observability, deploy, and test (Task 6)

### `configure_observability(runtime_arn)`
- Build a `loggingConfiguration` dict and pass it to the pre-written `apply_observability_config()`, **inside try/except** (rubric).
- `cloudWatchConfig`: `config.AGENT_LOG_GROUP`, `logLevel='INFO'`, `enabled=True`.
- `xRayConfig`: `enabled=True`, `samplingRate=1.0`.

**Checkpoint:** `python tests/test_agent.py task6`, output pasted.

### End-to-end
1. `python src/agent_orchestrator.py deploy`
2. Run the three scenarios and paste the output:

| Scenario | Expected routing |
|---|---|
| "I want to return my order ORD-27176" (as CUST-001) | Orchestrator → Inventory → Refund → Communication |
| "What is the return policy for premium customers?" | Orchestrator → Policy (3 parallel KB retrievers) → Communication |
| "How much are 5 items at $29.99 with 10% off?" | Orchestrator → Communication. Skip Inventory, Policy, Refund |

3. Guardrail checks: the math question is allowed; one negotiation, one competitor, and one legal-threat example are blocked. Paste the output.
4. `python tests/test_agent.py all`, output pasted.
5. **Screenshot A (Shara):** all tests passed, showing 120 scores.
6. **Screenshot B (Shara):** run `python src/agent_orchestrator.py test`, wait 30 to 60 seconds, then go to CloudWatch → X-Ray traces → Service map → Last 5 minutes. The map must show `NovaMart-Orchestrator` connected to the worker nodes, including PolicyAgent and the KnowledgeBase nodes.

### Scope labels for the stand-out suggestions

| Item | Label |
|---|---|
| Everything in Sections 4 through 9 | **MUST** |
| Adversarial guardrail inputs with screenshots of blocked responses | **STUB** until Shara decides. Note: Step 3 above already covers part of it |
| DynamoDB `agent-sessions` table plus `DynamoDbSessionStorage` | **STUB** |
| CloudWatch dashboard (invocations, latency per agent, guardrail triggers) | **STUB** |
| Cognito plus a frontend | **STUB** |

A STUB is a comment in the README describing the item and how it would be built. No code.

---

## 10. Submission package and README

- `evidence/`: pasted checkpoint outputs, one file per checkpoint, plus both screenshots.
- README sections:
  - What it is
  - How to run it
  - Architecture (agent graph plus routing table)
  - Test results
  - "Not built, and why": the stand-outs left as STUBs
  - Methodology: AI assisted, human approved. Name what Claude, Kiro, and Shara each did. Same pattern as Courses 2 and 3: permission was granted and the work is disclosed.
- Redact the 12-digit account ID and any runtime ARN or endpoint before anything goes in a public repo.

### Cost and teardown

- This project uses **S3 Vectors**, not OpenSearch Serverless (Phase 3 page: "serverless, no OpenSearch needed"). So the Course 3 cost trap should not apply. `UNVERIFIED` until Kiro confirms there's no OpenSearch resource in `starter_stack.yaml`.
- Cleanup per the Phase 4 page: `python infrastructure/cleanup.py` (dry run), then `python infrastructure/cleanup.py --yes`.
- **Before cleanup, Kiro reads `cleanup.py` and reports, with line numbers, whether it deletes each of these:**

| Resource | Created by |
|---|---|
| Three Knowledge Bases | Console, outside the stack |
| Guardrail | `create_guardrail()` |
| AgentCore Memory | `configure_memory()` |
| AgentCore Runtime | CLI deploy |
| CDK bootstrap stack and assets bucket | First runtime deploy |
| `udacity-agentcore` stack | CloudFormation |

  Anything it misses goes on a manual list.
- Teardown is done when every list command for those resource types returns empty, with the output pasted. Kiro confirms the exact CLI syntax with `aws <service> help` before running.
- Course 3 lesson: two Course 2 resources survived a teardown everyone believed was complete. Verify with output, never by memory.
- **Kiro's read of `cleanup.py` (Oct 25, 2026):** it deletes the KBs (plus the console-created KB service roles), Guardrail, Memory, Runtime (the `AgentCore-udacity-default` stack), the `udacity-agentcore` stack, and prefix-matched leftovers. Starter `README.md` line 363 confirms the grading suite is 120 points.
- **Manual teardown list (cleanup.py does not remove these):**

| Item | Why it stays | Action |
|---|---|---|
| `CDKToolkit` bootstrap stack | Left in place on purpose (`cleanup.py` docstring) | Shara decides. It is shared by any CDK project in the account |
| CDK assets bucket `cdk-hnb659fds-assets-<acct>-<region>` | Does not match the `udacity-agentcore-` prefix filter | Shara decides, together with the stack |
| CloudWatch Transaction Search and the X-Ray resource policy | Only removed with `--disable-transaction-search` | Run cleanup with that flag unless something else in the account uses it |

- **Timing:** the page says clean up after submitting, but a reviewer may ask for changes. Shara decides (Section 12).

---

## 11. Conflicts and gaps in the course pages

Found by comparing the eight pasted pages. Kiro: confirm each against the repo files and report.

| # | Conflict | Resolution |
|---|---|---|
| 1 | The agent graph image labels models **Claude 3 Haiku / Claude 3 Sonnet**. The Architecture table, Phase 1, and the rubric say **Haiku 4.5 / Sonnet 4.5** | Use the config constants only. Kiro pastes the actual IDs from `config.py` |
| 2 | The image names the retrievers `ReturnsPolicyRetriever` etc. The rubric names them `ReturnsPolicyRetrieverAgent`, `ShippingPolicyRetrieverAgent`, `WarrantyPolicyRetrieverAgent` | Use the rubric names |
| 3 | The image draws the three retrievers as a dotted chain. The text says they run in parallel | They are parallel siblings under `search_all_policies` |
| 4 | Phase 1 says "ThreadPoolExecutor." The rubric says `ThreadPoolExecutor(max_workers=3)` and `as_completed()` | Use the rubric version |
| 5 | Phase 2 memory has no name or description. The rubric requires both | Include both |
| 6 | The Overview lists four deliverables. Phase 4 adds a second screenshot showing "120 scores." "120 scores" is never defined | `UNVERIFIED`. Kiro reads `tests/test_agent.py` and reports what produces 120 |
| 7 | The Overview says TODO Tasks 2, 3, 4, 6 plus KB Task 5. No page describes a Task 1 | Mapping: Phase 1 = Task 2, Phase 2 = Tasks 3 and 4, Phase 3 = Task 5, Phase 4 = Task 6. Kiro confirms against the TODO markers in `agent_orchestrator.py` |
| 8 | The guardrail STANDARD tier and `crossRegionConfig` are on Phase 2 but not in the rubric | Do both. They don't conflict |
| 9 | The rubric wants try/except around `apply_observability_config()`. Phase 4 doesn't mention it | Include it |
| 10 | The `task3` test needs deployment, which needs Phases 3 and 4 | Expect `task3` to fail until the end. Don't "fix" it early |
| 11 | Python: the Local Machine section says 3.12+. Course 3's pages said 3.13 while the repo pinned 3.14 | Kiro pastes the actual pin from `requirements.txt` or `pyproject.toml` |
| 12 | The scenario routing table leaves out `initialize_session`. Rule 1 says it is always first | Rule 1 applies to all three scenarios |
| 13 | The Submit Project page and due date were not provided | Open item for Shara. (The starter repo URL is now in the Repos section.) |

---

## 12. Open items for Shara

| # | Item | Default if not answered |
|---|---|---|
| 1 | Project name | DONE: Nova Concilium |
| 2 | Own AWS account plus local machine, or the Udacity workspace | DONE: own account, local machine (confirmed Oct 25, 2026) |
| 3 | Submit Project page, due date | Repo URLs DONE (see Repos section). Due date still needed |
| 4 | Teardown timing: right after submitting, or after the project passes | After it passes. Also check what the resources cost while they sit |
| 5 | Which stand-out items, if any | None. All STUB |

---

## 13. For the dev.to article (NOT build input. Kiro ignores this section.)

### The two earlier projects

**Nova Trivium (Course 2, passed Aug 23, 2026).** A customer support chatbot on the AgentCore managed harness. Three routes, one system prompt: bug reports get filed to DynamoDB through a Gateway tool, platform questions get answered from an FAQ pasted into the prompt, and everything else is redirected to a human. The starter files were a generation behind the instructions, which were written for AgentCore while the starter was still Flows. It scored 0.929 on Bedrock Evaluations Correctness. The big lesson: a checkpoint you can describe is one an agent can fake.
Repo: https://github.com/earlgreyhot1701D/nova-trivium
Post: https://dev.to/earlgreyhot1701d/future-aws-agent-engineer-i-didnt-write-the-code-does-it-count-1epj

**Nova Adiutrix (Course 3, shipped Sep 20, 2026).** An AI support agent on AgentCore plus Strands. One agent with a Bedrock Knowledge Base, AgentCore Memory, six Gateway tools, Code Interpreter, and Browser. Order ownership is enforced in code, not in the prompt. 11 of 11 tests passed, and the account was torn down to zero the same day. The project built two of the three items on Trivium's "Not built, and why" list. The third, guardrails, was still not built.
Repo: https://github.com/earlgreyhot1701D/nova-adiutrix
Post: https://dev.to/earlgreyhot1701d/nova-adiutrix-my-second-agent-built-my-first-projects-to-do-list-3gb

**Thread to pick up:** the Adiutrix post ends with "The third item on that 'Not built, and why' list was guardrails. Still not built. I will let you know how project three goes." In the final project, a Bedrock Guardrail is a graded requirement.

### How the difficulty ramped

| | Course 2: Nova Trivium | Course 3: Nova Adiutrix | Final: Nova Concilium |
|---|---|---|---|
| Agents | 1 managed harness | 1 Strands agent | 5 agents plus 3 retriever sub-agents |
| Models | 1 (Nova Pro) | 1 (Nova 2 Lite) | 2 (Haiku 4.5 routes, Sonnet 4.5 works) |
| Where knowledge lives | FAQ pasted into the prompt | 1 Knowledge Base, OpenSearch Serverless | 3 Knowledge Bases on S3 Vectors, queried in parallel |
| State | None (single-turn tests) | AgentCore Memory | Shared DynamoDB WorkflowState with optimistic locking, plus AgentCore Memory |
| Routing | In the prompt | One agent picks its tools | Orchestrator with 6 written routing rules |
| Safety | Prompt only | Ownership check in code | Bedrock Guardrail (content, PII, topics, profanity) |
| Observability | Bedrock Evaluations | CloudWatch metric filter plus alarm | CloudWatch plus X-Ray service map across agents |

### Shara's notes
_(Add notes here as the build goes.)_
