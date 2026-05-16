# Automation Patterns

## Declarative-only

Every business rule in this reference is implemented with Flows, Validation Rules, formula fields, and roll-up summaries. Zero Apex on the revenue-recognition path. This is a design choice, not an accident — the team owning post-close automation is closer to admin than engineering, and flow-based code lowers the cost of evolution at the price of a slightly higher ceiling on what's expressible. For the shape of work in this domain (sequence a few records, copy a few values, gate a few stages), the ceiling never comes up.

## Inheritance and bootstrapping

**Net Terms inheritance (`Set_Opportunity_Net_Terms_From_Account`)** runs as a before-save flow on Opportunity Create. It uses cross-object dot-notation (`$Record.Account.Net_Terms__c`) to read the parent Account without an explicit Get Records step. Before-save is the right placement here: the value is set in the same transaction as the insert, so downstream automation sees it immediately, and there's no extra DML cost. Users can override at any time after create.

**Project bootstrap (`Create_Project`)** runs as an after-save record-triggered flow on Opportunity, fired when StageName transitions to "Commit." It queries related OpportunityLineItems, creates exactly one Project, then loops through OLIs creating one Project_Deliverable per OLI. The flow is idempotent — it skips if a Project already exists for the Opportunity — so a Stage rollback-and-forward doesn't double-create.

**Cross-transaction race avoidance (`Bootstrap_Deliverables_on_Project_Create`)** is the most subtle pattern in the architecture. When `Create_Project` inserts a `Project__c`, both `Create_Project` (still completing on the Opportunity transaction) and any flow triggered by Project Create want to read OpportunityLineItems. If both ran in the same transaction, the second flow would race against still-uncommitted state. The solution: a 0-minute scheduled path on Project Create. Zero minutes is enough to push execution to a separate transaction, eliminating the race. The flow checks for existing Deliverables before creating, so if `Create_Project` already populated them, it's a no-op.

## Quantity sync and the formula-trigger gotcha

`Project_Deliverable__c.Quantity__c` is a formula derived from `OpportunityLineItem.Quantity`. Salesforce does not fire record-triggered flows on formula recalculation, so a flow listening to `Project_Deliverable` Update with `ISCHANGED(Quantity__c)` will never fire when the upstream OLI Quantity changes.

The fix lives in `Sync_DLIs_On_OLI_Quantity_Change`, which sits on OpportunityLineItem (not Project_Deliverable). It listens for `ISCHANGED(Quantity)`, counts existing DLIs on the related Project_Deliverable, and either creates the delta (Quantity increase) or deletes oldest-Pending-first DLIs (Quantity decrease). It never deletes DLIs marked `Delivered` — a delivery already in the books is immutable from the automation's perspective.

The companion flow `Create_Deliverable_Line_Items_from_Quantity` handles the Create case: when a new Project_Deliverable is inserted with `Quantity > 0`, it builds a collection of N DLI records and writes them in a single DML to stay inside governor limits at high quantities.

## Status roll-up

`Roll_Up_Deliverable_Status_from_Line_Items` runs after every Project_Deliverable update. When `Delivered_Units__c` (a count rollup of DLIs marked Delivered) equals `Quantity__c`, the flow stamps the Deliverable as Delivered and sets `Actual_Delivery_Date__c = Today`. The Project's `Delivered_Percentage__c` is a downstream formula on the count of Delivered child Deliverables divided by total. Three layers of rollup or formula, zero Apex.

## Approval gating

The SOW lifecycle is gated by a sequence of validation rules, not by a single state machine. `SOW_Delivered_Requires_Reviews` blocks `SOW_Delivered__c = true` unless both `Legal_Reviewed__c` and `Finance_Reviewed__c` are true. `SOW_Executed_Requires_Reviews` blocks `SOW_Executed__c = true` unless all three (Legal, Finance, Delivered) are true. `Require_SOW_Executed_Before_ClosedWon` blocks the final Stage transition unless Executed is true. The result is a deterministic chain: a Closed Won record always has Legal Reviewed → Finance Reviewed → SOW Delivered → SOW Executed in its history. The `Finance_Approval_Process` autolaunched flow (kept as `Draft` in source) plugs into this chain when the team wants a formal approval submission rather than a manual checkbox flip.

## Restrictive validation

The `Restrict_Opportunity_Type` validation rule guards the `"Data Providers"` value on Opportunity.Type. It uses a profile-based allowlist (`$Profile.Name = "Data Provider Manager"` or `"System Administrator"`) rather than a username-based allowlist — profiles scale without per-user maintenance and survive personnel changes cleanly.

## Idempotent backfill pattern

Three flows (`Backfill_Projects_For_Legacy_Closed_Won_Opps`, `Create_Deliverables_for_Legacy_Project`, `Legacy_Closed_Won_Backfill_Project_Deliverables_DLI`) together implement a one-time backfill for legacy Closed Won Opportunities that pre-date the Project model. The pattern is worth calling out: each flow checks for existing downstream records before creating, so the trio can be re-run safely. They're shipped as `Draft` so an admin triggers them manually after data validation — no surprise mass-create on deploy.

## Discovery note: Finance_approval draft consolidation

During Phase 2 extraction, three flows in source — `Finance_approval`, `Finance_approval_flow`, `Finance_approval_opp` — were found to be **byte-identical, status `InvalidDraft`**, and contained only Salesforce-generated approval-flow template scaffolding (empty stages, no approver, no record reference). They were consolidated and excluded from this reference architecture; the canonical approval pattern is represented by `Finance_Approval_Process` (status `Draft`), which has actual approval logic wired to a `Finance_Team` group. The three boilerplate drafts are not load-bearing; they reflect setup-time exploration in the Salesforce UI that produced duplicate empty shells.
