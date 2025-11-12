# GCP Production Architecture - Veteran Architect Recommendations

## Executive Summary

**Current plan:** Cloud Run + Cloud SQL + Memorystore
**Risk level:** ⚠️ Medium - Missing critical HA, monitoring, and security components
**Recommended changes:** 8 additions for production-grade robustness

---

## 1. Compute Layer ✅ (Correct, with additions)

### **API Backend: Cloud Run** ✅
- **Why:** Serverless, auto-scaling, cost-effective ($0 at MVP scale)
- **Config:**
  ```bash
  gcloud run deploy tih-api \
    --source . \
    --region us-central1 \
    --min-instances 1 \        # ⚠️ ADD: Prevent cold starts (critical for UX)
    --max-instances 100 \       # Prevent runaway costs
    --cpu 2 \                   # 2 vCPU for CPE matching performance
    --memory 2Gi \              # MISP API calls need headroom
    --concurrency 80 \          # Request concurrency
    --timeout 60s \             # Digest generation can be slow
    --vpc-connector tih-vpc \   # ⚠️ ADD: Private network to Cloud SQL
    --set-env-vars DATABASE_URL=$DB_URL
  ```

- **⚠️ CRITICAL ADDITION: VPC Connector**
  - Cloud Run is public by default - connects to Cloud SQL via public IP (security risk)
  - **Add:** VPC Serverless Connector ($9/mo) for private database access
  - Prevents database exposure to internet

### **Background Jobs: Cloud Run + Cloud Scheduler** ✅ (Better than Celery)
- **Problem with Celery:** Requires persistent workers = Cloud Run not ideal
- **Better approach:**
  ```yaml
  # Deploy separate Cloud Run service for each background job

  # Daily digest service
  gcloud run deploy tih-digest-worker \
    --source ./workers/digest \
    --no-allow-unauthenticated \  # Only callable by Cloud Scheduler
    --timeout 600s                 # 10min for batch processing

  # Cloud Scheduler triggers daily
  gcloud scheduler jobs create http daily-digest \
    --schedule "0 8 * * *" \
    --uri "https://tih-digest-worker-xxx.run.app/generate" \
    --oidc-service-account-email digest-sa@project.iam.gserviceaccount.com
  ```

- **Why this is better:**
  - Zero cost when not running (vs 24/7 Celery workers)
  - Automatic retries and error handling
  - Scales to zero

### **MISP Deployment: Compute Engine** ⚠️ (NOT Cloud Run)
- **Why:** MISP is stateful, needs persistent storage
- **Config:**
  ```bash
  # Create VM for MISP
  gcloud compute instances create misp-server \
    --machine-type e2-standard-4 \       # 4 vCPU, 16GB RAM
    --boot-disk-size 100GB \             # MISP data grows
    --boot-disk-type pd-ssd \            # SSD for performance
    --image-family ubuntu-2204-lts \
    --subnet tih-private-subnet \        # Private network
    --no-address \                       # NO public IP (security)
    --metadata-from-file startup-script=misp-install.sh
  ```

- **Why NOT Cloud Run:** MISP has MySQL, Redis, background workers - container would be 5GB+
- **Cost:** ~$80/mo (acceptable for production data source)

---

## 2. Database Layer ⚠️ (Correct choice, wrong configuration)

### **Cloud SQL PostgreSQL** ✅ - BUT enable HA
- **Current plan risk:** Single instance = downtime during maintenance
- **Fix: Enable High Availability**
  ```bash
  gcloud sql instances create tih-db \
    --database-version POSTGRES_15 \
    --tier db-custom-2-7680 \           # 2 vCPU, 7.68GB RAM
    --region us-central1 \
    --availability-type REGIONAL \      # ⚠️ ADD: HA with automatic failover
    --backup-start-time 02:00 \         # Daily backups at 2am
    --retained-backups-count 7 \        # 7 days retention
    --retained-transaction-log-days 7 \ # Point-in-time recovery
    --enable-point-in-time-recovery \   # ⚠️ ADD: Recover to any second
    --database-flags max_connections=100 \
    --network tih-vpc \                 # Private IP only
    --no-assign-ip                      # ⚠️ ADD: NO public IP (security)
  ```

- **Why HA matters:**
  - GCP maintenance = automatic failover (30-60s downtime vs 10+ min)
  - Protects against zone failures
  - Cost: +50% (~$75/mo vs $50/mo) - **WORTH IT**

