# Specification: wave-creation-agent

> **Guidelines**: Read all applicable guidelines before executing ANY tasks below:
> - [guidelines.md](../guidelines.md) — Universal execution rules
> - [guidelines-agent.md](../guidelines-agent.md) — Universal agent patterns
> - [guidelines-agent-python.md](../guidelines-agent-python.md) — Python implementation details
> - [guidelines-agent-skills.md](../guidelines-agent-skills.md) — Runtime skills patterns
> - [guidelines-agent-mcp.md](../guidelines-agent-mcp.md) — MCP integration patterns

---

## Basic Setup

- [ ] Read `product-requirements-document.md` and `intent.md` before starting any implementation
- [ ] Bootstrap agent code in `assets/wave-creation-agent/` using the `sap-agent-bootstrap` skill (invoke from inside `assets/wave-creation-agent/`, use copy commands — do NOT create files manually)
- [ ] Install dependencies, validate the agent starts and responds at `/.well-known/agent.json`

---

## Runtime Skills

No runtime skills are needed for this agent. The wave creation logic is deterministic and self-contained: retrieve deliveries, group by date, create waves, log results. No multi-step approval workflows or domain-specific reference material required.

---

## Project-Specific Tasks

### Agent Identity & System Prompt

- [ ] Set the agent name to `wave-creation-agent` and description to `Automated daily wave creation agent for SAP EWM outbound deliveries`
- [ ] Write the system prompt in `assets/wave-creation-agent/app/agent.py` with the following instructions:
  - You are an automated warehouse management agent responsible for creating waves in SAP Extended Warehouse Management (EWM) for outbound deliveries.
  - Your job is to run daily: retrieve all open outbound deliveries not yet assigned to a wave, group them by their requested shipping/goods issue date, and create one wave per date group in SAP EWM.
  - IMPORTANT: You MUST use tools to retrieve live data. Never fabricate, guess, or invent delivery data or wave IDs.
  - When calling tools that support pagination, always set the page size parameter (top, limit, pageSize, etc.) to a maximum of 100 items to prevent context overflow.
  - If a tool returns an error, report the error message exactly as received. Do not suggest workarounds unless asked.
  - Deliveries with no shipping date set must be logged as skipped with the reason "missing shipping date" — do not include them in any wave.
  - Never assign a delivery to a wave if it is already in a completed or goods-issued status.
  - Never create duplicate waves for delivery groups that already have a wave assigned for that date.
  - After completing the run, emit a structured summary log covering all waves created, skipped deliveries, and any errors.

### Delivery Retrieval (R1)

- [ ] Implement tool/function `retrieve_open_deliveries` that:
  - Calls the warehouse outbound delivery order API (`WAREHOUSEOUTBDELIVERYORDER_0001`) to fetch all open outbound delivery orders not yet assigned to a wave
  - Filters for deliveries in an open/unassigned wave status
  - Returns a list of delivery records including: delivery ID, shipping date (requested goods issue date), warehouse number, and current wave assignment status
  - Handles empty results cleanly (returns empty list, does not raise)
  - On API error, retries up to 3 times with exponential backoff before raising
- [ ] Write unit test `tests/test_retrieve_open_deliveries.py` and run it immediately after writing

### Delivery Grouping (R2)

- [ ] Implement function `group_deliveries_by_date(deliveries: list) -> dict` that:
  - Groups delivery records by their requested shipping/goods issue date
  - Returns a dictionary mapping date string (ISO format) → list of delivery IDs
  - Deliveries with a missing or null shipping date are collected in a `skipped` list (not included in any group)
  - Logs a skip entry for each delivery excluded due to missing date: `"delivery {id} skipped: missing shipping date"`
  - Is a pure function — no API calls, no side effects
- [ ] Write unit test `tests/test_group_deliveries_by_date.py` covering: normal grouping, missing dates, all missing, empty input — run immediately after writing

### Wave Creation (R3)

- [ ] Implement tool/function `create_wave(shipping_date: str, delivery_ids: list) -> dict` that:
  - Calls the outbound delivery API (`API_OUTBOUND_DELIVERY_SRV_0002`) to create a wave for the given delivery group
  - Passes the list of delivery IDs and the target shipping date as wave parameters
  - Returns the created wave ID and confirmation details on success
  - On API failure, retries up to 3 times; if all retries fail, logs the error and returns a failure result without raising (so the calling loop continues with the next group)
  - Never creates a wave if the delivery group is already waved (check before calling the creation endpoint)
- [ ] Write unit test `tests/test_create_wave.py` covering: successful creation, API error with retry, all retries exhausted, duplicate wave guard — run immediately after writing

### Daily Run Orchestration (R5)

- [ ] Implement the main orchestration method `run_daily_wave_creation(self)` in the agent that:
  1. Calls `retrieve_open_deliveries` → emits M1 milestone log on success/failure
  2. Calls `group_deliveries_by_date` on the result → emits M2 milestone log
  3. Iterates over each date group and calls `create_wave` → emits M3 milestone log (after all groups processed)
  4. Emits M4 milestone log with final summary counts (waves created, deliveries skipped, errors)
  5. Returns a structured result dict with keys: `waves_created`, `deliveries_skipped`, `errors`
- [ ] The agent's `stream()` method must call `run_daily_wave_creation` via the agent's LLM/tool graph — it should not contain direct business logic (keep the graph as the orchestrator, with tools doing the work)

### Error Handling & Resilience (R6)

