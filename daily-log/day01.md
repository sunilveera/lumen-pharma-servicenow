# Week 1, Day 1 · Platform foundations

**Date:** 2026-11-17
**Instance:** Personal Developer Instance (PDI)

## What I built

- **Company:** Lumen Pharma (`core_company`)
- **Locations** (`cmn_location`), all linked to Lumen Pharma:
  - Lumen HQ (US), Indianapolis
  - Lumen Hyderabad Tech Centre, Hyderabad
  - Lumen Ireland Plant, Ringaskiddy (GMP manufacturing)
- **Departments** (`cmn_department`), all linked to Lumen Pharma: Manufacturing, Quality Assurance, Clinical Operations, Medical Affairs, R&D Informatics, IT
- **Application CI:** Lumen LIMS (`cmdb_ci_appl`)
- **Incidents** against the Lumen LIMS CI:
  - P1: "LIMS down, batch release blocked"
  - P3: "Request Access to LIMS is not working"
- **Saved favourite:** "Active P1/P2 incidents" (Active = true AND Priority is 1 or 2), with dot-walked columns Caller.Department and Caller.Location

## What I learned

- **Everything is a record.** Tables are rows in `sys_db_object`; fields are rows in `sys_dictionary`.
- **sys_id** is a 128-bit GUID (32 hex characters), the true unique key. Incident numbers come from a counter and can be edited or clash after imports.
- **Table extension:** `incident` extends `task`. Inherited fields include Correlation ID, Urgency, Opened, SLA due, Escalation and Additional comments. Incident-only fields include Subcategory, Probable cause and Last reopened at. Fields like Created by and Updated by are system fields that every table has.
- **Dot-walking** follows reference fields to related records (Caller → Department). Add dot-walked columns via column header right-click → Configure → List Layout → expand the reference field. The gear icon only offers the table's own fields.
- **Personal vs shared changes:** the list gear changes only my view; Configure → List Layout changes it for everyone.
- **Time zones:** ServiceNow stores times in UTC and displays them per user. Changing my time zone changed the display, not the data.
- **Impersonation** shows how roles shape the UI. Abel Tuter (no itil or admin role) only sees Self-Service apps.
- **Incident vs request:** "Request access to LIMS" is a service request (Service Catalog), not an incident. "Access to LIMS is not working" is an incident.

## What broke and how I fixed it

### Time zone missing from Preferences
Asia/Kolkata and Asia/Calcutta weren't listed. On a PDI, most time zone choices are inactive in `sys_choice` (Table = sys_user, Element = time_zone). I activated the choice, then selected it in Preferences.

### Lumen Pharma not offered in the Department → Company lookup
- **Diagnosis:** right-click the Company label → Configure Dictionary → Reference Specification. The field has a reference qualifier: **Customer = true**.
- **Fix:** ticked **Customer** on the Lumen Pharma company record. I didn't remove the qualifier, because changing base config affects every picker on the instance and would need agreement from the platform owner.
- **Lesson:** "I can't see my record in a lookup" usually means a reference qualifier. Check the dictionary first.

### Modelling mistake
I first created "Lumen HQ (US)" as a Company. Companies are organizations; sites belong in `cmn_location` with a reference to the company. I renamed the record to Lumen Pharma and created the HQ as a location.

## Discovery note (FDE practice)

Scenario: Lumen's Head of Manufacturing IT says *"When the LIMS goes down, QA can't release batches and we lose a day."*

1. **Impact:** LIMS outages hold finished batches in quarantine. Each lost day delays shipments (e.g. insulin) and risks supply. Cost per day to be confirmed.
2. **Affected:** QC analysts, QA reviewers, lab informatics and IT, the LIMS vendor, production planning, supply chain; downstream, patients.
3. **Today (assumed):** reported by email to IT, who escalate to the vendor. No priority, SLA or tracking; history lives in inboxes.
4. **Data:** LIMS-CI incidents (count, MTTR, reassignments), CMDB dependencies (DB, MES/SAP/CDS links), changes before outages, problem records.
5. **Next question:** In the last 12 months, how many outages blocked batch release, how long did each last, and what was the main cause?

**Value of ServiceNow here:** logged against the Lumen LIMS CI, every outage gets a priority, an owner and an SLA clock, and QA is notified automatically. Over time the CI builds a history that shows the real cost of LIMS downtime.

## Open items

- [ ] Assignment group on LIMS incidents (create Lab Informatics Support on Day 3)
- [ ] Department ID codes (MFG, QA, CLIN, MEDA, RDI, IT)
- [ ] Finish the "Welcome to ServiceNow" course on Day 7