- **Read Replicas** (add at 500+ users):
  ```bash
  # When dashboard queries slow down digest generation
  gcloud sql instances create tih-db-read-replica \
    --master-instance-name tih-db \
    --tier db-custom-1-3840 \           # Smaller for read-only
    --replica-type READ

  # Update app: read from replica for dashboards, write to primary
  ```

### **Alternative: AlloyDB** ❌ (NOT for MVP)
- 4x performance vs Cloud SQL
- **BUT:** $500+/mo minimum (PostgreSQL-compatible but expensive)
- **Verdict:** Overkill until 10K+ users

---

## 3. Caching & Queue Layer ✅

### **Memorystore for Redis** ✅
- **Config:**
  ```bash
  gcloud redis instances create tih-cache \
    --size 1 \                          # 1GB for MVP
    --region us-central1 \
    --tier STANDARD_HA \                # ⚠️ ADD: HA mode (99.9% SLA)
    --read-replicas-mode READ_REPLICAS_ENABLED \
    --network tih-vpc
  ```

- **Why HA Redis:**
  - Without HA: Cache eviction = GPT-4 re-translation = cost spike
  - With HA: Automatic failover, read replicas
  - Cost: $65/mo (HA) vs $35/mo (Basic) - **WORTH IT**

### **Cloud Tasks** (for async jobs)
- **Use case:** User marks action → trigger follow-up email 48 hours later
- **Why NOT Celery:** Serverless, no infrastructure
- **Example:**
  ```python
  from google.cloud import tasks_v2

  client = tasks_v2.CloudTasksClient()
  task = {
      'http_request': {
          'http_method': tasks_v2.HttpMethod.POST,
          'url': 'https://tih-api.run.app/send-followup',
          'body': json.dumps({'alert_id': alert_id}).encode(),
      },
      'schedule_time': datetime.now() + timedelta(hours=48)
  }
  client.create_task(parent=queue_path, task=task)
  ```

---

## 4. Storage Layer ⚠️ (MISSING)

### **Cloud Storage** (not mentioned in current plan)
- **Use cases:**
  1. Database backups (automated export)
  2. User uploads (proof screenshots for verification)
  3. MISP raw feed data (archive)
  4. Static assets (device icons)

- **Config:**
  ```bash
  # Create bucket with versioning
  gcloud storage buckets create gs://tih-backups \
    --location us-central1 \
    --uniform-bucket-level-access \
    --versioning \                      # Protect against accidental deletes
    --lifecycle-rule '{"action": {"type": "Delete"}, "condition": {"age": 90}}'

  # Auto-export Cloud SQL daily
  gcloud sql backups create --instance tih-db \
    --backup-location gs://tih-backups/sql/$(date +%Y%m%d).sql
  ```

- **Cost:** ~$5/mo for 100GB

---

## 5. Networking & Security ⚠️ (CRITICAL MISSING)

### **VPC Network** ⚠️ (Not configured in current plan)
```bash
# Create private VPC
gcloud compute networks create tih-vpc \
  --subnet-mode custom

# Private subnet for backend services
gcloud compute networks subnets create tih-private-subnet \
  --network tih-vpc \
  --region us-central1 \
  --range 10.0.0.0/24 \
  --enable-private-ip-google-access  # Access GCP APIs without public IP

# Serverless VPC connector (for Cloud Run → Cloud SQL)
gcloud compute networks vpc-access connectors create tih-connector \
  --region us-central1 \
  --subnet tih-private-subnet \
  --min-instances 2 \                   # Prevent cold starts
  --max-instances 10
```

### **Cloud Armor** (DDoS protection + WAF)
```bash
# Create security policy
gcloud compute security-policies create tih-armor \
  --description "DDoS and bot protection"

# Rate limiting rule (prevent brute force)
gcloud compute security-policies rules create 1000 \
  --security-policy tih-armor \
  --expression "origin.region_code == 'CN' || origin.region_code == 'RU'" \
  --action "deny-403" \
  --description "Block high-risk regions"

# Rate limit: 100 req/min per IP
gcloud compute security-policies rules create 2000 \
  --security-policy tih-armor \
  --expression "true" \
  --action "rate-based-ban" \
  --rate-limit-threshold-count 100 \
  --rate-limit-threshold-interval-sec 60
```

