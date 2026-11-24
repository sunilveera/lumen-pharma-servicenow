# Week 1, Day 4 · Users, groups, roles, ACLs

**Date:** 2026-11-20
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)

## What I built

### Roles (scoped, `x_1941695_medica_0.*`)

| Role | Purpose |
| --- | --- |
| `nurse` | Read clinical data; update appointments and nurse notes |
| `doctor` | Clinical data; prescriptions and discharge summaries |
| `admissions` | Register patients, book appointments, billing |
| `biomed` | Maintain medical equipment |
| `admin` | The app's existing default admin role, reused. Contains the four roles above |

### Groups (each group carries its role)

| Group | Role | Members |
| --- | --- | --- |
| Nursing | nurse | Deepa Nair, Ravi Kumar |
| Doctors | doctor | Arjun Mehta, Kavya Rao, Sameer Khan |
| Admissions Desk | admissions | Meena Joshi |
| Biomedical Engineering | biomed | Suresh Pillai |

### Doctor → User link
- Added a **User** field (Reference → `sys_user`) to Doctor, which fixes a Day 2 gap. ACL-D needs it.

### Menu
- The MediCare HMS menu requires the four business roles.
- Each module has its own roles.
- Modules are ordered 100 (Patients) to 700 (Medicines).

### Access matrix (ACLs)

| Table / field | Read | Create | Write | Delete |
| --- | --- | --- | --- | --- |
| Patient | nurse, doctor, admissions | admissions | nurse, doctor, admissions | admin |
| Patient.outstanding_balance | admissions | (falls back to write) | admissions | — |
| Patient.patient_portal_password | admissions | (falls back to write) | admissions | — |
| Appointment | nurse, doctor, admissions | admissions | nurse, doctor, admissions | admin |
| Appointment.discharge_summary | (table) | — | doctor | — |
| Prescription | nurse, doctor | doctor | **ACL-D:** doctor AND `Prescribed By.User is (dynamic) Me`; plus admin | admin |
| Medical Equipment | biomed, nurse, doctor | biomed | biomed | admin |
| Medical Equipment.condition | (table) | — | biomed | — |
| Doctor, Ward, Medicine | all four | admin | admin | admin |

The 28 generated table ACLs were changed from the generated `…_user` roles to business roles. The generated roles are no longer used.

## Test results (impersonation)

| Test | Deepa (nurse) | Arjun (doctor) | Meena (admissions) | Suresh (biomed) |
| --- | --- | --- | --- | --- |
| Sees the MediCare HMS menu | ✓ 7 modules | ✓ 7 modules | ✓ 5 modules | ✓ 4 modules |
| Opens a Patient record | ✓ | ✓ | ✓ | ✗ "Security constraints prevent access" |
| Sees Outstanding Balance | ✗ | ✗ | ✓ | ✗ |
| Can create a Patient | ✗ | ✗ | ✓ | ✗ |
| Can edit Discharge Summary | ✗ | ✓ | ✗ | ✗ |
| Can create a Prescription | ✗ | ✓ | ✗ | ✗ |
| Can edit Dr. Sameer Khan's prescription | ✗ | ✗ | ✗ | ✗ |
| Can change Equipment Condition | ✗ | ✗ | ✗ | ✓ |
| Can delete a Patient | ✗ | ✗ | ✗ | ✗ |

**Result: 36 out of 36 checks match the design.**

## What I learned

- **Roles go on groups, not users.** Joiners, movers and leavers are handled by group membership, and "who has access?" is answered by one member list.
- **An ACL has three checks: roles, condition and script.**
  - All three must pass, and an empty check counts as a pass.
  - **An ACL with no role, no condition and no script grants everyone.**
- **To see a field, a user must pass both the table ACL and the field ACL.**
- **A field read ACL doesn't stop writes.** Add a field write ACL too.
- **A field with no create ACL** falls back to its write ACL on new records.
- **Allow If ACLs combine with OR**, so passing any one matching ACL is enough.
  - That's how ACL-D (own prescriptions) and the admin ACL (all prescriptions) work together.
