# AURA — AI-Powered B2B Purchase Order Processing

> An agentic AI and workflow automation prototype for streamlining B2B hospital procurement, from lead qualification through purchase-order processing and inventory operations.

## Overview

AURA is a prototype automation system designed around a hospital mattress manufacturing and procurement workflow.

The system combines **LLM-based decision support**, **deterministic business rules**, **workflow orchestration with n8n**, and **Supabase-backed persistence** to automate repetitive B2B sales and procurement operations.

The current prototype contains two core workflows:

1. **AI Lead Qualification**
2. **Purchase Order Processing**

The goal is to demonstrate how AI can be integrated into an operational business workflow while keeping critical business rules deterministic and auditable.

---

## System Architecture

```text
                    ┌──────────────────────┐
                    │   Hospital / Lead    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │        n8n            │
                    │ Workflow Orchestrator │
                    └──────────┬───────────┘
                               │
             ┌─────────────────┴─────────────────┐
             │                                   │
             ▼                                   ▼
   ┌─────────────────────┐             ┌─────────────────────┐
   │  Lead Qualification │             │ PO Processing       │
   │       Workflow      │             │      Workflow       │
   └──────────┬──────────┘             └──────────┬──────────┘
              │                                   │
              ▼                                   ▼
       ┌──────────────┐                    ┌──────────────┐
       │ LLM Analysis │                    │ PO Validation│
       └──────┬───────┘                    └──────┬───────┘
              │                                   │
              ▼                                   ▼
       ┌──────────────┐                    ┌──────────────┐
       │ Deterministic│                    │  Inventory   │
       │  Validation  │                    │    Check     │
       └──────┬───────┘                    └──────┬───────┘
              │                                   │
       ┌──────┴──────┐                    ┌──────┴──────┐
       ▼             ▼                    ▼             ▼
    HOT/WARM       COLD              Available      Shortage
       │             │                    │             │
       ▼             ▼                    ▼             ▼
   Follow-up      Nurture            Reservation   Procurement
                                      / Production   Exception
Core Workflows
1. AI Lead Qualification

The lead qualification workflow retrieves new leads and enriches them with hospital and contact information.

An LLM evaluates the available information using factors including:

Hospital size / estimated bed count
Public vs private organization
Procurement involvement of the identified contact
Potential mattress purchasing opportunity
Overall commercial potential
Other explicitly available lead information

The LLM produces:

lead_score
priority
qualification
reason
next_action

The result is then passed through deterministic validation.

Lead priority is derived from the score:

Score	Priority
80–100	HOT
50–79	WARM
0–49	COLD

The workflow then routes leads into different follow-up paths and records the qualification and activity information in the database.

Why deterministic validation is used

The LLM is responsible for interpreting unstructured business information.

Critical business classification is subsequently validated using deterministic code.

This prevents the model from independently overriding the defined priority thresholds.

2. Purchase Order Processing

The purchase-order workflow receives a PO through an HTTP webhook.

The incoming order is validated for:

PO number
Supplier
Items
Product ID
Product SKU
Product name
Quantity
Unit price
Delivery date

Invalid orders are routed to rejection handling.

Valid orders are persisted and continue through inventory processing.

Inventory Processing

The workflow:

Creates the purchase order.
Checks product inventory.
Records the inventory check.
Determines whether sufficient stock exists.
Reserves inventory when stock is available.
Updates stock quantity.
Advances the order workflow.
Creates a production order.

When inventory is insufficient, AURA creates a procurement exception containing:

Requested quantity
Available quantity
Shortage quantity
Product information
Purchase-order information

The procurement team is then notified.

Technology Stack
Component	Technology
Workflow orchestration	n8n
AI / LLM	Groq-hosted openai/gpt-oss-20b
Database	Supabase
Email automation	Gmail / n8n
Business logic	JavaScript
Integration	HTTP / Webhooks
Data validation	Deterministic code + structured LLM output
Repository Structure
AURA-Purchase-Order-Processing/
│
├── architecture/
│
├── docs/
│
├── examples/
│
├── n8n/
│   ├── aura-lead-qualification.json
│   └── aura-purchase-order-processing.json
│
├── screenshots/
│
├── .env.example
├── .gitignore
└── README.md
Design Principles
AI where interpretation is required

LLMs are used for tasks involving interpretation and qualification of business information.

Deterministic rules where correctness matters

Business thresholds, validation rules, inventory calculations, and state transitions are handled using deterministic logic.

Auditability

Important workflow events are recorded in an activity log, including AI-driven lead routing and inventory checks.

Exception handling

The system does not assume that every transaction succeeds.

Examples include:

Invalid purchase orders
Missing required fields
Insufficient inventory
Invalid AI output
Unexpected workflow states
Example Lead Decision

Example output:

{
  "lead_score": 88,
  "priority": "HOT",
  "qualification": "QUALIFIED",
  "reason": "Strong commercial signals and procurement relevance.",
  "next_action": "Contact the procurement manager with a hospital mattress proposal."
}

The actual implementation additionally validates the score and ensures that the priority corresponds to the defined score range.

Example Purchase Order

A simplified PO payload can follow this structure:

{
  "po_number": "PO-1001",
  "supplier_name": "Example Supplier",
  "customer_name": "Example Hospital",
  "customer_email": "procurement@example.com",
  "delivery_date": "2026-10-15",
  "items": [
    {
      "product_id": "PROD-001",
      "product_sku": "MAT-001",
      "product_name": "Hospital Mattress",
      "quantity": 50,
      "unit_price": 5000
    }
  ]
}
Current Prototype Scope

The current repository represents a working prototype rather than a production deployment.

Current capabilities include:

AI-assisted lead qualification
Structured LLM output
Deterministic AI-output validation
Lead priority routing
Automated email workflows
Purchase-order webhook ingestion
PO validation
Inventory checking
Inventory reservation
Stock updates
Inventory shortage handling
Procurement exception creation
Production-order creation
Activity logging
Known Limitations

This is an evolving prototype.

The current PO inventory workflow operates on the first PO line item in several workflow expressions. A future iteration should process arbitrary numbers of PO line items using explicit item iteration and transactional inventory operations.

Additional production-oriented improvements would include:

Transactional inventory updates
Idempotent PO processing
Authentication and authorization
More comprehensive schema validation
Human approval gates for high-impact actions
Retry and failure recovery mechanisms
Comprehensive automated testing
Production observability
Multi-item purchase-order processing
Deployment and infrastructure documentation
Future Direction

AURA is intended to evolve toward a broader agentic B2B procurement platform connecting:

Lead
  ↓
Qualification
  ↓
Requirement Extraction
  ↓
Quotation
  ↓
Purchase Order
  ↓
Validation
  ↓
Inventory
  ↓
Production
  ↓
Quality Control
  ↓
Dispatch
  ↓
Delivery
  ↓
Product Passport / Verification

Future versions can introduce specialized agents for individual stages while maintaining deterministic controls around critical business operations.

Status

Prototype / Active Development

This repository is intended to document the architecture, workflows, engineering decisions, and evolution of the AURA system.

Author

Pranav Sridhar

Built as an exploration of Agentic AI, workflow orchestration, intelligent automation, and AI-assisted B2B operations.


