# 🔄 Power Automate — BPG Audit: Automatic Non-Compliance Creation

> Automation built to eliminate a recurring manual audit process — comparing deliveries registered in the project management system (Azure DevOps) against deliveries tracked in the performance dashboard (Power BI/BPG) and automatically creating **Non Compliance** work items for responsible managers on divergent items.

> 📖 [Leia em Português](README.md)

---

## 🖼️ Flow Overview

![Flow Diagram](assets/flow-diagram.png)

---

## 💡 Context and Motivation

During monthly audit cycles, it was necessary to manually identify which User Stories delivered in Azure DevOps were not reflected in the BPG (Business Performance Guide — Power BI dashboard), and vice versa. This process was time-consuming and prone to human error.

This automation solves the problem end-to-end: it identifies divergences, creates audit cards, generates NCs per responsible manager, and updates existing cards if they already exist — **with zero manual interaction required**.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│              TRIGGER: Monthly Recurrence                 │
└────────────────────────┬────────────────────────────────┘
                         │
            ┌────────────▼────────────┐
            │   Dynamic Month Calc     │
            │  (month, quarter, year)  │
            └────────────┬────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
   ┌──────▼──────┐              ┌───────▼──────┐
   │  BPG Scope   │              │  VSTS Scope  │
   │ (Power BI)   │              │(Azure DevOps)│
   │  DAX Query   │              │ WIQL Query   │
   └──────┬───────┘              └──────┬───────┘
          │                             │
          └──────────┬──────────────────┘
                     │
        ┌────────────▼────────────┐
        │      ID Comparison      │
        │      BPG ≠ VSTS?        │
        └──────┬──────────────────┘
               │ Yes (divergences found)
    ┌──────────▼──────────┐
    │    Audit Card Scope  │
    │  Find or create the  │
    │  month's audit card  │
    └──────────┬───────────┘
               │
    ┌──────────▼──────────────┐
    │  Fetch leader list       │
    │  (SharePoint → JSON)     │
    └──────────┬───────────────┘
               │
    ┌──────────▼───────────────────┐
    │  Fetch divergent card details │
    │  + manager per assignee       │
    └──────────┬────────────────────┘
               │
    ┌──────────▼───────────────────┐
    │  Per unique manager:          │
    │  • NC exists? → Update it    │
    │  • NC missing? → Create it   │
    └──────────────────────────────┘
