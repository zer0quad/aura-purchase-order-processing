# AURA System Architecture

## 1. Architecture Overview

AURA uses n8n as the workflow orchestration layer connecting AI decision-making, deterministic business logic, database persistence, communication, and procurement operations.

The current prototype consists of two primary workflows:

- AI Lead Qualification
- Purchase Order Processing

---

## 2. High-Level Architecture

```text
                         ┌─────────────────────┐
                         │ Hospital / Customer │
                         └──────────┬──────────┘
                                    │
                     ┌──────────────┴──────────────┐
                     │                             │
                     ▼                             ▼
             Lead / Contact                  Purchase Order
                     │                             │
                     ▼                             ▼
             ┌──────────────┐              ┌──────────────┐
             │     n8n      │              │     n8n      │
             │ Lead Workflow│              │ PO Workflow  │
             └──────┬───────┘              └──────┬───────┘
                    │                             │
                    ▼                             ▼
             ┌──────────────┐              ┌──────────────┐
             │     LLM      │              │ Deterministic│
             │ Qualification│              │  Validation  │
             └──────┬───────┘              └──────┬───────┘
                    │                             │
                    ▼                             ▼
             ┌──────────────┐              ┌──────────────┐
             │ Rule-Based   │              │   Inventory  │
             │ Validation   │              │    Check     │
             └──────┬───────┘              └──────┬───────┘
                    │                             │
             ┌──────┴──────┐                ┌─────┴──────┐
             ▼             ▼                ▼            ▼
          HOT/WARM       COLD          Available      Shortage
             │             │                │            │
             ▼             ▼                ▼            ▼
          Follow-up      Nurture        Reservation  Procurement
                                           │          Exception
                                           ▼
                                      Production Order
3. AI Lead Qualification
Input

The workflow retrieves new leads from the leads database table and enriches the lead with hospital and contact information.

AI Processing

The LLM evaluates commercially relevant information including:

Hospital size / estimated bed count
Public or private status
Procurement involvement
Potential mattress purchasing opportunity
Other explicitly available information

The LLM produces a structured result containing:

lead_score
priority
qualification
reason
next_action
Validation Layer

The AI result is passed through deterministic validation.

The system verifies:

Score is an integer between 0 and 100
Priority is one of HOT, WARM, or COLD
Priority corresponds to the score
Qualification is valid
Reason is present
Next action is present

Priority is determined using deterministic thresholds:

80–100 → HOT
50–79  → WARM
0–49   → COLD

This creates a separation between:

AI interpretation

and

deterministic business policy enforcement.

Routing

After validation, the lead is routed according to priority:

HOT  → Sales follow-up
WARM → Follow-up sequence
COLD → Nurture sequence

The workflow also records activity in the database.

4. Purchase Order Processing
Input

The workflow exposes an HTTP POST endpoint for receiving purchase orders.

The incoming payload contains order and item information.

Validation

The order is validated before business processing begins.

Validation includes:

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

5. Order Processing

A valid PO is persisted in the purchase_orders table.

The workflow calculates the order total from item quantity and unit price.

The order then proceeds to inventory processing.

6. Inventory Processing

The workflow retrieves product inventory and compares available stock against requested quantity.

Sufficient inventory

When sufficient inventory exists:

Check Inventory
      ↓
Reserve Inventory
      ↓
Update Stock
      ↓
Determine Next Status
      ↓
Update PO Status
      ↓
Create Production Order
Insufficient inventory

When inventory is insufficient:

Check Inventory
      ↓
Prepare Inventory Exception
      ↓
Create Procurement Order
      ↓
Notify Procurement

The exception records the requested quantity, available quantity, and shortage quantity.

7. Data Persistence

Supabase is used as the persistence layer for operational data.

The current workflows interact with tables including:

leads
ai_qualifications
purchase_orders
products
inventory_reservations
procurement_orders
production_orders
activity_log

The exact database schema is environment-specific and is not included in the workflow exports.

8. Auditability

AURA records workflow events through an activity log.

Examples include:

Lead routing
Inventory checks
Workflow exceptions

The activity records include contextual metadata so that downstream users can understand what action occurred and why.

9. Error and Exception Handling

The prototype explicitly models several failure conditions.

Lead qualification
Invalid AI score
Invalid priority
AI/business-rule disagreement
Missing qualification
Missing reason
Missing next action
Routing exception
Purchase orders
Missing PO number
Missing supplier
Missing items
Missing product information
Invalid quantity
Invalid unit price
Missing/invalid delivery date
Insufficient inventory
Invalid workflow state
10. AI vs Deterministic Logic

A core architectural principle is to avoid using an LLM for decisions that can be expressed reliably as deterministic rules.

AI is used for:
Interpreting lead information
Assessing commercial potential
Generating qualification reasoning
Recommending a next sales action
Deterministic logic is used for:
Score validation
Priority thresholds
PO schema validation
Inventory calculations
Stock updates
State transitions
Exception handling

This hybrid approach improves predictability and makes the system easier to audit.

11. Current Prototype Boundary

AURA is currently a prototype.

The PO inventory workflow operates on the first PO line item in several expressions. Supporting arbitrary numbers of PO line items requires explicit item iteration and transactional inventory operations.

Other production concerns include:

Authentication and authorization
Idempotent PO processing
Transactional inventory updates
Retry and recovery mechanisms
Comprehensive automated testing
Observability
Human approval gates for high-impact actions
Production deployment architecture

These are identified as future engineering work rather than being represented as completed capabilities.
