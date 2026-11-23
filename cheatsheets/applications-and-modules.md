# Cheat sheet: ServiceNow applications & modules

A quick reference to the navigator. Menu names can differ slightly between releases and between the classic UI and Next Experience. The **table name** always works: type `table_name.list` in the navigator filter.

## Navigator shortcuts

| Type in the filter navigator | Opens |
| --- | --- |
| `incident.list` | The list view of a table |
| `incident.LIST` | The list in a **new tab** |
| `incident.do` | A new, blank form |
| `incident.FORM` | A new form in a **new tab** |
| `incident.config` | The table's configuration: dictionary, layouts, rules |
| `sys.scripts.do` | Scripts - Background |
| `stats.do` | Instance version, node and build info |
| `sys_properties.list` | System properties |

**Tips:**
- Right-click a list breadcrumb → **Copy query** to reuse a filter in a script.
- Right-click a form header → **Copy sys_id**.
- Use the **globe icon** (top bar) to check the application scope and current update set before every build session.

## Building & configuring

| Application → Module | What it's for | Table |
| --- | --- | --- |
| System Definition → **Tables** | Create and view tables; open the **schema map**; Controls tab (auto-number, user role) | `sys_db_object` |
| System Definition → **Dictionary** | Field definitions: type, reference, max length, mandatory, read-only, default value | `sys_dictionary` |
| System Definition → **Choice Lists** | Choice values and labels for Choice fields | `sys_choice` |
| System Definition → **Application Menus** | Navigator menus and their modules, with role restrictions | `sys_app_application` / `sys_app_module` |
| System Definition → **Business Rules** | Server scripts that run on insert, update, delete or query | `sys_script` |
| System Definition → **Script Includes** | Reusable server-side classes and functions | `sys_script_include` |
| System Definition → **Client Scripts** | Browser scripts on forms (onLoad, onChange, onSubmit) | `sys_script_client` |
| System Definition → **UI Actions** | Form and list buttons, links and context-menu items | `sys_ui_action` |
| System Definition → **Scripts - Background** | Run a one-off server script (careful: it changes real data immediately) | — |
| System Definition → **Fix Scripts** | Saved, re-runnable one-off scripts, captured in update sets | `sys_script_fix` |
| System UI → **UI Policies** | Show, hide, make mandatory or read-only on forms (browser only) | `sys_ui_policy` |
| System UI → **Views** / **View Rules** | Alternative form and list layouts, and rules for who gets which | `sys_ui_view` / `sysrule_view` |
| System UI → **Form Layouts / Form Builder** | Arrange fields and sections (or use the form header → Configure → Form Layout) | `sys_ui_section` |
| System Policy → Rules → **Data Policies** | Server-side mandatory and read-only rules, applying to imports and APIs too | `sys_data_policy2` |
| **ServiceNow Studio** / **App Engine Studio** | Build and manage scoped applications in one place | `sys_app` |
| **CI Class Manager** | Create and extend CMDB classes (e.g. extend Hardware) | — |

## Users & security

| Application → Module | What it's for | Table |
| --- | --- | --- |
| User Administration → **Users** | People who log in. Set the time zone on the user record | `sys_user` |
| User Administration → **Groups** | Teams. **Give roles to groups**, not to users | `sys_user_group` |
| User Administration → **Roles** | Named permissions; roles can contain other roles | `sys_user_role` |
| Group members / user roles | Who's in which group, and who holds which role (look for *Inherited*) | `sys_user_grmember` / `sys_user_has_role` |
| System Security → **Access Control (ACL)** | Table and field rules for read, write, create and delete. **Elevate to security_admin** to edit | `sys_security_acl` |
| Access Roles | Which roles each ACL requires | `sys_security_acl_role` |
| Avatar menu → **Impersonate user** | Test as a real user. Admin passes most ACLs, so always test as someone else | — |
| Avatar menu → **Elevate role** | Temporary security_admin, for editing ACLs | — |
| Cross-scope privileges | APIs your scoped app may call outside its scope; captured in update sets | `sys_scope_privilege` |

## Data in & out

