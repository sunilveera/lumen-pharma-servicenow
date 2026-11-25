# Lumen Pharma on ServiceNow

A 6-week, hands-on ServiceNow + agentic AI learning portfolio, built while preparing for a ServiceNow Forward Deployed Engineer (FDE) role.

**Lumen Pharma is a fictional global pharmaceutical company** used as a practice customer, modeled on a large US pharma. All data in this repo is simulated; no real company or patient data is used.

## Structure

| Folder | Contents |
| --- | --- |
| `daily-log/` | One note per study day: what I built, what broke, what I learned |
| `cheatsheets/` | Quick references: [applications & modules](cheatsheets/applications-and-modules.md), [scripting](cheatsheets/scripting.md) |
| `deploy/` | [Deployment order](deploy/README.md) and XML for items update sets don't capture (scheduled job) |
| `projects/` | Portfolio projects (update sets, scripts, READMEs, demos) |
| `data/` | Simulated datasets (CSV) |

## Portfolio projects (planned)

- **P1 · GxP Incident Triage Agent**: AI-assisted classification and routing of lab/manufacturing incidents with GxP impact checks
- **P2 · Medical Inquiry & Adverse-Event Intake**: agentic intake that detects adverse events in medical inquiries and creates safety cases, with guardrails
- **P3 · Deviation → CAPA App**: scoped quality app with approvals, a Scripted REST endpoint and openFDA enrichment

## Progress

- [x] [Week 1, Day 1](daily-log/week1-day1.md): platform foundations, Lumen Pharma org set up
- [x] [Week 1, Day 2](daily-log/week1-day2.md): tables & fields deep dive, MediCare HMS scoped app (7 tables, CMDB extension)
- [x] [Week 1, Day 3](daily-log/week1-day3.md): forms, views, UI policies vs data policies (bypass test), dot-walked list filters
- [x] [Week 1, Day 4](daily-log/week1-day4.md): roles, groups, ACLs (table, field, condition-based); impersonation test matrix 36/36
- [x] [Week 1, Day 5](daily-log/week1-day5.md): import sets & transform maps (coalesce, choice actions, messy data), update set hygiene, Instance Scan
- [x] [Week 2, Day 1](daily-log/week2-day1.md): GlideRecord & business rules (age, allergy block, ward bed counts, discharge, full-ward block)
- [x] [Week 2, Day 2](daily-log/week2-day2.md): script includes (PatientUtils, PrescriptionUtils, WardUtils), refactor + regression test, read-only health check
- [x] [Week 2, Day 3](daily-log/week2-day3.md): GlideAggregate, fix script with dry run (idempotent), nightly reconciliation job, deployment order
