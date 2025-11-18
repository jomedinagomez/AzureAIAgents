# Claims Magentic Orchestration Plan

## 1. Objective
Build a Python-first **claims agent assessment system** that mirrors the "General Insurance Payment Claim Process" (`docs/general_insurance_payment_claim_process.jpg` as the reference rendering, with `docs/general_insurance_payment_claim_process.bpmn` available for edits). This is an **agent-facing application**—designed to assist claims agents (not end-user customers) in efficiently processing claims and making informed decisions. The solution will:
- leverage Semantic Kernel's Magentic orchestration to coordinate multiple AI agents that gather evidence, analyze data, and **produce assessment recommendations** for the claims agent;
- keep the BPMN flow purposeful yet readable, highlighting automated data gathering, validation, risk assessment, and—critically—the **claims agent** receiving AI-generated recommendations to inform their final decision;
- run locally via the **uv** tool for fast virtualenv + dependency management, with a pinned `requirements.txt` for reproducibility;
- surface the orchestration outputs through a front-end UI that presents a comprehensive claim assessment to the claims agent, captures their approve/deny choice, and hands that decision (plus rationale) to downstream carrier systems for execution.

## 2. Target User Journey (Claims Agent Workflow)
1. **Claim Intake**: customer submits a claim (free-form description + attachments). The **claims agent** receives the claim in their queue. The system captures the intake, issues an acknowledgment to the customer, and begins automated assessment.
2. **Automated Data Gathering**: orchestration fans out to query every available source (uploaded docs, internal policy data, claims history, external risk feeds) **on behalf of the claims agent** before presenting any assessment.
3. **Data Completeness Check**: system automatically identifies missing information and requests it from the customer. SLA timers track response times. If customer fails to respond within SLA or if certain data can only be obtained manually, the **claims agent** updates remaining info via the UI (BPMN userTask "Claims Agent Updates Remaining Info") before the system proceeds.
4. **AI Assessment Generation**: once data is complete, the system synthesizes a comprehensive assessment (risk signals, policy alignment, fraud indicators, payout calculations) and presents it to the **claims agent** as a decision support tool.
5. **Agent Decision**: the **claims agent** reviews the AI-generated assessment, applies their expertise and judgment, and makes the final approve/decline decision inside the claims UI.
6. **Handoff to Carrier Systems**: once the decision is recorded, the orchestration stops and emits a structured payload (`ready_for_settlement_handoff` or `denied_with_reason`). Existing finance/CRM platforms consume that payload to run settlements, notifications, and compliance workflows outside this project’s scope.

## 3. Agent Roster (Magentic Members)
| Agent | Responsibilities | Key Tools |
| --- | --- | --- |
| **IntakeAgent** | Normalize intake data, detect missing fields, request clarifications | PII scrubber, schema validator |
| **DataSourcingAgent** | Fan out to policy, claims-history, and external feeds; reconcile conflicts; raise structured data gaps | REST/SQL connectors (future), mocked adapters |
| **ValidationAgent** | Cross-check docs vs. policy metadata, run fraud heuristics once data contracts are met | Document parser (future), rules engine stubs |
| **AnalysisAgent** | Summarize claim context, compute simple risk/impact score | Lightweight vector lookup, reference data CSVs |
| **DecisionSupportAgent** | Produce **agent-ready assessment briefs** that explain policy coverage, fraud risk indicators, payout calculations, and recommendation confidence scores—enabling the claims agent to make informed decisions quickly | Policy rules, company thresholds, risk scoring models |
| **HandoffAgent** | Package the agent’s decision, rationale, and required artifacts into a structured payload (`ready_for_settlement_handoff` or `denied_with_reason`) for downstream carrier systems | Schema templates, webhook/outbox client |
| **ReviewerAgent** | Quality-check AI-generated assessment outputs (e.g., DecisionSupportAgent briefs) for compliance tone, completeness, and factual accuracy before presenting to the claims agent | Checklist tool, output validator |
| **Router/Manager** | Provided by `StandardMagenticManager`; maintains task ledger and orchestrates agents |

All agents will be loaded from a future `agents_config.yaml` (derived from the existing sample) so they can be tuned without code edits.

