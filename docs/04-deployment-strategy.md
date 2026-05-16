# Deployment Strategy

## Environment shape

The architecture was developed under a three-environment chain: a long-lived **Sandbox** for build, a **UAT** sandbox for stakeholder validation, and **Production**. Sandbox is refreshed from Production at irregular intervals (driven by data volume tolerance rather than cadence). The SFDX project is the source of truth for declarative metadata; manual changes in any environment are expected to be re-discovered on the next `sf project retrieve` and committed back.

The four custom objects, their fields, validation rules, flows, permission sets, and reports are all deployed via metadata-API deployment from this repository. There is no managed package; this is intentional — the architecture is meant to be readable, forkable, and trivially modifiable by the downstream owner.

## Track-based scope splitting

The build is organized into discrete **Tracks**, each representing a coherent slice of functionality that can be deployed and validated independently. The numbering is project-historical rather than priority-ordered:

- **Track 1** — Opportunity custom fields (Net Terms, Payment Terms, SOW lifecycle checkboxes, Kickoff/End dates) plus the Opportunity validation rules. Foundational; everything else depends on it.
- **Track 2** — Account custom fields (Net Terms, MSA/NDA gates, Account Business Unit). Independent of Track 1 except for the Net Terms inheritance flow, which is wired in Track 3.
- **Track 3** — Project / Project_Deliverable / Deliverable_Line_Item custom objects, plus the flows that bootstrap and reconcile them (`Create_Project`, `Bootstrap_Deliverables_on_Project_Create`, `Create_Deliverable_Line_Items_from_Quantity`, `Sync_DLIs_On_OLI_Quantity_Change`, `Roll_Up_Deliverable_Status_from_Line_Items`, `Prevent_Duplicate_Project_Per_Opportunity`). Also includes `Set_Opportunity_Net_Terms_From_Account`.
- **Track 4** — Payment_Schedule__c and the `Cash_Forecasting_Access` permission set. Standalone in the sense that nothing else depends on it; can be deployed (or retired) without disturbing Tracks 1–3.
- **Track 5** — SOW lifecycle gating (`Finance_Reviewed__c`, `Legal_Reviewed__c`, `SOW_Delivered__c`, `SOW_Executed__c` and their validation rules) and the Finance approval flows. Sits architecturally between Tracks 1 and 3.
- **Track 6** — Legacy backfill flows (`Backfill_Projects_For_Legacy_Closed_Won_Opps`, `Create_Deliverables_for_Legacy_Project`, `Legacy_Closed_Won_Backfill_Project_Deliverables_DLI`). Deployed once, run manually by admin against the Closed-Won-with-Project_Number population, then left inactive.

Tracks deploy independently because the dependency graph is mostly acyclic. The exception is the OpportunityLineItem dependency in Track 3 — the deliverable bootstrap flows read OLIs, so the standard Opportunity Products object must already be enabled (it ships standard, but Pricebook setup is a prerequisite).

## Permission set strategy

Profiles are used for license, app, and system permissions only. All custom object access, custom field FLS, and tab visibility is granted via permission sets. This decouples "what can a user do" from "what kind of user are they" and lets the security model evolve without profile cloning.

The four permission sets in this extract divide along functional lines:

- `Access_Projects_and_Deliverables` — full CRED on the project hierarchy, for the delivery team.
- `Cash_Forecasting_Access` — read on Net Terms across Account/Opp, full CRED on Payment_Schedule, for finance/forecasting users.
- `Edit_rights_to_Finance_reviewed` — narrow grant for the single Finance Reviewed checkbox, for the finance leads who flip it.
- `Edit_rights_to_PM_notes` — narrow grant for the PM Notes field, scoped to PMs.

A user typically holds one functional permset plus possibly one or two narrow edit-rights permsets, layered over their license profile.

## Migration and retirement

`Payment_Schedule__c` is designed for graceful retirement. Its field naming maps cleanly to standard ERP billing schedule structures, so the eventual migration is mechanical: a one-shot data export from Salesforce keyed on Opportunity, ingested by the ERP, after which `Cash_Forecasting_Access` is unassigned and the object enters read-only sunset. The architecture's deliberate avoidance of cross-object dependencies on Payment_Schedule (no other object's roll-ups or formulas read from it) makes the retirement non-load-bearing.

Similarly, `Opportunity.Project_Number__c` is a transitional field — once the legacy backfill completes and all closed-won Opps have a related Project__c, the field can be deprecated in favor of the lookup relationship that already exists on Project. The dependent flows (`Backfill_*`) and the validation rule (`Project_Number_Must_Be_Numeric`) can be retired alongside it.

## Change set considerations

When propagating changes via Change Set rather than SFDX (e.g., for an org without a CI/CD pipeline), the typical leakage points are: Lightning Pages (which may carry org-feature-specific component props that fail validation in a target org lacking those features), Profiles (which carry all field permissions and grow unboundedly), and Layouts (which are environment-specific and verbose). For the metadata in this repository, none of those types are in scope — the surface area is small enough to ship via raw metadata API without spelunking through Change Set components.
