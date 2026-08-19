# Noora Health Portal — Feature Documentation

Internal web-based admin/operations tool used to run, audit, and troubleshoot the Noora monitoring system (see companion doc `Noora-System.md` for what Noora itself is).

---

## 0. Overview

- Application name: **Noora Health Portal**, version **1.4**.
- Footer credit: "Ari Lipponen @ TeliaCompany 2018–2024" — suggesting this tool has been maintained/authored primarily by Ari Lipponen since 2018.
- It's a PHP-based internal web app (config/rule files it displays live under `/opt/provisioning/...`).
- Top navigation bar (8 primary tabs): **Action Required · Provisioning Control · System Tools · System Control · Change Reports · System Reports · External Data · Device Dashboard**
- Persistent controls in the top-right: **Auto Update** (toggle), **Toggle Width** (layout), **Logout**.
- Session shows the logged-in admin user by name (e.g. "Hello Tran [admin], welcome to Health portal!").

In short: the Health Portal is the **operational control tower** for Noora's automated device-discovery-to-monitoring pipeline (see §3.2 for the underlying phase model) — it lets an operator see what's broken, why, inspect raw config/rules driving automation, query raw data, and pull up a live per-device dashboard.

---

## 1. Action Required

Surfaces devices that need **manual** intervention because automation couldn't handle them. Three panels, each listing: date added, last discovery, SNMP/ICMP status, device id, device name, device description, IP address, RO (read-only SNMP) community, and a **re-provision** action button per row.

1. **New devices which did not respond to SNMP (mandatory)** — devices discovered but SNMP polling failed.
2. **New devices which are missing device OS support (mandatory)** — devices discovered successfully (SNMP/ICMP OK) but their OS isn't supported/mapped by the system yet.
3. **New devices which are missing device control grouping (mandatory)** — devices that haven't been pinned into a Control Device Group, so the automated rules (see §3.3) have nothing to act on.

This is effectively the **exception queue** for the otherwise-automated provisioning pipeline.

---

## 2. Provisioning Control

Live/recent view into the auto-provisioning pipeline.

- **Devices currently under provisioning (auto-provisioning)** — in-flight devices, showing device id, enabled object count so far, current provisioning phase (e.g. "phase 2 - device pinning"), last discovery time/age, discovery status, and whether it's "stuck".
- **Most recent finished provisionings (auto-provisioning completed)** — completed runs: finish time, device name/id, enabled object count, final provisioning state (e.g. "completed", "deleted"), last discovery info.
- **Most recent provisionings errors (auto-provisioning error)** — failed runs with a log entry explaining the failure, e.g.:
  - `DevicePinner: Unsupported device`
  - `DiscoveryPoller: looks like we got no XML document data`
  Each errored row also gets a re-provision button.
- **All devices currently in native discovery** — devices SevOne is natively discovering (panel present, not fully captured in detail).

This tab is the operational **pulse** of automated provisioning — what's running, what just finished, and what's failing right now.

---

## 3. System Tools

Six sub-tabs, all diagnostic/query tools for admins:

### 3.1 Device Data
- Query a specific device by name.
- Checkboxes to select which categories of data to pull:
  - **Basic/Provisioning Data**: basic device data, provisioning data.
  - **Object Data**: enabled objects, disabled objects.
  - **Group Data**: associated device groups, associated object groups.
  - **Link Data**: active link objects, deleted link objects.
  - **Alert Data**: alert config data, thresholds per objects, active alerts.
- "Select All" / "Clear All" helpers.
- Results render below in "Device Data Results".

### 3.2 Raw Data Query
- **Group Query Options**: find groups by object group name regex and/or parent group name regex ("Find Groups" button). Shows live pre-info counts:
  - "Selected group has ? objects (deleted included)"
  - "Selected group's all child groups have ? objects in summary (deleted included)"
  - "Running raw dump will generate estimate of ? rows if all selected indicators exist (deleted included)"
  - "Running raw dump from childs will generate estimate of ? rows..."
  - Count key: I = ICMP objects, S = SNMP objects, C = COC objects, O = other objects.
