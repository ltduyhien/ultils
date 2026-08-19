# Noora — System Documentation

Telia Finland's IP Backbone / network performance monitoring system, built on SevOne NMS.

---

## 1. What Noora Is

- Noora is Telia Finland's monitoring platform, built on top of the commercial SevOne NMS (Network Monitoring System) product.
- **Noora is distinct from Zaara**: Noora uses SevOne NMS as its core engine, while Zaara (the platform covered elsewhere in this documentation set) uses Zabbix. They are separate systems within Telia's OSS landscape.
- **Origin of the name**: "Noora" was originally just the project name for building a single-SevOne-instance IP Backbone monitoring solution in Finland. Over time it became the official name of the resulting system.
- **What Noora actually is**: SevOne NMS is a commercial, all-in-one performance monitoring product (built-in SOAP and REST APIs) designed for carrier/backhaul networks, SD-WAN, SDN, cloud, and datacenter environments. However, out of the box, a single SevOne instance can't handle the diversity of technologies, networks, inventories, and environments at operator scale. To solve this, Telia built extensive custom components around SevOne:
  - Components to gather and organize input data *before* it reaches SevOne NMS
  - Components to automate device provisioning *into* SevOne NMS
  - Components to automate device monitoring and alert management
  - Components to automate report management and data export
  - Components to manage the whole custom environment
  - Tailored reporting UIs for easier, more direct access to statistical data
- These custom components, together with the underlying SevOne NMS, collectively form the system called **Noora**. The SevOne installation itself is so heavily customized that it deviates greatly from standard SevOne instances used elsewhere in Telia — Noora can be thought of as its own separate system that merely uses SevOne (NMS) as a core component.
- The most meaningful of Noora's automated control functions are **data enrichment** and **categorization**, which provide most of the system's value.

---

## 2. Scale (as documented)

- ~4,000 network devices
- ~120,000 objects
- ~2,700,000 (2.7M) indicators
- Devices are polled every **180 seconds**
- Polled/raw data retention: **240 days**
- SevOne licensing model is based on number of *active objects*; a single license costs ~1.2€

---

## 3. Architecture

### 3.1 SevOne core databases
SevOne's core has two databases:
- **DATA** — contains raw statistics; unique per appliance (not replicated)
- **CONFIG** — contains everything else (device/object/group definitions, etc.); replicated across the whole cluster

### 3.2 Cluster / deployment model
- SevOne NMS can run on physical servers or virtual appliances (**vPAS100K**).
- A cluster consists of one **Master** and multiple **Collectors**.
  - Master: runs Modelling, Calculating, Provisioning, Grouping, Polling, Alerting, plus CONFIG and DATA databases.
  - Collectors: run Polling and their own local DATA database only (CONFIG is replicated from master).
- Master and Collectors can be placed in different security/network zones (e.g. Zone A for the master, facing the end user; Zone B for collectors, facing the devices, separated by a firewall).
- This lets the system scale up easily and lets device-facing polling sit closer to (or in a different security zone from) the actual network devices, which are spread across Finland.

### 3.3 Input data feed / integrated systems
Noora integrates with many upstream systems:
- **SurfManager & Operator (SM/O)** — the network orchestrator; the *main* input to Noora, provisions devices to the Noora system.
- **Satellite Systems** — feeds SNET data.
- **MIT/SAP ERP** — feeds Material Data / Stock Inventory Data.
- **Tomaatti** — feeds Device Data, Customer Data, Product and Subscription Data.
- **SM/O L2** — Termination and Service Data, SAP and Service Data.
- Also referenced: SNET, SMO, SAP more generally as inputs alongside SurfManager & Operator.
- Noora enriches this initial input data with information collected from other BSS & OSS systems' databases (e.g. Satellite Systems, Tomaatti).
- Device/connection/service provisioning from SM/O into Noora is fully automated.

### 3.4 Custom Provisioning layer
- Custom Provisioning: Modelling Control, Provisioning Control, Polling Control, Grouping Control, Constants, Rules.
- Custom scheduler (CRON-based).
- **Rasmus Admin**: management/provisioning/grouping control tool (distinct from "Rasmus", the reporting system — see §6).
- Common functions and custom scripts layer supporting the above.
- **Data collection is SNMP-only** — even though SevOne itself supports many collection methods, Noora exclusively uses SNMP to collect everything.

### 3.5 High-level data flow (five automated stages)
1. **Automated input data collection** — from inventories & network orchestration (external to Noora).
2. **Automated data enrichment and provisioning** — into SevOne.
3. **Automated statistic data collection** — SevOne polling.
4. **Automated data categorizing** — tagging into categories such as CPU, Power Supply, 100G, LAG, Mobile, Fixed IP, Edge, Switch, Metro, Tampere, DSL, Radiolink.
5. **Automated alert policy and report management** — categorized data automatically triggers pre-defined alert policies and reports.

