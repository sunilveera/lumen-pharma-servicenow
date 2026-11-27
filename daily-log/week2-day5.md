# Week 2, Day 5 · Debugging tools & Week 2 review

**Date:** 2026-11-27
**Instance:** Personal Developer Instance (PDI)
**Practice data:** MediCare HMS scoped app (fictional hospital, simulated data)
**Update set:** HMS Logging v1 (3 updates). HMS Client Scripts v1 is marked Complete.

**Goal:** learn which debugging tool answers which question, and add diagnostic logging that can be switched on in Prod without a deployment.

## Session debug: which rules ran?

**Admitting Priya to Cardiology:**

| Rule | Result | Why |
| --- | --- | --- |
| Block admission to a full ward | Ran (before) | Its condition passed. It didn't abort, because there were free beds |
| Calculate age from date of birth | Skipped | `date_of_birth VALCHANGES` not satisfied |
| Clear ward on discharge | Skipped | `is_admitted CHANGESTO false` not satisfied |
| keep ward bed counts right | Ran (after) | It triggered the **Ward** before rules for Cardiology Ward |
| Set available beds from total (Ward) | Skipped | `total_beds VALCHANGES` not satisfied, so it doesn't conflict with the bed rule |

**Discharging Priya:**
- *Block admission* was skipped.
- *Clear ward on discharge* ran and emptied `admission_date` and `ward` (shown as `old =>` with an empty new value).
- Then the bed rule ran.

**Lessons:**
- **One patient save ran rules on two tables.** The nesting in the output shows the cascade.
- **On discharge, `current.ward` is already empty** by the time the after rule runs, so the rule can only free the bed through **`previous.ward`**.
- **The display rule runs in the next transaction** (the form reload), after *"Preceding lines from previous transaction"*.

## Script Tracer: which line changed the field?

The tracer must be **started before** the action; my first attempt recorded nothing. Admitting Priya recorded two entries:

| # | Script | Table | Change |
| --- | --- | --- | --- |
| 1 | Update (UI Action) | Patient | Is Admitted 0 → 1, Admission Date, Ward (my form edits) |
| 2 | **WardUtils, line 41** | Ward | **Available Beds 28 → 27** |

The tracer points at **the line that changed the value** (WardUtils), not the business rule that called it.

## Script Debugger: breakpoints in `adjustBeds`

On this release, you set breakpoints **in the script editor gutter**, then start the debugger with **F2**. Shortcuts: F9 resume, Alt+F9 step over.

| | `change` | `bedsBefore` | `bedsAfter` | Called from |
| --- | --- | --- | --- | --- |
| Admit Kiran | -1.0 | 38 | 37 | `executeRule` line 11 |
| Discharge Kiran | 1 | 37 | 38 | `executeRule` line 8 |

**What the debugger showed:**
- **The call stack** (UI Action → business rule wrapper → `executeRule` → `adjustBeds`) shows **which branch** of the bed rule ran.
- **`this`** showed `WARD_TABLE` and `PATIENT_TABLE` from `initialize()`.
- **`ward` is a ScopedGlideRecord.**
- **Transaction Detail** showed the submitted form values, including the original value before the change.
- **`-1.0`:** the server's JavaScript engine (Rhino) stores numbers as decimals.

## Diagnostic logging switchable in Prod

- I added `gs.debug('HMS beds: adjustBeds called, ward=…, change=…')` to `WardUtils.adjustBeds`.
- I created the property **`x_1941695_medica_0.logging.verbosity`**:
  - choicelist: `debug, info, warn, error`
  - **ships as `info`**
- **Mistake caught:** I first typed the full name into **Suffix**. On a scoped app the form adds the prefix, so the Suffix must be `logging.verbosity`.

| Verbosity | Admit + discharge | System Log |
| --- | --- | --- |
| debug | Kiran, General Ward A | Debug line (inputs: ward, change) **plus** info line (result: 38 → 37) for each |
| info | Priya, Cardiology | Info lines only |