- **Raw Data Query Options** (right side): select a time period; select one or more indicators from a categorized list (e.g. under "Calculation related": Bits In, Bits Out, Packets In, Packets Out, Utilization In, Utilization Out).
- Safety/display options: Exclude deleted devices, Exclude deleted objects, Display data in browser (auto-disabled if result exceeds 10,000 rows), Bypass 10k display limit (explicit warning: "may cause browser to crash").
- This is essentially a **raw statistics export/query tool** for bulk indicator data across a group of devices/objects.

### 3.3 OID Validator
- Select target devices (by group), select tree branch (parent group), enter an OID, click Validate.
- Note: "Devices not included in here will be skipped"; "Only the first available device will be selected from each child group."
- Suggested workflow hint: use SMO Device Info > SMO Device Model to pick a representative device.
- Purpose: verify that a given SNMP OID actually resolves/returns data on representative devices before rolling out a monitoring rule that depends on it.

### 3.4 Device Deviations
Dashboard of large red counters, each with a drill-down list (row: date, device name, device id, IP, etc.):
| Metric | Example count seen |
|---|---|
| Devices not responding to SNMP during the last discovery | 76 |
| Devices not responding to ICMP during the last discovery | 193 |
| Active devices with zero objects | 307 |
| Deleted devices with active objects | 131 |
| All devices in new state | 12 |
| Active devices with SNMP plugin disabled | 3 |
| Active devices with ICMP plugin disabled | 2 |
| Active devices with polling disabled | 116 |

Purpose: a **data-quality / health scorecard** at the device level — spotlighting devices that are "wrong" in some operationally meaningful way (unreachable, disabled, orphaned, deleted-but-not-cleaned-up, etc.).

### 3.5 System Anomalies
Dashboard of large red counters focused on **grouping/structure** problems rather than individual devices, each with a drill-down list:
| Metric | Example count seen |
|---|---|
| Empty leaf device and device type groups | 882 |
| Empty Object Groups | 2268 |
| Object groups which do not follow custom hierarchy | 113 |
| Object groups over 1000 members | 67 |

Drill-down columns vary by panel: parent/class name, group name, group id, and (for the "over 1000 members" panel) member count.

Purpose: surfaces **group-hygiene** issues — empty groups cluttering the tree, groups that break the intended hierarchy convention, or groups that have grown unmanageably large (e.g. "Temperature Critical" with 28,028 members, "Metro Ports" with 23,969 members).

### 3.6 OG (Object Group) Tree Viewer
- Search by Group name (any words) and/or Group id.
- Toggle switches: Show class, Show members, Show bind type.
- "Search" / "Reset" buttons, results below in "Object Group Tree Results".
- Purpose: a general-purpose **explorer** for navigating/inspecting the object group hierarchy directly.

---

## 4. System Control

Seven sub-tabs — this is the deepest, most "engine internals" section of the portal, exposing both live statistics and the actual raw rule/config files that drive automated provisioning.

### 4.1 System Statistics
- **Generic Statistics** panel:
  - Number of active devices (e.g. 4097), virtual devices (5430), deleted devices (594), peers/appliances (5).
  - "Capacity <appliance-name> (#id)" for each collector appliance, shown as objects-used / 100,000 capacity and percentage (e.g. "Capacity aura08 (#11): 73423/100000 (73.4%)").
  - Number of device groups (3785), number of object groups (5732).
  - Average enabled object usage per device, average disabled object usage per device, average indicator usage per object.
  - Total enabled/disabled object counts, total enabled indicator count.
- **Object Statistics** panel: per-server (per SevOne appliance, e.g. `ferguson1.express.sonera.net`, `aura06–09.express.sonera.net`) breakdown of enabled / disabled / deleted / visible / hidden / total object counts, with a summary row totaling across all servers.
- **Indicator Statistics** panel: same per-server breakdown but for indicators (enabled / disabled / deleted / total), with summary row.
- Purpose: **capacity planning and system-wide health at a glance** — how full is each SevOne appliance, how much of the licensed capacity is used, and how object/indicator counts break down.

