# Week 2, Day 1 · Server-side scripting & business rules

**Date:** 2026-11-23
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)
**Update set:** HMS Scripting v1. It holds 5 business rules, 3 cross-scope privileges and 1 dictionary change.

## Background scripts (read-only practice)

| Exercise | Result | Lesson |
| --- | --- | --- |
| List admitted patients | 6 rows, with wards shown by dot-walking `gr.ward.getDisplayValue()` | Scoped log lines are prefixed `x_1941695_medica_0:` |
| Priya's allergies | `getValue()` gives 2 comma-separated **sys_ids**; `getDisplayValue()` gives "Penicillin, Aspirin" | Code compares sys_ids; people see display values |
| Encoded query from a list | First returned **nothing**: I'd forgotten `gr.query()`. It fails silently | new GlideRecord → addQuery → **query()** → while (next()) |
| Order | Rows came back unsorted until I added `orderBy()` | Never rely on default order |

## Business rules built

| Rule | When | Condition | Purpose |
| --- | --- | --- | --- |
| Calculate age from date of birth | before, insert/update | DOB changes | Age always correct; the field is now read-only |
| Block prescriptions for allergens | before, insert/update | Medicine changes OR Patient changes | Abort when the medicine's sys_id is in the patient's Known Allergies |
| Keep ward bed counts right | **after**, insert/update | Is Admitted changes OR Ward changes | Free the old bed (from `previous`), take the new bed (from `current`) |
| Clear ward on discharge | before, update | Is Admitted changes to false | No stale ward or admission date on discharged patients (my own addition) |
| Block admission to a full ward | before, insert/update | Is Admitted is true AND (Is Admitted or Ward changes) | Abort when Available Beds ≤ 0 |

### Bed count rule (readable version)
```javascript
(function executeRule(current, previous) {
    var isAdmittedNow = (current.is_admitted == true);
    var newWardId = current.getValue('ward');

    var wasAdmittedBefore = false;
    var oldWardId = '';
    if (previous != null) {                       // previous is null on insert
        wasAdmittedBefore = (previous.is_admitted == true);
        oldWardId = previous.getValue('ward');
    }

    if (wasAdmittedBefore) changeAvailableBeds(oldWardId, 1);   // give back the old bed
    if (isAdmittedNow)     changeAvailableBeds(newWardId, -1);  // take the new bed

    function changeAvailableBeds(wardId, change) {
        if (!wardId) return;
        var ward = new GlideRecord('x_1941695_medica_0_ward');
        if (!ward.get(wardId)) return;
        var bedsBefore = parseInt(ward.getValue('available_beds'), 10);
        if (isNaN(bedsBefore)) bedsBefore = 0;
        var bedsAfter = bedsBefore + change;
        ward.setValue('available_beds', bedsAfter);
        ward.update();
        gs.info('HMS beds: ' + ward.getValue('ward_name') + ' changed from ' + bedsBefore + ' to ' + bedsAfter);
    }
})(current, previous);
```

### Block admission to a full ward
```javascript
(function executeRule(current, previous) {
    var ward = current.ward.getRefRecord();
    if (!ward.isValidRecord()) return;
    var availableBeds = parseInt(ward.getValue('available_beds'), 10);
    if (availableBeds <= 0) {
        gs.addErrorMessage('Cannot admit to ' + ward.getValue('ward_name') +
            ': no beds available. Choose another ward.');
        current.setAbortAction(true);
    }
})(current, previous);
```

## Test results

**Age:**
- Meera, DOB 1986-03-07 → 40 ✓
- Test patient, DOB 1996-11-22 → 30 ✓
- Edge cases to cover: birthday today and birthday tomorrow (off-by-one)

**Allergy check:**

| Test | Result |
| --- | --- |
| Priya + Aspirin (new) | Blocked ✓ |
| Priya + Paracetamol | Saved ✓ |
| Change to Penicillin (update) | Blocked ✓ |
| Priya + Amoxicillin | Saved: the expected gap (exact-medicine match, no drug class) |

**Bed counts** (starting at ICU 5, General A 18, Cardiology 12):

| Step | ICU | Gen A | Cardio |
| --- | --- | --- | --- |
| Admit Sneha → General A | 5 | 17 | 12 |
| Move Sneha → Cardiology | 5 | 18 | 11 |
| Discharge Sneha (ward cleared) | 5 | 18 | 12 |
| **As Deepa** (no write on Ward), admit Kiran → ICU | 4 | 18 | 12 |
| **List edit** Kiran → not admitted | 5 | 18 | 12 |
| ICU set to 0: admit Kiran → ICU | **Blocked** | | |
| Admit Kiran → General A, then discharge | 0→5 (restored) | 17→18 | 12 |

The data ended exactly where it started.

## What I learned

- **Before rules are for validation; after rules are for side effects.**
  - Before rules change or check `current`, or abort the save.
  - After rules update other records once the save has succeeded.
- **`previous` holds the values before this save.** It's null on insert. The discharge rule clears `current.ward`, and the after rule still frees the right bed by reading `previous.ward`.
- **Don't call `current.update()` in a before rule.** Changes to `current` are saved automatically.
- **Aborting a save:** `setAbortAction(true)` plus `gs.addErrorMessage()`. Without the message, users see only "Invalid insert".
- **GlideRecord ignores ACLs.** That's why a nurse's admission updates a ward she can't edit. Use GlideRecordSecure to respect ACLs.
- **Business rules run for every source:** form, list edit, import and API. UI policies don't.
- **Cross-scope privileges** are created the first time a rule runs, and captured in the update set. An untested rule may lack its privilege in production.
- **Review update sets for strays.** A "Password Reset Process" record had slipped in; I moved it to Default [Global].
- **Rules check records only when they're saved.** RX0001004 (Priya + Amoxicillin, created before the rule) still exists. A one-off check needs a script.

## My mistake: the full-ward check

I first added the full-ward check inside the **after** bed-count rule. It had four bugs:
- `setAbortAction` does nothing in an after rule
- there was no `return` after aborting, so beds could go to −1
- the check also fired on discharges
- `gs.info` meant the user saw nothing

**Fix:** a separate **before** rule, plus removing the check from the after rule.

## FDE reflection

Scenario: the Head of Pharmacy says *"I want the system to stop dangerous prescriptions."*

**Today's rule:**
- It catches exact-medicine allergies only.
- It can't catch drug classes (penicillin/amoxicillin), drug-drug interactions or dose limits.

**Block or warn?**
- Block for severe risks; warn and allow an override for lower ones.
- Too many alerts cause **alert fatigue**.
- An override should need a **mandatory reason**, and maybe a pharmacist review, so the audit shows *why*, not just *who*.

**Data needed:**
- each medicine's drug class
- allergy severity and reaction type
- interaction rules
- dose ranges
- patient weight

**Next question:** *"Which drug database does your pharmacy system already use? Could we call it instead of keeping drug lists in ServiceNow?"* Don't rebuild clinical knowledge; integrate with the source of truth.

## Self-check

About 5½ out of 7.
- **Weak:** why GlideRecord can update records the user can't (it runs as the system and ignores ACLs).
- **Weak:** the purpose of the conditions (run only when a bed count could change).

## Requirements noted for later

- [ ] Recalculate Age for all existing patients and recount ward beds from actual admissions (background script, Day 3)
- [ ] One-off check of existing prescriptions for allergy conflicts (RX0001004)
- [ ] Drug class on Medicine, with the allergy check comparing classes
- [ ] Admission history: a separate admission/encounter table if the hospital needs stay history
- [ ] Default Prescription Date to today
