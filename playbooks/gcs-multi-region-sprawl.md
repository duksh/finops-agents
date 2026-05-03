---
name: GCS Multi-Region Sprawl
description: GCS buckets created in multi-region or dual-region locations when single-region would meet requirements -- a 2x storage cost multiplier applied to data that never needs geo-redundancy.
scope: gcp
color: "#DC2626"
emoji: 🪣
---

## Problem

GCP offers three storage location types for GCS buckets:
- **Single-region** (e.g., `us-central1`): Standard ~$0.020/GB/month
- **Dual-region** (e.g., `NAM4`): Standard ~$0.026/GB/month — 30% premium
- **Multi-region** (e.g., `US`): Standard ~$0.026/GB/month — 30% premium

The 30% premium buys geo-redundancy: data is replicated across at least two
geographically separated locations. For data that must survive a regional
outage, this is worth every cent.

For backup data, build artifacts, log archives, and ML training datasets —
which are write-once and never need < 1-second recovery — the premium is
pure waste. At petabyte scale, the difference is $60,000/year per PB.

The failure mode: engineers pick `US` or `EU` when creating buckets because it
sounds "safer" or because it avoids thinking about which specific region to
use. The bucket accumulates data for years.

## Symptoms

- Buckets with `locationType = MULTI_REGION` or `DUAL_REGION` containing
  data that has no documented geo-redundancy requirement
- Backup or archive buckets in multi-region (backups are already
  point-in-time recovery data; geo-replication adds cost with no RPO benefit
  beyond what snapshots already provide)
- ML training datasets in multi-region (training jobs are rerunnable;
  data loss tolerance is high)
- Build artifact storage (container images, compiled binaries) in multi-region
  when the CI/CD pipeline only runs in one region
- `locationType = MULTI_REGION` combined with `NEARLINE` or `COLDLINE`
  storage class (the cold-data discount is offset by the geo-premium)

## Detection

```bash
# List all buckets with their location type and storage class
gcloud storage buckets list \
  --format="table(name,location,locationType,storageClass,timeCreated)" \
  | grep -E "MULTI_REGION|DUAL_REGION"
```

Cost query via BigQuery billing export:

```sql
-- GCS multi-region storage cost vs single-region equivalent (last 30 days)
SELECT
  project.id AS project_id,
  sku.description AS sku,
  SUM(usage.amount) AS gb_months,
  SUM(cost + IFNULL((SELECT SUM(c.amount) FROM UNNEST(credits) c), 0)) AS net_cost_usd,
  -- Estimated single-region equivalent (20% lower for Standard class)
  SUM(cost + IFNULL((SELECT SUM(c.amount) FROM UNNEST(credits) c), 0)) * 0.77 AS single_region_equiv_usd
FROM `MY_PROJECT.MY_DATASET.gcp_billing_export_resource_v1_*`
WHERE DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND service.description = 'Cloud Storage'
  AND (sku.description LIKE '%Multi-Region%' OR sku.description LIKE '%Dual-Region%')
GROUP BY 1, 2
ORDER BY net_cost_usd DESC;
```

## Fix

1. **Inventory multi-region and dual-region buckets** and classify each
   by use case (production serving / backup / archive / CI artifacts / ML data)

2. **For buckets that don't need geo-redundancy**, migrate to single-region.
   GCS does NOT support changing a bucket's location after creation.
   Migration requires creating a new single-region bucket and transferring data:

   ```bash
   # Create a single-region replacement bucket
   gcloud storage buckets create gs://MY-BUCKET-single-region \
     --location=us-central1 \
     --default-storage-class=NEARLINE

   # Transfer data using Storage Transfer Service (handles large datasets)
   gcloud transfer jobs create \
     gs://MY-BUCKET-multi-region \
     gs://MY-BUCKET-single-region \
     --source-agent-pool=projects/MY_PROJECT/agentPools/transfer-pool \
     --include-modified-after-absolute="1970-01-01T00:00:00Z"
   ```

3. **For data that genuinely needs geo-redundancy** (e.g., serving data for
   a multi-region application), keep the multi-region location but apply
   the cheapest appropriate storage class (Nearline/Coldline for cold data)

4. **Set bucket creation policy** in Terraform/IaC to default to
   single-region, with multi-region requiring explicit justification:
   ```hcl
   variable "bucket_location" {
     type    = string
     default = "us-central1"   # single-region default
     description = "Use a multi-region code (US, EU, ASIA) only for data with documented geo-HA requirements"
   }
   ```

5. **Add an Organization Policy** to warn when multi-region buckets are
   created without a `geo-ha-required` label:
   Custom constraint on `storage.googleapis.com/Bucket` checking
   `resource.location` matches single-region pattern OR label present.

## Anti-patterns

- **"Multi-region is safer"**: Safer for what? A 30-day backup that nobody
  reads except during a disaster doesn't need sub-second geo-replication.
  Define the actual RTO/RPO requirement before paying the premium.
- **Migrating without Storage Transfer Service**: `gsutil cp -r` works but
  is slow and doesn't handle large buckets well. Use Storage Transfer Service
  for anything over a few hundred GB.
- **Keeping the old bucket "just in case"**: Set a 90-day deletion date on
  the source bucket after transfer completes. Two copies of petabytes is
  twice the problem.

## References

- [GCS bucket locations](https://cloud.google.com/storage/docs/locations)
- [Storage Transfer Service](https://cloud.google.com/storage-transfer/docs)
- [GCS pricing](https://cloud.google.com/storage/pricing)
- Agent: `waste-detection/s3-storage-class-auditor.md` (Object Storage Class Auditor)