## 4. Orchestration & Runtime
- Use `MagenticOrchestration` with a custom `ClaimsMagenticManager` (extending `StandardMagenticManager`) to map BPMN states to agent tasks.
- `CoreRuntime`: start with the in-process runtime bundled with SK (no external service bus required initially).
- Conversation state will mirror BPMN tokens. Metadata keys such as `state`, `missing_documents`, `risk_score`, `fraud_indicators`, `assessment_confidence`, `agent_decision`, `decision_confidence`, `ack_sent`, `info_request_sent`, `sla_breached`, `agent_reviewed`, and `handoff_status` let AI agents hand off context cleanly to the claims agent.
- **Data completeness loop exit**: the orchestration exits the "request additional info" loop when either (a) `missing_documents == []` (all required data obtained), (b) `sla_breached == True` (timeout forces escalation), or (c) `agent_reviewed == True` (agent manually completed the data via UI and marked ready to proceed).
- **Human-in-the-loop commitment**: the orchestration **never auto-adjudicates claims**; instead it produces AI-powered assessments and recommendations for the **claims agent**, records their final decision (`agent_decision`), and then emits a handoff payload for downstream settlement or denial systems. The claims agent remains the ultimate decision-maker.
- **Termination Policies**: custom manager implements termination logic tied to BPMN terminal nodes (see Section 11).

## 5. Project Structure (draft)
```
SemanticKernel/claims/
├── PLAN.md                     # this plan
├── docs/
│   ├── general_insurance_payment_claim_process.bpmn
│   └── general_insurance_payment_claim_process.jpg
├── requirements.txt            # dependency lock references for uv export
├── pyproject.toml              # uv-friendly config (project metadata + tool.uv)
├── claims/
│   ├── __init__.py
│   ├── runtime.py              # CoreRuntime bootstrap helper
│   ├── orchestration.py        # Magentic orchestration assembly
│   ├── agents.py               # agent factory + config loader
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── validators.py
│   │   ├── handoff.py
│   │   └── mocks/              # mock API implementations for testing
│   │       ├── __init__.py
│   │       ├── mock_policy_api.py
│   │       ├── mock_fraud_api.py
│   │       ├── mock_document_parser.py
│   │       └── mock_settlement_webhook.py
│   ├── data/                   # reference data & business rules (API-backed in production)
│   │   ├── policy_rules.yaml
│   │   ├── policies.csv
│   │   ├── coverage_matrix.csv
│   │   ├── risk/
│   │   │   ├── fraud_indicators.yaml
│   │   │   ├── risk_scoring_rules.json
│   │   │   └── blacklist.csv
│   │   ├── historical/
│   │   │   ├── claims_history.csv
│   │   │   └── payout_benchmarks.csv
│   │   └── external/
│   │       ├── weather_events.json
│   │       ├── police_reports.json
│   │       └── medical_codes.csv
│   ├── config/                 # agent & system configuration
│   │   ├── agents_config.yaml
│   │   ├── validation_checklist.yaml
│   │   └── handoff_schema.json
│   └── tests/
│       ├── __init__.py
│       ├── fixtures/           # sample claim scenarios
│       │   ├── claim_auto_simple.json
│       │   ├── claim_auto_fraud_suspected.json
│       │   ├── claim_home_missing_docs.json
│       │   ├── claim_health_high_value.json
│       │   └── claim_sla_breach.json
│       └── documents/          # simulated customer uploads (markdown format)
│           ├── document_metadata.json
│           ├── police_report_12345.md
│           ├── repair_estimate_001.md
│           ├── medical_receipt_summary.md
│           └── witness_statement.txt
└── cli.py                      # entry point: `python -m claims.cli --task "..."`
```

**Data Access Strategy**:
- **Mock/Development**: Files in `data/` are read directly by mock APIs (`tools/mocks/`) for local testing
- **Production**: Mock APIs replaced with real REST/SQL connectors to policy databases, fraud services, and external APIs
- **Document Processing**: `tests/documents/` contains markdown representations of typical customer uploads (police reports, estimates, receipts) that simulate extracted text from PDF/image OCR processing
- **Configuration**: `config/` files loaded at application startup; `tests/fixtures/` provide end-to-end test scenarios

## 6. Data Schemas & Reference Files

### 6.1 CSV File Schemas

