# Week 1, Day 2 · Tables & fields deep dive

**Date:** 2026-11-18
**Instance:** Personal Developer Instance (PDI)
**Exercise:** a friend's data dictionary for **MediCare HMS**, a fictional hospital (used for this day in place of Lumen Pharma). All data is simulated.

## What I built

- **Scoped app:** MediCare HMS (`x_1941695_medica_0_`), one application menu with a module per table
- **Update set:** HMS Data Model v1 (scoped). An earlier global set, "HMS Data Model", was set to Ignore.
- **7 tables, with sample data in each:**

| Table | Highlights |
| --- | --- |
| Medicine | Display = Medicine Name. Missing from the dictionary, added because Patient referenced it |
| Ward | Time, True/False, Translated HTML description |
| Doctor | List → Ward, Password (1 Way), Phone (E164), HTML bio |
| Patient | Auto-number PAT, Choice, List → Medicine, Journal, Currency (INR), Image, File Attachment, Reference → Ward |
| Appointment | Auto-number APT, References → Patient / Doctor / Ward, List → `sys_user` (nurses), Duration |
| Prescription | Auto-number RX, References → Patient / Doctor / Medicine |
| Medical Equipment | Extends `cmdb_ci_hardware`; reuses the inherited Cost field |

- **Instance-level change, outside the app:** `glide.i18n.single_currency` = true, `glide.i18n.single_currency.code` = INR (made in Global scope)

## Reviewing the dictionary first

I found 8 gaps before building:

- a referenced table that was never defined (Medicine)
- no doctor name field
- no medicine or dosage field on Prescription
- body temperature labelled °C with a °F example (98.6)
- an ICU report that needed a Ward field on Patient
- a business rule with no approval field to check
- no Visitor Hours End field on Ward
- Blood Group as free text instead of a Choice

## What I learned

- **Update sets** capture configuration, not data. Lifecycle: In progress → Complete → export or retrieve → Preview → Commit. Always use *Submit and Make Current*.
- **Scoped vs global:**
  - A scoped app has its own namespace and ships as one unit. It's the recommended approach for new apps.
  - Fields on your own scoped tables get no `u_` prefix.
  - Each scope has its own current update set.
- **Cross-scope protection:** a scoped app can't edit Global records such as system properties. Switch scope, edit, then switch back.
- **Reference vs List:**
  - Reference holds one record. It gets a schema-map line, dot-walking and related lists.
  - List holds many records, stored as a comma-separated list of sys_ids. The schema map draws no line for it, and its list column can't be searched.
- **Display field** decides how a record appears wherever it's referenced.
- **Auto-number** (table → Controls tab) creates a `number` field with a `getNextObjNumberPadded()` default. A number is assigned when a New form opens, so gaps are normal.
- **CMDB extension:** the child inherits the parent's fields, business rules, ACLs and scripts, and its records appear in `cmdb_ci_hardware` and `cmdb_ci` lists.
- **Schema map:** System Definition → Tables → table → Show Schema Map.

## What broke and how I fixed it

- **I could pick only one ward.** Assigned Wards had been saved as **Reference** from the Columns grid. I changed the Type to **List** in the dictionary. Lesson: check the type after creating a field.
- **The choice editor rejected values like `A+`.** I switched to values like `a_pos`. Labels can stay as they are.
- **Duplicate Patient Number.** I had created `patient_number` by hand and then turned on auto-number. I found the duplicate by filtering `sys_dictionary` on Column label and deleted it.
- **Currency showed USD.** I set the single-currency properties, which meant switching to Global scope ("Security prevents writing" otherwise).
- **One menu per table.** I merged them into one MediCare HMS menu, and from then on set *Add module to menu*.
- **A filter pulled in rows from other tables.** The Table operator was `>=`; it should have been *is*.
- **Not bugs:**
  - Phone (E164) shows a country dropdown plus a number box.
  - Journal fields are always empty in lists.

## Requirements noted for Week 2

- [ ] Block a prescription whose medicine is in the patient's Known Allergies (test case: Amoxicillin for a penicillin-allergic patient)
- [ ] Reference qualifier: limit Assisting Nurses to a Nurses group
- [ ] Validations: OPD End > Start; Available Beds ≤ Total Beds; Expiry > Prescription Date
- [ ] Calculate Age from Date of Birth
- [ ] Auto-confirm an appointment on doctor approval (needs a `doctor_approved` field)

## Self-check

My first attempt scored about 3 out of 5. Weak spot: **Reference vs List**.

## Open items

- [ ] Screenshot the schema map for the portfolio
- [ ] Fix labels: the "Needs Repair" choice, and the Medical Equipment plural label
- [ ] Day 3: forms, UI policies, data policies (e.g. show Ward only when Is Admitted = true)
