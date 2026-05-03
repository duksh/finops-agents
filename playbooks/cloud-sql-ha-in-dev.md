---
name: Cloud SQL HA in Dev
description: High Availability enabled on non-production Cloud SQL instances -- a silent cost doubler that accumulates across dev and staging environments.
scope: gcp
color: "#34A853"
emoji: 🗃️
---

## Problem

Cloud SQL High Availability doubles the instance cost (vCPU + RAM) by running
a permanent standby replica in a second zone. This is the right trade-off for
production databases with sub-hour RPO requirements. It is almost never the
right trade-off for development, staging, or QA databases -- where the
downside of a 10-minute outage is a developer rerunning a test, not a revenue
impact.

The pattern is born from convenience: developers clone the production Terraform
module, which has `availability_type = "REGIONAL"` by default, and the cost
doubles silently. There is no cloud alert for "HA enabled on a dev database."

A single `db-n1-standard-4` with HA costs ~$336/month.
The same instance without HA: ~$168/month.
10 dev/staging instances = ~$1,680/month wasted, ~$20k/year.

## Symptoms

- Cloud SQL instances with `availability_type = REGIONAL` in projects
  labelled `env=dev`, `env=staging`, or `env=test`
- Identical Terraform modules used for production and non-production
  without environment-specific overrides
- No HA-justification tag or comment on the non-production instance
- Cloud SQL cost growing proportionally with developer headcount
  (each new developer gets a personal dev DB with HA)

## Detection

```bash
# Find all HA-enabled Cloud SQL instances across all projects
gcloud sql instances list \
  --filter="settings.availabilityType=REGIONAL" \
  --format="table(name,project,region,settings.tier,settings.availabilityType)" \
  --project=MY_ORG_PROJECT
```

For multi-project audits, use the Cloud Asset Inventory:

```bash
gcloud asset search-all-resources \
  --scope="organizations/MY_ORG_ID" \
  --asset-types="sqladmin.googleapis.com/Instance" \
  --query="additionalAttributes.settings.availabilityType:REGIONAL" \
  --format="table(name,additionalAttributes.settings.tier,additionalAttributes.settings.availabilityType)"
```

Cross-reference with project labels to identify non-production:

```sql
-- BigQuery: Cloud SQL HA cost by project label (requires billing export)
SELECT
  project.id AS project_id,
  project.labels.env AS env,
  SUM(cost + IFNULL((SELECT SUM(c.amount) FROM UNNEST(credits) c), 0)) AS monthly_net_cost
FROM `MY_PROJECT.MY_DATASET.gcp_billing_export_resource_v1_*`
WHERE DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND service.description = 'Cloud SQL'
  AND sku.description LIKE '%Regional%'
GROUP BY 1, 2
ORDER BY monthly_net_cost DESC;
```

## Fix

1. **Audit now**: run the detection query above; classify each HA instance
   by environment (production / non-production)

2. **Disable HA on non-production instances** (maintenance window required;
   ~2 minutes of downtime):
   ```bash
   gcloud sql instances patch MY_INSTANCE_NAME \
     --availability-type=ZONAL
   ```

3. **Fix the Terraform module** to use environment-specific defaults:
   ```hcl
   variable "availability_type" {
     type    = string
     default = "ZONAL"  # safe default; production overrides to REGIONAL
   }

   resource "google_sql_database_instance" "this" {
     settings {
       availability_type = var.availability_type
     }
   }
   ```

4. **Add an Organization Policy** to warn (or deny) HA creation in
   non-production folders:
   ```yaml
   # Custom constraint: require justification tag for HA instances
   # Apply as 'warn' in non-prod folders, 'deny' in dev folders
   name: organizations/MY_ORG_ID/customConstraints/custom.requireHaJustification
   resourceTypes:
     - sqladmin.googleapis.com/Instance
   methodTypes: [CREATE, UPDATE]
   condition: >
     resource.settings.availabilityType == "REGIONAL" &&
     !resource.settings.userLabels.exists(label, label.key == "ha-justified")
   actionType: ALLOW
   ```

5. **Tag production instances** with `ha-justified=true` + reason to
   distinguish them from accidental HA in audits

## Anti-patterns

- **"HA is cheap insurance"**: HA on a dev DB costs as much as the DB
  itself. At 10 dev instances, the "insurance" costs $1,680/month.
- **Default-HA Terraform modules**: Modules that default to production
  settings are a gift that keeps costing. Environment-specific defaults
  belong in the module interface.
- **Fixing it manually each time**: The fix is one `--availability-type=ZONAL`
  flag. The prevention is one Terraform variable default. Do both.

## References

- [Cloud SQL high availability overview](https://cloud.google.com/sql/docs/postgres/high-availability)
- [Cloud SQL pricing](https://cloud.google.com/sql/pricing)
- Agent: `specialized/cloud-sql-cost-optimizer.md`
