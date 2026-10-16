# Salesforce Revenue Recognition — Reference Architecture

A declarative-first reference architecture for managing the post-Close Won handoff: project tracking, milestone-based deliverable decomposition, payment scheduling, and cash-collection forecasting. Anonymized and extracted from a production engagement.

## The problem this solves

Standard Salesforce covers the sales funnel (Lead → Opportunity → Closed Won) and the forecast roll-up, but it doesn't natively model the middle: turning a Closed Won Opportunity into a Project, decomposing the Project into Deliverables and unit-level work items, and computing when cash will arrive based on negotiated Net Terms. This reference fills that gap with four custom objects and fifteen flows — and zero Apex on the revenue-recognition path.

## Tech stack

- Salesforce Enterprise Edition (or any edition with Opportunity Products + Flow Builder + custom objects)
- Declarative metadata only: Flows, Validation Rules, formula and roll-up summary fields
- SFDX source format, API version 66.0
- No managed packages, no Apex on the revenue path

## Before you deploy

This repository is the architecture surface, not a complete org. Create these in the target org first, or the deploy will fail:

- **Tabs** for `Project__c`, `Project_Deliverable__c`, `Deliverable_Line_Item__c` and `Payment_Schedule__c`. The permission sets grant access to them.
- **Report folder** `SOWLifecycle`, for the four reports.
- **Email alert** `Opportunity.Notification_to_Finance_when_SOW_Delivered_TRUE`, used by the SOW notification flow.
- **Public group** `Finance_Team`, used by the approval flow.
- **Opportunity Splits** enabled, for the credit allocation flow.
- **Opportunity stage values** `Propose`, `Negotiate`, `Hold` and `Commit`.

Then deploy with `sf project deploy start --source-dir force-app --target-org <alias>`.

**Deployment status:** the metadata has not yet been re-validated against a scratch org since the object definitions were completed in October 2026.

## Folder structure

```
salesforce-revenue-recognition/
├── README.md                       (this file)
├── sfdx-project.json
├── SECURITY.md
├── docs/
│   ├── 01-architecture-overview.md
│   ├── 02-data-model.md            (includes Mermaid ERD)
│   ├── 03-automation-patterns.md
│   └── 04-deployment-strategy.md
├── force-app/main/default/
│   ├── objects/
│   │   ├── Account/                (in-scope custom fields + VRs only)
│   │   ├── Opportunity/            (in-scope custom fields + VRs only)
│   │   ├── Project__c/             (full)
│   │   ├── Project_Deliverable__c/ (full)
│   │   ├── Deliverable_Line_Item__c/ (full)
│   │   └── Payment_Schedule__c/    (full)
│   ├── flows/                      (15 flows)
│   ├── permissionsets/             (4 permission sets)
│   └── reports/SOWLifecycle/       (4 reports; no Apex classes, by design)
└── .gitignore
```

## Read order

1. **[01-architecture-overview.md](docs/01-architecture-overview.md)** — the problem, the design decisions, the tradeoffs (5 min read)
2. **[02-data-model.md](docs/02-data-model.md)** — objects, fields, relationships, Mermaid ERD (5 min read)
3. **[03-automation-patterns.md](docs/03-automation-patterns.md)** — flow patterns, the formula-trigger gotcha, the cross-transaction race idiom (8 min read)
4. **[04-deployment-strategy.md](docs/04-deployment-strategy.md)** — track-based deployment, permission set strategy, retirement path (5 min read)

## Status

Reference architecture extracted from a production Salesforce engagement. **Anonymized.** Stakeholder names, customer names, downstream-ERP names, and tenant-specific identifiers have been removed. Field names are preserved where neutral; API names were not renamed.

The metadata in this repository is the deployable architecture surface. Real production data, customer-specific record values, integration credentials, sandbox URLs, and personally-identifying information are not included.

## Not included

By design, this repository excludes:

- **Real data.** No records, no exports, no migration CSVs.
- **Customer-specific configurations.** Page layouts, Lightning pages, app menus, dashboards, custom apps, and any UI surfaces that were branded or tuned for the production org.
- **Industry-specific fields and labels.** A small set of Opportunity fields in the original org carried domain-specific naming and segmentation that doesn't generalize as reference architecture. These were excluded from the extract.
- **File-management automation.** The original org included ContentDocumentLink/ContentVersion triggers and an OpportunityFileCount batch for file-count rollups onto Opportunity. These are file-management plumbing, unrelated to revenue recognition, and excluded for clarity.
- **Notification flows tied to obsolete state.** Two flows that fired email alerts on Opportunity Amount changes and SOW PDF attachment events were marked `Obsolete` in source; they're excluded from the reference.
- **Integration credentials, named credentials, external services.** None included.
- **Dashboards, custom apps, custom labels, custom metadata types, profiles.** Out of scope.

## License and reuse

Forking and adapting this reference is encouraged. It's MIT-licensed: keep the copyright and license notice when you reuse it. The architecture patterns are not novel inventions — they're the result of running into the same Salesforce platform constraints (formula fields don't fire RTFs, lookup fields can't drive matching rules, same-transaction race conditions in cascading triggers) and arriving at the same canonical workarounds that any experienced admin/architect would arrive at independently. The value in this repository is in seeing all of them assembled in one coherent shape.

## Related

[salesforce-finance-layer](https://github.com/marilometa/salesforce-finance-layer) is the later, fuller version of this work: invoicing, collections and an Apex revenue recognition engine, with a scratch-org demo. The two repositories are separate references and are not meant to be installed in the same org. Some names overlap with different meanings: `Project__c.Opportunity__c` is a lookup here and a master-detail there, and payment terms are modeled differently.

Maintained by [Marilo Meta](https://marilometa.github.io/).