### 4.2 Device Statistics
- Per-device table: device name, last discovery date, days since last discovery, peer (which SevOne appliance/collector it's assigned to), enabled object count, disabled object count.
- Effectively a sortable/scrollable master list of every device with its discovery freshness and object counts — useful to spot stale or lopsided devices (e.g. one device with 2466 enabled objects vs. most others with ~15–25).

### 4.3 Control Groups
- "Get all control groups" (button) or "Get control groups where device is pinned" (enter a device name, button "Get by device").
- Shows the **Control Device Groups level structure**: `[PlatformCategory] → [PlatformClass] → [PlatformGroup] → [PlatformType]`.
- Purpose: inspect which Control Device Group(s) a given device is pinned to (which in turn determines which provisioning rules apply to it — see §4.5–4.7).

### 4.4 Constants
- Raw **File Display** of a PHP constants file (system-wide configuration constants), e.g.:
  ```php
  define("DEVICE_TYPE_ROOT", "Partial Discovery");
  define("DEVICE_GROUP_ROOT_PLATFORM_1", "Lab Equipment");
  define("DEVICE_GROUP_ROOT_PLATFORM_2", "Customer-Premises Equipment");
  define("DEVICE_GROUP_ROOT_PLATFORM_3", "Fixed IP Platforms");
  define("DEVICE_GROUP_ROOT_PLATFORM_4", "Virtual Environments");
  define("DEVICE_GROUP_ROOT_PLATFORM_5", "");
  define("DEVICE_GROUP_ROOT_MAINTENANCE", "SMO Maintenance Class");
  define("DISCOVERY_CONTROL_LOG", "/var/log/provisor/discoveryControl.log");
  ```
- Purpose: read-only view into the root-level naming/config constants that anchor the whole Control Device Group hierarchy and where discovery logs live.

### 4.5 Rules Generic
- Raw file display of `/opt/provisioning/cfg/_RULES_GENERIC.ini`.
- Documented purpose (from the file's own header comment): determines which **object types** are associated with devices during the **DEVICEPINNER provisioning process (phase 2)** — i.e. mass object-type binding for every device in a rule's target.
- Mechanics: devices are pinned to Device Type Groups, which are the file's key values; the object types listed under each Device Type Group get associated with those devices; during discovery, SevOne uses all associated object types to find the actual objects. **After discovery, ALL found objects are DISABLED by default** — enabling them is a separate, later step (see Rules Enable, §4.7).
- Usage convention example: `[Memory]` is a Device Type Group name that should exist under each vendor branch (Junos, TiMOS, etc.); the system automatically picks the correct `[Memory]` group since a device always belongs to exactly one vendor group.

### 4.6 Rules Advanced
- Raw file display of `/opt/provisioning/cfg/_RULES_ADVANCED.ini`.
- Documented purpose: determines which **additional** object types are associated with devices during DEVICEPINNER phase 2, but keyed **by TAGS** (device-specific object-type binding, on top of the generic rules).
- Rationale given in the file's own comments: e.g. in TiMOS, a "Logical Interface" (aka "SAP") SNMP query is relatively heavy, so it should only be enabled for devices that actually have one or more SAP interfaces requested to be monitored — achieved by tagging the device (e.g. tag `"sap"`) so this rule binds it under a Device Type Group ("Logical Interface") it wouldn't normally belong to.
- Same `[Group name]` usage convention as Rules Generic.

### 4.7 Rules Enable
- Raw file display of `/opt/provisioning/cfg/_RULES_ENABLE_OBJECTS.ini`.
- Documented purpose: determines which of the (previously discovered-but-disabled) objects actually get **enabled** during the **DISCOVERYPOLLER phase (phase 3)**.
- Rule structure (per rule block, e.g. `[Rule0202]`):
  - `match_objtype` — root object type name; retrieves that root plus all child types associated to the device.
  - `match_tag` — optional; enables objects based on a tag match (e.g. tag `"COS"` originating from SMO). Note: explicitly *not* the same mechanism as description-text tags.
  - `match_regex` — regex to match objects (`.*` = match all).
  - `match_column` — which column to match against: `"name"` or `"description"`.
  - `match_plugin` — which object plugin to require, e.g. `"ICMP"`, `"SNMP"`, `"COC"`.
  - `platformCategory[]`, `platformClass[]`, `platformType[]` — arrays defining which Control Device Groups (see §4.3/§6.4 in the companion doc) this rule activates for; only "Control Device Groups" can be used here, allowing precise targeting (e.g. specific SD-WAN platform classes, HAG, Sedn platform type).

**Summary of the three rule files together**: they form the **three-phase provisioning pipeline**:
1. Rules Generic + Rules Advanced (phase 2, DevicePinner) → decide *which object types* get associated/discovered per device (found objects start disabled).
2. Rules Enable (phase 3, DiscoveryPoller) → decide *which of those discovered objects* actually get turned on for active polling/monitoring.

---

## 5. Change Reports

- Select **Group Type**: Device or Object (radio buttons).
- Select **Date 1** and **Date 2** — dropdowns; the two most recent available dates are auto-selected, but any available dates can be chosen instead. Date 1 should be the older date being compared against (i.e. "what changed between Date 1 and Date 2").
- "Run" button produces results in the "Change Report Results" panel below.
- Purpose (per the tool's own description text): "Change reports will offer information of changes which has happened for devices, objects and groups between given dates."
- This is effectively a **diff/audit tool** — what devices/objects/groups were added, removed, or modified between two points in time.

---

## 6. System Reports

- A single dropdown listing many pre-built, named reports; select one, click "Run".
- Reports observed in the list (alphabetically, non-exhaustive — list scrolls further):
  - Ari testaa smo generic device category
  - Autosummary validation report
  - Device group rule report
  - Double device report
  - Dump snet latest
  - Dump spare part inventory latest
  - Dump spare part material latest
  - Exporter group report
  - Exporter group report locate to-be-removed
  - Generic device category against smo first two levels
  - Generic device category against smo full list
  - Generic device category against smo full with details
  - Generic device category against smo uncategorized
  - Generic device category against snet first two levels
  - Generic device category against snet full list
  - Generic device category against snet print uncategorized
  - Generic device category smo deviceroles.properties
  - Install base all by chassis model name
  - Install base all by chassis part number
  - (list continues beyond what was captured)
- Purpose: a library of **canned/scheduled-style reports** — largely focused on cross-checking Noora's device categorization against external inventory systems (SMO, SNET), validating install-base/hardware inventory, spotting duplicate or "double" devices, and locating stale exporter groups.

---

## 7. External Data

Two stacked panels tracking the health of every external data feed Noora depends on.

### 7.1 External Data Overview
Table columns: System, Content, Path, Filename, Rows, Cols, Filetime, Filesize. Feeds observed:
| System | Content | Example filename |
|---|---|---|
| SAP ERP | Material Data | sap_materials_*.csv |
| SAP ERP | Stock Inventory Data | sap_inv_fibc_*.csv |
| SM/O L2 | Termination and Service Data | sevone_termination_sync.csv |
| SM/O L2 | SAP and Service Data | sevone_sap_sync.csv |
| TOMAATTI | Device Data | device_record.csv |
| TOMAATTI | Customer Data | customer_record.csv |
| TOMAATTI | Product and Subscription Data | subscription_record.csv |
| MIT | Product and Subscription Data | sevone-export.csv |
| SATELLITE SYSTEMS | SNET Device Data | sevone_laitteetdumppi.csv |
| SM/O | Link Data | smo_link_dump.csv |
| NOORA (import) | Filtered L2 Service Data | refined.csv |
| NOORA (import) | Previous L2 Service Data | refined.old |

Row counts range widely (from a few thousand up to 537,690 for SM/O Link Data), confirming these are full nightly-scale data dumps, not incremental feeds.

### 7.2 External Data Validation
Same feed list, now validated column-by-column: **File Exists**, **Read Access**, **Date Check**, **Data Parsing**, **Parsed Rows** (as `parsed/total`), **Parsing Time** (seconds).
- Most feeds show OK across all checks.
- One observed exception: MIT / Product and Subscription Data showed a **WARNING!** on its Date Check while still parsing successfully — i.e. the file's timestamp looked stale/off even though its contents were readable and valid.
- Purpose: this is Noora's **input pipeline health check** — confirming every upstream file that feeds provisioning/enrichment actually landed, is readable, is fresh, and parsed cleanly before the automated pipeline trusts it.

---

## 8. Device Dashboard

The single-device, real-time/historical monitoring view — the "SevOne NMS UI" equivalent inside the Health Portal.

- **Device selector**: a large searchable dropdown of all devices (e.g. `fi-aljlans-csg1`, `fi-alv-rer1/2/3`, `fi-ams-rer1/2`, `fi-anjkanjl-rer1`, ...) or a free-text "or type here" field; "Run" button.
- Labeled **"Device Dashboard v1.6"** (a versioned sub-component within the v1.4 portal).
- Header row for the selected device shows: device name, location/description (e.g. "HyperScale, ALAJÄRVI"), and quick links out to other systems: **Splunk, Sense, Rasmus, Horizon, Health** (with an info icon).
- **Time range selector**: Hour / Day / 3 Days / Week / Month / 3 Months / Custom.
- **Staleness warning banner** (shown when applicable), e.g.: *"Last SM/O update is 518 days old. Port check or SevOne update is recommended."* — flags when the upstream SM/O source data backing this device hasn't refreshed in a long time.
- **Sub-links**: Basic, Hardware, Interface, Inventory, Info (different data views for the device).
- **Monitored Interfaces**: list of the device's monitored interfaces as clickable links, each annotated with what it connects to (e.g. `Bundle-Ether1 (me-alj-s01)`, `HundredGigE0/0/0/24 (fi-vli-rer1)`).
- **Graphs/panels rendered** (time-series, matching the selected time range):
  - ICMP Availability (%)
  - SNMP Responsiveness (%)
  - Uptime Counter (raw counter value over time)
  - ICMP Jitter (ms) — with Name/Min/Ave/Max summary table
  - ICMP Maximum ping time (ms) — with summary table
  - ICMP Packet loss (%) — with summary table
  - **CPU Utilization** (%) — multi-line chart, one line per logical CPU/process (e.g. "CPU 0/0-Virtual processor...", "CPU 0/RP0-...", "CPU 0/RP1-..."), each with Min/Ave/Max/Cur columns.
  - **Memory Utilization** (%) — same per-process multi-line breakdown with Min/Ave/Max/Cur columns.
  - **Active Alerts** panel at the bottom, e.g. "Active Alerts (showing 0 out of 0 alerts) — No alarms found."
- Purpose: this is the portal's **single-pane-of-glass device view** — everything an operator needs to triage one specific device's health (reachability, responsiveness, resource utilization, active alarms) without leaving the Health Portal.

---

## 9. Summary — What the Health Portal Is Capable Of

Putting it all together, the Noora Health Portal is an **admin/operations console** that gives an operator (like "Tran" in the observed screenshots) end-to-end visibility and control over the Noora provisioning-to-monitoring pipeline:

- **Triage** newly-discovered devices that automation couldn't fully onboard (Action Required).
- **Monitor** the live and historical state of the auto-provisioning pipeline itself, including errors (Provisioning Control).
- **Query** raw device/object/group/alert data on demand, run bulk raw statistics dumps, and validate SNMP OIDs before relying on them (System Tools).
- **Audit data quality** at both the device level (unreachable, disabled, orphaned devices) and the group/structure level (empty, malformed, or oversized groups) (Device Deviations / System Anomalies).
- **Explore** the object group hierarchy directly (OG Tree Viewer).
- **Inspect system-wide capacity and statistics** per SevOne appliance/collector (System Statistics, Device Statistics).
- **Trace exactly which Control Device Group(s) a device belongs to**, and view the underlying rule/config files (Generic, Advanced, Enable rules; Constants) that determine what gets discovered and enabled for that device — i.e. full transparency into the automation logic, not just its output.
- **Audit changes over time** for devices, objects, and groups (Change Reports).
- **Run a library of pre-built cross-check reports** against external inventory systems like SMO and SNET (System Reports).
- **Verify the health of every external data feed** the pipeline depends on, down to per-file parse status and row counts (External Data).
- **Drill into any single device's live/historical monitoring data** — reachability, performance, utilization, and active alarms — in one dashboard (Device Dashboard).