Example alert policy: *Target: Edge & 100G & LAG; Trigger: utilization > 90%; Action: generate warning alert.*
Example alert policy: *Target: Mobile & CPU; Trigger: CPU load > 95%; Action: generate critical alert.*
Example traffic report: *Target: Mobile & Tampere; Indicators: Bits In, Bits Out; Timespan: past 7 days; Scheduling: every Mon 8am; Action: create Top-10 report.*

---

## 4. Data Model

SevOne's data structure is built around three levels: **Devices → Objects → Indicators**.

- Example chain: `FI-HSSV-ASBR1` (device) → `CPU` / `MEMORY` / `TE1/1` / `COS QUEUE RT` (objects) → `CPU UTILIZATION`, `MEMORY UTILIZATION`, `MEMORY TOTAL`, `BITS IN`, `PACKETS OUT`, `ERRORS IN`, `DISCARDS OUT`, `MULTICAST PACKETS IN`, `TOTAL PACKETS OUT`, `VOLUME`, `FORWARDED BITS`, `DROPPED PACKETS`, etc. (indicators).
- **Raw data mechanics**: every 180 seconds, each device is polled (poll device → store value), producing ~2.7M values stored to the DATA database per interval. CONFIG holds the definitions (device, object, indicator, polling interval) that drive what gets polled.

### 4.1 Metadata and data enrichment
- "Metadata" = bindable data related to a given item (device, object, or indicator).
- Parsed from different source inventories, then bound to target devices/objects during provisioning in the Noora System. This parsing/storage is fully scripted and automated.
- **Device-level metadata**: Name, Description, IP Address, SNMP Community, Postal Code, SNET Cabinet, Manufacturer, Device Type, Coordinates, etc.
- **Object-level metadata**: Name, Description — for interfaces this can include Speed information, Connected device, Connected port; for hardware objects, Serial number, Product name, Manufacturer.
- **Indicator-level metadata**: Description.

---

## 5. Monitoring Scope

### 5.1 What Noora monitors (network domains)
- **Fixed network monitoring**: IP backbone and mobile CORE; Broadband, mobile and telco EDGE; Regional networks (Metro Ethernet).
- **NFVI/SDN/Virtual monitoring**: Virtual EDGE, SD-WAN nodes, Hyperscale nodes.
- **Special monitoring**: B2C IP address allocation and utilization, Hybrid Access virtual platform monitoring, Cable HFC RF signal monitoring.
- **Capacity reporting for infra management**: Peering and NNI traffic, Core traffic, Edge traffic, Metro traffic, Metro Ring traffic.
- **Service-based traffic reporting**: IP Transit service traffic, B2C and B2B service traffic, 3G/4G/5G service traffic, CTV/IPTV/VoD service traffic, Cable Internet service traffic, DNS service traffic.

### 5.2 What is monitored on a device
- **Device component monitoring**: device performance statistics; hardware status, inventory, serial numbers, part numbers, etc.
- **Physical interface monitoring**: copper, fiber, coaxial, wireless, etc.; traffic, drops, errors, etc.
- **Logical interface monitoring**: aggregated interfaces, subinterfaces, virtual interfaces.
- **Class of Service (CoS) monitoring**: CoS Queue 1 (Best Effort), CoS Queue 2 (Business Class), CoS Queue 4 (Real Time), CoS Queue 6 (Network Control).

---

## 6. Grouping

- A **group** = a set of devices or objects bound together by a common factor; groups themselves are items stored in the SevOne CONFIG database.
- Groups are used to: set user privileges per device group; automate alert policy bindings to device components; serve directly as reports; act as input for creating additional (derived) groups.
- **Metadata itself can be set as a group**, and all groups are searchable via the Noora reporting UIs.

### 6.1 Rule-based auto-grouping example
Given devices like `fi-hki-asbr1` (desc: "Core Asbr Router"), `metro-hki-s01` ("Helsinki Area Metro Switch"), `metro-tre-s01` ("Tampere Area Metro Switch"), rules such as:
- If device name matches "metro" → pin to **Metro** group
- If device description matches "Core" → pin to **Core** group
- If device description matches "Switch" → pin to **Switch** group
- If device description matches "Tampere" → pin to **Tampere** group

...automatically populate the corresponding groups.

### 6.2 Object grouping example
Objects (CPU, Power Supply, interfaces, etc.) across devices get pinned into object groups such as `CPU`, `POWER SUPPLY`, `100G`, `LAG`, `ASBR TO METRO`, `METRO TO FTTB`, based on object name/description matching (e.g. link speed "30G"/"100G", or connected-device naming patterns).

### 6.3 Hierarchical grouping (four parallel hierarchies)
1. **Device Type → Object Type control**: All Device Types → Basic/Advanced → vendor (Alcatel TiMOS, Juniper Junos, Cisco IOS, Huawei VRP) → Hardware/Interface.
2. **Object Type → Indicator Type control**: All Object Types → Card/Logical Interface → vendor-specific sub-objects (e.g. Alcatel TiMOS IOM/MDA, Juniper FPC/RE; Alcatel SAP, Cisco Subinterface).
3. **Device Groups (for alert, report, authentication control)**: All Device Groups → Fixed IP Platform → Core (Asbr, Cr) / Aggregation → Edge (Cagw, Mobiedge) / Metro; also Device Info → Device Model (e.g. Cisco-7609s) / Device Role (e.g. CPE).
4. **Object Groups (for alert policy and report control)**: All Object Groups → Device Statistics (ICMP Availability, Card, Power Supply, Processor) / Traffic Statistics (Asbr to Core → Inet Asbr to Core, L3vpn Asbr to Core; Core to Edge → Core to Consumer Edge → Core to Helsinki Edge / Core to Tampere Edge).

