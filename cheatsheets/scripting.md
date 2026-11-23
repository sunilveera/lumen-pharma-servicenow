# Cheat sheet: ServiceNow scripting

The most-used server and client APIs, with patterns and gotchas. Examples use the MediCare HMS scope prefix `x_1941695_medica_0`; replace it with your own.

## Where scripts run

| Script type | Runs on | When | Typical use |
| --- | --- | --- | --- |
| **Business rule** | Server | Insert, update, delete, query or display | Validate or set fields (before); update other records (after) |
| **Script include** | Server | When called | Reusable logic shared by rules, flows and APIs |
| **Client script** | Browser | onLoad, onChange, onSubmit, onCellEdit | Form behaviour and quick feedback |
| **UI policy script** | Browser | When its conditions match | Form logic beyond show, hide and mandatory |
| **UI action** | Server and/or browser | Button or link click | Custom buttons ("Discharge patient") |
| **Scheduled job** | Server | On a schedule | Nightly checks and clean-ups |
| **Background / fix script** | Server | When you run it | One-off data fixes (test reads first!) |
| **Scripted REST API** | Server | HTTP request | Custom inbound integrations |

**Protect data on the server.** Client-side logic is a convenience: list edits, imports and APIs bypass it.

## GlideRecord: read

```javascript
var gr = new GlideRecord('x_1941695_medica_0_patient');
gr.addQuery('is_admitted', true);            // field = value
gr.addQuery('age', '>', 60);                 // with an operator
gr.orderBy('patient_name');                  // orderByDesc() for descending
gr.setLimit(100);                            // cap the rows
gr.query();                                  // ← don't forget!
while (gr.next()) {
    gs.info(gr.getValue('number') + ' ' + gr.ward.getDisplayValue());
}
```

| Method | Does |
| --- | --- |
| `gr.get(sysId)` / `gr.get('number', 'PAT0001022')` | Fetches one record; returns true or false |
| `gr.addQuery(field, op, value)` | Operators: `=`, `!=`, `>`, `>=`, `<`, `<=`, `IN`, `NOT IN`, `STARTSWITH`, `ENDSWITH`, `CONTAINS`, `DOES NOT CONTAIN` |
| `gr.addNullQuery(f)` / `gr.addNotNullQuery(f)` | Field is empty / not empty |
| `q.addOrCondition(f, v)` | `var q = gr.addQuery(...); q.addOrCondition(...)` |
| `gr.addEncodedQuery('status=completed^ageGT60')` | Paste from list → **Copy query** |
| `gr.hasNext()` | Is there at least one row? (no loop needed) |
| `gr.getRowCount()` | Row count. Fine for small sets; use **GlideAggregate** for counting |
| `gr.getValue('field')` | The stored value as a string (a sys_id for references). **Use for comparisons** |
| `gr.field.getDisplayValue()` | What users see (a name for references) |
| `gr.getUniqueValue()` | The record's sys_id |
| `gr.ward.ward_name` | Dot-walking through a reference |
| `gr.ward.getRefRecord()` | The referenced record as a GlideRecord |
| `gr.isValidRecord()` | Did `get()` or `next()` land on a real record? |
| `gr.canRead()` / `gr.canWrite()` | Would the current user pass the ACLs? |

**Encoded query syntax:**

| Syntax | Meaning |
| --- | --- |
| `^` | AND |
| `^OR` | OR |
| `^NQ` | A new query (OR of whole blocks) |
| `LIKE` | Contains |
| `STARTSWITH` / `ENDSWITH` | Starts / ends with |
| `ISEMPTY` / `ISNOTEMPTY` | Empty / not empty |
| `IN` | Value is in a comma-separated list |

## GlideRecord: write

```javascript
// Insert
var gr = new GlideRecord('x_1941695_medica_0_medicine');
gr.initialize();
gr.setValue('medicine_name', 'Cetirizine');
gr.setValue('generic_name', 'Cetirizine hydrochloride');
var newId = gr.insert();                     // returns the new sys_id

// Update one
if (gr.get('medicine_name', 'Cetirizine')) {
    gr.setValue('generic_name', 'Cetirizine HCl');
    gr.update();
}

// Update many (no loop)
var bulk = new GlideRecord('x_1941695_medica_0_patient');
bulk.addQuery('is_admitted', false);
bulk.setValue('ward', '');
bulk.updateMultiple();

// Delete
gr.deleteRecord();                           // the current record
// bulk.deleteMultiple();                    // every record matching the query (careful!)
```