**What this gives you in Prod:**
- To diagnose, set one property to `debug`, wait for the problem to happen, and read the log.
- No code change, no deployment.
- The debug line shows **what came in**, and the info line shows **what happened**.

**Possible improvement:** log the ward's name, not its sys_id. Logs are read by people.

**The nightly job's first unattended run** (02:00 today) reported `wards fixed 0, ages fixed 0`, so nothing drifted after a full day of testing on Day 4.

**Payload check before marking Complete:**
- The property's payload has `<value>info</value>` and the name prefixed once.
- The client script's payload has no `console.log`.
- The **Created** column only shows when a record was first captured, so the payload is the source of truth.

## Client side

- **Console:** `HMS WardAjax answer for 07573ebe…: 18`. The surrounding platform lines (`fireAllChangeHandlers start/end`) show onChange scripts running during load, which is the platform side of `isLoading`.
- **I removed the console line,** and the removal is captured. That client script's net change in this set is none.
- **Network:** `xmlhttp.do` took **292 ms** for 1.0 kB. That's how long the nurse waits for the bed message.
- **Field Watcher: not available on this release.** Next Experience Preferences → Debugging only lists Script Debugger, Next Experience Developer Tools (for Workspaces and UI Builder) and the ATF Page Inspector. Use the Console and Sources tabs instead.

## Bug hunt (background scripts, read-only)

| Bug | My prediction | Actual | Root cause | Fix |
| --- | --- | --- | --- | --- |
| A | — | Nothing, no error | `query()` missing | Add `p.query()` |
| B | "In ICU" ❌ | "NOT in ICU" | `p.ward` is the sys_id, not the name | `p.ward.getDisplayValue()`, or better, compare sys_ids |
| C | Read up first | Last name repeated (expected) | `p.patient_name` is a GlideElement pointing at the current row | Copy the value: `getValue()` / `getDisplayValue()` |
| D | Missed the typo ❌ | **0 rows**, plus *"field name 'is_admited' not found"* | Invalid field in the query | Spell it correctly. Dry-run and check counts |

**Bug D on this PDI returned 0 rows, not all rows.** The behaviour depends on the instance (`glide.invalid_query.returns_no_rows`).
- 0 rows makes checks say *"nothing to do"*.
- All rows makes bulk changes hit the whole table.
- **Never assume which a customer's instance does.**

## FDE reflection

**Question from the ward manager:** *"After transfers from ICU, ICU shows one bed too few until it fixes itself overnight. What's going on?"*

**My answer:**
1. The nightly job fixes the count and logs it.
2. Look at System Log for the job's logs.
3. Then use session debug to see the business rules.
4. Add diagnostic logging to the util functions.
5. Reproduce in test with the Script Tracer and breakpoints.
6. Ask: are transfers done without updating the system?

**Review notes (6/10):**
- **Good:** the order, starting with read-only steps.
- **Wrong tool:** session debug only shows **my own session, live**. For past events, use the **record history** (audit) of the transferred patients, plus the `HMS beds` log lines around those times.
- **Missed today's point:** the diagnostic logging **already exists**. In Prod you flip the verbosity property, with no code change.
- **The better question back:** *how* do staff transfer a patient (form, discharge and re-admit, list edit, another system)? Each takes a different branch of the bed rule.

## Self-check: 6/7 (up from 4.5 on Day 4)

- Most answers now include the **why**.
- Half marks:
  - **Q4:** the tracer records one transaction across all scripts and shows which script changed which field.
  - **Q7:** after impersonating, turn on Debug Business Rule (Details) and Debug Security Rules in that session.

## Week 2 review: 12/12 marked for review

I marked every item ❓. Weekend plan: explain each item out loud in 2 minutes without notes, using these daily logs as the answer key, and rebuild anything I stumble on in the PDI.

## Deployment

[`deploy/README.md`](../deploy/README.md) now includes HMS Logging v1 (step 9), so there are 12 steps.

## Next

- **Day 6 (optional), freelance track:** portfolio item #1, MediCare HMS.
- **Week 3:** Flow Designer, Service Catalog, REST. Candidate: turn the nightly bed warning into an event and a notification.
