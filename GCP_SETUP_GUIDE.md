# GCP Managed Services Setup Guide - Step by Step

**Project:** Threat Intelligence Hub (TIH)
**Objective:** Set up production-grade managed services on GCP
**Timeline:** 1-2 days for complete setup
**Skill level:** Intermediate (copy-paste commands, understand what they do)

---

## Prerequisites Checklist

Before starting, ensure you have:

- [ ] GCP account created
- [ ] Billing enabled on your GCP project
- [ ] `gcloud` CLI installed locally ([install guide](https://cloud.google.com/sdk/docs/install))
- [ ] Project ID chosen (e.g., `tih-production`)
- [ ] Domain name purchased (optional for MVP, required for production)
- [ ] Credit card on file (you'll get $300 free credits for new accounts)

---

## Step 0: Initial GCP Setup (15 minutes)

### 1. Install gcloud CLI

```bash
# macOS
brew install --cask google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Windows
# Download from https://cloud.google.com/sdk/docs/install
```

### 2. Authenticate and set project

```bash
# Login to GCP
gcloud auth login

# Create new project (or use existing)
export PROJECT_ID="tih-production"
gcloud projects create $PROJECT_ID --name="Threat Intelligence Hub"

# Set as default project
gcloud config set project $PROJECT_ID

# Enable billing (required - you'll be prompted to link billing account)
# Go to: https://console.cloud.google.com/billing/linkedaccount?project=$PROJECT_ID

# Verify project is set
gcloud config get-value project
```

### 3. Enable required APIs

```bash
# Enable all APIs we'll need (takes 2-3 minutes)
gcloud services enable \
  compute.googleapis.com \
  sqladmin.googleapis.com \
  run.googleapis.com \
  redis.googleapis.com \
  storage-api.googleapis.com \
  secretmanager.googleapis.com \
  cloudscheduler.googleapis.com \
  cloudtasks.googleapis.com \
  vpcaccess.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com

# Verify APIs are enabled
gcloud services list --enabled
```

### 4. Set default region

```bash
# Choose region closest to your users
# Options: us-central1 (Iowa), us-east1 (S.Carolina), europe-west1 (Belgium)
export REGION="us-central1"
export ZONE="${REGION}-a"

gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

echo "Region set to: $REGION"
```

---

## Step 1: VPC Network Setup (20 minutes)

**Why first:** All other services will connect through this private network.

### 1.1 Create VPC Network

```bash
# Create custom VPC
gcloud compute networks create tih-vpc \
  --subnet-mode=custom \
  --description="TIH private network"

# Verify creation
gcloud compute networks list
```

### 1.2 Create Private Subnet

```bash
# Create subnet for backend services
gcloud compute networks subnets create tih-private-subnet \
  --network=tih-vpc \
  --region=$REGION \
  --range=10.0.0.0/24 \
  --enable-private-ip-google-access \
  --description="Private subnet for Cloud SQL, Redis, MISP"

# Verify subnet
gcloud compute networks subnets list --network=tih-vpc
```

### 1.3 Create Serverless VPC Connector

**Purpose:** Allows Cloud Run to connect to Cloud SQL via private IP (more secure).

```bash
# Create connector (takes 3-5 minutes)
gcloud compute networks vpc-access connectors create tih-connector \
  --region=$REGION \
  --subnet=tih-private-subnet \
  --min-instances=2 \
  --max-instances=10 \
  --machine-type=e2-micro

# Check status (wait until STATE = READY)
gcloud compute networks vpc-access connectors describe tih-connector \
  --region=$REGION

# Expected output:
# state: READY
```

**Cost:** ~$9/mo (2 instances × $0.15/hr × 730 hrs)

---

## Step 2: Secret Manager Setup (10 minutes)

**Why early:** You'll need secrets for database passwords, API keys.

### 2.1 Create Secrets

```bash
# OpenAI API key
echo -n "YOUR_OPENAI_API_KEY" | gcloud secrets create openai-api-key \
  --data-file=- \
  --replication-policy="automatic"

# SendGrid API key
echo -n "YOUR_SENDGRID_API_KEY" | gcloud secrets create sendgrid-api-key \
  --data-file=- \
  --replication-policy="automatic"

# Database password (generate strong password)
openssl rand -base64 32 | tr -d "=+/" | cut -c1-25 | \
  gcloud secrets create db-password --data-file=-

# Auth0 credentials (if using Auth0)
echo -n "YOUR_AUTH0_DOMAIN" | gcloud secrets create auth0-domain --data-file=-
echo -n "YOUR_AUTH0_CLIENT_ID" | gcloud secrets create auth0-client-id --data-file=-
echo -n "YOUR_AUTH0_CLIENT_SECRET" | gcloud secrets create auth0-client-secret --data-file=-

# Verify secrets
gcloud secrets list
```

### 2.2 Create Service Account for Cloud Run

```bash
# Create service account
gcloud iam service-accounts create tih-api \
  --display-name="TIH API Service Account" \
  --description="Service account for Cloud Run backend"

# Grant secret access
for SECRET in openai-api-key sendgrid-api-key db-password auth0-domain auth0-client-id auth0-client-secret; do
  gcloud secrets add-iam-policy-binding $SECRET \
    --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"
done

# Verify permissions
gcloud secrets get-iam-policy openai-api-key
```

**Cost:** Free (first 10,000 secret accesses/month)

---

## Step 3: Cloud SQL PostgreSQL Setup (30 minutes)

**This is your primary database.**

### 3.1 Create Cloud SQL Instance (High Availability)

```bash
# Get database password from Secret Manager
DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)

# Create PostgreSQL instance (takes 10-15 minutes)
gcloud sql instances create tih-db \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-7680 \
  --region=$REGION \
  --network=projects/$PROJECT_ID/global/networks/tih-vpc \
  --no-assign-ip \
  --availability-type=REGIONAL \
  --backup-start-time=02:00 \
  --retained-backups-count=7 \
  --retained-transaction-log-days=7 \
  --enable-point-in-time-recovery \
  --database-flags=max_connections=100 \
  --root-password=$DB_PASSWORD

# Monitor creation progress
gcloud sql operations list --instance=tih-db --limit=5

# Wait for "DONE" status before proceeding
```

**Configuration breakdown:**
- `db-custom-2-7680`: 2 vCPU, 7.68 GB RAM (good for 1000+ users)
- `availability-type=REGIONAL`: Automatic failover (99.95% SLA)
- `no-assign-ip`: Private IP only (security best practice)
- `backup-start-time=02:00`: Daily backups at 2am
- `enable-point-in-time-recovery`: Restore to any second within 7 days

**Cost:** ~$75/mo (with HA)

### 3.2 Create Database and User

```bash
# Create application database
gcloud sql databases create tih_db \
  --instance=tih-db \
  --charset=UTF8 \
  --collation=en_US.UTF8

# Create application user
gcloud sql users create tih_user \
  --instance=tih-db \
  --password=$DB_PASSWORD

# Get private IP address (save this)
gcloud sql instances describe tih-db \
  --format="value(ipAddresses[0].ipAddress)"

# Example output: 10.0.0.3
```

### 3.3 Build Connection String

```bash
# Format: postgresql://user:password@private-ip:5432/database
DB_PRIVATE_IP=$(gcloud sql instances describe tih-db --format="value(ipAddresses[0].ipAddress)")

DATABASE_URL="postgresql://tih_user:${DB_PASSWORD}@${DB_PRIVATE_IP}:5432/tih_db"

# Save to Secret Manager
echo -n "$DATABASE_URL" | gcloud secrets create database-url --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding database-url \
  --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

echo "Database URL saved to Secret Manager"
```

### 3.4 Test Connection (from local machine)

```bash
# Install Cloud SQL Proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.darwin.amd64
chmod +x cloud-sql-proxy

# Get connection name
CONNECTION_NAME=$(gcloud sql instances describe tih-db --format="value(connectionName)")

# Start proxy (leave running in separate terminal)
./cloud-sql-proxy $CONNECTION_NAME

# In another terminal, connect with psql
PGPASSWORD=$DB_PASSWORD psql -h 127.0.0.1 -U tih_user -d tih_db

# Test query
\dt  # Should show "Did not find any relations"
\q   # Quit
```

---

## Step 4: Memorystore Redis Setup (15 minutes)

**Purpose:** Caching GPT-4 translations, session storage, rate limiting.

### 4.1 Create Redis Instance (High Availability)

```bash
# Create Redis instance (takes 10-15 minutes)
gcloud redis instances create tih-cache \
  --size=1 \
  --region=$REGION \
  --tier=STANDARD_HA \
  --read-replicas-mode=READ_REPLICAS_ENABLED \
  --network=projects/$PROJECT_ID/global/networks/tih-vpc \
  --connect-mode=PRIVATE_SERVICE_ACCESS \
  --redis-version=redis_7_0

# Monitor creation
gcloud redis instances describe tih-cache --region=$REGION

# Wait for state: READY
```

**Configuration breakdown:**
- `size=1`: 1 GB memory (sufficient for 1000+ users)
- `tier=STANDARD_HA`: High availability with automatic failover
- `redis_7_0`: Latest stable version
- `PRIVATE_SERVICE_ACCESS`: Only accessible from VPC

**Cost:** ~$65/mo (HA tier)

**Cost optimization for MVP:** Use `BASIC` tier instead (single instance, no HA) = $35/mo

```bash
# MVP alternative (no HA, cheaper)
gcloud redis instances create tih-cache \
  --size=1 \
  --region=$REGION \
  --tier=BASIC \
  --network=projects/$PROJECT_ID/global/networks/tih-vpc \
  --connect-mode=PRIVATE_SERVICE_ACCESS \
  --redis-version=redis_7_0
```

### 4.2 Get Redis Connection Info

```bash
# Get Redis host and port
REDIS_HOST=$(gcloud redis instances describe tih-cache --region=$REGION --format="value(host)")
REDIS_PORT=$(gcloud redis instances describe tih-cache --region=$REGION --format="value(port)")

REDIS_URL="redis://${REDIS_HOST}:${REDIS_PORT}"

# Save to Secret Manager
echo -n "$REDIS_URL" | gcloud secrets create redis-url --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding redis-url \
  --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

echo "Redis URL: $REDIS_URL"
```

---

## Step 5: Cloud Storage Setup (10 minutes)

**Purpose:** Backups, user uploads (proof screenshots), MISP data archives.

### 5.1 Create Storage Buckets

```bash
# Backups bucket (with versioning)
gcloud storage buckets create gs://tih-backups-${PROJECT_ID} \
  --location=$REGION \
  --uniform-bucket-level-access \
  --versioning \
  --public-access-prevention

# Set lifecycle rule (delete after 90 days)
cat > lifecycle.json <<EOF
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "Delete"},
        "condition": {"age": 90}
      }
    ]
  }
}
EOF

gcloud storage buckets update gs://tih-backups-${PROJECT_ID} --lifecycle-file=lifecycle.json

# User uploads bucket
gcloud storage buckets create gs://tih-uploads-${PROJECT_ID} \
  --location=$REGION \
  --uniform-bucket-level-access \
  --public-access-prevention

# Static assets bucket (device icons, etc.)
gcloud storage buckets create gs://tih-assets-${PROJECT_ID} \
  --location=$REGION \
  --uniform-bucket-level-access

# Verify buckets
gcloud storage buckets list
```

### 5.2 Set IAM Permissions

```bash
# Grant Cloud Run service account access
for BUCKET in tih-backups-${PROJECT_ID} tih-uploads-${PROJECT_ID} tih-assets-${PROJECT_ID}; do
  gcloud storage buckets add-iam-policy-binding gs://$BUCKET \
    --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/storage.objectAdmin"
done
```

**Cost:** $0.02/GB/month (very cheap - ~$5/mo for 250GB)

---

## Step 6: Artifact Registry Setup (10 minutes)

**Purpose:** Store Docker images for Cloud Run deployment.

### 6.1 Create Docker Repository

```bash
# Create repository
gcloud artifacts repositories create tih-docker \
  --repository-format=docker \
  --location=$REGION \
  --description="TIH container images"

# Configure Docker to authenticate
gcloud auth configure-docker ${REGION}-docker.pkg.dev

# Verify repository
gcloud artifacts repositories list
```

### 6.2 Build and Push Test Image (optional - do this later)

```bash
# Example: Build your FastAPI app
cd /path/to/your/app

# Build image
gcloud builds submit --tag ${REGION}-docker.pkg.dev/${PROJECT_ID}/tih-docker/api:v1

# Image URL format:
# us-central1-docker.pkg.dev/tih-production/tih-docker/api:v1
```

**Cost:** $0.10/GB/month for storage

---

## Step 7: Cloud Scheduler Setup (5 minutes)

**Purpose:** Trigger daily digest generation, follow-up emails.

### 7.1 Create App Engine App (required for Scheduler)

```bash
# Cloud Scheduler requires App Engine to exist (even if you don't use it)
gcloud app create --region=$REGION

# This is a one-time setup, takes 2-3 minutes
```

### 7.2 Create Scheduler Jobs (do this AFTER deploying Cloud Run)

```bash
# Daily digest job (runs at 8am daily)
# NOTE: Replace CLOUD_RUN_URL with your actual Cloud Run URL after deployment
CLOUD_RUN_URL="https://tih-api-xxx.run.app"  # Update this later

gcloud scheduler jobs create http daily-digest \
  --schedule="0 8 * * *" \
  --uri="${CLOUD_RUN_URL}/internal/digest/generate" \
  --http-method=POST \
  --oidc-service-account-email=tih-api@${PROJECT_ID}.iam.gserviceaccount.com \
  --location=$REGION

# Follow-up reminders (runs at 8pm daily)
gcloud scheduler jobs create http follow-up-reminders \
  --schedule="0 20 * * *" \
  --uri="${CLOUD_RUN_URL}/internal/digest/follow-up" \
  --http-method=POST \
  --oidc-service-account-email=tih-api@${PROJECT_ID}.iam.gserviceaccount.com \
  --location=$REGION

# Verify jobs
gcloud scheduler jobs list --location=$REGION
```

**Cost:** Free (up to 3 jobs)

---

## Step 8: Compute Engine for MISP (30 minutes)

**Why Compute Engine:** MISP is stateful with MySQL + Redis + workers. Not suitable for Cloud Run.

### 8.1 Create VM Instance

```bash
# Create startup script
cat > misp-startup.sh <<'EOF'
#!/bin/bash
set -e

# Update system
apt-get update
apt-get install -y docker.io docker-compose git

# Start Docker
systemctl start docker
systemctl enable docker

# Clone MISP Docker
cd /opt
git clone https://github.com/MISP/misp-docker.git
cd misp-docker

# Configure environment
cat > .env <<ENVEOF
HOSTNAME=misp-server
BASE_URL=http://misp-server
MYSQL_ROOT_PASSWORD=$(openssl rand -base64 32)
MYSQL_PASSWORD=$(openssl rand -base64 32)
ENVEOF

# Start MISP
docker-compose up -d

# Wait for MISP to be ready
sleep 60

# Get admin key
docker-compose exec misp-core cat /var/www/MISP/app/Config/config.php | grep -A1 authkey

echo "MISP setup complete. Access at http://$(hostname -I | awk '{print $1}')"
EOF

# Create VM
gcloud compute instances create misp-server \
  --machine-type=e2-standard-4 \
  --zone=$ZONE \
  --subnet=tih-private-subnet \
  --no-address \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --metadata-from-file=startup-script=misp-startup.sh \
  --service-account=tih-api@${PROJECT_ID}.iam.gserviceaccount.com \
  --scopes=cloud-platform \
  --tags=misp-server

# Monitor startup (takes 5-10 minutes)
gcloud compute instances get-serial-port-output misp-server --zone=$ZONE

# Get private IP
MISP_IP=$(gcloud compute instances describe misp-server --zone=$ZONE --format="value(networkInterfaces[0].networkIP)")

echo "MISP private IP: $MISP_IP"
```

**Cost:** ~$80/mo (e2-standard-4 = 4 vCPU, 16GB RAM)

### 8.2 Access MISP (via SSH tunnel)

```bash
# Create SSH tunnel to access MISP web UI
gcloud compute ssh misp-server --zone=$ZONE -- -L 8080:localhost:80

# Open browser to: http://localhost:8080
# Default credentials: admin@admin.test / admin
```

### 8.3 Save MISP API Key

```bash
# After logging into MISP web UI:
# 1. Go to: Administration > List Users > View (admin)
# 2. Copy "Authkey"
# 3. Save to Secret Manager

MISP_API_KEY="YOUR_MISP_API_KEY"  # Replace with actual key
echo -n "$MISP_API_KEY" | gcloud secrets create misp-api-key --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding misp-api-key \
  --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Save MISP URL
MISP_URL="http://${MISP_IP}"
echo -n "$MISP_URL" | gcloud secrets create misp-url --data-file=-

gcloud secrets add-iam-policy-binding misp-url \
  --member="serviceAccount:tih-api@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

---

## Step 9: Cloud Run Deployment (20 minutes)

**Deploy your FastAPI backend.**

### 9.1 Create Dockerfile (if you haven't already)

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Run as non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Start server
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### 9.2 Deploy to Cloud Run

```bash
# Build and deploy in one command
gcloud run deploy tih-api \
  --source . \
  --region=$REGION \
  --platform=managed \
  --service-account=tih-api@${PROJECT_ID}.iam.gserviceaccount.com \
  --vpc-connector=tih-connector \
  --min-instances=1 \
  --max-instances=100 \
  --cpu=2 \
  --memory=2Gi \
  --timeout=60s \
  --concurrency=80 \
  --allow-unauthenticated \
  --set-secrets="DATABASE_URL=database-url:latest,REDIS_URL=redis-url:latest,OPENAI_API_KEY=openai-api-key:latest,SENDGRID_API_KEY=sendgrid-api-key:latest,MISP_URL=misp-url:latest,MISP_API_KEY=misp-api-key:latest"

# Get Cloud Run URL
CLOUD_RUN_URL=$(gcloud run services describe tih-api --region=$REGION --format="value(status.url)")

echo "API deployed at: $CLOUD_RUN_URL"
```

### 9.3 Test Deployment

```bash
# Test health endpoint
curl ${CLOUD_RUN_URL}/health

# Expected response:
# {"status":"ok","version":"1.0.0"}
```

**Cost:** ~$7/mo (min-instances=1) + request charges

---

## Step 10: Monitoring & Alerting Setup (15 minutes)

### 10.1 Create Uptime Check

```bash
# Create uptime check for API
gcloud monitoring uptime create tih-api-uptime \
  --display-name="TIH API Health Check" \
  --resource-type=uptime-url \
  --resource-host=$(echo $CLOUD_RUN_URL | sed 's|https://||') \
  --resource-path=/health \
  --period=60 \
  --timeout=10

# Verify
gcloud monitoring uptime list
```

### 10.2 Create Notification Channel

```bash
# Email notification
gcloud alpha monitoring channels create \
  --display-name="TIH Alerts Email" \
  --type=email \
  --channel-labels=email_address=YOUR_EMAIL@example.com

# Get channel ID
CHANNEL_ID=$(gcloud alpha monitoring channels list --format="value(name)" | head -1)

echo "Notification channel ID: $CHANNEL_ID"
```

### 10.3 Create Alert Policies

```bash
# Alert when API is down for 5 minutes
gcloud alpha monitoring policies create \
  --notification-channels=$CHANNEL_ID \
  --display-name="TIH API Down Alert" \
  --condition-display-name="Health check failing" \
  --condition-threshold-value=1 \
  --condition-threshold-duration=300s

# Alert on high error rate
gcloud alpha monitoring policies create \
  --notification-channels=$CHANNEL_ID \
  --display-name="High Error Rate" \
  --condition-display-name="Error rate > 5%" \
  --condition-threshold-value=0.05 \
  --condition-threshold-duration=60s
```

**Cost:** Free (included in GCP)

---

## Step 11: Security Hardening (20 minutes)

### 11.1 Set Up Cloud Armor (DDoS Protection)

```bash
# Create security policy
gcloud compute security-policies create tih-armor \
  --description="DDoS and bot protection for TIH"

# Rate limiting rule (100 requests/min per IP)
gcloud compute security-policies rules create 1000 \
  --security-policy=tih-armor \
  --expression="true" \
  --action="rate-based-ban" \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --ban-duration-sec=600 \
  --conform-action=allow \
  --exceed-action=deny-429

# Block common bot user agents
gcloud compute security-policies rules create 2000 \
  --security-policy=tih-armor \
  --expression="has(request.headers['user-agent']) && request.headers['user-agent'].contains('bot')" \
  --action="deny-403"

# Verify policy
gcloud compute security-policies describe tih-armor
```

**Note:** Cloud Armor requires Load Balancer. For MVP, you can skip this and add later.

**Cost:** $10/mo base + $1/1M requests

### 11.2 Enable Audit Logging

```bash
# Enable data access logs
cat > audit-config.json <<EOF
{
  "auditConfigs": [
    {
      "service": "allServices",
      "auditLogConfigs": [
        {"logType": "ADMIN_READ"},
        {"logType": "DATA_WRITE"},
        {"logType": "DATA_READ"}
      ]
    }
  ]
}
EOF

gcloud projects set-iam-policy $PROJECT_ID audit-config.json
```

**Cost:** Free for admin logs, $0.50/GB for data logs

---

## Step 12: Backup Automation (15 minutes)

### 12.1 Automated Database Backups (Already Configured)

```bash
# Verify backup schedule
gcloud sql instances describe tih-db --format="value(settings.backupConfiguration)"

# Manually trigger backup (test)
gcloud sql backups create --instance=tih-db

# List backups
gcloud sql backups list --instance=tih-db
```

### 12.2 Export Database to Cloud Storage (Additional Safety)

```bash
# Create export script
cat > export-db.sh <<'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BUCKET="gs://tih-backups-${PROJECT_ID}"

gcloud sql export sql tih-db ${BUCKET}/exports/tih-db-${DATE}.sql.gz \
  --database=tih_db

echo "Database exported to ${BUCKET}/exports/tih-db-${DATE}.sql.gz"
EOF

chmod +x export-db.sh

# Schedule weekly export via Cloud Scheduler
gcloud scheduler jobs create http weekly-db-export \
  --schedule="0 3 * * 0" \
  --uri="${CLOUD_RUN_URL}/internal/backup/export" \
  --http-method=POST \
  --oidc-service-account-email=tih-api@${PROJECT_ID}.iam.gserviceaccount.com \
  --location=$REGION
```

---

## Cost Summary

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| **Cloud SQL** | db-custom-2-7680 (HA) | $75 |
| **Memorystore Redis** | 1GB Standard HA | $65 |
| **Compute Engine (MISP)** | e2-standard-4 | $80 |
| **Cloud Run** | min-instances=1 | $7 |
| **VPC Connector** | 2-10 instances | $9 |
| **Cloud Storage** | 100GB | $5 |
| **Secret Manager** | 10 secrets | $1 |
| **Artifact Registry** | 10GB images | $1 |
| **Scheduler + Monitoring** | - | Free |
| **TOTAL** | | **$243/mo** |

### Cost Optimizations for MVP (<100 users):

```bash
# Use BASIC Redis (no HA) - saves $30/mo
--tier=BASIC

# Use smaller Cloud SQL - saves $40/mo
--tier=db-f1-micro

# Use min-instances=0 for Cloud Run - saves $7/mo (but adds cold starts)
--min-instances=0

# Skip MISP initially, use NVD API directly - saves $80/mo

# MVP optimized cost: $86/mo
```

---

## Verification Checklist

After setup, verify everything works:

- [ ] VPC network created: `gcloud compute networks list`
- [ ] Cloud SQL running: `gcloud sql instances list`
- [ ] Redis running: `gcloud redis instances list --region=$REGION`
- [ ] Secrets created: `gcloud secrets list`
- [ ] Storage buckets created: `gcloud storage buckets list`
- [ ] MISP accessible via SSH tunnel
- [ ] Cloud Run deployed: `gcloud run services list`
- [ ] Health endpoint returns 200: `curl ${CLOUD_RUN_URL}/health`
- [ ] Database connection works from Cloud Run
- [ ] Redis connection works from Cloud Run
- [ ] Monitoring alerts configured: `gcloud alpha monitoring policies list`

---

## Next Steps

1. **Develop application:**
   - Implement database schema (IMPLEMENTATION_GUIDE.md)
   - Build CPE matching engine
   - Integrate MISP API
   - Add translation engine

2. **Test thoroughly:**
   - Unit tests for matching engine
   - Integration tests for API endpoints
   - Load testing (1000+ concurrent users)

3. **Deploy frontend:**
   - Build React app
   - Deploy to Cloud Storage + Cloud CDN
   - Connect to Cloud Run backend

4. **Launch:**
   - Domain setup + SSL certificate
   - SendGrid domain verification
   - Beta testing with 20 users
   - Product Hunt launch

---

## Troubleshooting

### Cloud SQL connection fails

```bash
# Check VPC connector status
gcloud compute networks vpc-access connectors describe tih-connector --region=$REGION

# Verify Cloud Run has VPC connector attached
gcloud run services describe tih-api --region=$REGION --format="value(spec.template.spec.containers[0].vpcAccess)"

# Test connection with Cloud SQL Proxy
./cloud-sql-proxy $CONNECTION_NAME
```

### Redis connection fails

```bash
# Check if Redis is in READY state
gcloud redis instances describe tih-cache --region=$REGION --format="value(state)"

# Verify Cloud Run can access VPC
gcloud run services describe tih-api --region=$REGION --format="value(spec.template.metadata.annotations)"
```

### Secret access denied

```bash
# Check service account permissions
gcloud secrets get-iam-policy openai-api-key

# Verify service account email
gcloud run services describe tih-api --region=$REGION --format="value(spec.template.spec.serviceAccountName)"
```

### MISP not starting

```bash
# SSH into VM
gcloud compute ssh misp-server --zone=$ZONE

# Check Docker containers
sudo docker ps -a

# View logs
sudo docker-compose logs -f

# Restart MISP
cd /opt/misp-docker
sudo docker-compose restart
```

---

## Resource Cleanup (if needed)

**WARNING:** This deletes everything. Only use for testing/cleanup.

```bash
# Delete Cloud Run
gcloud run services delete tih-api --region=$REGION --quiet

# Delete Cloud SQL
gcloud sql instances delete tih-db --quiet

# Delete Redis
gcloud redis instances delete tih-cache --region=$REGION --quiet

# Delete Compute Engine
gcloud compute instances delete misp-server --zone=$ZONE --quiet

# Delete VPC connector
gcloud compute networks vpc-access connectors delete tih-connector --region=$REGION --quiet

# Delete VPC network
gcloud compute networks subnets delete tih-private-subnet --region=$REGION --quiet
gcloud compute networks delete tih-vpc --quiet

# Delete storage buckets
gcloud storage rm -r gs://tih-backups-${PROJECT_ID}
gcloud storage rm -r gs://tih-uploads-${PROJECT_ID}
gcloud storage rm -r gs://tih-assets-${PROJECT_ID}

# Delete secrets
gcloud secrets delete openai-api-key --quiet
gcloud secrets delete sendgrid-api-key --quiet
gcloud secrets delete database-url --quiet
# ... etc

# Delete project (nuclear option)
gcloud projects delete $PROJECT_ID
```

---

## References

- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud SQL Best Practices](https://cloud.google.com/sql/docs/postgres/best-practices)
- [Memorystore Redis](https://cloud.google.com/memorystore/docs/redis)
- [VPC Connector](https://cloud.google.com/vpc/docs/configure-serverless-vpc-access)
- [Secret Manager](https://cloud.google.com/secret-manager/docs)

---

**Setup complete! You now have a production-grade managed services infrastructure on GCP.**

Total setup time: ~3-4 hours (including wait times for service provisioning)

Next: Start building your application using the IMPLEMENTATION_GUIDE.md