**coverage_matrix.csv** - Coverage details by policy tier
```csv
policy_tier,claim_type,coverage_limit,deductible,exclusions,notes
basic,auto_collision,15000,1000,"racing|commercial_use","Standard collision coverage"
standard,auto_collision,25000,500,"racing|commercial_use","Enhanced collision coverage"
premium,auto_collision,50000,250,"racing","Comprehensive collision with low deductible"
basic,auto_comprehensive,10000,1000,"wear_and_tear|mechanical","Theft and vandalism only"
standard,auto_comprehensive,20000,500,"wear_and_tear|mechanical","Includes weather damage"
premium,auto_comprehensive,40000,250,"wear_and_tear","Full comprehensive coverage"
basic,home_fire,100000,2500,"arson|business_use","Structure only"
standard,home_fire,250000,1500,"arson|business_use","Structure + contents"
premium,home_fire,500000,1000,"arson","Full replacement value"
basic,health_surgery,50000,5000,"cosmetic|experimental","Emergency procedures only"
standard,health_surgery,150000,2500,"cosmetic|experimental","Planned and emergency"
premium,health_surgery,unlimited,1000,"cosmetic","All medically necessary procedures"
```

**blacklist.csv** - Flagged entities for fraud detection
```csv
entity_id,entity_type,business_name,tax_id,license_number,reason,date_flagged,severity,status,last_verified,notes
BL-001,customer,N/A,SSN-REDACTED,N/A,multiple_fraudulent_claims,2024-03-15,high,active,2024-11-01,"3 confirmed fraud cases 2023-2024, SIU investigation complete"
BL-002,repair_shop,QuickFix Auto LLC,EIN-12-3456789,MD-SHOP-4421,inflated_estimates,2024-06-20,medium,active,2024-10-15,"Estimates average 38% above market rate across 15 claims"
BL-003,medical_provider,HealthPlus Clinic,EIN-98-7654321,MD-MED-8832,billing_fraud,2024-01-10,high,active,2024-09-20,"Submitted invoices for services not rendered, FBI case #2024-MD-1122"
BL-004,customer,N/A,SSN-REDACTED,N/A,staged_accident,2024-09-01,critical,under_investigation,2024-11-10,"Organized fraud ring, 5+ staged accidents, police case #2024-PD-9955"
BL-005,repair_shop,Discount Body Shop,EIN-55-4433221,MD-SHOP-2109,unlicensed_operation,2024-07-15,medium,resolved,2024-10-01,"License expired 2024-05-30, re-licensed 2024-09-15"
BL-006,attorney,Legal Eagles PC,EIN-77-8899001,MD-BAR-55443,excessive_litigation,2024-05-22,medium,active,2024-10-30,"Pattern of inflated soft tissue injury claims, settlement mill indicators"
```

**claims_history.csv** - Historical claims per customer
```csv
claim_id,customer_id,policy_number,claim_type,incident_date,filed_date,closed_date,amount_requested,reserved_amount,amount_paid,claim_status,fraud_flag,assigned_adjuster,processing_days,notes
CLM-2023-00891,CUST-1234,AUTO-789456,auto_collision,2023-05-10,2023-05-12,2023-05-20,3200,3000,2700,closed_approved,false,AGT-112,8,"Rear-end collision, verified"
CLM-2023-01205,CUST-1234,AUTO-789456,auto_comprehensive,2023-08-22,2023-08-23,2023-08-28,1500,1500,1200,closed_approved,false,AGT-445,5,"Hail damage, verified weather event"
CLM-2024-00034,CUST-5678,HOME-445566,home_fire,2024-01-15,2024-01-16,2024-01-27,45000,0,0,closed_denied,true,AGT-223,12,"Arson suspected, police investigation ongoing"
CLM-2024-00567,CUST-9012,HEALTH-112233,health_surgery,2024-03-20,2024-03-21,2024-04-05,12000,11000,10500,closed_approved,false,AGT-334,15,"Appendectomy, routine procedure"
CLM-2024-01023,CUST-1234,AUTO-789456,auto_collision,2024-06-05,2024-06-08,2024-06-14,4500,0,0,closed_denied,false,AGT-112,6,"Aggregate limit exceeded for policy year"
CLM-2024-01456,CUST-3456,AUTO-998877,auto_comprehensive,2024-09-10,2024-09-11,2024-09-21,8000,7500,7200,closed_approved,false,AGT-445,10,"Vehicle theft, recovered with damage"
```

