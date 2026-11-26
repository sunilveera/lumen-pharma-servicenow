# Deploying MediCare HMS to another instance

Update sets don't carry everything. This is the full order for moving the MediCare HMS app from the PDI to a new instance (for example Test or Prod).

## Deployment order

| # | Item | How | Notes |
| --- | --- | --- | --- |
| 1 | HMS Global Config v1 | Retrieve, preview, commit the update set | Global-scope items the app depends on |
| 2 | HMS Data Model v1 | Update set | Tables, fields, choices |
| 3 | HMS Data Model v2 | Update set | Later field changes |
| 4 | HMS Scripting v1 | Update set | Business rules |
| 5 | HMS Script Includes v1 | Update set | `PatientUtils`, `PrescriptionUtils`, `WardUtils` |
| 6 | HMS Data Fixes v1 | Update set | Fix scripts, `reconcileBeds` / `recalculateAllAges`, the GlideAggregate cross-scope privilege |
| 7 | HMS Bed Integrity v1 | Update set | Field write ACL on Available Beds, *Block delete of admitted patient*, the `GlideRecordSecure.getValue` cross-scope privilege |
| 8 | HMS Client Scripts v1 | Update set | 6 client scripts, 4 business rules, `WardAjax` / `PrescriptionAjax` with their client-callable ACLs **and Access Roles** (nurse, doctor) |
| 9 | [Scheduled job: HMS nightly reconciliation](hms-nightly-reconciliation.xml) | **Import XML** (*System Definition → Scheduled Jobs → list context menu → Import XML*) | Not captured by update sets (see below). Import **after** the update sets: it belongs to the MediCare HMS scope and calls `WardUtils` and `PatientUtils` |
| 10 | Groups and users data pack | Import XML | **Not in this repo**: user exports contain password hashes |
| 11 | Verify | Run the fix script **HMS - Data health check (read-only)** | All sections should be clean |

Preview every update set before committing it, and commit them in this order.

**Check ACL roles travel with their ACLs.** The role on an ACL is a separate record (Access Roles). If a role is added in a later update set than the ACL, the ACL arrives without it and only admin passes.

## Why the scheduled job isn't in an update set

Scheduled jobs (`sysauto_script`) aren't tracked by update sets: the table has no `update_synch` attribute, so creating or editing the job writes no customer update. ([ServiceNow Community: Are scheduled jobs captured by update sets?](https://www.servicenow.com/community/developer-forum/are-scheduled-jobs-captured-by-update-sets/m-p/1582722))

Options:
- **XML export and import** (used here). Simple, but it's a manual step that has to be on the deployment checklist.
- **A scheduled Flow** in Flow Designer instead of a script job. Flows are captured in update sets. This is the better long-term option.
- Don't add `update_synch` to the platform table: it's an unsupported change to a platform table.

## Job details (from the XML)

| Field | Value |
| --- | --- |
| Run | Daily, 02:00 Asia/Kolkata (stored as 20:30 UTC) |
| Run as | System Administrator (standard sys_id, so it resolves on every instance) |
| Scope | MediCare HMS |

On a customer project, run it as a dedicated service account with only the roles it needs, so the audit trail shows the integration user, not admin.
