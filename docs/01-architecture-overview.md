# Architecture Overview

## The problem

Mid-market B2B services organizations carry a particular shape of revenue: long-cycle Opportunities, each closing into a Project with a Statement of Work that decomposes into a handful of Deliverables, and a milestone-tied or date-tied invoicing cadence determined by the customer's negotiated payment terms. Standard Salesforce gives you the front of that workflow (Opportunity, Opportunity Products, Stages) and the back of it (Closed Won, Forecasting), but it doesn't natively model the middle: the post-close handoff into delivery and cash-collection forecasting. The organization needs three things that don't ship in the box — a project shape, a unit-of-work shape under each deliverable, and a billing schedule that knows how to translate "Net 30 EOM" into a date you can put on a forecast.

This reference architecture is the middle. It introduces a `Project__c → Project_Deliverable__c → Deliverable_Line_Item__c` hierarchy plus a side-car `Payment_Schedule__c` object. It hangs all of that off the standard Opportunity and Account model so the front of the funnel doesn't need to change. And it does it declaratively — flows, validation rules, formulas, roll-up summaries — with zero Apex on the revenue-recognition path.

## Design decisions

**Custom objects, not Revenue Cloud.** Salesforce's native milestone billing was evaluated and declined. It's powerful, but it brings opinionated objects (`BillingSchedule`, `Order`, `Asset`) and licensing implications that are disproportionate for an organization whose milestone count per deal is low single digits and whose ERP-of-record sits elsewhere. The custom `Payment_Schedule__c` object is intentionally a thin shape — eleven fields — designed for graceful retirement once a downstream ERP assumes the cash forecasting role. Field names mirror standard ERP billing schedule structures to make that migration mechanical.

**Three-level deliverable hierarchy.** Project → Deliverable → Deliverable Line Item exists because organizations frequently sell deliverables in quantities (e.g., "five reports") but track delivery one unit at a time. A flat object can't reconcile "Quantity = 5 on the Opportunity Product" with "Unit 3 delivered last Tuesday." The DLI fan-out (one DLI per unit of OLI Quantity) is the smallest model that lets ops, finance, and sales tell the same story without spreadsheet exports.

**Declarative over Apex.** Every automation in this reference is a Flow or Validation Rule. The reason is operational, not theological: the team owning post-close handoff is closer to admin than to engineering, and the cost of evolving a flow is lower than the cost of evolving an Apex class with a test class behind it. The patterns documented in [03-automation-patterns.md](03-automation-patterns.md) — before-save inheritance, scheduled-path race avoidance, formula-aware re-trigger placement, idempotent backfill — are all things one would otherwise reach for triggers and queueable Apex to solve.

## Tradeoffs

The custom-object approach trades depth for fit. There's no native invoicing, no payment matching against an actual cash event — just expected and actual dates. The dual-format Net Terms handling (active "Net 30" values plus a tolerance for legacy "X days" strings) is a real cost paid for migration grace, and it's worth removing once the data has fully converged. The 0-minute scheduled path used to dodge same-transaction races (Project Create kicks off both `Create_Project` and `Bootstrap_Deliverables`) is a known idiom but feels like a workaround for anyone who hasn't run into the same race twice. None of these are fatal; all are visible in the metadata for future maintainers to inherit cleanly.
