# Product Requirements Document (PRD)

**Title:** Wave Creation Agent for SAP EWM Outbound Deliveries
**Date:** 2026-11-18
**Owner:** Warehouse Operations
**Solution Category:** AI Agent

---

## Product Purpose & Value Proposition

**Elevator Pitch:**
Warehouse planners today manually create waves for outbound deliveries in SAP EWM — every single day. This agent eliminates that work entirely by running automatically on a daily schedule, fetching open deliveries, grouping them by shipping date, and creating the waves directly in SAP EWM.

**Business Need:**
In SAP S/4HANA Cloud Private Edition with Extended Warehouse Management (EWM), wave creation is a prerequisite for picking and shipping outbound deliveries. Currently this is done manually by warehouse planners, introducing risk of omission, delays, and missed shipments. There is no native scheduling capability in EWM to automate this step.

**Expected Value:**
- Zero manual daily effort for wave creation
- Consistent, on-time wave creation ensuring deliveries are always picked on schedule
- Reduced missed shipments caused by forgotten or delayed wave creation
- Full auditability of waves created with structured logging

**Product Objectives (Prioritized):**
1. Fully automate daily wave creation — no manual steps required
2. Ensure waves are created reliably every day for all open outbound deliveries grouped by shipping date
3. Provide traceable, structured logs of all wave creation activity for auditability

---

## Business Metrics

| Metric | Baseline | Target | Timeline | Process / Capability | Source |
|--------|----------|--------|----------|----------------------|--------|
| Manual effort for wave creation | — | Fully automated (0 manual steps per day) | — | Outbound Warehouse Management | user |
| Daily wave creation frequency | Ad-hoc / manual | Once daily via scheduled agent run | — | Outbound Warehouse Management | user |

---

## User Profiles & Personas

### Primary Persona: Warehouse Planner / Supervisor

Marcus is a 38-year-old warehouse planner at a distribution centre running SAP EWM. He is responsible for ensuring that all outbound deliveries are picked and shipped on time. Every morning he logs into SAP, filters open deliveries, and manually creates waves — a task that takes 20-40 minutes and must not be forgotten. When he is on leave or overwhelmed by urgent issues, waves are sometimes created late or not at all, causing shipment delays. He is comfortable with SAP transactions but has no coding background. He wants processes to "just work" without him having to intervene every day.

### Secondary Persona: Warehouse Operations Manager

Priya is a 45-year-old operations manager overseeing the warehouse. She is accountable for on-time delivery KPIs and warehouse throughput metrics. She does not interact with the agent directly but expects full visibility into whether waves were created each day and for which deliveries. She needs exception reports when wave creation fails so she can act quickly.

---

## User Goals & Tasks

### For Marcus (Warehouse Planner):

**Goals:**
- Never miss a wave creation cycle, even when absent or busy
- Spend zero time on routine daily wave creation

**Key Tasks:**
- Review the daily wave creation log to confirm all deliveries were processed
- Investigate and resolve exceptions flagged by the agent

### For Priya (Operations Manager):

**Goals:**
- Maintain on-time shipment rates
- Have full visibility into daily wave creation status

**Key Tasks:**
- Review structured logs and exception reports
- Escalate unresolved failures that could impact shipments

---

## Product Principles

1. **Automation First**: The agent operates without human input on the happy path. Human involvement is only required when exceptions arise.
2. **No Silent Failures**: Every failure — whether an API error or a delivery skipped — must be logged explicitly. The agent must never fail silently.
3. **Data Integrity**: The agent reads and creates; it never modifies or deletes existing EWM data beyond wave assignment.
4. **Idempotency**: Running the agent more than once on the same day must not create duplicate waves for already-waved deliveries.

---

## Business Context

**Current State:**
Warehouse planners manually create waves in SAP EWM every day by filtering open outbound deliveries and assigning them to a wave. This is a repetitive, error-prone task with no native automation. If the planner is absent or delayed, waves are missed and shipments are delayed.

