# Workflow Automation — BPMN Process Modeling

RA2411003012179

This repository contains BPMN 2.0 models for three business process scenarios, built using [bpmn.io](https://bpmn.io) (Camunda's browser-based modeler). Each scenario folder contains the raw `.bpmn` XML source file and an exported image of the diagram.

## Repository structure

```
.
├── scenario1-leave-approval/
│   ├── leave-approval.bpmn
│   └── leave-approval.png
├── scenario2-purchase-order/
│   ├── purchase-order.bpmn
│   └── purchase-order.png
├── scenario3-it-service-request/
│   ├── it-service-request.bpmn
│   └── it-service-request.png
└── README.md
```

## Tools used

- **Modeler**: [bpmn.io](https://demo.bpmn.io) / Camunda Modeler
- **Notation**: BPMN 2.0

---

## Scenario 1: Employee Leave Approval

### Process description

An employee wants to apply for leave through the company's HR system. The process is designed so that the system validates the request *before* a manager's time is spent on it — leave balance is checked first, and only requests that pass that check are ever routed for approval. This means an employee with insufficient balance gets an immediate, automatic rejection without the manager being involved at all.

![Leave Approval Diagram](scenario1-leave-approval/leave-approval.png)

### BPMN elements used

| Element | Count | Notes |
|---|---|---|
| Start event | 1 | Triggered by employee submitting the request |
| Tasks | 5 | Check balance, manager approval, update balance + notify, send rejection, send insufficient-balance notice |
| Exclusive gateways | 2 | "Balance sufficient?" and "Manager approves?" |
| End events | 3 | One per distinct outcome — approved, rejected, insufficient balance |

### Detailed workflow explanation

**Step 1 — Start event.** The process is triggered the moment the employee submits a leave request through the HR system. Nothing is evaluated yet; this is purely the entry point.

**Step 2 — Check leave balance (task).** As soon as the request comes in, the system automatically looks up the employee's current leave balance and compares it against the number of days requested. No human is involved in this step — it's a straightforward data check.

**Step 3 — Gateway: "Balance sufficient?"** This is the first decision point, and it's an *exclusive* gateway, meaning exactly one of the two outgoing paths will fire, never both.
- If the balance is **insufficient**, the process takes the short path: it immediately sends an insufficient-balance notification to the employee and ends. The manager is never contacted, because there's nothing for them to approve — the system already knows the request can't be honored.
- If the balance is **sufficient**, the request moves forward to the manager.

**Step 4 — Manager approval (task).** This is the one human-driven step in the whole process. The request sits with the manager, who reviews it and makes a decision. Everything before this point was automated; this is where a person actually exercises judgment.

**Step 5 — Gateway: "Manager approves?"** The second decision point.
- If the manager **rejects** the request, the system sends a rejection notification and the process ends.
- If the manager **approves** the request, the system updates the employee's leave balance *and* sends an approval notification — both of these happen together in a single task, since the requirement describes them as one combined action with no decision in between.

**Step 6 — End.** The process ends after whichever notification was appropriate for the outcome. There are three separate end events — one for insufficient balance, one for rejection, one for approval — rather than merging everything into a single end event, because each of these is a genuinely different, independent terminal state for the employee's request.

### Task type justification

| Task | Type | Reason |
|---|---|---|
| Check leave balance | Service task | Automated system lookup |
| Manager approval | User task | Human decision |
| Update balance + notify approval | Service task | Fully automated |
| Send rejection notification | Send task | Sole purpose is sending a message |
| Send insufficient-balance notice | Send task | Sole purpose is sending a message |

### Modeling note

Three separate end events were used rather than converging all outcomes into one, since the requirement explicitly treats each notification as a distinct terminal state ("the process ends after the appropriate notification is sent").

---

## Scenario 2: Online Purchase Order Processing

### Process description

A customer places an online order, and the system runs it through two independent checks in sequence — stock availability first, then payment — before the order is actually fulfilled. Either check can fail on its own, and each failure ends the process at a different point. Only an order that clears both checks proceeds all the way through to shipment.

![Purchase Order Diagram](scenario2-purchase-order/purchase-order.png)

### BPMN elements used

| Element | Count | Notes |
|---|---|---|
| Start event | 1 | Triggered by customer placing the order |
| Tasks | 7 | Check availability, process payment, confirm order, prepare shipment, ship order, send shipping confirmation, plus 2 failure-notification tasks |
| Exclusive gateways | 2 | "Product available?" and "Payment successful?" |
| End events | 3 | Out of stock, payment failed, order shipped |

### Detailed workflow explanation

**Step 1 — Start event.** Triggered the moment the customer places the order. This is the entry point for the whole fulfillment pipeline.

**Step 2 — Check product availability (task).** Before touching payment at all, the system checks whether the product is actually in stock. This ordering matters: there's no point charging a customer for something that can't be shipped.

**Step 3 — Gateway: "Product available?"** The first decision point.
- If the product is **unavailable**, the system notifies the customer that the item is out of stock, and the process ends immediately — payment is never attempted.
- If the product is **available**, the flow moves on to payment.

**Step 4 — Process payment (task).** The system runs the payment through, e.g. a payment gateway. This only happens once availability is already confirmed.

**Step 5 — Gateway: "Payment successful?"** The second, independent decision point.
- If payment **fails**, the system notifies the customer about the payment failure, and the process ends there — no order gets shipped.
- If payment **succeeds**, the process continues into the fulfillment chain.

**Step 6 — Confirm order → Prepare shipment (tasks).** Once payment clears, the system confirms the order and prepares the product for shipment. These are modeled as two separate tasks rather than one combined step, since they're conceptually distinct actions — confirming is a system/customer-facing acknowledgment, while preparing shipment is a warehouse-side physical action — even though nothing branches between them.

**Step 7 — Ship order (task).** The prepared product is physically handed off to a carrier.

**Step 8 — Send shipping confirmation (task).** The customer receives confirmation that their order is on its way.

**Step 9 — End.** The process ends here, on the success path. Combined with the two earlier failure paths, this gives three distinct end events total — out of stock, payment failed, and successfully shipped — each representing a genuinely different outcome for the order.

### Task type justification

| Task | Type | Reason |
|---|---|---|
| Check product availability | Service task | Automated inventory query |
| Process payment | Service task | Automated payment gateway call |
| Confirm order | Service task | System-generated confirmation |
| Prepare shipment | Manual task | Physical warehouse work |
| Ship order | Manual task | Physical carrier handoff |
| Send shipping confirmation | Send task | Pure notification |
| Notify out of stock | Send task | Pure notification |
| Notify payment failure | Send task | Pure notification |

### Modeling note

"Confirms the order and prepares the product for shipment" (a single line in the original requirement) was split into two separate tasks — Confirm order and Prepare shipment — since they represent conceptually distinct actions, even though no decision separates them.

---

## Scenario 3: IT Service Request Handling

### Process description

An employee reports an IT problem, and the help desk routes it based on severity — but unlike the first two scenarios, every path through this process eventually leads to the same place: the ticket gets resolved and the employee gets notified. There's no "dead end" outcome here, which is why this model looks structurally different — it uses converging gateways to merge branches back together, rather than letting each branch run to its own separate end event.

![IT Service Request Diagram](scenario3-it-service-request/it-service-request.png)

### BPMN elements used

| Element | Count | Notes |
|---|---|---|
| Start event | 1 | Triggered by employee reporting the problem |
| Tasks | 10 | Submit, register, check severity, assign (x2), investigate, fix, escalate, update status, notify |
| Exclusive gateways | 4 | 2 diverging (severity, resolvable) + 2 converging (merge back after each split) |
| End event | 1 | All paths lead to the same outcome — ticket resolved and employee notified |

### Detailed workflow explanation

**Step 1 — Start event.** Triggered when the employee reports an IT problem.

**Step 2 — Submit IT support request (task).** The employee formally files the request — this is the human action that generates the ticket.

**Step 3 — Register request (task).** The IT help desk logs the request into their system, giving it an official record and identifier.

**Step 4 — Check severity (task).** The help desk assesses how serious the problem is. This is modeled as its own task — separate from the gateway that follows — because it's a distinct activity the help desk performs, the same way "check leave balance" was its own task in Scenario 1 before its gateway.

**Step 5 — Gateway: "Severity high?"** The first diverging decision.
- **Low severity** → the ticket is assigned to a general support technician.
- **High severity** → the ticket is assigned to a senior technician instead.

**Step 6 — Converging gateway.** This is the key structural difference from Scenarios 1 and 2. Both assignment paths — regardless of which technician tier picked it up — merge back into a single flow here, because from this point onward the process behaves identically no matter who's handling the ticket.

**Step 7 — Investigate problem (task).** Whichever technician was assigned now investigates the issue. Because the previous step merged the two paths, this is a single shared task rather than two duplicated ones.

**Step 8 — Gateway: "Resolvable internally?"** The second diverging decision.
- **Yes** → the technician fixes the problem directly.
- **No** → the technician escalates the problem to an external service provider.

**Step 9 — Converging gateway.** Just like after the severity split, both resolution paths — fixed internally or escalated externally — merge back into one flow, because the next steps (updating status, notifying the employee) happen identically either way.

**Step 10 — Update request status (task).** The help desk updates the ticket's status to reflect that it's been resolved, regardless of which route got it there.

**Step 11 — Notify employee (task).** The employee receives a resolution notification.

**Step 12 — End.** A single end event closes the process. This is deliberate and matches the requirement's element list, which specifies "End Event" in the singular (unlike Scenarios 1 and 2, which needed plural end events) — every branch in this process is a different *route*, not a different *outcome*.

### Task type justification

| Task | Type | Reason |
|---|---|---|
| Submit IT request | User task | Employee fills out a form |
| Register request | User task | Help desk logs it manually |
| Check severity | User task | Human judgment call (or Business rule task if automated by a rules engine) |
| Assign to technician / senior technician | User task | Help desk manually routes it (or Business rule task if a fixed decision table) |
| Investigate problem | Manual task | Offline diagnostic work outside the system |
| Fix problem | Manual task | Physical/technical work |
| Escalate to external provider | Send task | Sending ticket details out (or Service task if a fully automated API call) |
| Update request status | User task | Explicit human actor per the requirement text |
| Notify employee | Send task | Pure notification |

### Modeling note — why this scenario differs from Scenarios 1 and 2

In Scenarios 1 and 2, each branch leads to a genuinely different terminal state, so multiple end events were used. In Scenario 3, the branches (technician tier, resolvable vs. escalated) are different *routes* to the same destination — the ticket is always resolved and the employee is always notified. This is modeled with converging exclusive gateways (one for each split) rather than separate end events, which is the correct BPMN pattern for "different paths, same outcome."

---

## Summary: requirement compliance

| Requirement | Status |
|---|---|
| Complete BPMN diagrams for all 3 scenarios | ✅ |
| Modeled using bpmn.io / Camunda Modeler | ✅ |
| `.bpmn` source files + diagram images uploaded | ✅ |
| README explaining each scenario and its BPMN process | ✅ (this file) |
| Repository organized into clear folders | ✅ |
| Models verified for completeness and logical correctness | ✅ |