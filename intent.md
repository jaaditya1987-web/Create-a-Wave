# Wave Creation Agent for EWM Outbound Deliveries

Automated AI agent for daily wave creation in SAP S/4HANA EWM

## Business challenge

Warehouse planners manually create waves for outbound deliveries in SAP S/4HANA Cloud Private Edition with Extended Warehouse Management (EWM). This is time-consuming, error-prone, and can lead to missed shipments when it is forgotten. The goal is to fully automate daily wave creation by having an AI agent retrieve open deliveries, group them by shipping/due date, and create waves in EWM automatically on a scheduled basis.

## Business Goals & Success Criteria

| Metric | Baseline | Target | Timeline | Process / Capability | Source |
|--------|----------|--------|----------|----------------------|--------|
| Manual effort for wave creation | — | Fully automated (0 manual steps per day) | — | Outbound Warehouse Management | user |
| Daily wave creation frequency | Ad-hoc / manual | Once daily via scheduled agent run | — | Outbound Warehouse Management | user |

## Key Milestones

1. **Delivery Retrieval** — Agent successfully queries SAP EWM for open outbound deliveries pending wave assignment
2. **Delivery Grouping** — Agent groups deliveries by shipping/due date into candidate wave batches
3. **Wave Creation** — Agent creates waves in SAP EWM for each delivery group via API call
4. **Confirmation & Logging** — Agent logs wave creation results (wave IDs, delivery counts, any errors) for traceability

## Business Architecture (RBA)

### End-to-End Process

Deliver Product to Fulfill

### Process Hierarchy

```
Deliver Product to Fulfill
└── Deliver Product to Fulfill (basic outbound)
    └── Manage warehouse and inventory (basic outbound) [BPS-348_012]
        └── Ship products
└── Deliver Product to Fulfill (outbound without dock appointment)
    └── Manage warehouse and inventory (outbound without dock appointment) [BPS-348_040]
        └── Ship products
└── Deliver Product to Fulfill (outbound, omnichannel commerce)
    └── Manage warehouse and inventory (outbound, omnichannel commerce) [BPS-348_020]
        └── Ship products
```

### Summary

The wave creation challenge maps to the "Manage warehouse and inventory (outbound)" sub-processes within the "Deliver Product to Fulfill" E2E process. SAP S/4HANA Cloud Private Edition with EWM provides the outbound warehouse management capabilities; the gap is the absence of intelligent, automated daily wave grouping and creation.

## Fit Gap Analysis

| Requirement (business) | Standard asset(s) found | API ORD ID | MCP Server ORD ID | MCP Server Version | Webhook API ORD ID | Data Product ORD ID | Gap? | Notes / assumptions |
|------------------------|------------------------|------------|-------------------|--------------------|--------------------|---------------------|------|---------------------|
| Retrieve open outbound deliveries from EWM | SAP S/4HANA Cloud Private Edition — Outbound Warehouse Management (SC5105) | `sap.s4:apiResource:WAREHOUSEOUTBDELIVERYORDER_0001:v1` | — | — | — | — | No | OData API available |
| Read warehouse orders and tasks | SAP S/4HANA Cloud Private Edition — Internal Warehouse Management (SC5130) | `sap.s4:apiResource:WAREHOUSEORDER_0001:v1` | — | — | — | — | No | OData API available |
| Read/update outbound deliveries | SAP S/4HANA Cloud Private Edition — Outbound Delivery Management (SC6221) | `sap.s4:apiResource:API_OUTBOUND_DELIVERY_SRV_0002:v2` | — | — | — | — | No | OData API available |
| Group deliveries by shipping date and create waves automatically | SAP S/4HANA Cloud Private Edition — EWM (no native scheduling/automation for wave creation) | — | — | — | — | — | Yes | No native automated wave creation scheduler; custom AI agent required |

### Key findings
- SAP S/4HANA Cloud Private Edition with EWM covers all outbound warehouse management capabilities needed
- Three OData APIs are available to read deliveries and warehouse orders; no MCP servers found for these APIs
- The core gap is the absence of an automated, intelligent wave creation scheduler — this requires a custom AI agent
- The agent will run on a daily schedule, grouping open deliveries by shipping/due date and calling EWM APIs to create waves
- No MCP servers are pre-built for these EWM APIs; the agent will call them directly via the available OData services

## Recommendations

### Automated Daily Wave Creation Agent for SAP EWM

#### Executive Summary

Python AI agent runs daily, creates EWM waves from open deliveries by date

#### Recommended Solution

A pro-code Python AI agent (A2A protocol) that runs on a daily schedule. The agent:
1. Calls the SAP S/4HANA EWM OData API to retrieve all open outbound deliveries not yet assigned to a wave
2. Groups those deliveries by their requested shipping / goods issue date
3. Creates one wave per date group via the SAP EWM API
4. Logs all created wave IDs and delivery assignments for auditability

The agent is deployed on SAP BTP and triggered via a scheduled job (cron). It uses the available OData APIs: `WAREHOUSEOUTBDELIVERYORDER_0001` and `API_OUTBOUND_DELIVERY_SRV_0002`.

#### Recommended solution category

AI Agent

#### Intent fit
92%
