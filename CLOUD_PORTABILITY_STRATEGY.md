# Cloud Portability Strategy - Pragmatic Approach

## Executive Summary

**Question:** If I containerize everything, can I easily switch clouds?
**Answer:** No. Containers are 40% of your stack. Managed services are 60% and NOT portable.

**Reality Check:**
- Migration time: 20-35 days (even with containers)
- Migration cost: $20K-50K in engineering time
- Probability you'll actually switch: <5% over 5 years

**Recommendation:** Build cloud-pragmatic, not cloud-agnostic. Use managed services, but architect for data portability.

---

## The Portability Spectrum

```
Cloud-Native          Cloud-Pragmatic         Cloud-Agnostic
(Lock-in)            (Recommended)           (Expensive)
│                    │                       │
├─ Cloud Run         ├─ Containers           ├─ Kubernetes
├─ Cloud SQL         ├─ PostgreSQL           ├─ Self-hosted DB
├─ Memorystore       │   (managed)           ├─ Self-hosted Redis
├─ Cloud Tasks       ├─ Standard protocols   ├─ Custom queue
├─ Secret Manager    ├─ Data export ready    ├─ Abstraction layers
│                    │                       │
Cost: $87/mo         Cost: $87/mo            Cost: $300/mo + ops time
Dev speed: Fast      Dev speed: Fast         Dev speed: Slow
Migration: 30 days   Migration: 25 days      Migration: 10 days
```

**Sweet spot:** Cloud-Pragmatic (middle column)

---

## TIH-Specific Portability Assessment

### Layer 1: Application (80% portable)

**Your FastAPI app:**
```python
# main.py - runs anywhere
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

**Portability score:** 90%
- Docker container runs on GCP/AWS/Azure without changes
- Python code is cloud-agnostic
- HTTP protocol is universal

**Lock-in risk:** LOW

---

### Layer 2: Database (70% portable)

**PostgreSQL data:**
```sql
-- Your schema works on any PostgreSQL instance
CREATE TABLE users (...);
CREATE TABLE threats (...);
```

**Portability score:** 70%
- SQL is standard (works on Cloud SQL, RDS, Azure Database)
- Data export: `pg_dump` works everywhere
- Migration path: dump → import (~2 hours for 1GB)

**Lock-in risk:** LOW (if you architect for export)

**Critical: Automate exports**
```bash
# Daily export to Cloud Storage (works as escape hatch)
pg_dump tih_db | gzip > backup-$(date +%Y%m%d).sql.gz
gsutil cp backup-*.sql.gz gs://tih-backups/
```

**Why this matters:**
- You can restore to AWS RDS in 2 hours if needed
- Data is YOUR asset, not locked in GCP

---

### Layer 3: Managed Services (30% portable)

**GCP services with equivalents:**

| Service | Abstraction Difficulty | Migration Time |
|---------|------------------------|----------------|
| **Cloud Run** → ECS/Container Apps | Medium | 2-3 days |
| **Cloud SQL** → RDS/Azure DB | Low | 2 hours |
| **Memorystore** → ElastiCache/Azure Cache | Low | 1 day |
| **Cloud Storage** → S3/Blob Storage | Medium | 1-2 days |
| **Secret Manager** → AWS Secrets/Key Vault | Medium | 1 day |
| **Cloud Scheduler** → EventBridge/Logic Apps | High | 3 days |

**Total migration time: 10-15 days** (with good abstractions)

**Lock-in risk:** MEDIUM (requires code changes)

---

### Layer 4: Cloud-Specific Services (0% portable)

**Services that DON'T have equivalents:**

| GCP Service | Why It's Unique | Workaround |
|-------------|-----------------|------------|
| **VPC Connector** | Serverless-to-VPC bridge | AWS PrivateLink (different architecture) |
| **Cloud Armor** | Integrated WAF | AWS WAF (separate service, different rules) |
| **Cloud Tasks** | Deferred execution model | AWS SQS + Lambda (rebuild queue logic) |

**Lock-in risk:** HIGH (architectural changes required)

---

## Pragmatic Architecture Decisions

### ✅ DO: Isolate cloud-specific code

**Bad (tightly coupled):**
```python
# main.py - GCP SDK everywhere
from google.cloud import storage, secretmanager, tasks