- [ ] Implement retry logic (max 3 attempts, exponential backoff) for all API tool calls
- [ ] Ensure a failure in one wave group does not abort processing of subsequent groups
- [ ] Ensure the agent never exits silently — every run must emit at least one log entry (even if no deliveries found)
- [ ] Validate that deliveries already in a completed/goods-issued status are excluded before grouping

### Idempotency Guard

- [ ] Before calling `create_wave`, check if a wave already exists for the given date group via the delivery query API
- [ ] If a wave already exists for that date group, log `"wave already exists for date {date}, skipping"` and do not call the creation endpoint

---

## MCP Tool Integration

> Read [guidelines-agent-mcp.md](../guidelines-agent-mcp.md) for complete MCP integration patterns.

### API Discovery Results (from Specification Step 2b)

Two OData API spec files are downloaded and available (Path A — no existing MCP server):

| API Name | ORD ID | Spec File |
|----------|--------|-----------|
| Warehouse Outbound Delivery Order — Read, Update (A2X) | `sap.s4:apiResource:WAREHOUSEOUTBDELIVERYORDER_0001:v1` | `specification/wave-creation-agent/api-specs/WAREHOUSEOUTBDELIVERYORDER_0001.edmx` |
| Outbound Delivery (A2X) | `sap.s4:apiResource:API_OUTBOUND_DELIVERY_SRV_0002:v2` | `specification/wave-creation-agent/api-specs/API_OUTBOUND_DELIVERY_SRV_0002.edmx` |

**MCP API scope:**

```text
MCP API: sap.s4:apiResource:WAREHOUSEOUTBDELIVERYORDER_0001:v1
MCP tool scope: read-only
Resources in scope: WarehouseOutboundDeliveryOrder, WarehouseOutboundDeliveryOrderItem
```

```text
MCP API: sap.s4:apiResource:API_OUTBOUND_DELIVERY_SRV_0002:v2
MCP tool scope: read-write
Resources in scope: A_OutbDeliveryHeader, A_OutbDeliveryItem, A_OutbDeliveryHeaderWave
```

- [ ] Invoke `mcp-translation-file` skill for each downloaded API spec file in `specification/wave-creation-agent/api-specs/` — do NOT manually create translation files
- [ ] Invoke `setup-solution` skill to create/register MCP server assets from the generated translation cards
- [ ] Wire MCP tool loading in `agent.py` using `get_mcp_tools()` from the `mcp_tools` module (never import directly from `sap_cloud_sdk.agentgateway`)
- [ ] Add MCP server dependencies to `assets/wave-creation-agent/asset.yaml` under `requires` — read the exact `ordId` from each generated MCP server's `asset.yaml` using `grep '^ordId:' assets/<mcp-server-asset-name>/asset.yaml` — NEVER invent or guess an ORD ID
- [ ] Generate `mcp-mock.json` using the `mcp-mock-config` skill (required before tests can run)

---

## Business Instrumentation

- [ ] Implement the following milestones in `assets/wave-creation-agent/app/agent.py` with structured logging and OpenTelemetry spans. Extract all business logic from `stream()` into the plain async helper `run_daily_wave_creation()` to avoid `GeneratorExit` context errors:

  | ID | Achieved log | Missed log |
  |----|-------------|-----------|
  | M1 | `M1.achieved: delivery retrieval complete — {count} open deliveries fetched from EWM` | `M1.missed: delivery retrieval failed or returned empty — no deliveries to process; check EWM connectivity` |
  | M2 | `M2.achieved: deliveries grouped into {group_count} date groups covering {delivery_count} deliveries` | `M2.missed: delivery grouping produced no valid groups — all deliveries may be missing shipping dates` |
  | M3 | `M3.achieved: {wave_count} wave(s) created in EWM — wave IDs: {wave_ids}` | `M3.missed: wave creation failed for all groups — no waves created in EWM; check API availability` |
  | M4 | `M4.achieved: run complete — {wave_count} waves created, {skip_count} deliveries skipped, {error_count} errors` | `M4.missed: agent run did not complete cleanly — partial results logged; manual review required` |

- [ ] Add an OpenTelemetry span for each milestone using `@tracer.start_as_current_span("M1-delivery-retrieval")` etc. on the respective helper methods (decorator form on regular async methods, context-manager form inside non-generator async functions only)
- [ ] Verify `bootstrap(app)` is called after `app = server.build()` in `main.py`

---

## Testing

> See [guidelines-agent-python.md](../guidelines-agent-python.md) for Python testing setup and patterns.

- [ ] `conftest.py` only sets `IBD_TESTING=true` — this causes the agent to run with mock MCP tool results during tests
- [ ] Unit tests (one per tool, run each immediately after writing):
  - `tests/test_retrieve_open_deliveries.py` — mock API returning deliveries, empty result, API error with retry
  - `tests/test_group_deliveries_by_date.py` — normal grouping, missing dates, all missing, empty input
  - `tests/test_create_wave.py` — success, API retry, all retries exhausted, duplicate guard
- [ ] Write `tests/test_integration.py`: end-to-end agent flow calling `run_daily_wave_creation` with mocked LLM responses and mocked MCP tools — assert returned dict contains `waves_created`, `deliveries_skipped`, `errors` keys and that M1–M4 log statements are emitted
- [ ] Run `pytest` from `assets/wave-creation-agent/` (no args) — if coverage < 70%, add tests until threshold met
- [ ] Run `pytest` again from `assets/wave-creation-agent/` (no args) to generate final `test_report.json`
- [ ] Verify `test_report.json` exists in `assets/wave-creation-agent/`