```

---

## 🔌 Connectors Used

| Connector | Purpose |
|---|---|
| **Power BI** | Runs DAX query on the performance dataset to retrieve BPG deliveries |
| **Azure DevOps (VSTS)** | Queries delivered User Stories via WIQL, fetches work item details, creates and updates work items |
| **SharePoint Online** | Retrieves result center list containing assignee and manager email mappings |

---

## 📋 Prerequisites

Before importing and configuring this flow in your environment, you need:

**Power Automate Connections:**
- Active **Power BI** connection authenticated with a user who has access to the workspace and dataset
- Active **Azure DevOps** connection with read and write permissions on the involved projects
- Active **SharePoint Online** connection with access to the result center list

**Azure DevOps Structure:**
- A source project with User Stories (e.g., engineering team)
- A management project with `Audit` and `Non compliance` work item types
- Custom fields configured (delivery value flag, integration project field)

**SharePoint Structure:**
- A list with a JSON column containing the assignee matrix: `email`, `managerEmail`, `resultCenterName`, `businessUnitName`

**Power BI Structure:**
- Dataset with a `fact_workitem` table containing: `code_source`, `code_target`, `status`, `closed_date`, `created_date`, `target_date`, `title`, `user_account`
- Dimension table `dim_result_center_process` with `ecosystem`, `business_unit_name`, `set_process`
- Local date table with `Year`, `Quarter`, `Month` fields

---

## 🔧 Configuration Guide

### Step 1 — Replace the placeholders

Open `flow_sanitized.json` and replace all `YOUR_*` values according to the table below:

| Placeholder | Description | Example |
|---|---|---|
| `YOUR_DEVOPS_ORG` | Azure DevOps organization name | `mycompany` |
| `YOUR_SOURCE_PROJECT` | Source project for User Stories | `MyProduct` |
| `YOUR_MANAGEMENT_PROJECT` | Project where Audit and NC items live | `PerformanceManagement` |
| `YOUR_POWERBI_WORKSPACE_GROUP_ID` | Power BI workspace GUID | `xxxxxxxx-xxxx-...` |
| `YOUR_POWERBI_DATASET_ID` | Power BI dataset GUID | `xxxxxxxx-xxxx-...` |
| `YOUR_TENANT.sharepoint.com` | SharePoint tenant URL | `mycompany.sharepoint.com` |
| `YOUR_SHAREPOINT_SITE` | SharePoint site name | `Performance` |
| `YOUR_RESULT_CENTER_LIST` | SharePoint list name | `ResultCenters` |
| `YOUR_AREA_PATH` | Azure DevOps area path | `Directorate\Engineering` |
| `YOUR_PROJECT_NAME` | Project name used in card titles | `MyProduct` |
| `YOUR_DELIVERY_FLAG` | Custom field flagging value deliveries | `Custom.DELIVERY_FLAG` |
| `YOUR_PROJECT_FIELD` | Custom field with project classification | `Custom.PROJECT_TYPE` |
| `YOUR_PROJECT_1..N` | Project field values to include in WIQL | `'PROJ_EVOLUTION'` |
| `YOUR_EXCLUDED_AREA_1..N` | Area paths to exclude from WIQL | `'MyProduct\\Analysis'` |
| `YOUR_RESULT_CENTER_1..N` | Valid result center names | `'RC_ENGINEERING'` |
| `YOUR_NC_TITLE_V1` / `YOUR_NC_TITLE_V2` | NC card titles to search for / create | `'Are VSTS deliveries equal to BPG?'` |
| `YOUR_BU_FIELD` | Custom BU field in Azure DevOps | `BU_FIELD` |
| `YOUR_BUSINESS_UNIT_VALUE` | BU value for Audit cards | `MyBU` |
| `YOUR_TEAM_TYPE` | Audit type for the Type field | `Engineering` |
| `YOUR_ECOSYSTEM` | Ecosystem value in DAX filter | `E-COMMERCE` |
| `YOUR_THROUGHPUT_MEASURE` | Throughput measure in Power BI model | `m_average_throughput` |
| `YOUR_THROUGHPUT_TABLE` | Throughput selection table | `select_throughput` |
| `YOUR_DATE_TABLE_ID` | Date table name in Power BI model | `LocalDateTable_xxxx` |
| `YOUR_TIMEZONE` | Trigger timezone | `E. South America Standard Time` |
| `YOUR_ENVIRONMENT_ID` | Power Platform environment ID | `xxxxxxxx-xxxx-...` |

### Step 2 — Configure connections

When importing the flow into Power Automate:

1. Go to **Power Automate → My Flows → Import**
2. Select the JSON file
3. For each connection shown (`shared_powerbi`, `shared_visualstudioteamservices`, `shared_sharepointonline`), select or create the corresponding connection in your environment

### Step 3 — Validate the SharePoint list

The result center list must have an item with a `JSON` column containing an array in this format:

```json
[
  {
    "email": "assignee@company.com",
    "managerEmail": "manager@company.com",
    "resultCenterName": "RC_NAME",
    "businessUnitName": "BU_NAME"
  }
]
```

### Step 4 — Validate custom fields in Azure DevOps

Ensure the `Audit` and `Non compliance` work item types exist in the management project and that all referenced custom fields are configured in the process (via **Organization Settings → Processes**).

### Step 5 — Configure the Power BI dataset

The DAX query in the `BPG` scope assumes the semantic model contains the tables and measures listed in the prerequisites. Adjust table and measure names to match your model before enabling the flow.

### Step 6 — Activate and test

1. Save and enable the flow
2. Run it manually once by clicking **Run** to validate behavior
3. Monitor the execution in the Power Automate run history panel

---

## 🔁 Detailed Execution Logic

### 1. Trigger and Variable Initialization

The flow fires monthly. On startup, the following variables are initialized:

- `varTotalDeItens` — integer for general counting
- `varItensFaltandoBPG` — array of IDs present in VSTS but missing from BPG
- `varItensFaltandoVSTS` — array of IDs present in BPG but missing from VSTS
- `varItensDetalhados` — array of objects with details for each divergent card
- `varManagersUnicos` — array of unique manager emails
- `varAuditCardId` — ID of the month's audit card (found or created)

### 2. Dynamic Month Calculation

The `Escopo_-_calculo_de_mes_dinamico` scope dynamically calculates the previous month in three formats: full month name in Portuguese, quarter (e.g., `Trim 1`), and year as integer. These values are used as filters in both the Power BI query and Azure DevOps searches.

### 3. BPG Query (Power BI)

The `BPG` scope runs a DAX query on the configured dataset filtering by ecosystem, business unit, and the calculated month/quarter/year. Results are processed to extract only the numeric ID from each item (separated by `:` in the `code_source` field), filtering out subtotal rows (`IsGrandTotalRowTotal = false`).

### 4. VSTS Query (Azure DevOps)

The `VSTS` scope runs a WIQL query on the source project for User Stories not in `Removed` state, with the delivery value flag active, closed in the previous month, belonging to configured integration projects, and excluding specific area paths.

### 5. Comparison and Divergence Detection

The flow first checks if totals are already equal — if so, it terminates successfully with no further processing. Otherwise, it iterates over both ID sets and populates the missing-item arrays for each system.

### 6. Audit Card

Searches for an existing `Audit` card for the current month. If found, reuses the ID; if not, creates a new one titled `BPG Audit - [PROJECT] - [MONTH_YEAR]`. If no divergences are found after comparison, the flow terminates. Otherwise, it continues to NC creation.

### 7. Data Enrichment

For each divergent ID, fetches full work item details from Azure DevOps and cross-references with the SharePoint list to identify the manager's email for the assignee of each card.

### 8. NC Creation/Update per Manager

For each unique manager identified:
- Searches for an existing open NC linked to the month's audit card
- **If none exists:** creates a new `Non compliance` work item with an HTML table of divergent cards, hierarchically linked to the audit card and assigned to the manager
- **If one exists:** updates the description of the existing card with the refreshed divergence list

---

## 📁 Repository Structure

```
.
├── flow_sanitized.json   # Flow definition with sensitive data removed
├── README.md             # Documentation (Portuguese)
├── README.en.md          # Documentation (English)
├── .gitignore            # Prevents accidental sensitive data commits
└── assets/
    └── flow-diagram.png  # Visual screenshot of the flow in Power Automate
```

---

## 🧠 Key Techniques Demonstrated

- **Data-driven automation** — no manual intervention throughout the monthly cycle
- **Cross-system comparison** — data cross-referencing between Power BI (DAX) and Azure DevOps (WIQL)
- **Idempotency** — checks for existing records before creating, preventing duplicates
- **Dynamic grouping** — NCs generated per unique manager, consolidating multiple cards per responsible
- **Dynamic temporal calculation** — period always computed relative to execution time
- **Runtime lookup enrichment** — manager data fetched from SharePoint and cross-referenced at execution time
- **Dynamic HTML generation** — styled tables programmatically generated for work item descriptions

---

## ⚠️ Notes

- The flow uses serial concurrency (`repetitions: 1`) on main loops to ensure consistency in shared variables
- The DAX query uses `TOPN(502, ...)` as an upper limit — adjust based on expected monthly delivery volume
- The `Vsts_AssignedToEmail` field is specific to the Azure DevOps connector for Power Automate; verify the exact field name in your environment