@app.post("/upload")
def upload(file):
    # Direct GCP calls in business logic
    client = storage.Client()
    bucket = client.bucket('tih-uploads')
    bucket.blob(file.name).upload_from_string(file.read())
```

**Good (abstracted):**
```python
# main.py - business logic
@app.post("/upload")
def upload(file):
    storage.save(file.name, file.read())

# adapters/storage.py - cloud-specific adapter
from abc import ABC, abstractmethod

class StorageAdapter(ABC):
    @abstractmethod
    def save(self, key: str, data: bytes) -> None:
        pass

class GCPStorageAdapter(StorageAdapter):
    def save(self, key: str, data: bytes) -> None:
        from google.cloud import storage
        client = storage.Client()
        bucket = client.bucket('tih-uploads')
        bucket.blob(key).upload_from_string(data)

class AWSStorageAdapter(StorageAdapter):
    def save(self, key: str, data: bytes) -> None:
        import boto3
        s3 = boto3.client('s3')
        s3.put_object(Bucket='tih-uploads', Key=key, Body=data)

# factory.py
def get_storage() -> StorageAdapter:
    provider = os.getenv('CLOUD_PROVIDER', 'gcp')
    if provider == 'gcp':
        return GCPStorageAdapter()
    elif provider == 'aws':
        return AWSStorageAdapter()
    raise ValueError(f"Unsupported provider: {provider}")
```

**Migration time reduction:** 30 days → 15 days

---

### ✅ DO: Use standard protocols

| Instead of | Use |
|------------|-----|
| Cloud Pub/Sub | Redis Streams (works anywhere) |
| Firestore | PostgreSQL JSONB (SQL standard) |
| Cloud Tasks | Celery + Redis (Python standard) |

**Why:** Standard protocols have libraries for every cloud.

---

### ✅ DO: Automate data exports

```python
# scheduled_exports.py
from google.cloud import storage
import subprocess
from datetime import datetime

def export_database():
    """Daily full database export"""
    timestamp = datetime.now().strftime('%Y%m%d')
    filename = f'tih-db-{timestamp}.sql.gz'

    # Export PostgreSQL
    subprocess.run([
        'pg_dump',
        os.getenv('DATABASE_URL'),
        '--format=custom',
        '--file', filename
    ])

    # Upload to Cloud Storage (escape hatch)
    client = storage.Client()
    bucket = client.bucket('tih-backups')
    bucket.blob(f'exports/{filename}').upload_from_filename(filename)

    # ALSO upload to AWS S3 (multi-cloud backup)
    import boto3
    s3 = boto3.client('s3',
        aws_access_key_id=os.getenv('AWS_KEY'),
        aws_secret_access_key=os.getenv('AWS_SECRET')
    )
    s3.upload_file(filename, 'tih-backups-aws', f'exports/{filename}')

    print(f"✅ Database exported to GCP + AWS: {filename}")

# Run daily at 2am
```

**Why this matters:**
- Your data lives in 2 clouds simultaneously
- If GCP has outage, you can restore to AWS in 2 hours
- If you need to migrate, you already have recent exports ready

**Cost:** $10/mo for redundant storage (WORTH IT)

---

### ✅ DO: Document cloud-specific dependencies

```yaml
# cloud-dependencies.yml
gcp_specific:
  - service: Cloud Run
    alternative: AWS ECS Fargate
    migration_effort: 2 days
    code_changes: deployment configs only

  - service: Cloud SQL
    alternative: AWS RDS PostgreSQL
    migration_effort: 2 hours
    code_changes: connection string only

  - service: Secret Manager
    alternative: AWS Secrets Manager
    migration_effort: 1 day
    code_changes: adapters/secrets.py

  - service: Cloud Storage
    alternative: AWS S3
    migration_effort: 1 day
    code_changes: adapters/storage.py

portable_components:
  - FastAPI application (containerized)
  - PostgreSQL schema (SQL standard)
  - Redis caching (protocol standard)
  - Celery workers (Python standard)

