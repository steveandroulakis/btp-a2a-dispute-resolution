# BTP A2A Dispute Resolution

Multi-agent orchestration demo. Three agents (SAP, Google, Azure) collaborate via A2A protocol to resolve a customer dispute. SAP's agent runs on BAF (Business Agent Foundation), wrapped by a CAP service connector.

## Why we care

This is the codebase for **Workshop Hour 3**. We're forking the SAP agent connector, replacing BafAgentClient's crash-vulnerable `while(true)` polling loop with a Temporal Workflow. See `../../workshop-exercises.md` for full exercise spec.

**The key file:** `agents/sap-agent-builder-a2a/agent-builder-a2a-agent-connector/srv/BafAgentClient.ts` — `while(true)` loop holds `chatId`, `historyId`, loop position in memory. Crash = everything lost.

**The BAF agent itself** (`agents/sap-agent-builder-a2a/agent-builder-agent-exports/dispute-resolution-agent.json`) has `"tools": []` — it's pure LLM roleplay (GPT-4o faking S/4HANA lookups via prompt instructions). No real API calls behind BAF.

## Architecture

```
User → Joule (orchestrator on BAF)
  → A2A Router (discovers agents via ORD)
  → agent-builder-a2a-agent-connector (CAP service)
    → BAFAgentExecutor.execute()
      → BafAgentClient.invokeAgentSync()  -- creates BAF chat, sends message
      → BafAgentClient.triggerStatusUpdate()  -- while(true) polling loop ← CRASH VULNERABLE
        → GET /api/v1/Agents/{id}/chats/{chatId}?$select=state
        → switch(state): pending|running|success|error
        → publish A2A events via ExecutionEventBus
        → sleep(1500)
        → loop
```

Three agents in the scenario:
- **Dispute Resolution (SAP/BAF)** — confirms invoice vs shipment data. THIS is what we're making durable.
- **Warehouse Insights (Google/ADK on Cloud Run)** — reports 900 shipped not 1000. Python.
- **Dispute Email (Azure/LangGraph on Web Apps)** — drafts resolution email per policy. Python.

## Project structure

```
agent-catalog/                    # ORD aggregator + A2A router (CAP, TS)
  srv/a2a-router.ts              # routes A2A requests to discovered agents
  srv/ord-aggregator-service.ts  # fetches agent cards from all 3 platforms
agents/
  sap-agent-builder-a2a/
    agent-builder-agent-exports/
      dispute-resolution-agent.json  # BAF agent config (GPT-4o, no tools, roleplay)
      orchestrator.json              # Joule orchestrator config
    agent-builder-a2a-agent-connector/  # ← THIS IS THE FOCUS
      srv/
        server.ts            # A2A Express app setup, agent card, wires up executor
        AgentExecutor.ts     # BAFAgentExecutor — calls invokeAgentSync + triggerStatusUpdate
        BafAgentClient.ts    # THE KEY FILE — polling loop
        baf/AgentClient.ts   # HTTP client + OAuth token fetching for BAF
        well-known/document.json  # ORD discovery document
  gcp-adk-a2a/              # Google warehouse agent (Python, FastAPI, ADK)
  azure-ai-foundry-a2a/     # Azure email agent (Python, FastAPI, LangGraph)
```

## Running locally

### Agent Catalog
```bash
cd agent-catalog
npm install
npm run watch  # port 4005
```
Needs `.cdsrc.json` (copy from `.cdsrc-sample.json`) with agent host URLs.

### SAP Agent Connector
```bash
cd agents/sap-agent-builder-a2a/agent-builder-a2a-agent-connector
npm install
npm run watch  # hybrid profile, needs BAF credentials
```
Needs:
- `.cdsrc.json` (copy from `.cdsrc-sample.json`) — `agentId`
- `.cdsrc-private.json` (copy from `.cdsrc-private-sample.json`) — BAF OAuth credentials (`clientid`, `clientsecret`, `tokenUrl`, `apiUrl`, `agent_api_url`)

### We won't have BAF credentials

Workshop participants (and us) likely won't have live BAF access. The forked version will include a **mock BAF server** — simple Express app simulating the state machine (`pending → running → success`). Configurable via env var to point at real BAF or mock.

## Key dependencies

| Package | Role |
|---------|------|
| `@sap/cds` ^9 | CAP framework (Node.js app model) |
| `@a2a-js/sdk` ^0.2.2 | A2A protocol (agent cards, task events, execution) |
| `@cap-js/ord` ^1.3.3 | ORD discovery support |
| `axios` ^1.9.0 | HTTP client for BAF API |
| `express` ^4 | Web server |

Node 22.x.x required.

## Deployment (not needed for workshop)

Cloud Foundry via MTA. Each component has `mta.yaml` + `npm run deploy`. BAF service binding (`unified-agent-runtime-dev`) required for the SAP agent connector. GCP/Azure agents deploy to Cloud Run / Web Apps respectively.

## References

- `../../references/baf-agent-client-analysis.md` — crash vulnerability analysis + Temporal mapping
- `../../references/temporal-ai-sdk-and-sap-adapter.md` — how Temporal AI SDK + SAP adapter compose
- `../../resources/baf-architecture-diagram.md` — mermaid diagram of BAF/Agent Builder/A2A architecture
- `../../workshop-exercises.md` — exercise spec for all 4 hours
