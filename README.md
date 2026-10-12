# Cold-Chain Logistics AI Assistant

An agentic AI assistant for cold-chain logistics that helps control-tower operators investigate shipment risks through natural-language questions. The system is designed to bring fleet telemetry, weather context, and standard operating procedures (SOPs) into one workflow so operators can make faster, evidence-based decisions when cargo conditions may be at risk.

**Repository:** [himanshu2285/cold-chain](https://github.com/himanshu2285/cold-chain)

## The Problem

Cold-chain dispatchers often need to switch between fleet dashboards, SQL databases, weather information, and compliance documents to understand a temperature excursion and decide what to do next. This manual process can delay escalation and increase the risk of perishable-goods spoilage.

## The Solution

The assistant provides a conversational interface backed by an agent orchestration layer and controlled data-access tools. Rather than giving the language model unrestricted database access, the design uses narrowly scoped tools and database permissions.

### Key Capabilities

- **Natural-language operations queries** about active shipments and current conditions.
- **Fleet telemetry access** through an approved, read-only SQL view.
- **Weather context** when the relevant integration is configured and available.
- **SOP and compliance guidance** through operational-procedure retrieval when configured.
- **Agentic tool orchestration** using a LangGraph ReAct workflow.
- **Human-readable incident summaries** synthesized from tool results.
- **Resilience and fallback handling** that reports unavailable or stale data instead of inventing status.
- **Auditability** through an append-only log of prompts, tool activity, outputs, and agent decisions.

> This assistant supports dispatchers; it does not replace human oversight for safety-critical or compliance-sensitive decisions.

## Architecture

The system is designed as a decoupled, three-tier application.

```text
Dispatcher / Control-Tower Operator
                |
                v
       Streamlit Presentation
                |
                v
   LangGraph ReAct Orchestration
       |          |          |
       v          v          v
 Fleet SQL     Weather     SOP / Compliance
   Tool          Tool       Retrieval Tool
       |          |          |
       v          v          v
 Read-only SQL   External   SOP documents /
 View            API        Vector retrieval
       |
       v
 SQL Server + Append-only Agent Audit Log
```

### Main Components

| Component | Responsibility |
| --- | --- |
| Streamlit | Operator-facing conversational interface. |
| LangGraph ReAct agent | Maintains workflow state, interprets intent, selects tools, and synthesizes results. |
| SQL Server | Stores and serves structured fleet telemetry. |
| `VW_ACTIVE_FLEET` | Read-only view exposing approved active-fleet fields. |
| Weather integration | Supplies external weather context when configured. |
| SOP retrieval | Supplies operational and compliance guidance when configured. |
| `AgentAuditLog` | Append-only record of prompts, tool activity, raw outputs, and agent decisions. |

The technical design specifies these fields in `VW_ACTIVE_FLEET`: `Shipment_ID`, `Latitude`, `Longitude`, `Cargo_Type`, and `Current_Temperature`.

## Technology Stack

- **Language:** Python
- **UI:** Streamlit
- **Agent orchestration:** LangGraph (ReAct pattern)
- **Structured data:** Microsoft SQL Server
- **Database access:** SQLAlchemy / ODBC-based connectivity
- **AI layer:** LLM provider configured for the project
- **External context:** Weather API and SOP retrieval integration, where configured
- **Deployment direction:** Docker / infrastructure as code and AWS-style split-tier deployment

Use the repository's dependency and environment configuration files for exact package versions and enabled integrations.

## Security Design

Security should be enforced at the database, network, and application layers.

- **Least privilege:** use a database account with only the permissions required for approved read operations.
- **View-based access:** agent telemetry queries should use `VW_ACTIVE_FLEET`, not unrestricted underlying tables.
- **No direct write access:** the agent's database role should not be able to modify operational data or schema.
- **Immutable auditing:** record each decision cycle in `AgentAuditLog` as an append-only audit trail.
- **Network isolation:** the design exposes the application on port `8501`; SQL Server port `1433` should accept traffic only from the application's private network address.
- **Safe failure behavior:** explicitly report missing or stale data instead of guessing.

A README or prompt alone does not provide these security guarantees; enforce them in the real configuration.

## Project Structure

The legacy-data ingestion workflow referenced during development is:

```text
scripts/
└── ingest_legacy_data.py
```

It loads `data/raw/dynamic_supply_chain_logistics_dataset.csv` into the SQL Server table `TBL_SC_FLEET_HIST_RAW`. Other paths and application entry points may vary; consult the current repository for the authoritative file tree.

## Getting Started

### Prerequisites

- Python 3.10+ (use the version supported by the project's dependencies)
- Git
- Microsoft SQL Server or access to a configured SQL Server instance
- A compatible Microsoft SQL Server ODBC driver
- Credentials/API keys for any LLM, weather, or retrieval integrations you enable
- Docker Desktop, if using the containerized setup

### 1. Clone the repository

```bash
git clone https://github.com/himanshu2285/cold-chain.git
cd cold-chain
```

### 2. Create a Python environment

**Windows PowerShell**

```powershell
python -m venv cold-chain-env
.\cold-chain-env\Scripts\Activate.ps1
```

**macOS / Linux**

```bash
python3 -m venv cold-chain-env
source cold-chain-env/bin/activate
```

### 3. Install dependencies

If the repository contains `requirements.txt`, run:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

If dependencies are managed through another project file, follow that file's instructions.

### 4. Configure environment variables

Create a local `.env` from `.env.example`, if provided. Do not commit `.env` or real credentials.

The SQL ingestion workflow uses these environment-variable names:

```dotenv
SQL_SERVER_HOST=
SQL_SERVER_PORT=1433
SQL_SERVER_ADMIN_USER=
SQL_SERVER_ADMIN_PASSWORD=
```

Set these for your SQL Server instance. For normal agent queries, prefer a separate least-privilege read-only account rather than an administrator account.

Also configure the LLM provider, weather API, vector database, and embedding settings required by the integrations you enable. Use the exact variable names expected by the current code.

### 5. Prepare the source data

The ingestion workflow expects:

```text
data/raw/dynamic_supply_chain_logistics_dataset.csv
```

Make sure the CSV exists and the configured SQL Server is reachable before ingestion.

### 6. Ingest the legacy data

From the repository root:

```bash
python scripts/ingest_legacy_data.py
```

The script is intended to load data into `TBL_SC_FLEET_HIST_RAW`. Verify the destination table and row counts after ingestion.

### 7. Start the application

The UI is designed for Streamlit and the architecture specifies port `8501`. Run the actual Streamlit entry-point file in the repository:

```bash
streamlit run <your_streamlit_entry_file.py>
```

Replace the placeholder with the real filename. If using Streamlit's default port, open `http://localhost:8501`.

## Operational Workflow

1. The operator asks a question in natural language.
2. The agent identifies whether the request concerns telemetry, weather, SOPs, or a combination.
3. If the request is ambiguous, the agent asks for clarification.
4. The agent invokes the appropriate scoped tool or tools.
5. Tools return information from their approved sources.
6. The agent synthesizes the results into a contextual answer with supporting evidence.
7. The interaction and tool activity are recorded in the audit log.

## Reliability

- External API failures should produce a clear stale-data or unavailable-data message.
- SQL errors should be logged; a simplified query may be attempted before escalation.
- Tool functions should remain isolated and scoped to their documented responsibilities.
- Operators should validate recommended interventions against current SOPs and policy.

## Deployment Notes

The design document describes this network boundary:

- Public ingress: application port `8501` only.
- Internal database traffic: SQL Server port `1433`, restricted to the application's private network address.
- Database access: read-only view and least-privilege service account for agent queries.

Before production deployment, store secrets securely, disable development/debug settings, restrict database network access, verify audit-log controls, configure monitoring and health checks, and test behavior when APIs or SQL queries fail.

## Documentation

Supporting project material covers:

- **Technical Design Document (TDD):** architecture, agent orchestration, data security, auditing, and deployment topology.
- **Business presentation:** the cold-chain problem, dispatcher workflow, AI assistant concept, and governance/security approach.

If these PDFs are not already in the repository, consider adding them under `docs/` and linking them here.

## Limitations

- Results depend on the freshness and quality of telemetry and external sources.
- Weather and SOP features require their respective integrations and credentials.
- LLM-generated summaries may be incomplete; verify operational actions against source data and approved procedures.
- The assistant supports decisions and does not independently authorize high-impact operational actions.

## Author

**Himanshu Mishra**

- GitHub: [@himanshu2285](https://github.com/himanshu2285)
- Repository: [cold-chain](https://github.com/himanshu2285/cold-chain)

---

If you find this project useful, consider giving the repository a ⭐.
