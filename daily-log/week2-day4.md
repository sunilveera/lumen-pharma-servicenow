# Week 2, Day 4 · Client scripts, GlideAjax & g_scratchpad

**Date:** 2026-11-26
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)
**Update sets:** HMS Bed Integrity v1 (3 updates) and HMS Client Scripts v1 (16 updates). Both are marked Complete.

**Goal:**
- Close yesterday's gaps on bed counts.
- Move feedback onto the form so nurses see problems before they save, without weakening the server rules.

## What was built

| # | Name | Type | Applies to | Purpose | Update set |
| --- | --- | --- | --- | --- | --- |
| 1 | Available Beds write lock | Field ACL, write, `answer = false`, Admin overrides off | Ward → Available Beds | No one edits bed counts by hand. Scripts still can | Bed Integrity v1 |
| 2 | Block delete of admitted patient | BR: before delete, Is Admitted true | Patient | Discharge first, so the bed isn't left counted as taken | Bed Integrity v1 |
| 3 | `GlideRecordSecure.getValue` | Cross-scope privilege (auto) | — | Granted by the platform during a patient delete | Bed Integrity v1 |
| 4 | Future DOB check | Added to the BR *Calculate age from date of birth* | Patient | Server enforcement for the form, list and imports | Client Scripts v1 |
| 5 | date of birth can't be in the future | Client script: onChange | Patient | Instant feedback | Client Scripts v1 |
| 6 | Prescription: tablet count sanity check | Client script: onSubmit | Prescription | Blocks 0, confirms over 60 | Client Scripts v1 |
| 7 | Validate tablet count | BR: before insert/update, Tablet Count changes | Prescription | Closes the list-edit bypass | Client Scripts v1 |
| 8 | Set available beds from total | BR: before insert/update, Total Beds changes | Ward | Available = Total − admitted on new wards and capacity changes | Client Scripts v1 |
| 9 | `WardAjax` + ACL (nurse) | Client-callable script include | Reads Ward (GlideRecordSecure) | Free beds for the form | Client Scripts v1 |
| 10 | show free beds when a ward is picked | Client script: onChange (Ward) | Patient | "18 bed(s) free" or "ward is full" | Client Scripts v1 |
| 11 | `PrescriptionAjax` + ACL (doctor) | Client-callable script include, reuses `PrescriptionUtils.isAllergic()` | Reads Patient (GlideRecordSecure) | Returns only 'true'/'false' | Client Scripts v1 |
| 12 | Allergy warning on medicine pick / on patient pick | Client scripts: onChange × 2 | Prescription | Warning whichever field is picked last | Client Scripts v1 |
| 13 | Ward free beds to scratchpad | BR: display | Patient | Puts the ward's free beds in `g_scratchpad` | Client Scripts v1 |
| 14 | Show ward beds on load | Client script: onLoad | Patient | Banner on admitted patients, no extra call | Client Scripts v1 |

## Block 2 · Bed integrity tests

| # | Test | Result |
| --- | --- | --- |
| 1 | ICU form, Available Beds | Read-only, even for admin |
| 2 | List edit of Available Beds | "Security prevents writing to this field" |
| 3 | Fix script and nightly job | Both still write the field: 0 changed |
| 4 | Admit and discharge Sneha | General Ward A 38 → 37 → 38 |
| 5 | Delete an admitted test patient | Blocked with my message |
| 6 | Discharge, then delete | **Failed at first:** still blocked. The rule's condition hadn't saved, so it blocked every delete. After fixing the condition, the delete worked |

**Lessons:**
- **Test both paths of a blocking rule.** Test 5 alone would have passed a rule that blocks everything.
- **A field ACL is the most specific match, so it wins over `ward.*`.** Unticking Admin overrides locks the field for admin too. GlideRecord in business rules and jobs ignores ACLs, so automation still works.
- **The cross-scope privilege came from the platform, not my code.** I searched every HMS script include, business rule and client script for `GlideRecordSecure` and found none. The grant happened during the list delete, so I kept it in this update set: on a target where cross-scope access is enforced, deleting patients could otherwise fail.

## Block 3 · Client scripts and the bypass

| # | Test | Result |
| --- | --- | --- |
| 1 | DOB next year on the form | Cleared, with a red field message |
| 2 | Valid DOB (2024-11-25) | Saved. The server rule calculated Age 2 |
| 4 | Tablet count 0 → Save | Error, not saved |
| 5 | 90 → Cancel | Not saved |
| 6 | 90 → OK | Saved |
| **7** | **List edit of tablet count to 0** | **Saved.** The onSubmit check never ran |

