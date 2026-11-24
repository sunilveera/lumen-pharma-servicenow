# Week 2, Day 2 · Script includes & refactoring

**Date:** 2026-11-24
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)
**Update set:** HMS Script Includes v1. All earlier HMS sets are marked Complete.

## Script includes built

| Script include | Methods | Notes |
| --- | --- | --- |
| `PatientUtils` | `calculateAge(dob, onDate)` | An optional `onDate` makes edge cases testable |
| `PrescriptionUtils` | `isAllergic(patientId, medicineId)` | Exact-medicine match on the patient's Known Allergies (sys_ids) |
| `WardUtils` | `getAvailableBeds(wardId)`, `hasFreeBed(wardId)`, `adjustBeds(wardId, change)` | Read methods kept separate from the one method that **writes** |

All three: *Accessible from* = this application scope only; not client callable.

## Unit tests (background script, no records saved)

| Test | Expected | Result |
| --- | --- | --- |
| Birthday yesterday / today / tomorrow | 30 / 30 / 29 | ✓ |
| Leap-day baby (2000-02-29 on 2026-02-28) | 25 | ✓ |
| No DOB | null | ✓ |
| Priya + Aspirin / Paracetamol | true / false | ✓ |
| ICU beds, free? | 5, true | ✓ |

## Refactor: the business rules now call the script includes

| Rule | Now calls |
| --- | --- |
| Calculate age from date of birth | `PatientUtils.calculateAge` |
| Block prescriptions for allergens | `PrescriptionUtils.isAllergic` |
| Keep ward bed counts right | `WardUtils.adjustBeds` |
| Block admission to a full ward | `WardUtils.getAvailableBeds` |
| Clear ward on discharge | *Unchanged*: two lines, used once, so not worth extracting |

```javascript
// Keep ward bed counts right (after), after the refactor
(function executeRule(current, previous) {
    var wardUtils = new WardUtils();
    var isAdmittedNow = (current.is_admitted == true);
    var wasAdmittedBefore = (previous != null) && (previous.is_admitted == true);
    if (wasAdmittedBefore) wardUtils.adjustBeds(previous.getValue('ward'), 1);
    if (isAdmittedNow)     wardUtils.adjustBeds(current.getValue('ward'), -1);
})(current, previous);
```

**Regression test: every result matched Day 1.**

| Test | Result |
| --- | --- |
| Age recalculated on a DOB change | ✓ |
| Priya + Aspirin blocked (same message) | ✓ |
| Priya + Paracetamol saved (then deleted) | ✓ |
| Admit / move / discharge Sneha: 18→17, then 18/11, then 12 | ✓ |
| Full-ward block (ICU at 0) | ✓ |
| Wards restored to 5 / 18 / 12 | ✓ |

## Health check (read-only, reusing the script includes)

```
=== 1. Patients whose stored Age is wrong ===
PAT0001024 Ananya Iyer: stored 24, should be 25
=== 2. Existing prescriptions that conflict with allergies ===
(none)
=== 3. Ward beds: stored vs actual ===
ICU - Block B: stored 5, actual 18
General Ward A: stored 18, actual 38
Cardiology Ward: stored 12, actual 28
```

**What the report shows:**

- **Age:** Ananya turned 25 on 22 Nov. Her DOB didn't change, so the rule never ran. **Stored derived values go stale.** The options are a nightly recalculation job, or calculating age on display instead of storing it.
- **Allergies:** none found. RX0001004 (Priya + Amoxicillin) isn't an exact match. The report is only as good as the rule behind it.
- **Beds:** the starting values were typed in by hand on Day 2, so the +1/−1 updates have been correct but off a wrong baseline. **Event-driven counters drift.** They need a periodic reconciliation from the source of truth (Day 3).
- **Performance note:** section 3 runs one query per ward. That's fine for 3 wards; GlideAggregate is the fix (Day 3).

## What I learned

- **A script include** is reusable server code: one place to fix, unit-testable with any inputs, and it keeps business rules short and readable.
- **The class shape:**
  - `Class.create()` declares the class.
  - The `prototype` holds `initialize` (runs on `new`) and the methods.
  - `type` must equal the name.
  - Inside the class, use `this.` to reach other methods and shared values.
- **Calling one:** `new WardUtils()` from the same scope; `new x_1941695_medica_0.WardUtils()` from another, if *Accessible from* allows it.
- **Design for testability:** optional parameters (like `onDate`) let tests use fixed inputs.
- **Refactor plus regression test:** change the structure, not the behaviour, then prove it by re-running the old tests.
- **Rule order:** all my rules have Order 100. That's fine while they're independent; it matters when one rule depends on another's result.
- **Not everything needs extracting.**

## FDE reflection

Scenario: the integration team says *"We'd like to call your allergy check from our pharmacy portal and a Flow."*

**Access:**
- *Accessible from: All application scopes* lets other apps call it.
- That makes it a **public API**: it must stay stable.
- It also exposes patient data, because GlideRecord ignores ACLs. Return the minimum, and consider GlideRecordSecure.

**Don't pass in table names** *(my first idea; corrected)*:
- Callers shouldn't know internal table names (encapsulation).
- It would also let callers query arbitrary tables, a security risk.
- The real gap is the **inputs**: accept business identifiers (patient number or MRN, medicine name or code) and look up the sys_ids inside the method.

**Don't break existing callers:**
- Keep `isAllergic` unchanged.
- Add a new method (e.g. `checkAllergy`) that returns a richer result (which allergen matched; "patient not found").
- Document both, and re-run the regression tests before every change.

**How they call it:**
- A Flow calls it through a custom Flow action.
- An outside system calls it through a Scripted REST API (Week 3).

**Question to ask:** *"What identifiers does your portal have for patients and medicines, and what should happen when the patient can't be found?"*

## Self-check

About 6 out of 7.
- **Q4:** the *purpose* of `onDate` is testability: fixed dates for edge cases, without waiting for the real date.
- **Q7:** *Accessible from* controls which **applications** can call the code, not which **users**. Users are controlled by roles and ACLs.

## Requirements noted for later

- [ ] Day 3: GlideAggregate recount and fix script (set Available Beds = Total − admitted; recalculate ages)
- [ ] Nightly job to recalculate ages, or stop storing Age
- [ ] Periodic bed-count reconciliation
- [ ] `checkAllergy(patientNumber, medicineName)` returning a structured result; drug-class matching
