# Week 1, Day 5 · Import sets, transform maps, update sets & instance hygiene

**Date:** 2026-11-21
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data). The practice CSVs were deliberately messy.

## Import 1: Medicines (5 rows, clean file)

| Run | Result | Lesson |
| --- | --- | --- |
| 1: coalesce forgotten | Paracetamol **Inserted**, creating a duplicate | Set coalesce before the first load |
| 2: coalesce set, duplicate still present | Paracetamol **Skipped**: "Multiple coalesce matching target records" | Duplicates block future updates |
| 3: duplicate removed | 4 **Ignored** ("No field values changed"), 1 **Updated** | A re-runnable (idempotent) load |

- **Deleted the newer duplicate,** not the original. Prescriptions and allergies reference the original's sys_id.
- **Accepted the "coalesce field not indexed" prompt.** Without an index, every row's lookup scans the whole table.

## Import 2: Patients (9 rows with planted problems)

The transform map coalesced on email. Ward and choice fields were set to **reject**, and dates used **yyyy-MM-dd**. A source script translates blood group and gender labels to choice values.

| # | Row | Planted problem | Result |
| --- | --- | --- | --- |
| 1–3 | Vikram, Sneha, Arif | none | Inserted |
| 4 | Priya | Already exists | **Inserted (duplicate!)** at first; Updated after the fix |
| 5 | Neha | Ward "ICU Block B" (typo) | Skipped. No junk ward was created, because the action was reject |
| 6 | Rohan | Admitted with no ward | **Error**: data policy DP1 enforced during the import |
| 7 | Kiran | Blood group "B positive" | Inserted, blood group empty |
| 8 | Meera | DOB 07/03/1988 | **Inserted with an error**: no DOB (a partial success) |
| 9 | Sneha again | Duplicate inside the file | Updated, so the last row wins |

**Problems found and fixed:**
- **Priya duplicated:** her existing email had a **trailing space**, so coalesce didn't match. I fixed the record, deleted the duplicate and re-ran.
- **Every gender came through blank:** the gender source script read `u_blood_group`, a copy-paste bug. I fixed it and re-ran; coalesce made the re-run safe.
- **Mobile number not mapped:** I added the field map.
- **Corrected rows** (`hms_patients_fixes.csv`) loaded 4 out of 4 cleanly. The result is 12 patients and still exactly 3 wards.
- **Meera's date was ambiguous** (dd/mm or mm/dd?). I assumed 7 March. In a real project I'd ask the source owner.

**Noticed:** four patients were admitted by import, but each ward's Available Beds didn't change. That needs a business rule (Week 2), and imports will run it.

## Update set & instance hygiene

| Finding | Fix |
| --- | --- |
| The **app record** and admin/user roles were in Default [MediCare HMS], created before v1 existed | Moved to HMS Data Model v1 |
| The currency properties were in Default [Global] | Packaged in **HMS Global Config v1** (Global scope) |
| Platform "ECC Queue Authorization Policy" updates in Default [Global] | Recognised as platform side effects; left alone |
| Today's import work landed in v1, not a separate set | Accepted and noted; in a real project I'd split by feature |
| An empty old "Ward" menu (left over from the Day 2 menu merge) | Deleted |
| 7 unused generated `…_user` roles | Checked ACLs, users, menus and modules, then deleted |
| Deleting those roles **emptied each table's mandatory *User role* field** | Set it to the HMS admin role on all 7 tables |
| That change **auto-created 14 ACLs** (read/create/write where admin had no ACL yet) | Confirmed they were admin-only and redundant, deleted them, re-tested as Deepa |
| **Point Scan**: "A table's elements are on compatible columns" on Doctor.Assigned Wards | The Day 2 Reference → List type change left a mismatched database column. Recreated the field; rescan found 0 issues |
| Ward values were swapped when re-entered | Caught by checking against notes before the delete |

**Update sets and deployment order:**
1. **HMS Global Config v1** (Global, Complete): the currency properties
2. **HMS Data Model v1** (MediCare HMS, Complete): the app, Days 2–5
3. **HMS Data Model v2** (MediCare HMS, in progress): the clean-up (menu and role deletes, the field rebuild, the ACL deletes)

**Practised:**
- **Back-out:** on a one-record throwaway set. The module reverted to its previous version; it wasn't deleted.
- **Export to XML** of the completed update sets.
- **"Import Update Set from XML" happens on the target instance**, followed by Preview and then Commit. It wasn't practised here, because a real move needs a second instance.

**Data pack (data isn't in update sets):**
- Exported users (selected by User ID), groups, group roles and memberships as XML.
- **Load order:** update sets → users → groups → group roles → memberships.
- On a real project, users come from AD/SSO with different sys_ids, so memberships should load by import set, **coalescing on User ID**.
- **User exports contain password hashes.** They're kept out of this public repo.

## What I learned

- **The import pipeline:** data source → import set (staging) → transform map (field maps, coalesce, scripts) → target table.
- **Coalesce:** match on a business key. Normalise it first (trim, letter case) and index it. Without coalesce, every load duplicates records.
- **Choice action on reference fields:** use reject or ignore, never create. Otherwise typos become new reference records.
- **Read every row's state *and* its error text.** "Inserted" with an error is a silent partial failure.
- **Data policies apply to imports** when *Apply to import sets* is ticked. The row fails with an Error.
- **Retiring something:** check everything that refers to it (ACLs, users, menus, modules *and table records*). After the change, compare counts and re-test.
- **Changing a field's type later can leave a hidden database mismatch.** Get the type right the first time, and use Instance Scan.
- **When a scan check has no documentation,** read its script to see what it tests.
- **Re-entering data by hand is error-prone.** Export first, and reload by import.

## FDE reflection

Scenario: the IT lead says *"We'll send you a nightly CSV of new and updated patients from our admissions system."*

1. **Safe to re-run:** make the load idempotent.
   - Coalesce on a stable key. Ask for the hospital's **MRN** rather than email.
   - Normalise the key, and index it.
2. **Bad rows:**
   - Reject invalid choices and references.
   - Report Error, Skipped *and* "inserted with error" rows to the admissions data owner every day.
   - Fix rows at the source, not by hand in ServiceNow.
3. **File format to agree:**
   - ISO dates
   - codes rather than labels (with their full code list)
   - ward codes rather than names
   - full file or changes only
   - how discharges and merged patients appear
   - header row, delimiter, UTF-8, and file name and delivery time
4. **Next question:** *"When a row has an invalid date of birth, should we reject the whole patient, or create them without a DOB and flag it?"* It's a patient-safety decision for the business.

## Self-check

About 4½ out of 7.
- **Weak:** what a data policy does to an import row. The row fails with an Error, as Rohan's did. *Reverse if false* is unrelated.
- **Weak:** the full import pipeline, including the data source and import set (staging) steps.

## Requirements noted for later

- [ ] Business rule: update Ward Available Beds on admission and discharge (Week 2)
- [ ] Source script: trim and lowercase email before coalescing
- [ ] Load Known Allergies (a List field) by looking up medicine names, which needs a script
- [ ] Practise a real cross-instance update set move when a second instance is available

## Open items

- [ ] Week 1 wrap-up: Day 6 mini-project, or start Week 2 (scripting)