**Strategic Alignment:**
Automating repetitive operational tasks in warehouse management directly supports warehouse efficiency and on-time delivery goals. This aligns with the broader drive to reduce manual touchpoints in the Plan to Fulfill process.

**Success Criteria:**
See the Business Metrics section above.

---

## Goals and Non-Goals

### Goals (In Scope)

- Automatically retrieve all open outbound deliveries in SAP EWM not yet assigned to a wave
- Group deliveries by requested shipping / goods issue date
- Create one wave per date group in SAP EWM via OData API
- Run on a daily schedule without manual intervention
- Log all wave creation results with wave IDs, delivery counts, and any errors

### Non-Goals (Out of Scope)

- Manual or on-demand wave creation triggered by a user
- Inbound delivery wave management
- Wave release, confirmation, or post-creation steps
- Carrier assignment or transport planning
- A user-facing UI or dashboard (logs only)

---

## Requirements

### Must-Have Requirements

**R1: Retrieve Open Outbound Deliveries**

- **Problem to Solve**: The agent needs to know which deliveries are open and not yet waved in EWM before it can act.
- **User Story**: As a scheduled agent, I need to query SAP EWM for all open outbound deliveries without a wave assignment so that I can process them.
- **Acceptance Criteria**:
  - Given the agent is triggered, when it calls the EWM OData API, then it returns a list of open outbound deliveries with their shipping dates
  - Given no open deliveries exist, when the agent runs, then it logs a "no deliveries to process" message and exits cleanly
- **Maps to Objective**: Objective 1 — full automation
- **Priority Rank**: 1

**R2: Group Deliveries by Shipping Date**

- **Problem to Solve**: Deliveries for the same shipping date should be batched into one wave to optimise picking efficiency.
- **User Story**: As a scheduled agent, I need to group open deliveries by their requested shipping/goods issue date so that one wave covers all deliveries for the same date.
- **Acceptance Criteria**:
  - Given a list of deliveries with varying shipping dates, when the agent groups them, then each group contains only deliveries sharing the same date
  - Given deliveries with no shipping date set, when the agent encounters them, then they are logged as skipped with a reason
- **Maps to Objective**: Objective 2 — reliable, correct waves
- **Priority Rank**: 2

**R3: Create Waves in SAP EWM**

- **Problem to Solve**: Waves must be created in EWM so that picking can proceed; the agent must do this via API.
- **User Story**: As a scheduled agent, I need to create one wave per delivery date group in SAP EWM so that picking teams can begin work without manual planner intervention.
- **Acceptance Criteria**:
  - Given a delivery group, when the agent calls the EWM wave creation API, then a wave is created and its wave ID is returned
  - Given an API failure, when the agent encounters an error, then it logs the error with details and continues processing remaining groups
- **Maps to Objective**: Objective 1 & 2
- **Priority Rank**: 3

**R4: Structured Result Logging**

- **Problem to Solve**: Planners and managers need visibility into what the agent did each day without having to query SAP directly.
- **User Story**: As a warehouse operations manager, I need to see a structured log of wave creation results so that I can confirm all deliveries were waved and act on any failures.
- **Acceptance Criteria**:
  - Given the agent completes a run, when it finishes, then it emits a structured log entry per wave created (wave ID, delivery count, shipping date)
  - Given any delivery is skipped or any API call fails, when this occurs, then the log includes the reason and affected delivery IDs
- **Maps to Objective**: Objective 3 — auditability
- **Priority Rank**: 4

**R5: Daily Scheduled Execution**

- **Problem to Solve**: The agent must run automatically every day without a human trigger.
- **User Story**: As a warehouse planner, I need the agent to run automatically each day so that I do not need to remember to trigger wave creation manually.
- **Acceptance Criteria**:
  - Given the agent is deployed, when the daily cron schedule fires, then the agent executes end-to-end without manual input
  - Given the agent fails mid-run, when this occurs, then it logs the failure and exits without leaving partial state
