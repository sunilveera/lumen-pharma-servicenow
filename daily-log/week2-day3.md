# Week 2, Day 3 · GlideAggregate, fix scripts & scheduled jobs

**Date:** 2026-11-25
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)
**Update set:** HMS Data Fixes v1. HMS Script Includes v1 is marked Complete.

**Goal:** fix what yesterday's health check found (ward bed counts far off, one stale age), the way it's done on real projects: count in the database, fix with a tracked and re-runnable script, and stop it drifting again.

## GlideAggregate practice (background scripts, read-only)

| Query | Result |
| --- | --- |
| Admitted patients per ward (COUNT, `groupBy('ward')`) | ICU 2, General Ward A 2, Cardiology Ward 2 |
| Appointments by status | Completed 2, Confirmed 2, No-show 1, Scheduled 2 |
| Tablets prescribed per medicine (SUM, `orderByAggregate`) | Aspirin 30, Paracetamol 25, Amoxicillin 21 |
| Total patients (COUNT, no grouping) | 12 |

- The first scoped GlideAggregate call auto-granted the **`ScopedGlideAggregate` cross-scope privilege**, and it was captured in the update set.
- `orderByAggregate('SUM', …)` sorted **largest first** on this PDI. I corrected the comment in my notes: test the sort order before relying on it.

## Reconciliation methods added

| Script include | New method | What it does |
| --- | --- | --- |
| `WardUtils` | `getAdmittedCounts()` | One GlideAggregate query → `{ wardSysId: admittedCount }` |
| `WardUtils` | `reconcileBeds(dryRun)` | Available = Total − admitted. Logs every change, warns on over capacity, returns the number changed |
| `PatientUtils` | `recalculateAllAges(dryRun)` | Reuses `calculateAge()`, so there's still one age formula in the app |

## Fix script: HMS - Reconcile ward beds and patient ages

**Dry run (`DRY_RUN = true`).** I ran it twice and got identical output both times. The Wards list was unchanged.

| Log line |
| --- |
| ICU - Block B available beds 5 → 18 (DRY RUN, not saved) |
| General Ward A available beds 18 → 38 (DRY RUN, not saved) |
| Cardiology Ward available beds 12 → 28 (DRY RUN, not saved) |
| 3 ward(s) would change |
| PAT0001024 Ananya Iyer age 24 → 25 (DRY RUN, not saved) |
| 1 patient(s) would change |

**Real run (`DRY_RUN = false`):** the wards are now 18 / 38 / 28, and Ananya is 25.

**Idempotency proof:** the third run logged `0 ward(s) changed` and `0 patient(s) changed`.

**Health check re-run:** all three sections were clean.

**Regression check:** I admitted Sneha to General Ward A (38 → 37) and discharged her (→ 38). The business rules still work from the corrected baseline.

## Scheduled job: HMS nightly reconciliation

Daily at 02:00 (Asia/Kolkata). It calls `reconcileBeds(false)` and `recalculateAllAges(false)`, and **warns** if it had to fix any beds.

| Test | Result |
| --- | --- |
| Execute Now after the fix | `wards fixed 0, ages fixed 0` |
| Drift test: set ICU to 10 by hand, then Execute Now | Warning logged (*corrected 1 ward bed count(s). Investigate what bypassed the rules.*), and ICU went back to 18 |

**Ages vs beds:**
- Ages *should* change overnight, because birthdays happen.
- Beds should *never* need fixing if the rules work, so a bed correction is a symptom of something else and gets a warning.

## Finding: scheduled jobs aren't captured by update sets

The job wasn't in HMS Data Fixes v1, and it didn't appear in Customer Updates either.
- **Cause:** `sysauto_script` has no `update_synch` attribute, so the table isn't tracked ([ServiceNow Community thread](https://www.servicenow.com/community/developer-forum/are-scheduled-jobs-captured-by-update-sets/m-p/1582722)).
- **What I did:** exported the job as XML into [`deploy/`](../deploy/README.md) and wrote down the full deployment order.
- **Long-term option:** a scheduled Flow, which update sets do capture.
- **What not to do:** add `update_synch` to the platform table.

## Gaps found: other ways a bed gets taken or freed

**System gaps**
- An admitted patient is deleted: no rule runs, so the bed stays taken.
- A ward's Total beds changes: Available doesn't follow.
- An update skips business rules: imports with rules off, `setWorkflow(false)`, careless fix scripts.
- Is Admitted is true but Ward is empty.
- A ward is deleted while patients still reference it.
- Two admissions to the last bed at the same moment.
- Available Beds is **still editable by hand**: the drift test proved it. The field needs a write ACL or to be made read-only.

**Business questions for the ward staff:** beds out of service (cleaning, isolation), reserved beds, and discharge reasons (death, transfer out, leaving against advice).

**Design lesson:** every gap exists because the app **stores** a number it could **calculate**. Keeping the counter with a nightly reconciliation as the safety net is a deliberate trade-off.

## FDE reflection

**Question from the Head of Operations:** *"Last week the dashboard showed 5 free ICU beds when there were really 18. How did that happen, and how do we trust the numbers now?"*

**My answer:** the ward counts were entered by hand at setup and never matched the patients actually admitted. The rules then added and removed beds correctly, but always from that wrong starting number. Every ward is now corrected (dry run, real run, re-run, health check). Every night the system counts the admitted patients and resets the numbers, and it warns if it ever has to correct a bed count, because that means something went around the normal process.

**Review notes (7/10):**
- I claimed bed counts can't be edited by hand any more, but they still can. Don't promise a control until it exists.
- A warning in the system log isn't visible to operations. It should become an event and a notification to the ward admin team.

## Self-check: 5/7

- Strong: GlideAggregate vs GlideRecord, `groupBy`, dry run, the warning instead of a silent fix.
- Half marks:
  - Fix scripts: the key point is that they're captured in the update set and travel to Test and Prod.
  - Idempotency: I defined it but didn't give the proof (the third run changed 0).
- Missed: why updating Age didn't fire the bed rule. The rule's condition is *Is Admitted changes OR Ward changes*, and an Age-only update matches neither.

## Next

- Mark HMS Data Fixes v1 Complete.
- Lock Available Beds (write ACL), and handle patient deletes and Total beds changes.
- Turn the nightly warning into an event and a notification.