- **Parent table ACLs** (e.g. CMDB) don't apply when the child table has its own ACLs.
- **Admin overrides** let the platform admin through most ACLs, so test only by impersonating.
- **Editing ACLs needs elevation to `security_admin`.** Impersonating ends the elevation.
- **Hiding a menu isn't security.** Suresh had no Patients link, and the URL was still blocked by the read ACL.
- **Menu and module roles:**
  - A user needs the menu's roles *and* the module's roles to see a link.
  - *Override application menu roles* changes that.
- **Read each "blocked" message before deciding why:**
  - *"Security prevents writing"* is an ACL.
  - *"This element type is not editable from the list"* is the field type (e.g. Time).
  - A data policy exception is a data policy.
- **Personal vs shared list layouts:**
  - The gear changes only your own layout.
  - Configure → List Layout changes it for everyone.
  - Impersonation shows the difference, and *Reset to column defaults* clears your own.
- **Roles and ACLs are in the update set; groups, users and memberships aren't.** Those are data and need a separate migration step.
- **Time zones:**
  - Time-only fields (OPD hours) are stored in UTC too.
  - Set the time zone on the **user record**, not just the session.

## What broke and how I fixed it

- **Deepa couldn't see the menu.** Its modules required the generated `…_user` roles. I set the menu and module roles to the business roles.
- **Suresh was in "Conditional Script Writer".** Every user has it, so it's a platform-managed group. I left it alone. Lesson: tell platform config apart from project config.
- **A nurse could edit the patient portal password.** I found this by testing, not from the spec. I added field read and write ACLs (admissions).
- **Meena couldn't edit the portal password.** The ACL had been created without its role. After fixing it, I re-tested **both** sides: Meena can edit and Deepa sees nothing.
- **Users saw different list columns.** The admin layouts were personal (gear). I configured shared layouts and reset my own.
- **Admin timestamps were 12½–13½ hours off.** The time zone wasn't saved on the admin user record.

## FDE reflection

Scenario: the compliance officer says *"Clinical staff must never see patient billing, and we must be able to prove who changed a prescription."*

My answer, with feedback:
- **Billing:**
  - *My answer:* UI and data policies.
  - *Correction:* **field ACLs** (read and write on `outstanding_balance`). Policies control *when* a field shows, not *who* may see it.
- **Proof of change:**
  - *My answer:* the Activity formatter.
  - *Better:* **auditing** (`sys_audit`) is the evidence; the Activity stream only displays it. Together with ACL-D, that covers prevention and evidence.
- **Evidence pack for an auditor:**
  - record history (≡ → History → List)
  - a `sys_audit` report on prescriptions for a period
  - the ACL design and group memberships
  - the impersonation test matrix
- **Why groups:** joiner/mover/leaver handling, and an auditable list of who has access.
- **Better next questions:**
  - *"How long must change history be kept, and who reviews it?"*
  - *"Does billing include Is Insured? Nurses can see it today."*

## Self-check

About 5 out of 7.
- **Q3, ACL anatomy:** the three checks are roles, condition and script. Operation and table are what an ACL applies to, not checks.
- **Q2, seeing a field:** you need the table read ACL plus the field read ACL. The write ACL is a separate protection.

## Requirements noted for later

- [ ] Set Prescribed By to the logged-in doctor on create (Week 2, default value or business rule)
- [ ] Admissions view: hide the empty Account tab for clinical staff and the Prescriptions related list for admissions
- [ ] View rule: nurses automatically get the Nurse view
- [ ] Retire the 7 unused `…_user` roles
- [ ] Migration plan for groups and memberships (not captured in update sets)
- [ ] Decide whether OPD hours are hospital-local or shown in each user's time zone

## Open items

- [ ] Day 5: import sets, transform maps, update sets and instance hygiene