### **Secret Manager** (NOT environment variables)
- **Problem:** Current plan uses env vars for API keys (line 1211)
- **Risk:** Exposed in Cloud Run revision history, logs
- **Fix:**
  ```bash
  # Store secrets
  echo -n "sk-xxx" | gcloud secrets create openai-api-key --data-file=-
  echo -n "SG.xxx" | gcloud secrets create sendgrid-api-key --data-file=-

  # Grant Cloud Run access
  gcloud secrets add-iam-policy-binding openai-api-key \
    --member serviceAccount:tih-api@project.iam.gserviceaccount.com \
    --role roles/secretmanager.secretAccessor

  # Mount in Cloud Run
  gcloud run deploy tih-api \
    --set-secrets OPENAI_API_KEY=openai-api-key:latest
  ```

---

## 6. Load Balancing & CDN ⚠️ (MISSING)

### **Global Load Balancer** (add at 1000+ users)
- **Why:** Cloud Run gives you regional URL - single point of failure
- **Fix:**
  ```bash
  # Create load balancer with multi-region backends
  gcloud compute backend-services create tih-backend \
    --global \
    --load-balancing-scheme EXTERNAL_MANAGED \
    --enable-cdn \                      # Cache static responses
    --health-checks tih-health

  # Add Cloud Run as backend
  gcloud compute backend-services add-backend tih-backend \
    --global \
    --network-endpoint-group tih-api-neg \
    --network-endpoint-group-region us-central1

  # Add second region for failover
  gcloud run deploy tih-api \
    --region europe-west1 \             # EU failover
    --tag europe
  ```

- **Result:**
  - 99.95% SLA (vs 99.5% single Cloud Run)
  - CDN caching = faster device catalog loads
  - Multi-region failover

---

## 7. Monitoring & Observability ⚠️ (CRITICAL MISSING)

### **Cloud Monitoring**
```bash
# Create uptime check
gcloud monitoring uptime create tih-api-uptime \
  --display-name "API Health Check" \
  --resource-type uptime-url \
  --resource-labels host=tih-api-xxx.run.app,path=/health \
  --period 60 \
  --timeout 10

# Alert policy: API down > 5 min
gcloud alpha monitoring policies create \
  --notification-channels $EMAIL_CHANNEL \
  --display-name "API Down Alert" \
  --condition-display-name "Health check failing" \
  --condition-threshold-value 1 \
  --condition-threshold-duration 300s
```

### **Cloud Logging**
- **Structured logs:**
  ```python
  import google.cloud.logging

  client = google.cloud.logging.Client()
  logger = client.logger('tih-api')

  logger.log_struct({
      'event': 'threat_matched',
      'threat_id': threat.id,
      'user_id': user.id,
      'confidence': 0.85,
      'severity': 'info'
  })
  ```

- **Log-based metrics:**
  - Count of translations per day (cost tracking)
  - False positive reports
  - Email bounce rate

### **Error Reporting**
- **Sentry vs Cloud Error Reporting:**
  - Sentry: Better UI, grouping, source maps ($26/mo)
  - Cloud Error Reporting: Free, GCP-native
  - **Verdict:** Use Cloud Error Reporting for MVP, add Sentry if needed

---

## 8. Backup & Disaster Recovery ⚠️ (MISSING)

### **Automated Backups**
```bash
# Cloud SQL automated backups (ALREADY configured above)
# Cloud Storage lifecycle (ALREADY configured above)

# MISP backup script (run daily)
gcloud compute instances add-metadata misp-server \
  --metadata startup-script='
    #!/bin/bash
    docker exec misp-container mysqldump misp > /backup/misp-$(date +%Y%m%d).sql
    gsutil cp /backup/*.sql gs://tih-backups/misp/
    find /backup -mtime +7 -delete
  '
```

### **Disaster Recovery Runbook**
1. **Database corruption:**
   - Restore from Cloud SQL automated backup: `gcloud sql backups restore --backup-id=<ID> --instance=tih-db`
   - RTO: 10 minutes, RPO: 0 (point-in-time recovery)

2. **Region outage:**
   - Failover to Cloud Run in `europe-west1`
   - Restore Cloud SQL from backup in secondary region
   - RTO: 30 minutes, RPO: 24 hours (daily backups)