**vendors.csv** - Approved repair shops and medical providers
```csv
vendor_id,vendor_type,business_name,license_number,license_state,license_expiry,rating,total_claims_processed,avg_estimate_accuracy,contact_phone,address,city,state,zip,last_audit_date,audit_status,notes
VND-001,repair_shop,Honest Auto Body,MD-SHOP-8821,MD,2026-12-31,4.8,523,0.95,410-555-0100,"123 Main St",Baltimore,MD,21201,2024-09-15,passed,"Preferred DRP partner since 2020"
VND-002,repair_shop,Premium Collision Center,MD-SHOP-7743,MD,2025-08-30,4.5,312,0.92,410-555-0200,"456 Oak Ave",Columbia,MD,21044,2024-08-10,passed,"Certified for luxury vehicles"
VND-003,medical_provider,Metro Orthopedic Clinic,MD-MED-5512,MD,2026-06-30,4.7,892,0.98,410-555-0300,"789 Medical Plaza",Towson,MD,21204,2024-10-05,passed,"Workers comp and auto injury specialist"
VND-004,medical_provider,Valley Physical Therapy,MD-MED-6634,MD,2025-11-15,4.6,445,0.94,410-555-0400,"321 Health Blvd",Rockville,MD,20850,2024-07-22,passed,"Post-accident rehabilitation focus"
VND-005,repair_shop,Main Street Garage,VA-SHOP-3309,VA,2026-03-31,4.2,198,0.89,703-555-0500,"555 Commerce Dr",Arlington,VA,22201,2024-06-18,conditional,"Minor billing discrepancies noted, re-audit scheduled"
```

**medical_codes.csv** - ICD-10 medical procedure codes for validation
```csv
icd10_code,description,category,typical_cost_min,typical_cost_max,common,notes
K35.80,Unspecified acute appendicitis,surgery,8000,15000,true,"Emergency appendectomy"
S72.001A,Fracture of unspecified part of neck of right femur,orthopedic,25000,50000,true,"Hip fracture requiring surgery"
I21.9,Acute myocardial infarction unspecified,cardiac,30000,75000,true,"Heart attack treatment"
M17.11,Unilateral primary osteoarthritis right knee,orthopedic,15000,35000,true,"Knee replacement indication"
Z00.00,Encounter for general adult medical examination,preventive,200,500,true,"Routine checkup - not claimable"
J44.1,Chronic obstructive pulmonary disease with exacerbation,respiratory,5000,12000,true,"COPD treatment"
E11.9,Type 2 diabetes mellitus without complications,chronic,0,0,true,"Ongoing management - special coverage"
C50.911,Malignant neoplasm of unspecified site of right female breast,oncology,50000,150000,false,"Cancer treatment - special review"
```

**payout_benchmarks.csv** - Average payout statistics by claim type
```csv
claim_type,severity,avg_payout,std_deviation,percentile_25,percentile_50,percentile_75,percentile_90,sample_size,last_updated
auto_collision,minor,2500,800,1800,2400,3100,4200,1523,2024-11-01
auto_collision,moderate,8500,2200,6500,8200,10500,13000,892,2024-11-01
auto_collision,severe,25000,8500,18000,24000,31000,42000,234,2024-11-01
auto_comprehensive,minor,1800,600,1200,1700,2200,2800,1876,2024-11-01
auto_comprehensive,moderate,5500,1800,4000,5200,6800,8500,723,2024-11-01
auto_comprehensive,severe,18000,7500,12000,16500,23000,30000,189,2024-11-01
home_fire,minor,8500,3200,5500,8000,11000,14500,456,2024-11-01
home_fire,moderate,35000,12000,25000,33000,43000,55000,287,2024-11-01
home_fire,major,75000,35000,45000,68000,95000,125000,156,2024-11-01
health_surgery,routine,12000,4500,8500,11500,15000,19000,2341,2024-11-01
health_surgery,complex,45000,18000,30000,42000,58000,75000,892,2024-11-01
health_surgery,critical,95000,42000,60000,88000,125000,160000,234,2024-11-01
```