| Method | Does |
| --- | --- |
| `gr.setWorkflow(false)` | Skip business rules for this operation (careful: bypasses your logic) |
| `GlideRecordSecure` | Same API as GlideRecord but **enforces the user's ACLs**. Plain GlideRecord runs as the system |

## GlideAggregate: count, sum, group

```javascript
var ga = new GlideAggregate('x_1941695_medica_0_patient');
ga.addQuery('is_admitted', true);
ga.addAggregate('COUNT');
ga.groupBy('ward');
ga.query();
while (ga.next()) {
    gs.info(ga.ward.getDisplayValue() + ': ' + ga.getAggregate('COUNT'));
}
```
Other aggregates: `SUM`, `AVG`, `MIN`, `MAX`, e.g. `ga.addAggregate('SUM', 'tablet_count')`. Counting happens in the database, so it's much faster than looping.

## Business rule essentials

```javascript
(function executeRule(current, previous /*null on insert & async*/) {
    if (current.status.changesTo('completed')) { /* ... */ }
    if (somethingWrong) {
        gs.addErrorMessage('Explain why to the user');
        current.setAbortAction(true);            // stop the save (before rules only)
    }
})(current, previous);
```

| API | Meaning |
| --- | --- |
| `current` / `previous` | The record now / before this save (`previous` is null on insert) |
| `current.field.changes()` | Did this field change in this save? |
| `current.field.changesTo(v)` / `changesFrom(v)` | Changed to or from a specific value |
| `current.field.nil()` | Is the field empty? |
| `current.operation()` | `'insert'`, `'update'` or `'delete'` |
| `current.isNewRecord()` | Is this an insert? |
| `current.setAbortAction(true)` | Cancel the save (in a **before** rule) |