3. **Data loss:**
   - Cloud Storage versioning recovers deleted files
   - Cloud SQL transaction logs recover to any second

---

## Cost Breakdown (Monthly)

### MVP (0-100 users):
| Service | Configuration | Cost |
|---------|--------------|------|
| Cloud Run (API) | min-instances=1 | $7 |
| Cloud Run (Workers) | On-demand | $2 |
| Cloud SQL (HA) | db-custom-2-7680 | $75 |
| Memorystore Redis (HA) | 1GB Standard | $65 |
| Compute Engine (MISP) | e2-standard-4 | $80 |
| VPC Connector | 2-10 instances | $9 |
| Cloud Storage | 100GB | $5 |
| Secret Manager | 10 secrets | $1 |
| Cloud Armor | 1M requests | $10 |
| **TOTAL** | | **$254/mo** |

### Production (1000 users):
| Service | Configuration | Cost |
|---------|--------------|------|
| Cloud Run (API) | min-instances=3 | $25 |
| Cloud Run (Workers) | Daily batches | $10 |
| Cloud SQL (HA) | db-custom-4-15360 + replica | $200 |
| Memorystore Redis (HA) | 5GB Standard | $250 |
| Compute Engine (MISP) | e2-standard-4 | $80 |
| Load Balancer + CDN | 10TB egress | $100 |
| VPC Connector | 2-10 instances | $9 |
| Cloud Storage | 500GB | $12 |
| **TOTAL** | | **$686/mo** |

**Revenue at 1000 users (5% conversion):** 50 × $4.99 = $249.50/mo
**⚠️ Problem:** Infrastructure costs > revenue at MVP scale

**Optimization for MVP (<500 users):**
- Skip Load Balancer + CDN → saves $100/mo
- Use Basic Redis (not HA) → saves $30/mo
- Single Cloud SQL instance (not HA) → saves $37/mo
- **Revised MVP cost:** $87/mo (sustainable)

---

## Architecture Decision Summary

### ✅ Keep from current plan:
1. Cloud Run for API backend
2. PostgreSQL (Cloud SQL)
3. Memorystore Redis

### ⚠️ Add for robustness:
1. **Cloud SQL HA mode** (99.9% uptime)
2. **VPC private networking** (security)
3. **Secret Manager** (protect API keys)
4. **Cloud Storage** (backups)
5. **Compute Engine for MISP** (stateful workload)
6. **Cloud Scheduler + Cloud Tasks** (replace Celery)
7. **Cloud Armor** (DDoS protection)
8. **Monitoring & alerting** (observability)

### ❌ Skip for MVP:
1. Load Balancer (add at 1000+ users)
2. Multi-region deployment (add at 5000+ users)
3. AlloyDB (too expensive)
4. GKE (overkill)

---

## Implementation Priority

**Week 1 (Must-have for security):**
- [ ] VPC network with private subnets
- [ ] Cloud SQL with HA + private IP
- [ ] Secret Manager for API keys
- [ ] Compute Engine for MISP (NOT Cloud Run)

**Week 2 (Must-have for reliability):**
- [ ] Cloud Run with VPC connector
- [ ] Memorystore Redis HA
- [ ] Cloud Storage for backups
- [ ] Cloud Scheduler for digest jobs

**Week 3 (Monitoring):**
- [ ] Uptime checks + alerting
- [ ] Cloud Logging structured logs
- [ ] Error reporting integration

**Post-MVP (scale optimizations):**
- [ ] Load Balancer + CDN (at 1000+ users)
- [ ] Cloud SQL read replica (at 500+ users)
- [ ] Multi-region failover (at 5000+ users)

---

## Final Verdict

**Your original plan (Cloud Run + Cloud SQL + Memorystore) was 70% correct.**

**Missing 30%:**
1. High Availability configurations (Cloud SQL HA, Redis HA)
2. Private networking (VPC, no public IPs)
3. Security hardening (Secret Manager, Cloud Armor, IAM)
4. Observability (monitoring, alerting, backups)

**With these additions, you have a production-grade architecture that:**
- 99.9% uptime SLA
- SOC2/ISO27001 ready (for future SMB customers)
- Handles 0-10,000 users without redesign
- Costs $87/mo at MVP, $686/mo at 1000 users

**Risk assessment:** Without these additions, you have a **prototype**, not a product.