- **Maps to Objective**: Objective 1
- **Priority Rank**: 5

**R6: Error Handling and Resilience**

- **Problem to Solve**: API calls can fail transiently; the agent must not crash or leave work half-done.
- **User Story**: As a warehouse operations manager, I need the agent to handle API errors gracefully so that partial failures do not silently block wave creation for all delivery groups.
- **Acceptance Criteria**:
  - Given a transient API error, when it occurs, then the agent retries up to 3 times before marking the group as failed
  - Given a persistent failure, when all retries are exhausted, then the agent logs the failure and continues with the next group
- **Maps to Objective**: Objective 2 & 3
- **Priority Rank**: 6

---

## Non-Functional Requirements

### Performance

- **Latency**: Full daily run completes within 5 minutes for up to 500 open deliveries
- **Throughput**: Handles up to 500 deliveries per run, batched into waves

### Reliability

- **Availability**: Agent runs must not silently fail; every run must produce a log entry
- **Fallback**: On complete API unavailability, the agent logs the outage and exits cleanly without data corruption

### Explainability

- **Traceability**: Each wave creation is logged with the delivery IDs included, the shipping date, and the resulting wave ID
- **Decision Logging**: All grouping decisions and skipped deliveries are recorded with reasons
- **Uncertainty Communication**: Deliveries with missing or ambiguous shipping dates are flagged explicitly in logs

---

## Solution Architecture

**Architecture Overview:**
A Python AI agent (A2A protocol) is deployed on SAP BTP. It is triggered daily by a cron-based scheduler. The agent calls SAP S/4HANA EWM OData APIs to retrieve deliveries and create waves. All results are written to structured logs accessible via BTP.

**Key Components:**

- **Wave Creation Agent** — Python AI agent (A2A) containing the core business logic: retrieve, group, create, log
- **SAP EWM OData APIs** — `WAREHOUSEOUTBDELIVERYORDER_0001` (read warehouse outbound delivery orders) and `API_OUTBOUND_DELIVERY_SRV_0002` (read/update outbound deliveries)
- **BTP Scheduler** — Cron-based trigger for daily automated execution
- **Structured Logger** — OpenTelemetry-instrumented logging for milestone tracking and auditability

**Integration Points:**

- SAP S/4HANA EWM (Cloud Private Edition): read open outbound deliveries, create waves — outbound API calls, daily frequency
- SAP BTP: agent runtime and scheduler hosting

**Deployment Environments:**

- Dev: test with mock EWM API responses
- QA: integration-tested against EWM sandbox
- Prod: live daily execution against EWM production

---

### Agent Extensibility & Instrumentation

**Agent Extensibility:**
- The agent is designed with extension points so that future grouping strategies (e.g., by warehouse zone, carrier, or priority) can be plugged in without rewriting core logic
- The delivery query filter is externalised as a configuration parameter to allow future criteria changes
- Pre-wave and post-wave hooks are available for custom validation or notification steps

**Business Step Instrumentation:**
- All four business milestones (see Milestones section) are instrumented with structured log statements using OpenTelemetry spans
- Log pattern: `[MILESTONE_ID].[achieved|missed]: [description]`
- Each span captures the number of deliveries processed at that step for observability

---

### Automation & Agent Behaviour

**Automation Level:** Autonomous agent (fully automated, no human in the loop on the happy path)

**Actions the system performs without human approval:**
- Query SAP EWM for open outbound deliveries
- Group deliveries by shipping date
- Create waves in SAP EWM

**Actions that require human review or approval:**
- None on the happy path; failures are logged and require human follow-up

**Model or engine used:** Python AI agent (A2A protocol) on SAP BTP; SAP Generative AI Hub for any reasoning steps

