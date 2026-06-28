# Tazama — Alert Navigator & Rule Trigger Discovery

**Delta Analytics Fellowship | April – June 2026**
**Supported by the Bill & Melinda Gates Foundation | Linux Foundation Project**

---

## About Delta Analytics

[Delta Analytics](http://www.deltanalytics.org/) is a non-profit organisation that builds data capacity in the social sector by connecting data science volunteers with mission-driven organisations. This project was completed as part of the Delta Analytics Fellowship programme, contributing technical expertise to support Tazama's open-source financial crime detection platform.

**Fellow:** Shikha Pamnani — BI Strategy Lead

---

## About Tazama

[Tazama](https://tazama.org/) is an open-source, real-time transaction monitoring system designed to detect financial crime — including fraud and money laundering — in digital payment ecosystems. Built under the Linux Foundation and supported by the Bill & Melinda Gates Foundation, Tazama is designed to serve financial inclusion goals by making AML/fraud detection accessible to payment systems in low- and middle-income countries.

Tazama evaluates every financial transaction against a network of rules and typologies to produce a risk score. Transactions that breach configured thresholds are flagged as alerts and routed to a Case Management System (CMS) for investigation by compliance officers.

---

## Project Background

Tazama's evaluation engine assesses each transaction against up to 33 rules across 31 typologies. When a transaction is flagged, a compliance investigator needs to understand **which rules triggered, why they triggered, and what the underlying transaction data looked like** at the time of evaluation.

This notebook was built to answer that question — allowing investigators to immediately identify the specific behavior (constituent transactions or metadata) that triggered each rule, eliminating the need to manually search through a participant's entire transaction history.

---

## Project Goals

- Build an interactive **Alert Navigator** that allows investigators to select any alert and explore the rules that triggered it
- Build a **Rule Trigger Discovery** tool covering 15 rules across two delivery phases
- Surface **EFRUP status** (override / block / none) for each alert — the external fraud rule that can override typology scoring
- Integrate directly with the Tazama **Data Lakehouse** (Apache Hudi on S3) using PySpark
- Provide a clean, investigator-friendly visualization using Plotly and Jupyter Widgets
- Establish a reusable framework that can be extended to cover all 33 Tazama rules

---

## Scope

### Phase 1 — Six Core Rules

| Rule | Description |
|------|-------------|
| Rule-030 | Transfer to Unfamiliar Creditor — counts prior transactions between debtor-creditor pair |
| Rule-044 | Debtor Lifetime Transaction Count — counts all prior debtor transactions |
| Rule-045 | Creditor Lifetime Transaction Count — counts all prior creditor transactions |
| Rule-002 | Debtor Transaction Convergence — counts debtor transactions in a 24h window |
| Rule-016 | Creditor Transaction Convergence — counts creditor transactions in a 24h window |
| Rule-017 | Debtor Transaction Divergence — counts debtor transactions in an 8h window |

### Phase 2 — Nine Additional Rules

| Rule | Description |
|------|-------------|
| Rule-001 | Creditor Account Age — time since first ever creditor transaction |
| Rule-003 | Time Since Most Recent Creditor Activity — dormancy gap for creditor account |
| Rule-004 | Time Since Most Recent Debtor Activity — dormancy gap for debtor account |
| Rule-028 | Age Classification Debtor — debtor age in years from date of birth |
| Rule-076 | Time Since Last Transaction Debtor — gap between consecutive debtor transactions |
| Rule-078 | Transaction Type — purpose code classification from pacs008 |
| Rule-083 | Multiple Accounts Associated with Debtor — distinct accounts per debtor identity |
| Rule-084 | Multiple Accounts Associated with Creditor — distinct accounts per creditor identity |
| Rule-091 | Transaction Amount vs Regulatory Threshold — amount vs configured limit |

---

## Data Architecture

The notebook reads from the Tazama **Data Lakehouse** — a set of Apache Hudi tables on S3, accessed via PySpark on JupyterLab (Server C). Eight tables are loaded at startup in Cell 4 and reused throughout.

| Table | Layer | Purpose |
|-------|-------|---------|
| `gold/evaluation` | Gold | Primary source — alert dropdown, debtor/creditor IDs, transaction amount, report status |
| `bronze/alerts` | Bronze | Full evaluation result — all rule results, weights, independent variables, typology scores, EFRUP status |
| `gold/pacs002` | Gold | Transaction history for rule query replay |
| `gold/pacs008` | Gold | Transaction details for Phase 2 rules (purpose code) |
| `bronze/pacs008` | Bronze | Raw ISO 20022 message — date of birth extraction for Rule-028 |
| `gold/rule` | Gold | Rule configuration — band thresholds, max query range, rule descriptions |
| `gold/account_holder` | Gold | Account-identity relationships for Rules 083 and 084 |

**Key architectural decisions:**
- `gold/evaluation` is the primary source for alert metadata — has correctly populated debtor/creditor IDs and transaction amounts. No pacs002 join needed for IDs.
- `bronze/alerts` is joined via `evaluationID` field inside `alert_data` JSON to retrieve full evaluation results including rule weights, IVs, and typology scores.
- `bronze/pacs008` is used instead of `gold/pacs008` for Rule-028 because `BirthDt` is only available in the raw ISO 20022 JSON — not extracted into the gold layer.
- `gold/account_holder` is pre-loaded in Cell 4 and reused across Rule-083 and Rule-084 — not reloaded inside rule functions.

---

## How the Notebook Works

### 1. Alert Navigator

The notebook opens with a **text filter box and searchable alert dropdown** populated from the `gold/evaluation` table. The investigator can type to filter alerts by tenant, report status, amount, or date. Each alert label shows:

```
TAZAMA | NALT | 2026-05-21 06:25:50 | XTS 1,000.00 | 019e4937...
```

The investigator selects an alert to begin investigation.

### 2. Alert Metadata

Once an alert is selected, the notebook displays:
- Evaluation ID, Transaction ID, Timestamp
- Transaction type and status
- Transaction amount and currency
- Report status (ALRT = alerting, NALT = non-alerting)
- Number of typologies evaluated
- Tenant

### 3. EFRUP Status

Immediately after alert metadata, the notebook displays the **EFRUP (External Fraud Rule Universal Processor) status** for the alert. EFRUP is a control rule that fires before typology scoring and can override the entire evaluation. Three possible values:

| Status | Meaning |
|--------|---------|
| `none` | No external signal — proceed with normal typology evaluation |
| `override` | External signal overrides typology scores — flag this transaction |
| `block` | External signal blocks this transaction immediately |

EFRUP status is read from `bronze/alerts` via the `evaluationID` join.

### 4. Typologies Evaluated

The notebook reads the full evaluation result from `bronze/alerts` via the `evaluationID` join key and displays all typologies evaluated for this transaction, sorted by typology score descending. Each typology shows:
- Typology configuration ID
- **Typology score**
- Alert threshold and interdiction threshold
- Whether the threshold was breached

### 5. Rules Triggered

All rules that fired during the evaluation are displayed with their actual weight, independent variable (recomputed using RULE_QUERIES where implemented), and sub-rule reference — read from `bronze/alerts`. The rule dropdown is filtered to show only:
- Rules with weight > 0
- Rules implemented in the RULE_QUERIES dictionary

### 6. Rule Trigger Discovery

The investigator selects a rule from the dropdown and clicks **Run Rule Discovery**. The notebook:
1. Reads debtor/creditor account IDs directly from `gold/evaluation`
2. Replays the rule query against `gold/pacs002`, `gold/pacs008`, `bronze/pacs008`, or `gold/account_holder` depending on the rule
3. Displays the independent variable with a human-readable label
4. Shows the rule band thresholds from `gold/rule`
5. Renders the matching transactions in an interactive Plotly table

---

## Technical Stack

| Component | Technology |
|-----------|-----------|
| Notebook environment | JupyterLab on Tazama Server C |
| Data processing | PySpark 3.4.2 with Apache Hudi |
| Storage | Apache Hudi Data Lakehouse on S3 |
| Visualization | Plotly, Jupyter Widgets (ipywidgets) |
| Rendering | Voilà (v0.5.11) |
| Language | Python 3 |

---

## Known Limitations

- **Synthetic data placeholders** — the current TAZAMA tenant test data uses placeholder values (`__DBTR_ID__`, `DBTR_DOB`) for some identity fields. Rule-028 (Age Classification) returns an exit condition until real date of birth data is available.
- **Bronze layer dependency** — the full evaluation result (rule weights, independent variables) is read from `bronze/alerts`. This is a temporary workaround — a dedicated gold layer evaluation results table is planned for a future build.
- **15 of 33 rules implemented** — this notebook covers 15 rules across Phases 1 and 2. The remaining 18 rules can be added by extending Cell 9 (rule query functions) and the four dictionaries at the bottom of that cell.
- **Rules grouped by typology** — currently all triggered rules are displayed in a single flat table. Grouping rules per typology with expandable sections is planned for the next iteration.
- **Sub-rule ref reasons** — band descriptions are not available for sub-rule refs that fall outside the configured bands in `gold/rule` (e.g. Rules 008, 076, 045). This is a configuration gap in the source data, not a notebook issue.

---

## Repository Structure

```
Rule_Discovery.ipynb    — Main notebook (11 cells)
README.md               — This file
```

### Notebook Cell Structure

| Cell | Purpose |
|------|---------|
| Cell 1 | Imports |
| Cell 2 | SparkSession configuration |
| Cell 3 | Lakehouse paths — all 8 table paths defined |
| Cell 4 | Load all Hudi tables — evaluation, bronze_alerts, pacs002, pacs008, bronze_pacs008, rules, account_holder |
| Cell 5 | Standard column definitions for pacs002 and pacs008 |
| Cell 6 | Alert input extraction, bronze/alerts parsing, EFRUP extraction functions |
| Cell 7 | Rule configuration helpers (bands, max query range, descriptions) |
| Cell 8 | Timestamp utility function |
| Cell 9 | Rule query functions and dispatch dictionaries (RULE_QUERIES, RULE_DESCRIPTIONS, RULE_DISPLAY_LABELS, RULE_SHORT_NAMES) |
| Cell 10 | Visualization function — displays IV, band thresholds, transaction table |
| Cell 11 | Alert Navigator widget — filter box, dropdown, metadata, EFRUP, typologies, rules, rule discovery |

---

## Extending the Notebook

To add a new rule:

1. Add a query function `query_rule_XXX(inputs)` to **Cell 9** — it must return a tuple `(df, iv_value, iv_label)`
2. Add the rule ID to `RULE_QUERIES` dict
3. Add a description to `RULE_DESCRIPTIONS` dict
4. Add a display label to `RULE_DISPLAY_LABELS` dict
5. Add a short name to `RULE_SHORT_NAMES` dict

No changes to Cells 10 or 11 are needed — the framework handles new rules automatically.

---

## Acknowledgements

This project was made possible through the support of:
- **Justus Ortlepp** — Head of Product, Tazama
- **Sandy Labuschagne** — Data Architecture, Paysys Labs
- **Jai** — Delta Analytics Fellowship Sponsor
- The Tazama and Paysys Labs engineering teams for sandbox access, data ingestion, and technical guidance

---

*Tazama is a Linux Foundation project. This notebook was developed as part of the Delta Analytics Fellowship programme and is intended as a contribution to the Tazama open-source ecosystem.*