### 6.4 Control Device Groups (as surfaced in the Health Portal)
Level structure: **[PlatformCategory] → [PlatformClass] → [PlatformGroup] → [PlatformType]**.
Root platform categories include: Lab Equipment, Customer-Premises Equipment, Fixed IP Platforms, Virtual Environments (plus one unnamed root), and a separate maintenance root ("SMO Maintenance Class").

---

## 7. Output Paths — Three Automated Reporting/Alerting Paths

The enriched, categorized data automatically feeds three parallel output paths:

1. **Operational Reporting (real-time)** — custom, Telia-built user interfaces tailored to network operator needs; aimed at implementation and fault-management staff for real-time troubleshooting and data analysis. Surfaces: SMO Dashboards, Rasmus UI, Horizon UI.
2. **Network Surveillance (alarms)** — Noora monitors and triggers alarms based on created thresholds. Alarms are sent to the **Nelli** system (Health portal / wiki nav also references a "Nalli" — possibly a spelling variant, not fully confirmed — see open questions), from which Control Center acts on them. Also connects to a Splunk System for surveillance-related data.
3. **Capacity and Business Reporting (long-term)** — statistics are sent to the **Qlik System** (QlikView UI, Qlik Sense UI), based on the initial data categorization structure set up via the Rasmus Admin group-management tool. This data is highly aggregated and does not offer the precision of the Operational Reporting path; it's mainly used by network/infra planning.

---

## 8. Reporting Surfaces

- **Rasmus Reporting System** — Device and Interface dashboards, Group reporting dashboards.
- **Horizon Reporting System** — Device and interface statistics, Group reporting statistics.
- **SMO Embedded Graphs** — Static Device/Interface dashboards, Inventory and Spare Part dashboard.
- **Qlik** (QlikView / Qlik Sense) — 24-hour average/max data; data volume is manageable and precision is sufficient for long-term/trend reporting; drill-down into finer detail is available via the Noora reporting UIs (to inspect what happened within a given 24h window).
- Also mentioned as part of the broader reporting/surveillance landscape: **Splunk System**, **SMO Dashboards**, **SevOne NMS UI** itself.

### 8.1 Qlik reporting details
- Designed for Planning and Business use.
- Data is prepared inside the Noora System itself:
  - Device validation and data enrichment (Noora Engine)
  - Reporting structure management (Rasmus Admin)
  - Report data selection automation (Rasmus Admin)
  - Data summary calculations (Rasmus Admin)
- Exported statistics are 24h average and max values — manageable data volume, sufficient precision for long-term/trend reporting; drill-down detail available via Noora reporting UIs.

---

## 9. Automation From the Network Engineer's Perspective

- A network engineer only needs to configure the mandatory OSS system (e.g. SM/O) — they do **not** need to know anything about the alerting or reporting side of Noora.
- From that point, with **no manual work required**, within **5–15 minutes**:
  1. Devices, connections and services are automatically provisioned to Noora.
  2. Data polling starts, and groups are created/updated automatically.
  3. Data becomes updated and present in the OSS system and in Noora reporting.
  4. Network monitoring is automatically enabled.
  5. Data is automatically added and exported to external systems.

---

## 10. Documentation Source / Provenance

- Internal Telia wiki: `wiki.nmn.telia.fi/index.php/Category:Noora`, page "Noora in a nutshell", last edited 8 Feb 2024.
- The wiki's left navigation lists Noora alongside sibling Telia tools: SMO, SMC, **Nalli**, **Hirvi**, **Lupus** (note the wiki spelling is "Nalli", not "Nelli" as seen in slide diagrams — possibly the same alarm-handling system under two spellings, not fully confirmed).
- A companion internal slide deck ("What is SevOne and why do we call it Noora") was also used as a source, covering scope, data concept, cluster setup, metadata/enrichment, grouping, and data-flow automation slides.

---

## 11. Open Questions / To Confirm

- Is Hien's InfoSys role covering Noora as well as Project Zaara, or is Noora presented purely as broader ecosystem/KT context?
- Relationship between "Nelli" (per slide diagrams) and "Nalli" (per wiki nav) — same system, a spelling variant, or genuinely two different systems?
- What Hirvi and Lupus are (listed as sibling Telia tools in the wiki nav, not yet explained).
- Whether "Rasmus" (the Reporting System) is the same underlying system as "Rasmus Admin" (used for Qlik data preparation / group management) — likely yes (one back-end, two UI facets: reporting vs. admin), but not explicitly confirmed in source material.
