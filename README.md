# Cletrics FinOps Agents

**The open-source reference library for cloud cost operations.** A curated set
of specialist AI agent personas and named-pattern playbooks that drop into
any modern coding assistant -- Claude Code, OpenAI Codex CLI, Gemini CLI,
GitHub Copilot, Cursor, Windsurf, Aider, and OpenCode -- plus ChatGPT Custom
GPTs, Claude Projects, and Gemini Gems on the web.

Built by [Cletrics](https://realtimecost.com) and the FinOps community.
MIT-licensed. Vendor-neutral where possible. Contributions welcome.

**Framework-aligned. 22 of 22 Capabilities covered (100%), 26 agents.** Every
agent carries explicit
[FinOps Framework](https://www.finops.org/framework/) metadata -- Domain,
Capability, Phase, Personas, Maturity entry point -- and references a
shared [doctrine](./doctrine/) layer (Iron Triangle, Data in the Path,
Crawl/Walk/Run, FCP Canon Anchors, **FOCUS Essentials**). See
[`fcp-coverage.md`](./fcp-coverage.md) for the live matrix, or run
`./scripts/fcp-coverage.sh`.

**FOCUS-first.** The library defaults to the
[FinOps Open Cost & Usage Specification (FOCUS)](https://focus.finops.org/)
column semantics -- `BilledCost`, `EffectiveCost`, `ListCost`,
`ContractedCost`, `ServiceCategory`, `CommitmentDiscountStatus`, etc. --
so agent recommendations carry across providers. Provider-native
schemas (AWS CUR, Azure exports, GCP billing export) are documented
fall-back layers, not the primary frame.

---

## Why this exists

Cloud cost operations is a practice: tagging, allocation, forecasting,
commitment management, waste detection, unit economics, and the human
cadences that tie them together. It's also full of specialized knowledge --
CUR schema quirks, GCP SUD dynamics, Azure enrollment types, Kubernetes
attribution math -- that every team ends up re-learning from scratch.

This repo packages that knowledge as agent personas your coding assistant
can load on demand. Instead of "help me write a query against my multi-
cloud billing data," you ask the **Cloud Billing Analyst** agent, and it
knows the FOCUS schema, the cost-column semantics (Billed / Effective /
List / Contracted), and the common gotchas before you type the first
line.

---

## Install

```bash
git clone https://github.com/cletrics/finops-agents.git
cd finops-agents

# Install for all detected tools
./scripts/install.sh

# Or pick one
./scripts/install.sh --tool claude-code
./scripts/install.sh --tool codex         # OpenAI Codex CLI
./scripts/install.sh --tool gemini-cli    # Gemini CLI (skill format)
./scripts/install.sh --tool copilot
./scripts/install.sh --tool cursor
./scripts/install.sh --tool windsurf
./scripts/install.sh --tool aider
./scripts/install.sh --tool opencode
./scripts/install.sh --tool chatgpt       # prints Custom GPT / Projects bundle path
```

### Web-only surfaces (no CLI required)

ChatGPT Custom GPTs, Claude Projects, and Gemini Gems can load the agents
directly. See [`integrations/chatgpt/README.md`](./integrations/chatgpt/README.md)
for setup instructions for each.

The installer only does local file copies into your tool's standard config
directory. No network calls, no sudo, no elevated privileges.

---

## Agent roster (26 agents, 7 categories)

> **v2 consolidation note.** The roster was consolidated from 43 agents
> in v1 to 24 in v2 by merging provider-specific variants behind
> FOCUS-shaped capability agents. v1 agents (e.g. `AWS Cost Explorer
> Analyst`, `GCP Billing Interpreter`) are now covered as
> provider-native deep-cuts inside the consolidated agents
> (e.g. `Cloud Billing Analyst`). Two additional GCP-specialized agents
> (BigQuery Cost Optimizer, Cloud SQL Cost Optimizer) were added in the
> GCP enhancement pass. If you previously installed v1, see
> the migration table at the bottom of this section.

### `cloud-cost/` -- 5 agents
Day-to-day cloud cost analysis. FOCUS-first, multi-cloud by default.

- **Cloud Billing Analyst** -- FOCUS-shaped reporting and analysis;
  AWS CUR / Azure Cost Management / GCP BigQuery export as documented
  fall-back layers
- **Budget & Anomaly Operator** -- forecast-based trajectory budgets +
  segment-aware seasonal anomaly detection (covers Budgeting and
  Anomaly Management)
- **Forecast & Estimation Analyst** -- driver-based forecasts +
  pre-deployment workload cost estimates (covers Forecasting and
  Planning & Estimating)
- **Unit Economics Modeler** -- cost per tenant, per request, per GB,
  per AI feature -- shipped at GA, not retroactively
- **FinOps Benchmarking Analyst** -- KPI selection + internal +
  external comparison, FOCUS-normalized

### `commitments/` -- 2 agents
Rate optimization across all clouds.

- **Commitment Discount Strategist** -- cross-cloud RI / SP /
  Reservation / CUD portfolio using FOCUS Commitment Discount columns
  (`CommitmentDiscountId`, `CommitmentDiscountStatus`,
  `PricingCategory='Committed'`); Effective vs Contracted savings math
- **EDP Negotiation Coach** -- private pricing prep, BATNA modeling,
  non-price terms

### `data-platforms/` -- 3 agents
The pipelines and models behind every cost dashboard.

- **FOCUS Data Engineer** -- ingest, transform, and validate
  FOCUS-conformant cost datasets across providers; runs the FOCUS
  Validator and Requirements Analyzer; includes a full GCP FOCUS
  pipeline section with BigQuery export → FOCUS column mapping,
  credit restatement handling, and tags normalization
- **Cost Warehouse Modeler** -- dimensional models with FOCUS as the
  conformed-dimension source of truth; dbt, semantic layer
- **BigQuery Cost Optimizer** *(GCP)* -- on-demand vs capacity slot
  pricing decisions, partition/clustering strategy, INFORMATION_SCHEMA
  query audits, slot commitment sizing, BI Engine ROI analysis

### `governance/` -- 7 agents
Practice operations: showback, chargeback, tagging, maturity, policy,
tooling, enablement, migration, allied-discipline coordination.

- **FinOps Practice Lead** -- governance + maturity assessment +
  allied-discipline integration (ITAM / ITSM / ITFM / Security /
  Sustainability)
- **Allocation & Policy Architect** -- tag taxonomy + policy-as-code
  enforcement (SCPs, Azure Policy, OPA / Gatekeeper)
- **Platform & SRE Cost Lead** -- embeds cost in ADRs and PR reviews;
  quantifies the reliability-cost curve and SLO trade-offs
- **Showback / Chargeback Architect** -- allocation model maturity
  (`EffectiveCost`-driven; `InvoiceId` reconciliation)
- **Cloud Onboarding Coordinator** -- migration-time cost hygiene +
  intake gate
- **FinOps Enablement Lead** -- training, Champions program,
  onboarding integration
- **FinOps Tooling Evaluator** -- build-vs-buy, vendor selection,
  FOCUS-conformance filter

### `kubernetes/` -- 2 agents
Container-level attribution and in-cluster optimization.

- **Kubernetes FinOps Engineer** -- cluster-level allocation,
  OpenCost / Kubecost, FOCUS-emitting allocation
- **Kubernetes Workload Optimizer** -- rightsizing (VPA-informed)
  and node-level autoscaling tuning (Karpenter / CAS)

### `waste-detection/` -- 3 agents
The hunters.

- **Idle & Orphaned Resource Hunter** -- compute, snapshots, load
  balancers, and zombie NAT Gateways in one inventory; includes GCP
  idle detection patterns (Compute Engine, Cloud SQL, static IPs,
  orphaned persistent disks)
- **Cross-AZ Egress Investigator** -- network cost hidden across
  storage / database / managed-service categories
- **Object Storage Class Auditor** *(renamed from S3 Storage Class Auditor)* --
  object-storage class alignment + lifecycle policies across S3, GCS
  (Nearline/Coldline/Archive/Autoclass), and Azure Blob

### `specialized/` -- 4 agents
Higher-leverage niches.

- **Workload Cost Optimizer** -- ML training/inference + serverless
  + spot strategies (compute-pattern specialist); includes GCP Cloud
  Run CPU allocation model, min-instances cost traps, Vertex AI
  dedicated vs shared endpoint pricing, TPU vs GPU selection
- **License & SaaS Cost Optimizer** -- BYOL, marketplace, entitlement
  audits, FOCUS Provider/Publisher/Invoice Issuer for marketplace
  attribution
- **Cloud Sustainability Analyst** -- carbon accounting, region-carbon
  trade-offs, demand shifting
- **Cloud SQL Cost Optimizer** *(GCP)* -- right-sizing Cloud SQL
  instances (PostgreSQL/MySQL/SQL Server), HA configuration audits,
  idle instance detection, read replica audits, CUD analysis, and
  AlloyDB migration evaluation

### v1 → v2 migration map

| v1 agent | v2 destination |
|---|---|
| AWS Cost Explorer Analyst | Cloud Billing Analyst (AWS deep-cut) |
| Azure Cost Management Navigator | Cloud Billing Analyst (Azure deep-cut) |
| GCP Billing Interpreter | Cloud Billing Analyst (GCP deep-cut) |
| Multi-Cloud Cost Comparator | Cloud Billing Analyst (default frame) |
| Budget Alert Tuner | Budget & Anomaly Operator |
| Cost Anomaly Detector | Budget & Anomaly Operator |
| Forecast Model Builder | Forecast & Estimation Analyst |
| Cloud Workload Cost Estimator | Forecast & Estimation Analyst |
| AWS Reserved Instance Optimizer | Commitment Discount Strategist (AWS RI deep-cut) |
| AWS Savings Plans Strategist | Commitment Discount Strategist (AWS SP deep-cut) |
| Azure Reservation Planner | Commitment Discount Strategist (Azure deep-cut) |
| GCP CUD Optimizer | Commitment Discount Strategist (GCP deep-cut) |
| Billing Data Pipeline Architect | FOCUS Data Engineer |
| CUR & FOCUS Data Engineer | FOCUS Data Engineer |
| FinOps Governance Lead | FinOps Practice Lead |
| FinOps Intersections Coordinator | FinOps Practice Lead |
| FinOps Practice Maturity Assessor | FinOps Practice Lead |
| Tag Hygiene Enforcer | Allocation & Policy Architect |
| FinOps Policy Architect | Allocation & Policy Architect |
| Platform Team Cost Lead | Platform & SRE Cost Lead |
| SRE SLO/Cost Tradeoff Analyst | Platform & SRE Cost Lead |
| Cluster Autoscaler Tuner | Kubernetes Workload Optimizer |
| Container Rightsizer | Kubernetes Workload Optimizer |
| ML Workload Cost Optimizer | Workload Cost Optimizer |
| Serverless Cost Profiler | Workload Cost Optimizer |
| Spot Orchestrator | Workload Cost Optimizer |
| Idle Resource Hunter | Idle & Orphaned Resource Hunter |
| EBS Snapshot Gardener | Idle & Orphaned Resource Hunter |
| Orphaned Load Balancer Hunter | Idle & Orphaned Resource Hunter |
| Zombie NAT Gateway Detector | Idle & Orphaned Resource Hunter |

---

## Playbooks (10 named patterns)

Named-pattern writeups for specific failure modes. Cite them by name.

**Cloud resource waste:**
- [Zombie NAT Gateway](./playbooks/zombie-nat-gateway.md)
- [Snapshot Sprawl](./playbooks/snapshot-sprawl.md)
- [Cross-AZ Chatterbox](./playbooks/cross-az-chatterbox.md)
- [Idle Load Balancer](./playbooks/idle-load-balancer.md)
- [Oversized RDS](./playbooks/oversized-rds.md)

**Reporting / allocation / governance patterns (FCP-anchored):**
- [Untagged Spend Drift](./playbooks/untagged-spend-drift.md)
- [Month-Length Illusion](./playbooks/month-length-illusion.md) -- the February "win" that isn't
- [Masked Anomaly](./playbooks/masked-anomaly.md) -- real cost blown in by a coincident commitment offset
- [Chargeback Revolt](./playbooks/chargeback-revolt.md) -- skipping showback costs 12+ months of credibility

**FOCUS adoption:**
- [FOCUS Adoption -- Parallel Run](./playbooks/focus-adoption-parallel-run.md) -- migrate to FOCUS without losing reconciliation or stakeholder trust (STMicroelectronics, GitLab, Zoom, UnitedHealth Group, European Parliament patterns)

---

## Usage examples

**Claude Code**
```
> Use the Cloud Billing Analyst agent to audit last month's spend
  increases. Focus on the AWS deep-cut.
```

**Cursor**
```
Reference the Budget & Anomaly Operator rule to design our alerting pipeline.
```

**Any tool**
Reference the agent by name in a conversation. The agent's persona and rules
become the working context for that task.

---

## GCP usage examples and expected results

The following examples show prompts and the kind of output you should
expect when using the GCP-enhanced agents with a GCP-focused billing
dataset.

### 1. Diagnose a GCP billing spike

**Agent:** Cloud Billing Analyst

**Prompt:**
```
Our GCP bill jumped $12k month-over-month. The BigQuery billing export
is in `my-project.billing.gcp_billing_export_resource_v1_*`.
Identify the top movers and explain the credit handling.
```

**Expected output:**
- FOCUS SQL query against the BigQuery export using the correct
  `cost + SUM(credits[].amount)` pattern for EffectiveCost
- Breakout by `service.description` and `project.id` showing top
  movers week-over-week
- Identification of any `cost_type = 'adjustment'` rows (corrections)
  that should be excluded from trend
- Explicit note on which credits (`SUD`, `CUD`, `PROMOTION`) are
  included in the net cost number and whether SUDs are masking the
  true usage spike

---

### 2. Set up GCP budget alerts that actually fire

**Agent:** Budget & Anomaly Operator

**Prompt:**
```
We have 8 GCP projects under billing account XXXXXX-YYYYYY-ZZZZZZ.
Set up project-level budget alerts using the Cloud Billing Budgets API.
We want trajectory alerts, not just threshold alerts.
```

**Expected output:**
- Python code using `google.cloud.billing_budgets_v1` with
  `thresholdRules` including a `FORECASTED_SPEND` basis rule at 100%
- Pub/Sub notification channel configuration
- Guidance on `credit_types_treatment` choice (include vs exclude)
  and how it affects the trajectory number Finance sees
- Warning about `allUpdatesRule` noisiness vs targeted `thresholdRules`

---

### 3. Right-size Cloud SQL instances

**Agent:** Cloud SQL Cost Optimizer

**Prompt:**
```
We have 14 Cloud SQL instances in us-central1. I suspect we're
over-paying on HA configuration and oversized tiers. Run a cost audit.
```

**Expected output:**
- `gcloud sql instances list` command to enumerate instances and
  their `availabilityType`
- Cloud Monitoring metric queries for `database/cpu/utilization` and
  `database/memory/utilization` over 14 days
- Table showing each instance: current tier, HA status, p95 CPU/RAM,
  recommended tier, monthly savings from right-sizing
- List of dev/staging instances with HA enabled and estimated monthly
  waste per instance (typically $168-$400/instance depending on tier)
- CUD break-even calculation for the production fleet

---

### 4. Optimize GKE cluster costs

**Agent:** Kubernetes FinOps Engineer

**Prompt:**
```
We run GKE Standard clusters. We're seeing high idle node cost.
Should we migrate to Autopilot? What changes to our workloads would
be required?
```

**Expected output:**
- Side-by-side billing model comparison: Standard (per node-hour) vs
  Autopilot (per pod resource request) with break-even analysis
- Identification of workloads that would cost more on Autopilot
  (over-requested pods that don't consume their requests) vs less
  (workloads currently running on mostly-idle nodes)
- GKE Cost Allocation feature setup instructions to quantify per-namespace
  spend before migration
- NEG load balancer annotation recommendation to reduce LCU charges
- Specific `scheduling.gke.io/gke-spot: "true"` approach for Autopilot
  Spot vs Standard Spot node pools

---

### 5. Reduce BigQuery query costs

**Agent:** BigQuery Cost Optimizer

**Prompt:**
```
Our BigQuery on-demand bill is $8,000/month. What are the top
optimization opportunities?
```

**Expected output:**
- `INFORMATION_SCHEMA.JOBS_BY_PROJECT` query to identify top 10
  most expensive queries by TB billed in the last 30 days
- Partition pruning audit: for each expensive query, check whether
  the `WHERE` clause filters on the partition column
- `TABLE_STORAGE` query identifying tables with high active storage
  and zero recent reads (candidates for deletion or archival)
- Break-even analysis: at $8k/month, capacity pricing (slots) is
  competitive -- model the break-even point with flex + monthly
  commitment layering
- `require_partition_filter = true` recommendation for the top 3
  scanned tables
- BI Engine sizing recommendation for dashboard queries (expected
  to eliminate 30-50% of scan cost)

---

### 6. Audit GCS storage classes

**Agent:** Object Storage Class Auditor

**Prompt:**
```
We have 200 TB in GCS buckets. Most were created with Standard storage.
Audit and recommend lifecycle policies.
```

**Expected output:**
- `gcloud storage buckets list` command to find buckets by location
  type (flag multi-region buckets as 2x premium candidates)
- GCS lifecycle policy JSON for Standard → Nearline (30 days) →
  Coldline (90 days) → Archive (365 days) transition chain
- Autoclass vs manual lifecycle policy recommendation: for buckets
  with mixed access patterns, Autoclass; for uniformly cold data,
  direct lifecycle rule to Coldline/Archive is cheaper
- Versioning + missing `noncurrentVersionExpiration` audit with
  example of the bound-growing cost trap
- Minimum storage duration penalty warning for premature transitions

---

### 7. Enforce GCP label policy at project creation

**Agent:** Allocation & Policy Architect

**Prompt:**
```
40% of our GCP spend is untagged. We need to enforce labels at
resource creation, not after-the-fact. Explain the GCP approach.
```

**Expected output:**
- Organization Policy custom constraint YAML enforcing required labels
  (`env`, `team`, `cost-center`) at resource creation time
- Explanation of GCP label inheritance gap: labels do NOT flow from
  folders to projects -- enforcement must be at each resource level
- `gcloud billing projects describe` audit command to find projects
  with no labels on any resources
- Cloud Asset Inventory pattern to build a `dim_project_hierarchy`
  table mapping `project.id` to folder chain for allocation roll-up
- Warning about Shared VPC host-project billing attribution and how
  to split networking costs across service projects

---

### 8. Onboard a new GCP project with cost controls from day zero

**Agent:** Cloud Onboarding Coordinator

**Prompt:**
```
We're launching a new GCP project for the payments team next week.
What needs to be in place before it goes live?
```

**Expected output:**
- Billing account linkage verification (`gcloud billing projects link`)
- Mandatory label Organization Policy custom constraint deployment
- Budget creation via Cloud Billing Budgets API with trajectory alert
  wired to Pub/Sub
- Detailed billing export to BigQuery enabled (with note: not
  retroactive, must be enabled before first charge)
- Service account convention (no default compute service account in
  production; dedicated SA per workload with `team` label)
- FOCUS pipeline check: is the new project's billing account already
  covered by the existing BigQuery export dataset?

---

## About Cletrics

[Cletrics](https://realtimecost.com) builds real-time cloud cost
observability for teams that want to stop waiting for yesterday's data.
This repo is our open-source contribution to the FinOps community -- use it
freely, with or without Cletrics the product.

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). Pull requests welcome for new
agents, new playbooks, tooling improvements, and localizations.

Please respect the 90/10 rule: agents and playbooks are generic FinOps
resources. Product references stay minimal.

## License

MIT. See [LICENSE](./LICENSE). Trademark and attribution terms in
[NOTICE](./NOTICE).

"Cletrics" is a trademark of Cletrics. Fork freely; don't imply endorsement.

## Security

See [SECURITY.md](./SECURITY.md) for the disclosure policy.
