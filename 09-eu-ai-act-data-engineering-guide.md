# EU AI Act Article 10: The Data Engineering Compliance Guide Nobody Wrote

**Author:** [Arjunkumar Kansagara](https://ak-dataweda.github.io/Portfolio/) — Data Engineer & AI-Ready Data Infrastructure Consultant, Germany  
**Last updated:** April 2026  
**Reading time:** ~25 minutes

> Lawyers, researchers, and policy teams dominate EU AI Act discourse. This guide is for the people who actually have to **build** the systems that make high-risk AI defensible: data engineers, analytics engineers, and platform teams.

---

## Table of Contents

1. [Why This Guide Exists](#why-this-guide-exists)
2. [EU AI Act Timeline & What's at Stake](#eu-ai-act-timeline--whats-at-stake)
3. [Article 10 — The Data Engineer's Scene](#article-10--the-data-engineers-scene)
4. [Mapping Article 10 to Engineering Work](#mapping-article-10-to-engineering-work)
5. [Architecture: An Article 10-Compliant Data Pipeline](#architecture-an-article-10-compliant-data-pipeline)
6. [Quality Gates & Pseudocode](#quality-gates--pseudocode)
7. [Metric Governance Is Now a Legal Requirement](#metric-governance-is-now-a-legal-requirement)
8. [The Semantic Layer as Compliance Infrastructure](#the-semantic-layer-as-compliance-infrastructure)
9. [Agentic AI Needs a Trust Layer](#agentic-ai-needs-a-trust-layer)
10. [The €35M Audit — What Regulators Will Check](#the-35m-audit--what-regulators-will-check)
11. [German Manufacturing: The Biggest Compliance Market](#german-manufacturing-the-biggest-compliance-market)
12. [90-Day Compliance Playbook for Data Teams](#90-day-compliance-playbook-for-data-teams)
13. [I Audited My Own Platforms Against Article 10](#i-audited-my-own-platforms-against-article-10)
14. [Article 10 Data Readiness Checklist](#article-10-data-readiness-checklist)
15. [From Data Engineer to Data Trust Engineer](#from-data-engineer-to-data-trust-engineer)
16. [About the Author](#about-the-author)

---

## Why This Guide Exists

When I first read Article 10 of the EU AI Act, my instinct was to skim it and forward it to legal. That was a mistake.

Article 10 is not a privacy addendum or a vague "ethics" clause. It is a **specification for training, validation, and testing data** for high-risk AI systems. If you are the person who owns ingestion, transformation, quality, and consumption layers, you are on the hook for making those requirements true in production — not just "documented somewhere."

Here's the problem: **nobody is writing about the EU AI Act from the data engineering perspective.** Lawyers write about it. AI researchers write about it. Policy people write about it. But nobody is saying "here's how to actually BUILD the data infrastructure that makes your AI compliant."

This guide fills that gap.

---

## EU AI Act Timeline & What's at Stake

### The Deadline

The EU AI Act entered into force on August 1, 2024. The main obligations for high-risk AI systems apply from **August 2, 2026**. That's not an abstract date on a slide deck — it's when companies placing high-risk AI on the EU market need to demonstrate compliance.

### The Fines

| Violation Category | Maximum Fine |
|-------------------|-------------|
| Prohibited AI practices | €35M or 7% of global annual turnover |
| High-risk AI obligations (incl. Article 10) | €15M or 3% of global annual turnover |
| Providing incorrect information to authorities | €7.5M or 1% of global annual turnover |

These are not rounding errors for any company shipping AI into the EU.

### Who Is Affected

- **Providers** placing AI systems on the EU market
- **Deployers** using AI systems in the EU
- **Anyone whose data work enables those systems** — that's us

### The Blind Spot

What I'm seeing across DACH:

- Teams are racing on model cards, guardrails, and vendor demos
- Almost nobody is hardening the **data layer** the model actually learned from
- Lineage, representativeness, error baselines — still "Phase 2"

The data stack is the blind spot. Models get the spotlight. Pipelines get ignored until an auditor asks a question nobody can answer in one sentence.

---

## Article 10 — The Data Engineer's Scene

Article 10 is the data engineer's scene. It requires that training, validation, and testing data sets for high-risk AI are:

- **Relevant and representative** of the intended purpose and target population
- **Free of errors** and **complete** to the extent appropriate for the intended purpose
- **Statistically sound** where biases could affect health, safety, or fundamental rights — with possible need for bias identification and mitigation
- **Documented** with regard to **origin**, **preparation**, **labelling** (if applicable), **assumptions**, **limitations**, and **gaps**

The legal text uses words like "appropriate" and "to the extent possible." In practice, regulators and notified bodies will ask: **What did you do, how did you prove it, and can you reproduce it?**

### Article 10 Is Not GDPR

A nuance I stress in workshops: Article 10 is **not the same problem as GDPR data minimization**, though teams confuse them.

| | GDPR | EU AI Act Article 10 |
|---|------|---------------------|
| **Core question** | Should you hold this personal data? | Is the data you used **fit for purpose**? |
| **Focus** | Privacy and consent | Quality, representativeness, and bias |
| **Scope** | Personal data | All data (including non-personal: machine signals, images, logs) |
| **What you need** | Legal basis, consent, minimization | Governance, lineage, documentation |
| **Skill required** | Legal/DPO | Data Engineering + Governance |

**GDPR asked: are you protecting data? The EU AI Act asks: is your data even true?**

Different problem. Different skill set.

---

## Mapping Article 10 to Engineering Work

Each Article 10 requirement maps directly to data engineering work:

| Legal Requirement | Data Engineering Translation |
|------------------|------------------------------|
| Relevant & representative | Sampling design, cohort definitions, drift monitoring, domain coverage in the warehouse |
| Free of errors & complete | Validation rules, anomaly detection, reconciliation to source systems, null/duplicate policies |
| Statistical soundness & bias | Fairness metrics on slices, stratified evaluation sets, documented trade-offs |
| Documentation of origin | Lineage from source systems to model inputs |
| Documentation of preparation | Versioned SQL/dbt, transformation code in version control |
| Documentation of assumptions | Written assumptions signed by domain experts |
| Documentation of gaps | Known limitations, missing dimensions, temporal holes |
| Labelling (if applicable) | Label provenance, inter-rater agreement, guideline versioning |

**If your ML team trains models in notebooks while the "real" business data lives in Snowflake or BigQuery with a different definition of "customer" or "defect," you already have a compliance gap** — not because the model is bad, but because the evidentiary chain is broken.

### Labelling Deserves Special Attention

Many high-risk systems in industry are supervised — humans mark defects, approve loans, classify documents. From an engineering standpoint, labels are **data products** with their own error rates, inter-rater disagreement, rework rules, and temporal drift ("we tightened the standard in 2024").

If your pipeline treats labels as ground truth without provenance, you are one uncomfortable question away from a crisis: *Who labeled this, with which instructions, and did those instructions change?*

---

## Architecture: An Article 10-Compliant Data Pipeline

If I had to defend my training data in front of a regulator tomorrow, this is the architecture I would not want to be without.

**Design principle:** Separate analytical flexibility from regulated exports. Analysts can explore freely in a sandbox; only named, tested, and versioned artifacts cross into model training. That boundary is your compliance firewall.

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Sources     │────▶│ Ingestion    │────▶│ Raw / Landing   │
│ (ERP, MES,  │     │ (CDC, batch, │     │ (immutable)     │
│  IoT, CRM)  │     │  events)     │     └────────┬────────┘
└─────────────┘     └──────────────┘              │
                                                  ▼
                                        ┌─────────────────────┐
                                        │ Integration &       │
                                        │ Conformed Dimensions│
                                        │ (dbt, SQL)          │
                                        └──────────┬──────────┘
                                                   │
                     ┌─────────────────────────────┼─────────────────────────────┐
                     ▼                             ▼                             ▼
            ┌────────────────┐           ┌────────────────┐            ┌────────────────┐
            │ Quality Gates  │           │ Lineage &      │            │ Semantic /     │
            │ (GE, dbt test, │           │ Catalog        │            │ Metrics Layer  │
            │  observability)│           │ (DataHub,      │            │ (dbt Metrics,  │
            └───────┬────────┘           │  OpenLineage)  │            │  Cube.dev)     │
                    │                    └────────┬───────┘            └────────┬───────┘
                    │                             │                            │
                    └─────────────────────────────┴────────────────────────────┘
                                                   │
                                                   ▼
                                        ┌─────────────────────┐
                                        │ Model-Ready Exports │
                                        │ (versioned snapshots│
                                        │  + manifest)        │
                                        └──────────┬──────────┘
                                                   │
                                                   ▼
                                        ┌─────────────────────┐
                                        │ Training / Eval in  │
                                        │ ML Platform         │
                                        └─────────────────────┘
```

### How Each Layer Maps to Article 10

| Pipeline Layer | Article 10 Requirement |
|---------------|----------------------|
| Sources → Ingestion | **Origin** evidence starts here — which systems, which contracts, which refresh SLAs |
| Integration (dbt/SQL) | **Preparation** is code. Assumptions become versioned and reviewable |
| Quality Gates | **Errors, completeness, statistical soundness** — automated, logged, retained |
| Lineage & Catalog | **Traceability** from business definition to exported feature/label tables |
| Semantic / Metrics Layer | **Definitions** that match what the business thinks "defect rate" means |
| Model-Ready Exports | Frozen snapshots with **hashes and manifests** — "what we trained on" is unambiguous |

### Tooling I Would Actually Pick

I work across all four major cloud platforms. The compliance story is identical regardless of vendor — the shape matters more than the logo.

| Layer | GCP | AWS | Azure | Cross-Cloud |
|-------|-----|-----|-------|-------------|
| **Warehouse / Lakehouse** | BigQuery | Redshift, S3 + Athena | Azure Synapse, Azure Databricks | Snowflake (runs on all three) |
| **Ingestion / Orchestration** | Cloud Composer (Airflow) | AWS Glue, Step Functions, MWAA | Azure Data Factory, Azure Databricks Workflows | Airflow, Dagster |
| **Transform** | dbt on BigQuery | dbt on Redshift/Glue | dbt on Databricks/Synapse | dbt (cloud-agnostic) |
| **Quality** | dbt tests + GE | dbt tests + GE, AWS Glue DQ | dbt tests + GE, Azure Databricks expectations | Monte Carlo, Metaplane, Soda |
| **Lineage & Catalog** | Dataplex, OpenLineage | AWS Glue Data Catalog, OpenLineage | Microsoft Purview, Unity Catalog | DataHub, OpenLineage |
| **Semantic Layer** | dbt Metrics, Cube.dev | dbt Metrics, Cube.dev | dbt Metrics, Cube.dev | Cube.dev (cloud-agnostic) |
| **Storage (exports)** | GCS | S3 | ADLS Gen2 | Versioned snapshots + manifests on any object store |

**How I choose:** I pick based on the client's existing enterprise contracts, latency to source systems, and ecosystem maturity (e.g., OpenLineage support, orchestrator integration). Most enterprises are multi-cloud already — a manufacturing client might land OT data in Azure via Data Factory, transform in Databricks, and serve analytics from Snowflake. The Article 10 compliance architecture stays the same: time-travel or snapshot tables, object storage exports with checksums, and IAM that proves who could alter what.

**Key principle:** The pipeline pattern is cloud-agnostic. What matters is that every layer produces **auditable artifacts** — lineage, test results, manifests, and versioned definitions — regardless of whether they run on GCP, AWS, or Azure.

---

## Quality Gates & Pseudocode

Treat the handoff to ML like a production deploy. Quality gates should be **blocking**, not warnings in a log nobody reads.

### Conceptual Quality Gate Bundle

```text
# Quality gate for model-ready data export (not framework-specific)

# 1. COHORT INTEGRITY
assert cohort.definition_version == "v2025-03-01"
assert row_count(train) + row_count(val) + row_count(test) == row_count(base_cohort)

# 2. COMPLETENESS
assert max(null_rate(train.feature_x) for feature_x in NUMERIC_FEATURES) < 0.02
assert missing_label_rate(train) < 0.001

# 3. REPRESENTATIVENESS
for slice in PROTECTED_SLICES:
    assert min_rows(val, slice) >= SLA_MIN_ROWS
    record_metric("prevalence", slice, value=prevalence(val, slice))

# 4. DRIFT DETECTION
if drift_score(train, prod_snapshot) > DRIFT_THRESHOLD:
    block_or_flag("training_data_drift", evidence=drift_report)

# 5. LABEL INTEGRITY
assert label_cardinality(train) == EXPECTED_CLASSES
assert inter_rater_agreement >= MIN_KAPPA

# 6. MANIFEST GENERATION
write_manifest(
    dataset_version = generate_version_id(),
    git_sha = current_commit(),
    warehouse_snapshot = snapshot_timestamp(),
    row_counts = {train, val, test},
    quality_report = test_results,
    owner = cohort_owner()
)
```

You would implement this with Great Expectations, dbt tests, custom SQL assertions, or data observability tools — the Act does not prescribe tooling; it prescribes **outcomes**.

### The Quality Scorecard

I recommend a "quality scorecard" artifact per snapshot: a single JSON or table row stored next to the export that records pass/fail counts, thresholds, who ran the job, and links to logs. Auditors love narrative documents, but engineers win trust faster when narrative is backed by machine-generated receipts.

---

## Metric Governance Is Now a Legal Requirement

Before the EU AI Act, metric governance was sold as culture and efficiency: fewer arguments in meetings, cleaner dashboards, faster decisions. Those benefits are real. But they were still optional.

**That era is ending for high-risk AI.**

Article 10(2) effectively says: you must be able to explain **how** data was chosen, **how** it was prepared, what you **assumed**, and where the **gaps** are. That is metric and data-definition governance, written into law.

### What Changed

"Documented assumptions, preparation processes, and gap identification" sounds legal. In a warehouse, it translates to:

- **Canonical definitions** for labels and features (not five conflicting SQL snippets)
- **Change control** when a metric definition shifts (and awareness of downstream impact on models)
- **Explicit acknowledgment** of missing dimensions, historical holes, or proxy labels

If your organization cannot answer "what is an active customer this quarter, and who approved that definition?" you will struggle to answer regulator-grade questions about representativeness and bias.

### The TRUST Framework

I use TRUST as a shorthand for how I structure governance work with teams:

| TRUST Pillar | What It Is | Article 10(2) Link |
|-------------|-----------|-------------------|
| **T — Traceable** | Every metric/label has a documented path from source to consumption | Preparation processes & origin |
| **R — Reviewed** | Definitions pass business + technical review; changes are versioned | Documented assumptions & accountability |
| **U — Understood Limits** | Known gaps, proxies, and exclusions are explicit | Gap identification |
| **S — Stable Interfaces** | Semantic layer / metrics APIs reduce ad-hoc redefinition | Consistent preparation for training vs. ops |
| **T — Tested** | Automated checks + monitoring on the metrics that matter | Error/completeness & ongoing validity |

### Worked Example

Suppose "OEE" (overall equipment effectiveness) feeds a scheduling recommender. If operations change how planned downtime is classified, the metric moves — silently — unless you have Reviewed change control. If the recommender's training data still uses last quarter's definition while production dashboards use the new one, your model is optimizing a ghost KPI.

TRUST is not paperwork; it is preventing silent divergence between human and machine decisions.

---

## The Semantic Layer as Compliance Infrastructure

The semantic layer was marketed as governance for BI consistency. The subplot, especially under the EU AI Act, is **compliance infrastructure**: a stable, versionable interface between raw tables and every downstream consumer — including models and agents.

### Without vs. With a Semantic Layer

| Without Semantic Layer | With Semantic Layer |
|----------------------|-------------------|
| Definitions live in **dashboards** | Definitions live in **code** |
| ML features **re-specify** joins | ML consumes **approved** views/measures |
| Changes are **invisible** | Changes are **PR-reviewed** |
| Audits are **archaeology** | Audits are **navigation** |

### Generic YAML Examples (Illustrative)

**dbt-style metrics sketch:**

```yaml
metrics:
  - name: defect_rate
    label: "Defect rate (line QA)"
    description: >
      Ratio of units failing visual inspection at Station 3.
      Excludes rework loops within 24h (see assumption log AD-14).
    type: ratio
    type_params:
      numerator: ref('defect_events')
      denominator: ref('units_produced')
    dimensions:
      - site_id
      - product_family
    meta:
      owner: quality_ops@company.example
      regulation_notes: "Used as label source for HR-AI-vision-v2"
```

**Cube-style model sketch:**

```yaml
cubes:
  - name: quality_inspection
    sql_table: prod_mart.quality_inspection_facts
    dimensions:
      - name: site_id
        sql: site_id
        type: string
    measures:
      - name: defect_rate
        sql: "{defect_count} / nullif({inspection_count}, 0)"
        type: number
        title: Defect rate
```

These snippets are not magic compliance buttons. They are **evidence-friendly**: explicit descriptions, owners, dimensions for slice analysis, and hooks for regulatory metadata.

### CI/CD for Meaning

Treat semantic YAML like application code: pull requests, required reviewers, staging environments for breaking changes. When a metric changes, your pipeline should surface downstream consumers — especially ML feature views — so you do not "improve analytics" while silently retraining on a shifted target.

---

## Agentic AI Needs a Trust Layer

Agentic AI is the current hype cycle. Under the hype is a boring truth: an agent is only as safe as the signals it reads and the actions it is allowed to take. If those signals are wrong, stale, or inconsistently defined, autonomy does not multiply value — it multiplies incidents.

### Why Agents Break Faster Than Dashboards

Dashboards embarrass you in a meeting. Agents can trigger workflows: file tickets, change states, send messages, call tools. The feedback loop is tighter and the blast radius larger.

### The Agentic Trust Stack

```
┌──────────────────────────────────────────────────────────────┐
│ Agent (planner + tools)                                      │
└───────────────────────────────────┬──────────────────────────┘
                                    │ tool calls (metrics, search, actions)
                                    ▼
┌──────────────────────────────────────────────────────────────┐
│ Trust Layer                                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ ┌────────┐│
│  │ Semantic    │ │ Quality     │ │ Observability│ │ Policy ││
│  │ (Cube, dbt) │ │ (GE, tests) │ │ (drift, logs)│ │ / RBAC ││
│  └─────────────┘ └─────────────┘ └──────────────┘ └────────┘│
└───────────────────────────────────┬──────────────────────────┘
                                    ▼
┌──────────────────────────────────────────────────────────────┐
│ Warehouse / Lake / Streams + Lineage Catalog                  │
└──────────────────────────────────────────────────────────────┘
```

**The four pillars:**

1. **Semantic Layer (meaning)** — Agents consume approved metrics, not freestyle SQL
2. **Data Quality (correctness)** — Blocking checks on freshness, volume, uniqueness
3. **Observability (behavior over time)** — Drift, anomaly detection, run history
4. **Governance (who may do what)** — RBAC on semantic views, PII policies, tool allow-lists

For high-risk systems, regulators will not care whether your automation was "an agent" or "a script." They will care whether data for training and operation is documented, appropriate, and controlled. A trust layer is how you operationalize those requirements.

---

## The €35M Audit — What Regulators Will Check

Everyone quotes penalty ranges. I am less interested in the ceiling than in the mechanism: competent authorities will ask for **evidence**. The expensive failures will not be "we were evil." They will be "we could not demonstrate what we did with data."

### Generalized Audit Scenario

A high-risk AI system is deployed in an industrial quality context (vision + sensor features). A regulator or notified body requests documentation on training, validation, and testing data under Article 10.

### What They Will Ask

| Auditor Question | Strong Answer |
|-----------------|--------------|
| "Show me the training population." | Cohort SQL + version + row counts + manifest |
| "Prove label quality." | Rater guidelines + disagreement stats + cleaning rules in code |
| "How do you monitor drift?" | Dashboards + alert history + incident tickets |
| "Who approved assumptions?" | Named owners + dated sign-off + change log |
| "What changed between v1.3 and v1.4 in the data layer?" | Git diff + manifest comparison + quality scorecard deltas |

### Common Failure Modes

1. **No lineage** — The model was trained on "a CSV from 2022" with no path back to operational truth
2. **Undocumented assumptions** — "We removed outliers" — how, why, and with what side effects on minority cases?
3. **No bias testing narrative** — Not "no tests," but no coherent story tied to deployment context
4. **Inconsistent metrics** — Finance's "defect" ≠ MES's "defect" ≠ the label in the training table
5. **Tribal knowledge** — Only one data scientist knows how labels were cleaned — and they switched jobs

### The Reconstruction Drill

Pick one production model, pick a random past training date, and try to reconstruct the exact inputs in one working day. If you cannot, you have found your roadmap.

Record the drill like an incident postmortem: time to reconstruct, tools used, missing artifacts, owner. That document becomes your board-ready justification for platform investment.

---

## German Manufacturing: The Biggest Compliance Market

I live and work in Germany. When people talk about AI regulation, they often picture chatbots and foundation models. On the ground in DACH manufacturing, the more urgent story is: **high-risk AI** embedded in inspection, predictive maintenance, scheduling, and automated decisions — systems where bad data is not a bad recommendation but scrap, downtime, or harm.

### Density of High-Risk Use Cases

- Computer vision on the production line
- Models prioritizing maintenance spend
- Systems influencing human oversight in safety-critical workflows
- Quality inspection AI (automated accept/reject decisions)

### The Gap

Industrie 4.0 accelerated sensorization, connectivity, and analytics. It did **not** automatically create:

- Canonical industrial data models across sites
- Label governance for quality decisions
- Cross-OT/IT lineage that survives vendor changes
- Documentation habits that satisfy Article 10-style scrutiny

### The Opportunity

Germany has thousands of manufacturing firms with serious digital footprints — from Mittelstand suppliers to global OEMs. Even if only a fraction deploy high-risk AI in the regulatory sense, that fraction still implies hundreds to thousands of systems needing audit-grade data practices.

**The bottleneck is not willingness to "do AI." It is trustworthy data infrastructure at scale.**

Most manufacturers have excellent engineering culture but zero data governance in the regulatory sense. The opportunity: bring data governance to Industrie 4.0.

---

## 90-Day Compliance Playbook for Data Teams

When a team realizes their high-risk AI story is strong in demos and weak in data evidence, I use a 90-day playbook. The goal is not perfection — it is credible momentum and a repeatable template.

### Roles You Need

- **Engineering lead:** warehouse/pipelines/quality
- **ML lead:** model lifecycle, evaluation protocols
- **Domain owner:** accepts definitions and limitations
- **Legal/risk:** interprets obligations and interfaces with authorities
- **PM / program:** keeps the evidence pack coherent

### Phase 1: Weeks 1-2 — Audit (Inventory Reality)

**Deliverables:** Data map, model inventory, "evidence gap" list

**Activities:**
- Pick one flagship high-risk system
- Trace data from source to model
- Run the one-day reconstruction drill
- Exit with a single-page heat map of risk

### Phase 2: Weeks 3-4 — Gaps (Prioritize Brutally)

**Deliverables:** Prioritized backlog with owners; risk register

**Activities:**
- Define cohort contracts
- List assumptions
- Identify missing lineage
- Standardize train/val/test methodology

**Prioritization rule:** Fix traceability and snapshots before you polish narratives. A beautiful limitations memo on top of mystery exports is still weak evidence.

### Phase 3: Weeks 5-8 — Build (Make Compliance Ordinary)

**Deliverables:** Automated tests, snapshot manifests, semantic definitions, evaluation slice policy

**Activities:**
- Implement blocking quality gates
- Wire OpenLineage (or equivalent)
- Publish limitations memo v1
- Align ML exports to versioned tables

**Cadence:** Two-week sprints with demos showing artifacts (manifest, test report, catalog page), not just "we refactored SQL."

### Phase 4: Weeks 9-12 — Validate (Pretend It's Real)

**Deliverables:** Mock audit pack, remediation tickets, executive summary

**Activities:**
- External-style review (red team)
- Fix documentation holes
- Train support teams on where evidence lives

### Success Metrics

| Metric | Target |
|--------|--------|
| Time-to-reconstruct a historical training snapshot | < 1 business day |
| Model-critical metrics defined in semantic layer | 100% of top N |
| Test pass rate on export jobs (trailing 30 runs) | 100% or explicit waivers |
| Open critical gaps in risk register | Trending down week over week |

### Minimum Viable Compliance (Startups)

- One golden cohort definition in code
- Snapshot + hash per model release
- Ten tests that truly matter
- One-page limitations signed by domain

### Full Compliance (Enterprise)

- Catalog + lineage as default
- Change management for metrics
- Continuous monitoring for drift and freshness
- Formal evaluation governance with retention policies

---

## I Audited My Own Platforms Against Article 10

I preach audit-ready data for clients. So I decided to hold myself to the same standard: I took Article 10, printed the obligations in plain language, and walked my own past projects like a forensic review.

### The Honest Scorecard

| Article 10 Theme | What Was Already Strong | What Needed Work |
|-----------------|----------------------|-----------------|
| **Representativeness** | Clear cohort SQL, documented exclusions | "Production-like" data was convenient rather than demonstrably representative — known skew wasn't written down |
| **Errors & completeness** | dbt tests on keys and not-null fields; reconciliation reports | Label noise treated as "model robustness," not a documented limitation with quantified impact |
| **Bias & statistical soundness** | Some slice evaluations existed; performance by product family | Slices were ad-hoc, not tied to a policy. Ad-hoc looks evasive in hindsight |
| **Documentation** | Version control for transformations; decent README culture | Assumptions lived in tickets and Slack — retrievable, but not packaged as a coherent narrative |

### The Headline Number

If I compress everything into one uncomfortable metric: most production data platforms I have touched were **maybe 60% "compliant in spirit"** — solid engineering, partial paperwork. The missing 40% was not fancy ML; it was discipline around evidence: slice policies, limitation memos, immutable snapshots, and boring sign-offs.

### What I Would Do Differently on Day One

- Bundle assumptions into a living doc linked from the repo root — not "we will write it later"
- Name slice policies before the first serious model review — even if the list is short
- Treat label guidelines as a versioned artifact next to the labeling UI spec
- Run the reconstruction drill at milestone zero, not when a client asks a scary question

**Your platform is probably closer than you think — and the last mile is not more Kubernetes, it is more discipline around what you already built.**

---

## Article 10 Data Readiness Checklist

Use this to self-assess your readiness. If you can't answer "yes" to most of these, your AI isn't compliant.

### Data Lineage & Origin

- [ ] Can you trace training data from source system to model input?
- [ ] Is the path automated, versioned, and reproducible (not manual)?
- [ ] Do you know which systems contribute data and which contracts govern them?

### Data Quality & Completeness

- [ ] Do you have automated quality gates that BLOCK (not warn) on failures?
- [ ] Can you state your known error rate / null rate baseline?
- [ ] Are quality test results retained with run history (not overwritten)?

### Representativeness & Bias

- [ ] Does your training data reflect the actual production population / user groups?
- [ ] Have you tested for protected group disparities in the data?
- [ ] Can you explain sampling methodology and known coverage gaps?

### Metric Definitions & Assumptions

- [ ] Are your KPIs defined in ONE place (semantic layer / code), not multiple dashboards?
- [ ] Do you have change control when a metric definition shifts?
- [ ] Are assumptions documented, versioned, and signed by domain experts?

### Versioning & Reproducibility

- [ ] Can you reconstruct any historical training snapshot within 1 business day?
- [ ] Do you use snapshot manifests (paths, row counts, hashes, timestamps)?
- [ ] Are train/val/test split methodologies documented and reproducible?

### Governance & Accountability

- [ ] Is there a named owner for training data quality?
- [ ] Do you have a limitations memo per use case (plain language, gaps, known weaknesses)?
- [ ] Can you produce an evidence pack for an auditor (not scattered across Slack)?

**Scoring:**
- **15-18 "yes":** You're ahead of 95% of teams. Fine-tune and document.
- **10-14 "yes":** Solid foundation, but gaps that need 60-90 days of work.
- **5-9 "yes":** Start the 90-day playbook immediately.
- **Below 5:** This is your most urgent engineering priority.

---

## From Data Engineer to Data Trust Engineer

The EU AI Act is creating a new role: someone who combines data engineering + governance + compliance. This is different from a DPO (privacy), a data architect (blueprints), or a compliance officer (legal interpretation). It is the intersection: you can read Article 10, translate it to dbt tests and lineage, and stand next to legal without bluffing.

### What a Data Trust Engineer Does

- Designs cohorts, snapshots, and manifests for ML
- Implements quality gates and monitoring with audit trails
- Aligns semantic definitions across BI, ops, and models
- Partners with risk, quality, and legal on evidence packs
- Says "no" credibly when a shortcut breaks traceability

### Skills Stack

- Lineage & metadata (OpenLineage, catalog patterns)
- Testing strategy (dbt, Great Expectations, contract tests)
- Semantic modeling (metrics layers, slowly changing dimensions for labels)
- Risk communication — translating engineering facts into executive memos without dilution

### Rate Implications (German Market)

| Positioning | Typical Daily Rate (Germany) |
|------------|---------------------------|
| "Pipeline builder" (conventional ETL/analytics) | €600-800/day |
| Data engineer + governance + compliance | €1,000-1,500/day |
| Fixed-scope "Article 10 Readiness Audit" | €8,000-15,000 per engagement |

The gap: most data engineers position as "pipeline builders" instead of "data trust architects." The market is starving for engineers who understand both code and compliance.

---

## About the Author

**Arjunkumar Kansagara** — Data Engineer & AI-Ready Data Infrastructure Consultant based in Germany.

I build data systems that organizations can actually trust. I specialize in metric governance, data platform architecture, and AI-driven analytics with deep expertise in EU regulatory compliance (GDPR, EU AI Act, DORA).

4+ years hands-on experience designing and governing data systems serving tens of millions of users across payment processing, AI products, and cybersecurity domains.

### My Approach: The TRUST Framework

**T**ransparency, **R**egulation, **U**nified Metrics, **S**ystematic Validation, and **T**ransfer — transforming data teams from service functions into strategic business partners.

### What I Offer

- **Article 10 Data Readiness Audit** — 6-8 week fixed-scope engagement
- **Metric Governance Frameworks** — semantic layer + quality + documentation
- **AI-Ready Data Infrastructure** — Snowflake, GCP, BigQuery, dbt, Cube.dev
- **Data Trust Architecture** — lineage, observability, compliance evidence packs

### Get in Touch

- **Portfolio:** [ak-dataweda.github.io/Portfolio](https://ak-dataweda.github.io/Portfolio/)
- **Case Studies:** [github.com/ak-dataweda/case-studies](https://github.com/ak-dataweda/case-studies)
- **LinkedIn:** [linkedin.com/in/arjunkumar-kansagara](https://www.linkedin.com/in/arjunkumar-kansagara-80a5a7136/)
- **Free 15-min Discovery Call:** [Book via portfolio](https://ak-dataweda.github.io/Portfolio/#contact)

*Freiberufliche Tätigkeit nach §18 EStG · Scope-based pricing · Based in Germany, available across the EU.*

---

*This guide is professional opinion from a data engineering perspective, not legal advice. Verify dates, fines, and obligations against current official EU sources. Confirm legal interpretation with qualified counsel.*
