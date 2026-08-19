# Rasmus — Feature Documentation

Rasmus is Noora's operator-facing **reporting UI** (part of the "Operational Reporting (real-time)" output path described in the companion `Noora-System.md` doc — see §7 there). Where the Health Portal is the admin/engine-internals console, Rasmus is what network operators actually use day-to-day to pull performance reports for devices, groups, and alerts.

---

## 0. Overview

- Application name: **RASMUS**, version **2.7.0**.
- Accessed at a URL under `massey.express.sonera.net/custom/rasmus/`.
- Top navigation (persistent across all pages): **Run Device Report · Run Group Report · Run Alert Report · My Group Reports · Support · Logout**.
- The landing/home page displays a rotating tip banner with usage hints, e.g.:
  - "Open My Group Reports to display your stored reports"
  - "Report settings will be automatically loaded if they exist"
  - "Try Graph TopN Only option in Group Reports with large groups"
  - "Open additional Rasmus views to separate browser windows to compare different reports"
  - "Report calculation time can be greatly shorten by using filters"
- Below the banner, the home page shows a set of sample/default graphs (line charts, a pie chart, bandwidth-style area/area-under-curve charts) — likely a "last report" or example preview.

---

## 1. Run Device Report

Pulls performance data for a single device (and optionally a specific interface on it).

- **Search By** — dropdown for how to locate the device (e.g. "Directly By Name").
- **Select Device** — dropdown/search, e.g. `fi-trehara-rer2`.
- **Select Interface** — dropdown of that device's interfaces, e.g. `Bundle-Ether2 (fi-trehara-csg1 Bundle-Ether1 10G)` — shows the interface name plus what it connects to and its speed.
- **Quick-access shortcut buttons**, in two columns:
  - Personal shortcuts (orange): **My Device Report**, **My Device Detailed**, **My Interface Report**, **My Detailed Interface**.
  - Org-wide shortcuts (grey): **Most Used Report**, **Most Used Detailed** (shown twice, once per report type — device-level and interface-level).
- Purpose: quick, ad-hoc lookup of a single device or interface's performance graphs without building a full custom report.

---

## 2. Run Group Report

Pulls aggregated performance data across a **group** of devices/objects (i.e. one of Noora's device or object groups — see the grouping section of `Noora-System.md`).

- **Search By** — dropdown for report category (e.g. "Device Statistics").
- **Object Group** — dropdown/search for the target group (e.g. "Core ICMP Availability").
- **Enable Filter** — optional checkbox to further filter group members.
- **Number of group members** — live count shown once a group is selected (e.g. 60 members).
- **Quick-access shortcut buttons**: **My Group Report** / **My Detailed Report** (personal, orange) and **Most Used Report** / **Most Used Detailed** (org-wide, grey).
- **Report configuration fields**:
  - Report Name (free text, e.g. "Group Report")
  - Report Description (free text, e.g. "Core ICMP Availability")
  - Select Time Frame (e.g. "Past 24 Hours")
  - Additional Options (Yes/No toggle + "Show Options" expander)
  - Show Top N (Yes/No + Show Options) — limit the report to the top N members by some metric (per the home-page tip, recommended for large groups)
  - Show Graph (Yes/No + Show Options)
  - Show Summary (Yes/No + Show Options)
  - Show Table (Yes/No + Show Options)
  - Show Alarms (Yes/No + Show Options)
- **Save selected options for this group**: "Save Options" (persists this exact report configuration for the group, so it reappears under "My Group Reports" — see §4) and "Reset Options" (revert to defaults).
- **Run Report** button (orange) generates the report using the current configuration.
- Purpose: build and optionally save a reusable, configurable performance report for any device/object group — with fine control over what's shown (graph, summary stats, raw table, active alarms) and how much data (Top N filtering) for large groups.

---

## 3. Run Alert Report

Pulls a filtered view of active/historical alerts.

- **Select Alert View** — dropdown, e.g. "Alert View By Severity" (implies other grouping views likely exist, e.g. by device, by category — not confirmed).
- **Select Severity** — dropdown, e.g. "Critical".
- **Select** button — applies the view/severity filter.
- **Report configuration fields**:
  - Report Name (e.g. "Alert Report")
  - Report Description (e.g. "Severity - Critical")
  - Select Time Frame (e.g. "Past 24 Hours")
  - Additional Options (Yes/No + Show Options)
  - Alert State — e.g. "Active alerts" (implies other states, e.g. cleared/historical alerts, likely selectable)
- **Run Report** button (orange) generates the alert report.
- **Results table** columns: Start, Latest, Severity, Category, Device Name, Object Name, Object Description, Alert Name, Alert Description, Clear Message. Each row is color-coded (critical alerts shown in red).
- Example real report observed: *Severity – Critical, 2026/08/18 05:16 – 2026/08/19 05:16 — 90 unique alert(s) found*, including rows such as:
  - `cagw-tkukes` / `Ca6/0/15` / "Registered Mode" / "Below 25% (0.0..."
  - `me-jns-s02` / `Card Sfm 2` / "[VE Nokia] [MN..." / "Card Down" / "Admin Up and O..."
  - `me-ymlyc-s01` / `Card Sfm 2` / "Card Down" / "Admin Up and O..."
  - `me-hkihrif-s02` / `2/1/30` / "mnni061032653" / "Physical Interfac[e]" / "Availability Down"
  - `fi-hkibdc-nm1` / `BGP peer A` / "AS1759 to AS65" / "BGP Peer Down" / "Admin Up and B..."
  - `fi-kkp-rer1` / `TenGigE0/0/2` / "mnni061037498" / "Physical Interfac[e]" / "Availability Down"
- Purpose: the primary tool for reviewing what's actively alarming across the network, filterable by severity and time window — this is the operator-facing counterpart to the "Network Surveillance (alarms)" output path in the Noora architecture (alerts populated via SevOne thresholds, surfaced here and also routed to the Nelli/Nalli system and Control Center).

---

## 4. My Group Reports

- Lists group reports where the current user has previously saved settings (via the "Save Options" button in Run Group Report, §2).
- Empty-state message observed: *"You have no stored reports. Use Save Options button in Run Group Reports to store reports here."*
- Purpose: a personal library of pre-configured, reusable group reports — save a report's full configuration once (group, time frame, display options) and re-run it later without re-entering all the settings.

---

## 5. Support

- A dedicated Support tab exists in the top nav (contents not yet captured in detail).

---

## 6. Summary — What Rasmus Is Capable Of

Rasmus is the **operator-facing performance and alert reporting front-end** for Noora:

- **Single-device / single-interface performance lookups**, on demand or via saved personal/org-wide shortcuts (Run Device Report).
- **Group-level performance reporting** with fine-grained configurability — time frame, Top N limiting, graph/summary/table/alarm inclusion — and the ability to persist a report's configuration for reuse (Run Group Report + My Group Reports).
- **Alert/alarm reporting**, filterable by severity and alert state, over a selectable time window, surfaced in a detailed, color-coded results table (Run Alert Report).
- A **personal report library** so recurring reports don't need to be reconfigured each time.

Compared to the Noora Health Portal (which is about pipeline health, provisioning internals, and raw data/config access for admins), Rasmus is squarely aimed at the **day-to-day operator workflow**: "how is this device/group performing right now, and what's currently alarming."
