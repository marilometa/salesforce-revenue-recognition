# Data Model

## Object map

```mermaid
erDiagram
    Account ||--o{ Opportunity : "has"
    Account ||--o{ Project : "has"
    Opportunity ||--o{ OpportunityLineItem : "products"
    Opportunity ||--o| Project : "spawns (one-to-one via flow)"
    Opportunity ||--o{ Payment_Schedule : "billing milestones"
    Project ||--|{ Project_Deliverable : "decomposes into"
    OpportunityLineItem ||--|| Project_Deliverable : "1:1 backing"
    Project_Deliverable ||--|{ Deliverable_Line_Item : "unit-level"
    Project_Deliverable ||--o{ Payment_Schedule : "milestone-tied (optional)"

    Account {
        picklist Net_Terms__c "10 active values"
        date MSA_Execution_Date__c
        date MSA_Expiration_Date__c
        date NDA_Execution_Date__c
        date NDA_Expiration_Date__c
        multipicklist Account_Business_Unit__c
    }
    Opportunity {
        picklist Net_Terms__c "inherited from Account"
        multipicklist Payment_Terms__c
        checkbox Finance_Reviewed__c
        checkbox Legal_Reviewed__c
        checkbox SOW_Delivered__c
        date SOW_Delivered_Date__c
        checkbox SOW_Executed__c
        checkbox Opportunity_Credit_Allocated__c
        url Budget_SharePoint_Folder__c
        url SOW_SharePoint_Folder__c
        date Kickoff_Date__c
        date End_Date__c
        text Project_Number__c "legacy backfill"
    }
    Project {
        lookup Account__c
        lookup Opportunity__c
        lookup Project_Manager__c
        picklist Project_Status__c
        date Kickoff_Date__c
        date Estimated_End_Date__c
        currency Project_Total_Revenue__c "formula"
        number Number_of_Project_Deliverables__c "rollup"
        number Delivered_Deliverables__c "rollup"
        number Delivered_Percentage__c "formula"
        text Delivery_Progress__c "formula"
    }
    Project_Deliverable {
        masterDetail Project__c
        lookup Deliverable__c "OpportunityLineItem"
        text Deliverable_Name__c "formula from Product Name"
        picklist Deliverable_Status__c
        number Quantity__c "formula from OLI.Quantity"
        number Quantity_RollUp__c "rollup from DLI count"
        number Delivered_Units__c "rollup of DLIs marked Delivered"
        currency Unit_Price__c "formula from OLI.UnitPrice"
        currency Amount__c "formula from OLI.TotalPrice"
        date Expected_Delivery_Date__c "formula from OLI.ServiceDate"
        date Actual_Delivery_Date__c
    }
    Deliverable_Line_Item {
        masterDetail Project_Deliverable__c
        text Account__c "formula via Project lookup"
        text Deliverable_Item_Name__c "formula"
        picklist Delivery_Status__c
        number Quantity__c "fixed at 1"
        currency Unit_Value__c
        currency Total_Value__c "formula"
        date Due_Date__c
        date Delivered_Date__c
        text Due_Month__c "formula YYYY-MM"
    }
    Payment_Schedule {
        lookup Opportunity__c "required"
        lookup Deliverable__c "Project_Deliverable"
        text Milestone_Name__c
        picklist Trigger_Type__c "Date / Deliverable / Manual"
        date Expected_Trigger_Date__c
        currency Expected_Amount__c
        date Invoice_Date__c
        date Expected_Payment_Date__c "formula"
        date Actual_Payment_Date__c
        currency Actual_Payment_Amount__c
        picklist Status__c
    }
```

## Cardinality and ownership

Project sits in a one-to-one relationship with Opportunity, enforced at create time by a before-save flow (`Prevent_Duplicate_Project_Per_Opportunity`) rather than by a Duplicate Rule — lookup fields don't carry the custom search index required by matching rules, so the rule-based approach is unavailable. The same idea on the side-car: Payment Schedule sits in a many-to-one relationship with Opportunity, marked `Restrict` on delete so a Closed Won Opp can never quietly lose its billing trail.

The Project → Project Deliverable and Project Deliverable → Deliverable Line Item relationships are both master-detail. This is a deliberate choice over lookup: it gives roll-up summary fields (`Quantity_RollUp__c`, `Delivered_Units__c`), inherits sharing from the parent (so the Project sharing model dictates downstream access), and cascades deletion in the rare case where a Project is removed in admin cleanup.

## Sharing model

`Project__c` and `Payment_Schedule__c` are both set to `Private` with `Public` visibility on the object itself — meaning users see the tab but only see the records the sharing model lets them see. The intent is to let a Project Manager see only their projects (driven by `Project_Manager__c` lookup-based sharing) and to let Finance see all Payment Schedules via the `Cash_Forecasting_Access` permission set, which doesn't carry `viewAllRecords` so it still respects record-level sharing.

Deliverables and DLIs inherit sharing from their parent Project via master-detail. There's no separate sharing surface to manage for the lower levels of the hierarchy — a deliberate simplification.

## Field naming conventions

API names are intentionally neutral and self-describing. Lookup fields use the standard `_c` suffix with no relationship label decoration; relationship names (`Projects`, `Project_Deliverables`, `Deliverable_Line_Items`, `Payment_Schedules`) are pluralized so they read naturally in SOQL. Formula fields carry a `description` that names the upstream source field explicitly, so a future maintainer running `sf project retrieve` and reading the XML can reconstruct the data flow without opening the Lightning Setup UI.