**policies.csv** - Active customer policies (database table export)
```csv
policy_number,customer_id,policy_holder_name,dob,license_number,license_state,policy_type,tier,status,effective_date,expiration_date,annual_premium,payment_status,collision_limit,comprehensive_limit,deductible_collision,deductible_comprehensive,liability_bi_per_person,liability_bi_per_accident,liability_pd,uninsured_motorist,medical_payments,aggregate_limit_per_year,claims_count_this_year,claims_paid_this_year,remaining_aggregate,vehicle_make,vehicle_model,vehicle_year,vehicle_vin,vehicle_usage,garaging_address
AUTO-789456,CUST-1234,John Smith,1985-03-15,S123-4567-8901-23,MD,auto,standard,active,2024-01-01,2026-01-01,1200,current,25000,20000,500,500,100000,300000,50000,100000,5000,50000,1,2700,47300,Toyota,Camry,2020,4T1BF1FK5CU123456,personal,"123 Main St, Baltimore, MD 21201"
AUTO-998877,CUST-3456,Sarah Johnson,1990-07-22,J789-0123-4567-89,MD,auto,premium,active,2023-06-15,2025-06-15,1850,current,50000,40000,250,250,250000,500000,100000,250000,10000,100000,1,7200,92800,Honda,Accord,2022,1HGCV1F39MA123789,personal,"456 Oak Ave, Annapolis, MD 21401"
HOME-445566,CUST-5678,Michael Davis,1978-11-30,D456-7890-1234-56,MD,home,standard,active,2023-01-01,2025-01-01,2400,current,250000,250000,1500,1500,500000,500000,100000,0,0,500000,1,0,500000,N/A,N/A,N/A,N/A,primary_residence,"789 Maple Dr, Columbia, MD 21044"
HEALTH-112233,CUST-9012,Emily Rodriguez,1992-04-18,R234-5678-9012-34,MD,health,standard,active,2024-03-01,2025-03-01,4800,current,150000,150000,2500,2500,0,0,0,0,0,300000,1,10500,289500,N/A,N/A,N/A,N/A,individual,"321 Pine St, Rockville, MD 20850"
AUTO-556677,CUST-1234,John Smith,1985-03-15,S123-4567-8901-23,MD,auto,standard,lapsed,2022-01-01,2024-01-01,1150,overdue,25000,20000,500,500,100000,300000,50000,100000,5000,50000,0,0,0,Honda,Civic,2018,2HGFC2F59JH123456,personal,"123 Main St, Baltimore, MD 21201"
```

### 6.2 JSON File Schemas

**fraud_indicators.yaml** - Red flag patterns (configuration file, not database)
```yaml
patterns:
  duplicate_claims:
    threshold: 2
    timeframe_days: 90
    severity: high
    description: "Multiple claims for same incident type within 90 days"
  
  high_value_claim:
    threshold: 50000
    severity: medium
    description: "Claim amount exceeds $50k requires additional review"
  
  blacklisted_entity:
    severity: critical
    description: "Customer, provider, or shop on blacklist"
  
  inconsistent_statements:
    severity: high
    description: "Conflicting dates, locations, or narratives in documentation"
  
  suspicious_timing:
    patterns:
      - "Policy activated <30 days before incident"
      - "Claim filed >14 days after incident without explanation"
    severity: medium
  
  missing_corroboration:
    severity: medium
    description: "No police report, witness, or third-party verification"
```

**risk_scoring_rules.json** - Point-based risk assessment
```json
{
  "risk_factors": [
    {
      "factor": "customer_claims_history",
      "ranges": [
        {"min": 0, "max": 0, "points": 0, "label": "no_history"},
        {"min": 1, "max": 2, "points": 10, "label": "low_frequency"},
        {"min": 3, "max": 5, "points": 25, "label": "moderate_frequency"},
        {"min": 6, "max": 999, "points": 50, "label": "high_frequency"}
      ]
    },
    {
      "factor": "claim_amount_ratio",
      "description": "Ratio of claim to policy coverage limit",
      "ranges": [
        {"min": 0.0, "max": 0.25, "points": 0, "label": "low"},
        {"min": 0.25, "max": 0.50, "points": 5, "label": "moderate"},
        {"min": 0.50, "max": 0.75, "points": 15, "label": "high"},
        {"min": 0.75, "max": 1.0, "points": 30, "label": "very_high"}
      ]
    },
    {
      "factor": "documentation_completeness",
      "ranges": [
        {"min": 90, "max": 100, "points": -10, "label": "complete"},
        {"min": 70, "max": 89, "points": 5, "label": "mostly_complete"},
        {"min": 50, "max": 69, "points": 15, "label": "incomplete"},
        {"min": 0, "max": 49, "points": 30, "label": "severely_incomplete"}
      ]
    }
  ],
  "score_interpretation": [
    {"min": 0, "max": 20, "risk_level": "low", "recommendation": "approve"},
    {"min": 21, "max": 50, "risk_level": "medium", "recommendation": "review"},
    {"min": 51, "max": 100, "risk_level": "high", "recommendation": "deny"}
  ]
}
```