total_migration_estimate: 15-20 days
```

**Purpose:** When your CEO asks "can we move to AWS?", you have an answer in 5 minutes.

---

### ❌ DON'T: Build abstraction layers prematurely

**Anti-pattern:**
```python
# Over-engineered abstraction
class CloudProvider(ABC):
    @abstractmethod
    def store_file(self): pass

    @abstractmethod
    def get_secret(self): pass

    @abstractmethod
    def send_email(self): pass

    @abstractmethod
    def enqueue_task(self): pass

    # ... 20 more methods

class GCPProvider(CloudProvider):
    # 500 lines of adapter code

class AWSProvider(CloudProvider):
    # 500 lines of adapter code (never used)

class AzureProvider(CloudProvider):
    # 500 lines of adapter code (never used)
```

**Problems:**
- 1500 lines of abstraction code
- 2x development time for every feature
- Abstraction leaks (AWS SQS ≠ GCP Cloud Tasks)
- You're maintaining 3 implementations, using 1

**YAGNI principle:** You Aren't Gonna Need It (until you actually switch clouds)

---

### ❌ DON'T: Use Kubernetes "for portability"

**Kubernetes myth:**
> "Kubernetes runs everywhere, so I'm cloud-agnostic!"

**Reality:**
- GKE, EKS, AKS have different networking models
- Load balancers work differently on each cloud
- Persistent volumes are cloud-specific
- Autoscaling configs are different
- You still need cloud-specific IAM, DNS, monitoring

**Plus you add complexity:**
- Learning curve: 3-6 months to become proficient
- Ops burden: 10-20 hrs/week cluster management
- Cost: 3-5x more than serverless ($300/mo vs $87/mo)

**When to use Kubernetes:**
- You already have K8s expertise
- You need multi-tenant isolation
- You have 5+ microservices

**For TIH at MVP stage:** Cloud Run is 10x simpler.

---

## Migration Scenario Planning

### Scenario 1: GCP has regional outage

**Without portability prep:**
- Downtime: Hours to days
- Action: Wait for GCP to recover

**With portability prep:**
```bash
# 1. Restore latest DB export to AWS RDS (30 min)
aws rds restore-db-instance-from-s3 \
  --db-instance-identifier tih-db-failover \
  --s3-bucket-name tih-backups-aws

# 2. Deploy containers to AWS ECS (1 hour)
ecs-cli compose --file docker-compose.yml up

# 3. Update DNS to point to AWS (5 min)
# Total downtime: 90 minutes
```

**Portability ROI:** HIGH (disaster recovery)

---

### Scenario 2: Your company gets acquired, must move to Azure

**Without portability prep:**
- Timeline: 2-3 months
- Risk: Complete rewrite, high bug risk
- Cost: $50K-100K engineering time

**With portability prep:**
```bash
# Week 1: Infrastructure
terraform apply -var provider=azure  # Rewrite infra

# Week 2: Data migration
pg_dump | az postgres restore  # 2 hours

# Week 3: Testing
# Test all endpoints, validate data

# Week 4: Cutover
# DNS switch, monitor for 1 week

