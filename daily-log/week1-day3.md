# Week 1, Day 3 · Forms, lists, UI policies, data policies, views

**Date:** 2026-11-19
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app from Day 2 (fictional hospital, simulated data)

## What I built

### Forms
- **Appointment sections:**
  - first section (unnamed): Number, Patient, Doctor, Date/Time, Type, Status, Consultation Fee
  - *Clinical*: Ward, Assisting Nurses, durations, Nurse Notes and Activity
  - *Discharge*: Discharge Summary, Signed Consent Form
- **Patient sections:**
  - *Patient Status*: vitals, Clinical Notes and Activity
  - *Admission Details*: Is Admitted, Ward, Admission Date, Emergency Arrival Time
  - *Reports*: Lab Reports, Health Records Portal Link
  - *Account*: Outstanding Balance, Patient Portal Password
- **Activity (filtered) formatter** on Appointment and Patient. Journal entries and field-change history now show on the form.
- **Number set to read-only** on Appointment and Prescription.
- **Related lists:**
  - Patient: Appointments, Prescriptions
  - Doctor: Appointments, Prescriptions
  - Ward: Patients, Medical Equipment, Appointments
- **Nurse view** of Appointment: Number, Patient, Ward, Date/Time, Status, Assisting Nurses, Nurse Notes, Activity

### UI policies

| # | Table | Condition | Actions |
| --- | --- | --- | --- |
| UP1 | Patient | Is Admitted is true | Show Ward, Admission Date, Emergency Arrival Time; Ward and Admission Date mandatory |
| UP2 | Appointment | Type is Tele-consult | Hide Ward, Assisting Nurses, Surgery Duration, Signed Consent Form |
| UP3a | Appointment | Status is Completed | Consultation Duration mandatory; show Discharge Summary |
| UP3b | Appointment | Status is not Completed | Hide Discharge Summary |
| UP4 | Medical Equipment | Condition is Decommissioned | Assigned Ward, Requires Calibration, Last Serviced On, Maintenance Window read-only |

### Data policies

| # | Table | Condition | Rules |
| --- | --- | --- | --- |
| DP1 | Patient | Is Admitted is true | Ward and Admission Date mandatory (import sets, SOAP, client) |
| DP2 | Prescription | none (always) | Medicine, Dosage, Tablet Count, Expiry Date mandatory |

### Dictionary
- **Mandatory:** Patient and Prescribed By on Prescription

### Lists
- **Appointment list layout:** Number, Patient, Doctor, Date/Time, Type, Status
- **Medicine:** List Control set
- **Favourite:** "Cardiology appointments" (`Doctor.Specialization = Cardiology`), a dot-walked condition
- **Grouping:** Appointment list grouped by Status

## What I learned

### Where a rule should live

| Field(s) | Mechanism | Why |
| --- | --- | --- |
| Patient, Prescribed By | Dictionary Mandatory | Always required, unconditionally, everywhere |
| Medicine, Dosage, Tablet Count, Expiry Date | Data policy | Server-side, and switchable without a schema change |
| Ward, Admission Date when admitted | UI policy and data policy | Conditional: the UI policy shows and hides, the data policy enforces |

### The bypass test
- With only UP1, a **list edit** set Ananya to Is Admitted = true with no ward, and it **saved**. UI policies run only on forms in the browser.
- With DP1 in place, the same list edit was **rejected**: *"Data Policy Exception: The following fields are mandatory: Ward, Admission Date."*
- A data policy checks records only when they're saved. It doesn't fix bad records already in the table, so cleaning up existing data is a separate job.

### UI policy settings
- **Reverse if false** undoes the actions when the condition becomes false, live on the form.
- **On load** evaluates the policy when the form opens.
- **Global** applies the policy in every view.
- **Order:** a higher number runs later and wins.
- One policy with Reverse if false can replace two opposite policies. Use two only when the true and false states need different actions.
- **A data policy can't hide fields.** It does only mandatory and read-only. "Convert this to Data Policy" appears only on UI policies that have mandatory or read-only actions.
- The red dot next to a field in the UI policy actions list just marks a field a policy acts on.

### Forms and views
- **The first form section has no tab.** Its name becomes the form header ("Booking", "Patient Info", "Nurse"), so keep it named after the table.
- **Views** are named alternative layouts, not per-user. Create one from Form Layout → View name → New. **View Rules** assign views by role.
- **Related lists** need a **Reference** field back to the parent. A List field (Doctor → Assigned Wards) can't drive one.
- The **Activity formatter** shows journal entries plus the field-change audit history. That audit trail matters for GxP and healthcare customers.
- A tab containing mandatory fields gets an `*` on its label.

### Lists
- **Dot-walked conditions:** use the › arrow on a reference field, or *Show Related Fields*.
- **A favourite saves the filter, not the grouping.**

## What broke and how I fixed it

- **Wrong form headers ("Booking", "Patient Info", "Nurse").** Each was the first section's name. I renamed them back to the table name.
- **No history under the journal fields.** The Activity formatter was missing. I added *Activity (filtered)* to the layout.
- **Number was editable on Appointment.** I set Read only in the dictionary.
- **Activity timestamps were wrong** (US Pacific default). I set my user's time zone to Asia/Kolkata.
- **The list View menu has no "New".** Views are created from Form Layout → View name → New.
- **Patient and Prescribed By weren't mandatory.** I noticed this on the DP2 test form and fixed it in the dictionary (an unconditional rule).

## FDE reflection

Scenario: the Head of Nursing says *"Nurses keep admitting patients without a ward, and our bed counts are wrong."*

My answer, with feedback:
1. **Mechanisms:** a UI policy for the form experience, plus a data policy so list edits, imports and APIs can't bypass it.
   - *Feedback:* the bed count lives on **Ward**. Keeping it accurate needs a **business rule** that updates Available Beds on admission and discharge (Week 2).
2. **Sources of bad data:** forms, lists, import sets, APIs.
   - *Feedback:* name the customer's own systems, e.g. the admissions (ADT/HL7) integration, scripts and bulk updates.
3. **Demo:** make it concrete, as a before and after:
   - admit a patient on the form, where the fields appear and are mandatory
   - try a list-edit bypass, which is rejected
   - show the clean patient list grouped by ward
4. **Next question:** make it specific, not "any cases we missed?". For example:
   - *"When a patient is discharged or moved, who updates the ward today? Should Available Beds change automatically?"*
   - *"What should happen when a ward is full?"*

## Self-check

About 4.5 out of 6.
- **Weak spot:** dictionary Mandatory vs a data policy. Both apply across every input source. A data policy can be **conditional** and **switched off without a schema change**.
- **Also corrected:** a view is a named layout anyone can switch to, not something specific to a user.

## Requirements noted for later

- [ ] Business rule: update Ward Available Beds on admission and discharge (Week 2)
- [ ] Default Prescription Date to today (Week 2)
- [ ] Restrict who can change Medical Equipment Condition (ACL, Day 4)
- [ ] View Rule: nurses automatically get the Nurse view (Day 4, roles)
- [ ] Remove the duplicated mandatory actions from UP1, now covered by DP1, so each rule has one source

## Open items

- [ ] Day 4: users, groups, roles, ACLs