**weather_events.json** - External validation data (mock API response format)
```json
{
  "events": [
    {
      "event_id": "WX-2025-11-10-001",
      "date": "2025-11-10",
      "location": "I-95 Corridor, Maryland",
      "event_type": "heavy_rain",
      "severity": "moderate",
      "description": "Persistent rain 2-4 inches, reduced visibility",
      "verified_sources": ["NOAA", "local_news"]
    },
    {
      "event_id": "WX-2025-09-15-002",
      "date": "2025-09-15",
      "location": "Houston, TX",
      "event_type": "hail",
      "severity": "severe",
      "description": "Golf ball sized hail, widespread vehicle damage",
      "verified_sources": ["NOAA", "insurance_industry_cat"]
    }
  ]
}
```

**police_reports.json** - Simulated police database
```json
{
  "reports": [
    {
      "report_number": "2025-PD-8821",
      "incident_date": "2025-11-10",
      "location": "I-95 Northbound, Mile Marker 42",
      "incident_type": "vehicle_collision",
      "parties": [
        {"name": "John Smith", "role": "victim", "vehicle": "Toyota Camry"},
        {"name": "Jane Doe", "role": "at_fault", "vehicle": "Honda Accord"}
      ],
      "narrative": "Vehicle 2 failed to stop at red light, rear-ended Vehicle 1",
      "verified": true,
      "officer_badge": "12345"
    }
  ]
}
```

### 6.3 Configuration File Schemas