| Application → Module | What it's for | Table |
| --- | --- | --- |
| System Import Sets → **Load Data** | Upload CSV or Excel into a staging table | — |
| System Import Sets → **Transform Maps** | Field maps, **coalesce**, choice actions and scripts to target tables | `sys_transform_map` |
| System Import Sets → **Data Sources** | Where import data comes from (file, JDBC, REST); can be scheduled | `sys_data_source` |
| Import set rows | The state of each row: Inserted, Updated, Ignored, Skipped, Error | `sys_import_set_row` |
| Any list → right-click header → **Export** | XML, CSV or Excel export of the filtered list | — |
| Any list → right-click header → **Import XML** | Load an XML export (on the **target** instance) | — |

## Moving work between instances

| Application → Module | What it's for | Table |
| --- | --- | --- |
| System Update Sets → **Local Update Sets** | Capture configuration changes. One set per feature; never build in Default | `sys_update_set` |
| Customer Updates (on an update set) | The individual changes in a set; move strays by changing their *Update set* | `sys_update_xml` |
| System Update Sets → **Retrieved Update Sets** | On the **target**: Import Update Set from XML → **Preview** → **Commit** | `sys_remote_update_set` |
| System Update Sets → **Merge Update Sets** | Combine sets | — |
| Update set → **Back Out** | Undo a completed local set on this instance (last resort) | — |
| Source control / app repository | Preferred for scoped apps: publish versioned apps | — |

## Automation & integration

| Application → Module | What it's for | Table |
| --- | --- | --- |
| **Flow Designer / Workflow Studio** | No-code and low-code flows, subflows and actions | `sys_hub_flow` |
| System Scheduler → **Scheduled Jobs** | Run a script on a schedule | `sysauto_script` |
| System Policy → **Events** → Registry | Define events fired from scripts with `gs.eventQueue()` | `sysevent_register` |
| System Notification → Email → **Notifications** | Emails triggered by record changes or events | `sysevent_email_action` |
| System Web Services → **Scripted REST APIs** | Build your own inbound REST endpoints | `sys_ws_definition` |
| System Web Services → **REST Message** | Define outbound REST calls | `sys_rest_message` |
| **REST API Explorer** | Try the platform's Table API and others | — |
| **IntegrationHub** / Spokes | Pre-built connectors used in flows | — |

## Troubleshooting & health

| Application → Module | What it's for | Table |
| --- | --- | --- |
| System Logs → **System Log → All** | `gs.info` / `gs.warn` / `gs.error` output (filter on Source or Message) | `syslog` |
| System Diagnostics → **Session Debug** | Debug Security Rules, Business Rules, SQL and more, shown on screen | — |
| **Script Debugger** | Breakpoints in server scripts | — |
| **Instance Scan** / Run Point Scan | Health checks (e.g. a field type that doesn't match its database column) | `scan_result` |
| Record → ≡ → **History** | Every change to a record: who, when, old and new values | `sys_audit` / `sys_history_set` |
| System Properties | Instance settings (e.g. `glide.i18n.single_currency`) | `sys_properties` |

## Common out-of-box apps (ITSM and others)

| Application | Main tables | One line |
| --- | --- | --- |
| **Incident** | `incident` | Unplanned interruptions; restore service fast |
| **Problem** | `problem` | The root cause behind incidents |
| **Change** | `change_request` | Planned changes with approvals and risk |
| **Service Catalog** | `sc_cat_item`, `sc_request`, `sc_req_item`, `sc_task` | Things people can order or request |
| **Knowledge** | `kb_knowledge` | Articles and how-tos |
| **CMDB** | `cmdb_ci` and its children | Configuration items and their relationships |
| **Task** (parent table) | `task` | Parent of incident, problem, change and more; shared fields |
| **Service Portal** | `sp_portal`, `sp_page` | Self-service web portal |
| **Reports / Platform Analytics** | `sys_report` | Lists, charts and dashboards |

## Handy system tables

| Table | Holds |
| --- | --- |
| `sys_attachment` | Files attached to any record |
| `sys_journal_field` | Journal entries (work notes, comments, your Nurse Notes) |
| `sys_audit` | Field-level change history |
| `sys_email` | Sent and received emails |
| `sysevent` | The queue of fired events |
| `sys_metadata` | Parent of all configuration records |