**Closing the bypass:** I added the before rule *Validate tablet count*.
- A list edit to 0 is now blocked with "Tablet count must be at least 1."
- A list edit to 15 saves (the allow path). This also restored Rahul's original value.

**Client = convenience, server = truth,** shown with evidence: first the gap, then the fix.

## Block 4 · GlideAjax

**New finding:** the Available Beds ACL also blocks the value on **new** wards. A new ward got an empty Available Beds, and WardAjax returned `''`, so no message showed.
- **Fix:** a before rule on Ward that sets Available = Total − admitted. It reuses `WardUtils.getAdmittedCounts()`, so it always agrees with the nightly job.
- This also closes yesterday's *"Total beds changes, Available doesn't follow"* gap.

| # | Test | Result |
| --- | --- | --- |
| 1 | Pick ICU | "18 bed(s) free" |
| 2 | New ward Test (1 bed) → admit Sneha → Ananya picks Test | Available 1 → 0, then red "This ward is full" |
| 2b | Ananya → Update anyway | Server block: "Cannot admit to Test: no beds available", plus the platform's "Invalid update" |
| 3 | Clear Ward | Message gone |
| 4a | Priya + Penicillin | Red allergy message |
| 4b | Penicillin first, then Priya | Same message, from the patient-pick script |
| 5 | Update anyway | Server: "Prescription blocked: Priya Sharma is allergic to Penicillin." |
| 6 | Omeprazole | No message |
| 7 | Network tab, `xmlhttp.do` | One request. The response was only `answer="18"` |

**Roles (impersonation):**

| Role | WardAjax | PrescriptionAjax |
| --- | --- | --- |
| Nurse | ✅ "18 bed(s) free" | Not needed: nurses have read-only prescriptions |
| Doctor | **Not granted (design decision)** | ✅ Allergy warning (Rahul + Aspirin) |

**Design decision:** only nurses get the free-bed message. Doctors can admit, but a full ward is still blocked on save by the server rule. The trade-off gives up a convenience, not a control, and reversing it means adding one role to one ACL.

**Lessons:**
- **Client-callable script includes get an ACL on this release.** The role on it is a separate **Access Roles** record and travels in the update set separately.
- **Check every Ajax response in the Network tab:** return only what the form needs.
- **Impersonation writes the impersonated user's name into the audit history:** Sneha's admission shows "Arjun Mehta". Test with dedicated test users, never real staff, never in Prod.
- **A failed save leaves the form showing the unsaved values.** Reload before believing what you see.

## Block 5 · g_scratchpad

| # | Test | Result |
| --- | --- | --- |
| 1 | Lakshmi (admitted, ICU) | "ICU - Block B: 18 bed(s) free right now." |
| 2 | Priya (not admitted) | No banner |
| 3 | Network tab on Lakshmi | **0 of 62** `xmlhttp.do` requests. The data came with the page, and the ward onChange skipped the load (`isLoading`) |

## Clean-up

- Test patients and the Test ward were deleted.
- Sneha was discharged.
- RX0001003 is back to Paracetamol 15, and RX0001004 is still Amoxicillin 21.
- Wards are back to 18 / 38 / 28.

## FDE reflection

**Question from the ward manager:** *"Can we drop the server rules and do all checks on the form? It's faster."*

**My answer:** no. Entries made from the list or by imports would bypass the form checks. g_scratchpad can load information as the form opens, for the nurses.

**Review notes (6/10):**
- The answer and the bypass paths were right.
- **Missing the proof:** today a list edit saved 0 tablets.
- **Missing the speed argument:** server rules run once, on save, in milliseconds. Nurses feel *when* they hear about a problem, and the form warnings already fix that. Keep both.
- g_scratchpad fits as an offer of *more* speed, not a replacement for the rules.

## Self-check: 4.5/7

**Pattern:** I said *what* happens but often not *why* it matters.
- Q1: say which one **enforces**.
- Q2: why `isLoading` is checked.
- Q3: onSubmit doesn't wait, so the save goes ahead before the GlideAjax reply.
- Q5: `AbstractAjaxProcessor` + Glide AJAX enabled.
- Q6: anyone logged in can call it with any parameters.

**Strong:** UI policy vs client script; g_scratchpad at load.

## Deployment

[`deploy/README.md`](../deploy/README.md) now includes HMS Bed Integrity v1 and HMS Client Scripts v1, plus a check that ACL roles travel with their ACLs.

## Next

- Debugging tools: script debugger, session debug, logs.
- Turn the nightly bed warning into an event and a notification.