**Knowledge & data sources accessed:**
- SAP EWM (via `WAREHOUSEOUTBDELIVERYORDER_0001`): open outbound delivery orders — read-only
- SAP EWM (via `API_OUTBOUND_DELIVERY_SRV_0002`): outbound delivery details — read/write for wave assignment

**Tools or connectors invoked:**
- `WAREHOUSEOUTBDELIVERYORDER_0001` OData API (read-only): fetch open warehouse outbound delivery orders pending wave assignment
- `API_OUTBOUND_DELIVERY_SRV_0002` OData API (read/write): read delivery details and trigger wave creation in EWM

**Guardrails & fail-safes:**
- Never modify deliveries that are already goods-issued or in a completed status
- Never create duplicate waves for delivery groups that already have a wave assigned
- On API error after 3 retries, log and skip the affected group — do not crash the entire run
- Log every skipped delivery explicitly with the reason

---

## Milestones

### M1: Delivery Retrieval

- **Description**: The agent has successfully queried SAP EWM and obtained the list of open outbound deliveries not yet assigned to a wave.
- **Achieved when**: The OData API call returns a non-empty result set with valid delivery records
- **Log on achievement**: `M1.achieved: delivery retrieval complete — {count} open deliveries fetched from EWM`
- **Log on miss**: `M1.missed: delivery retrieval failed or returned empty — no deliveries to process; check EWM connectivity`

### M2: Delivery Grouping

- **Description**: The agent has grouped the retrieved deliveries into date-based batches ready for wave creation.
- **Achieved when**: All deliveries with a valid shipping date are assigned to a date group
- **Log on achievement**: `M2.achieved: deliveries grouped into {group_count} date groups covering {delivery_count} deliveries`
- **Log on miss**: `M2.missed: delivery grouping produced no valid groups — all deliveries may be missing shipping dates`

### M3: Wave Creation

- **Description**: The agent has created waves in SAP EWM for all delivery date groups.
- **Achieved when**: At least one wave is successfully created in EWM via the OData API
- **Log on achievement**: `M3.achieved: {wave_count} wave(s) created in EWM — wave IDs: {wave_ids}`
- **Log on miss**: `M3.missed: wave creation failed for all groups — no waves created in EWM; check API availability`

### M4: Confirmation & Logging

- **Description**: The agent has completed its run and emitted a structured summary log covering all waves created, skipped deliveries, and any errors.
- **Achieved when**: The agent exits cleanly after emitting a complete run summary
- **Log on achievement**: `M4.achieved: run complete — {wave_count} waves created, {skip_count} deliveries skipped, {error_count} errors`
- **Log on miss**: `M4.missed: agent run did not complete cleanly — partial results logged; manual review required`

---

## Risks, Assumptions, and Dependencies

### Risks

- EWM OData APIs may require specific authorisation scopes not yet configured in BTP — setup effort required before first run
- Deliveries with missing shipping dates will be skipped — if many deliveries lack this field, wave coverage will be incomplete

### Assumptions (Validate These)

- SAP EWM OData APIs (`WAREHOUSEOUTBDELIVERYORDER_0001`, `API_OUTBOUND_DELIVERY_SRV_0002`) are accessible from BTP via the Business Connector
- Deliveries to be waved have their requested shipping / goods issue date populated in EWM
- One wave per shipping date per daily run is an acceptable grouping strategy

### Dependencies

- SAP S/4HANA Cloud Private Edition with EWM — production system access
- SAP BTP — runtime for agent deployment and scheduling
- EWM API credentials and authorisations configured in BTP Destination service

---

## Appendix

### References

- SAP API: Warehouse Outbound Delivery Order — Read, Update (A2X): `sap.s4:apiResource:WAREHOUSEOUTBDELIVERYORDER_0001:v1`
- SAP API: Outbound Delivery (A2X): `sap.s4:apiResource:API_OUTBOUND_DELIVERY_SRV_0002:v2`
- SAP Help: Extended Warehouse Management — Outbound Warehouse Management
- SAP BTP: A2A Agent Protocol documentation