**handoff_schema.json** - Settlement system payload template
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["claim_id", "decision", "agent_id", "decision_timestamp"],
  "properties": {
    "claim_id": {"type": "string", "pattern": "^CLM-[0-9]{4}-[0-9]{5}$"},
    "decision": {"type": "string", "enum": ["approve", "deny"]},
    "payout_amount": {"type": "number", "minimum": 0},
    "agent_id": {"type": "string", "pattern": "^AGT-[0-9]{3}$"},
    "decision_timestamp": {"type": "string", "format": "date-time"},
    "confidence_score": {"type": "integer", "minimum": 0, "maximum": 100},
    "fraud_risk": {"type": "integer", "minimum": 0, "maximum": 100},
    "rationale": {"type": "string", "minLength": 10},
    "attachments": {"type": "array", "items": {"type": "string"}},
    "denial_reason": {"type": "string", "enum": ["fraud_suspected", "insufficient_documentation", "not_covered", "policy_limit_exceeded", "other"]}
  }
}
```

## 7. Environment & Tooling Plan
1. **uv setup**
   - `uv venv` to create `.venv/` in `SemanticKernel/claims` (kept out of git via root `.gitignore`).
   - `uv pip install -r requirements.txt` for day-to-day usage.
   - `requirements.txt` will be generated from `pyproject.toml` via `uv pip compile` once dependencies settle.
   - Quick commands:
     ```bash
     uv venv
     uv pip install -r requirements.txt
     uv run python -m claims.cli --help
     ```
2. **Dependencies (initial)**
   - `semantic-kernel>=1.12.0`
   - `pydantic>=2.6`
   - `python-dotenv` for env loading
   - `rich` for CLI UX
   - `typer` for CLI commands
   - `pandas` (optional) for quick tabular summaries
   - **Observability**: `opentelemetry-sdk`, `opentelemetry-exporter-otlp-proto-grpc`, `opentelemetry-instrumentation-logging`, `opentelemetry-distro` for Aspire dashboard export
3. **Secrets**
   - rely on `.env` (loaded via `python-dotenv`) for Azure OpenAI keys (`AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_DEPLOYMENT`, `AZURE_OPENAI_API_VERSION`).

## 7. Implementation Phases
1. **Scaffold**: create package layout, `pyproject.toml`, `requirements.txt`, and CLI skeleton; wire uv instructions into README snippet.
2. **Agent Config Loader**: parse YAML, instantiate `ChatCompletionAgent`s with Azure settings, default instructions.
3. **Tool Layer**: implement stubbed validation + handoff payload utilities; ensure they're callable via SK function tools. Implement SLA timer as async background task (using `asyncio.create_task` with timeout) that sets `sla_breached=True` in shared context after 5 days if agent hasn't completed data update.
4. **Magentic Assembly**: build orchestration + runtime bootstrap; map BPMN states to agent tasks and integrate termination logic.
5. **E2E Flow**: craft sample claim scenario, run CLI to produce acceptance and rejection paths; add tests.
6. **Docs & Diagrams**: embed BPMN diagram references and usage steps in README.

## 8. Risks & Open Questions
- **Structured Output Support**: ensure chosen Azure model (e.g., `gpt-4o`) handles JSON schema for Magentic manager.
- **Tool Completeness**: initial release will mock certain integrations (fraud detection, carrier handoff webhooks). Document these as future enhancements.
- **Runtime Scalability**: in-process runtime suffices now; consider distributed runtime if hitting concurrency limits.
- **Front-End UI Scope**: a web-based front-end UI is planned as part of this project to surface orchestration outputs to claims agents, capture their decisions (`agent_decision`), and allow manual data updates (BPMN userTask). UI will consume REST endpoints exposed by the orchestration backend and display real-time assessment briefs, confidence scores, and decision options.

## 9. Next Actions
1. Add `pyproject.toml` + `requirements.txt` aligned with uv workflow.
2. Implement the file/folder scaffold from Section 5.
3. Build agent loader + orchestration bootstrap in incremental PRs.
4. Create telemetry bootstrap helper + Aspire instructions (Section 10) in codebase docs.

## 10. Observability (Aspire Dashboard)
1. **Telemetry packages** already tracked in `pyproject.toml`/`requirements.txt` (see Section 6) to pull OpenTelemetry core + OTLP exporters.
2. **Environment variables**
   - `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317`
   - `OTEL_EXPORTER_OTLP_PROTOCOL=grpc`
   - `OTEL_SERVICE_NAME=claims-orchestrator`
   - `ASPIRE_ALLOW_UNSECURED_TRANSPORT=1` (required for local dashboard certs)
3. **Bootstrap helper** (planned `claims/observability.py`): configure OTLP span + metric exporters, set `Resource(service.name="claims-orchestrator")`, and hook Python logging via `LoggingInstrumentor().instrument(set_logging_format=True)` so SK + app logs surface in Aspire.
4. **Local workflow**
   ```bash
   dotnet aspire dashboard --open false  # start dashboard on http://localhost:18888
   uv run python -m claims.cli --task "example claim"
   ```
   Inspect traces/logs under the dashboard's **Traces** and **Structured** tabs per Microsoft "Telemetry with Aspire" guidance.

5. **Key Event Mapping**: emit span attributes such as `claim.ack_sent`, `claim.data_sources_complete`, `claim.additional_info_requested`, `claim.agent_updated_info`, `claim.agent_decision_recorded`, `claim.ready_for_settlement_handoff`, `claim.denied_with_reason`, and `claim.sla_breached` so Aspire timelines mirror the updated BPMN swim-lanes (acknowledgment, auto data checks, info loop, agent manual update, agent review, decision capture, and SLA timer firing).

## 11. Termination Policies
To align with the BPMN "General Insurance Payment Claim Process" diagram (which defines multiple terminal states), the orchestration must implement explicit termination conditions. This section outlines the strategy and implementation approach.

### 11.1 BPMN Terminal Nodes
The updated BPMN diagram exposes two explicit terminal nodes that the orchestration must honor:
1. **Approved – Ready for Settlement Handoff**: reached the moment the agent approves the claim and the system packages the handoff payload for downstream carrier systems.
2. **Denied**: reached when the claim is declined (either by agent choice or because the SLA timer forced an escalation) and the system produces a denial rationale packet.

Everything after the agent’s decision (settlement execution, payments, notifications) occurs outside this orchestration.

### 11.2 Termination Strategy Design
The custom `ClaimsMagenticManager` (extending or replacing `StandardMagenticManager` from SK) will implement the following termination logic:

**Primary Termination Conditions**
| Condition | Metadata Check | BPMN Node | Outcome |
| --- | --- | --- | --- |
| **Approved – Handoff Ready** | `agent_decision == "approve"` AND `handoff_status == "ready_for_settlement"` | Approved – Ready for Settlement Handoff | Return the packaged payload (decision rationale + evidence pointers) for downstream systems |
| **Denied (Manual)** | `agent_decision == "deny"` AND `denial_package_ready == True` | Denied | Return denial rationale + evidence pointers |
| **Denied via SLA Breach** | `sla_breached == True` AND `agent_decision == "deny"` | Denied | Emphasize that timeout triggered the denial and include escalation context |
| **Stall Detection** | Same agent loops ≥3 times with no ledger progress | N/A | Return partial result + stall warning |
| **Max Iterations** | Total orchestration rounds exceed `max_rounds` (default: 15) | N/A | Return current state + timeout warning |

**Secondary Policies**
- **Human-in-the-Loop Pause**: if `state == "request_additional_data"` AND `missing_documents != []`, pause orchestration (don't terminate), request user input via CLI, then resume.
- **Emergency Cancellation**: allow user to invoke `orchestration_result.cancel()` from CLI interrupt handler (Ctrl+C) to stop all agents mid-process.
- **Partial Success**: if `agent_decision` is captured but `handoff_status` stays `"pending"` due to a tool error, mark as partial success and request manual intervention rather than loop indefinitely.

### 11.3 Implementation Plan
1. **Create custom manager** `claims/managers.py` with `ClaimsMagenticManager(StandardMagenticManager)`:
   - Override `should_terminate(context, task_ledger) -> bool` to check the metadata conditions above.
   - Add helper `_is_stalled(task_ledger) -> bool` to detect loops (track agent call counts + ledger snapshot deltas).
   - Implement `_gather_final_result(context, task_ledger) -> str` to format the terminal response based on which condition was met.
2. **Metadata state machine**: ensure agents update shared context keys (`agent_decision`, `decision_confidence`, `handoff_status`, `denial_package_ready`, `state`, `missing_documents`, `sla_breached`) at each handoff so the manager can evaluate termination conditions accurately.
3. **Testing scenarios**:
   - Happy path: intake → validation → analysis → agent approval → HandoffAgent produces payload → termination at "Approved – Ready for Settlement Handoff".
   - Agent manual update path: intake → missing data detected → auto-request sent → agent manually updates via UI (`agent_reviewed=True`) → validation → analysis → agent approval → termination.
   - Rejection path: intake → validation → fraud flag → agent denial or SLA breach → denial packet → termination at "Denied".
   - Stall path: DecisionSupportAgent loops 3 times unable to produce a confident brief → stall termination with recommendation to add missing policy data.
   - Timeout path: orchestration exceeds max_rounds → return partial ledger and flag for operator review.
4. **Telemetry integration**: emit custom span attributes (`claim.agent_decision`, `claim.handoff_status`, `claim.termination_reason`) in the termination hook so Aspire dashboard traces show why each orchestration ended.

### 11.4 Configuration
The manager's termination settings will be exposed via constructor params (eventually loaded from YAML):
```python
ClaimsMagenticManager(
    max_rounds=15,
    stall_threshold=3,
    enable_human_in_loop=True,
    timeout_behavior="partial_success"  # or "fail_fast"
)
```

This ensures operational flexibility: strict production workflows can set lower thresholds; exploratory dev flows can allow more iterations.

### 11.5 Example Termination Flow (Code Stub)
```python
class ClaimsMagenticManager(StandardMagenticManager):
   async def should_terminate(self, context, task_ledger) -> bool:
      meta = context.metadata
      if meta.get("agent_decision") == "approve" and meta.get("handoff_status") == "ready_for_settlement":
         return True
      if meta.get("agent_decision") == "deny" and meta.get("denial_package_ready"):
         return True
      if self._is_stalled(task_ledger) or context.round_count >= self.max_rounds:
         return True
      return False
```

This design mirrors the SK GroupChat termination pattern (per migration docs) while adapting it for Magentic's task-ledger-driven workflow and the specific BPMN nodes in the claims diagram.