| When | Use for | Never |
| --- | --- | --- |
| **before** | Set, validate or abort on `current` | `current.update()` (causes a double save or a loop) |
| **after** | Update **other** records | Rely on aborting; the record is already saved |
| **async** | Slow work (integrations), run in the background | Expect `previous` (it's null) |
| **display** | Pass server data to the form via `g_scratchpad` | Heavy queries |

## gs: GlideSystem

| Call | Does |
| --- | --- |
| `gs.info(msg)` / `gs.warn()` / `gs.error()` / `gs.debug()` | Write to the System Log. **Scoped apps can't use `gs.log()`** |
| `gs.addInfoMessage(msg)` / `gs.addErrorMessage(msg)` | Message at the top of the user's form |
| `gs.getUserID()` / `gs.getUserName()` | The current user's sys_id / user name |
| `gs.getUser()` | The current user object |
| `gs.hasRole('x_1941695_medica_0.nurse')` | Does the user have the role? (admin always returns true) |
| `gs.getProperty('name', 'default')` | Read a system property |
| `gs.eventQueue('x_….event.name', current, parm1, parm2)` | Fire an event (drives notifications and script actions) |

## Dates: GlideDateTime / GlideDate

```javascript
var now   = new GlideDateTime();                   // now (UTC internally)
var today = new GlideDate().getValue();            // 'yyyy-MM-dd'
var gdt = new GlideDateTime(current.getValue('appointment_datetime'));
gdt.addDaysLocalTime(1);                           // +1 day (addDaysUTC also exists)
var dur = GlideDateTime.subtract(now, gdt);        // GlideDuration between two times
gs.info(gdt.getDisplayValue());                    // in the user's format and time zone
if (gdt.after(now)) { /* in the future */ }        // also before(), compareTo()
```
Dates are stored in **UTC** and displayed in each user's time zone, Time-only fields included.

## Script include (server, reusable)

```javascript
var WardUtils = Class.create();
WardUtils.prototype = {
    initialize: function() {},

    getAvailableBeds: function(wardId) {
        var w = new GlideRecord('x_1941695_medica_0_ward');
        return w.get(wardId) ? parseInt(w.getValue('available_beds'), 10) : 0;
    },

    type: 'WardUtils'
};

// Call it from a business rule (same scope):
var beds = new WardUtils().getAvailableBeds(current.getValue('ward'));
// From another scope, use the full API name:
// new x_1941695_medica_0.WardUtils()
```

## Client scripts: g_form, g_user

```javascript
function onChange(control, oldValue, newValue, isLoading, isTemplate) {
    if (isLoading || newValue === '') return;      // skip on form load and when cleared
    if (newValue == 'true') {
        g_form.setMandatory('ward', true);
        g_form.showFieldMsg('ward', 'Choose a ward with free beds', 'info');
    }
}

function onSubmit() {
    if (!g_form.getValue('ward')) {
        g_form.addErrorMessage('Ward is required');
        return false;                              // stops the submit
    }
}
```

| g_form | Does |
| --- | --- |
| `getValue(f)` / `setValue(f, v)` / `clearValue(f)` | Read, set or clear a field |
| `setMandatory(f, bool)` / `setReadOnly(f, bool)` / `setDisplay(f, bool)` | Like a UI policy, in code |
| `showFieldMsg(f, msg, 'info'/'error')` / `hideFieldMsg(f)` | A message under a field |
| `addInfoMessage(msg)` / `addErrorMessage(msg)` | A message at the top of the form |
| `getReference(f, callback)` | Fetch a referenced record **asynchronously** (always pass a callback) |
| `addOption(f, value, label)` / `clearOptions(f)` | Change choice options |

| g_user | Does |
| --- | --- |
| `g_user.userID` / `g_user.userName` | The current user |
| `g_user.hasRole('role')` | Role check in the browser (UI only, **not security**) |

## GlideAjax: the browser asks the server

```javascript
// Script include: tick "Client callable"
var WardAjax = Class.create();
WardAjax.prototype = Object.extendsObject(global.AbstractAjaxProcessor, {
    getBeds: function() {
        var w = new GlideRecord('x_1941695_medica_0_ward');
        return w.get(this.getParameter('sysparm_ward')) ? w.getValue('available_beds') : '0';
    },
    type: 'WardAjax'
});

// Client script
var ga = new GlideAjax('x_1941695_medica_0.WardAjax');   // scoped: use the full API name
ga.addParam('sysparm_name', 'getBeds');                   // the method to call
ga.addParam('sysparm_ward', g_form.getValue('ward'));
ga.getXMLAnswer(function(answer) {                        // asynchronous; never use getXMLWait
    if (parseInt(answer, 10) <= 0) g_form.showFieldMsg('ward', 'Ward is full', 'error');
});
```

## Outbound REST (RESTMessageV2)

```javascript
try {
    var r = new sn_ws.RESTMessageV2();
    r.setEndpoint('https://api.example.com/drugs/123');
    r.setHttpMethod('get');
    r.setRequestHeader('Accept', 'application/json');
    var resp = r.execute();
    if (resp.getStatusCode() == 200) {
        var data = JSON.parse(resp.getBody());
    }
} catch (e) {
    gs.error('Drug lookup failed: ' + e.message);
}
```
Prefer a saved **REST Message** record (credentials, endpoints) over hard-coded URLs.

## Debugging

| Tool | Use |
| --- | --- |
| `gs.info('HMS: ...')` with a prefix | Then filter System Log → All on the prefix |
| Session Debug → Debug **Business Rule** / **Security** | See which rules or ACLs ran, and why |
| **Script Debugger** | Breakpoints and variable inspection in server scripts |
| Scripts - Background | Test a query (read-only) before putting it in a rule |
| `try { } catch (e) { gs.error(e.message); }` | Keep one bad record from breaking a whole job |

## Gotchas (all of these bit me)

- **A missing `gr.query()`** means no rows and no error.
- **Compare with `getValue()`,** not the field object. `getValue()` on a reference returns its sys_id.
- **Booleans:** use `current.active == true`. `getValue()` returns `'1'`/`'0'` or `'true'`/`'false'` depending on context.
- **Never call `current.update()` in a before rule.** In an after rule, `setAbortAction` is useless.
- **Validation goes in before rules, side effects in after rules.** Don't mix them.
- **Rows have no guaranteed order** without `orderBy()`.
- **GlideRecord ignores ACLs.** Use GlideRecordSecure when the user's access should apply.
- **Scoped apps:** `gs.log` isn't available. **Cross-scope privileges** are created the first time a script runs, so test before deploying.
- **A List field** stores comma-separated sys_ids: `getValue('known_allergies').split(',')`.
- **Don't query inside a loop.** Use GlideAggregate or one query with `IN`.
- **Background scripts change real data immediately.** Test reads first, and use `setLimit()`.