# Total: 4 weeks, $20K cost
```

**Portability ROI:** MEDIUM (50% time savings)

---

### Scenario 3: GCP raises prices 50%

**Without portability prep:**
- You're stuck, no leverage

**With portability prep:**
- Show GCP your AWS migration plan
- Negotiate discount ("We'll switch if you don't match")
- Likely outcome: 20-30% discount

**Portability ROI:** LOW (unlikely scenario, but good negotiation leverage)

---

## Recommended Implementation

### Phase 1: MVP (Build fast, document dependencies)

**DO:**
- ✅ Use GCP managed services (Cloud Run, Cloud SQL, Memorystore)
- ✅ Containerize your FastAPI app
- ✅ Use PostgreSQL (not Firestore)
- ✅ Document cloud-specific code in `cloud-dependencies.yml`
- ✅ Set up daily database exports to Cloud Storage

**DON'T:**
- ❌ Build abstraction layers
- ❌ Use Kubernetes
- ❌ Write AWS adapters "just in case"

**Timeline:** 6 weeks
**Cost:** $87/mo
**Migration readiness:** 70% (data is portable, code needs 2-3 weeks)

---

### Phase 2: 500+ Users (Add data redundancy)

**DO:**
- ✅ Export databases to AWS S3 daily (multi-cloud backup)
- ✅ Isolate cloud SDKs into adapter modules
- ✅ Use Redis Streams instead of Cloud Pub/Sub

**Timeline:** +1 week
**Cost:** +$10/mo (S3 storage)
**Migration readiness:** 85% (1-2 weeks to migrate)

---

### Phase 3: 5000+ Users (Prepare for acquisition)

**DO:**
- ✅ Build storage/secrets adapters (GCP + AWS implementations)
- ✅ Test AWS deployment in staging
- ✅ Document 2-week migration runbook

**Timeline:** +2 weeks
**Cost:** +$50/mo (staging environment on AWS)
**Migration readiness:** 95% (3-5 days to migrate)

---

## Decision Matrix

**Should I invest in portability?**

| Your Situation | Portability Investment | Why |
|----------------|----------------------|-----|
| **Pre-revenue MVP** | LOW (10% effort) | Focus on product-market fit, not infrastructure |
| **500-1000 users** | MEDIUM (20% effort) | Add data exports, isolate SDKs |
| **Planning acquisition** | HIGH (40% effort) | Build adapters, test multi-cloud |
| **Regulatory requirement** | HIGH (40% effort) | Must operate in specific cloud/region |
| **Just worried about lock-in** | LOW (10% effort) | Worry is not a business case |

---

## Cost-Benefit Analysis

### Option A: No portability effort
- Dev time saved: 100% (build fast)
- Migration cost: $50K, 2 months
- Risk: High coupling to GCP
- **Best for:** MVP stage, fast iteration

### Option B: Pragmatic portability (RECOMMENDED)
- Dev time cost: +15% (isolate SDKs, document dependencies)
- Migration cost: $20K, 3-4 weeks
- Risk: Medium coupling
- **Best for:** Most startups (1-10K users)

### Option C: Full abstraction
- Dev time cost: +100% (build everything twice)
- Migration cost: $10K, 1 week
- Risk: Over-engineered, slow development
- **Best for:** Enterprise with multi-cloud mandate

---

## TIH Recommendation

**For your Threat Intelligence Hub project:**

### Use Pragmatic Portability (Option B)

**Week 1-6 (MVP):**
- Build on GCP with managed services (fast)
- Document dependencies in `cloud-dependencies.yml`
- Set up automated database exports

**Week 7+ (Post-launch):**
- Isolate cloud SDK calls into `adapters/` module
- Test database restore to AWS (validate escape hatch)
- Maintain export automation

**Result:**
- Fast development (6 weeks to MVP)
- 85% portable (can migrate in 2-3 weeks if needed)
- Low ops burden (managed services)
- Negotiation leverage (credible AWS alternative)

---

## Final Verdict

**Question:** If I containerize everything, can I easily switch clouds?

**Answer:**

**Containers alone:** 40% portable (20-30 days to migrate)
**Containers + data exports:** 70% portable (10-15 days to migrate)
**Containers + adapters + exports:** 85% portable (3-5 days to migrate)
**Full Kubernetes abstraction:** 90% portable (but 3x cost, 2x dev time)

**For TIH, you should:**
1. ✅ Containerize your app (you're doing this)
2. ✅ Use PostgreSQL (already planned)
3. ✅ Automate daily exports to Cloud Storage
4. ✅ Document cloud dependencies
5. ❌ DON'T build abstraction layers yet (YAGNI)
6. ❌ DON'T use Kubernetes for "portability"

**Your migration timeline with this approach:**
- To AWS: 10-15 days
- To Azure: 10-15 days
- To self-hosted: 5-7 days (you own the data)

**Probability you'll actually migrate:** <5% over 5 years

**ROI of portability effort:** Positive IF you:
- Maintain data exports (disaster recovery)
- Document dependencies (due diligence for acquisition)
- Isolate cloud SDKs (reduces migration time 50%)

**ROI is NEGATIVE if you:**
- Build full abstraction layers speculatively
- Use Kubernetes just for portability
- Spend 2x dev time "just in case"

---

**Bottom line:** Build on one cloud, but architect for data portability. Your data is your moat, not your infrastructure.
