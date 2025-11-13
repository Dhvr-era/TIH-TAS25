# Phase 1: Core Services - Detailed Execution Guide

**Goal:** Set up database, cache, and networking infrastructure
**Time:** 2-3 hours (including 30 min wait time)
**Cost:** $149/mo (SQL + Redis + VPC)
**End Result:** Working PostgreSQL database + Redis cache accessible via private network

---

## Prerequisites Checklist

Before starting, make sure you have:

- [ ] Computer with terminal/command line access
- [ ] Internet connection (stable for 2-3 hours)
- [ ] Credit card for GCP billing
- [ ] Email address for GCP account
- [ ] OpenAI API key ([get one](https://platform.openai.com/api-keys))
- [ ] SendGrid API key ([sign up](https://signup.sendgrid.com/))
- [ ] 2-3 hours of uninterrupted time

**Missing API keys?** That's OK - you can add them later. We'll create placeholder secrets.

---

## Phase 1 Task List (16 steps)

```
Step 0: Initial Setup (20 min)
  ├─ 0.1 Install gcloud CLI          [⏸️ Not Started]
  ├─ 0.2 Create GCP project          [⏸️ Not Started]
  ├─ 0.3 Enable APIs                 [⏸️ Not Started]
  └─ 0.4 Set default region          [⏸️ Not Started]

Step 1: VPC Network (15 min)
  ├─ 1.1 Create VPC                  [⏸️ Not Started]
  ├─ 1.2 Create subnet               [⏸️ Not Started]
  └─ 1.3 Create VPC connector        [⏸️ Not Started] ⏰ 5 min wait

Step 2: Secret Manager (20 min)
  ├─ 2.1 Create secrets              [⏸️ Not Started]
  ├─ 2.2 Create service account      [⏸️ Not Started]
  └─ 2.3 Grant permissions           [⏸️ Not Started]

Step 3: Cloud SQL (40 min)
  ├─ 3.1 Create SQL instance         [⏸️ Not Started] ⏰ 15 min wait
  ├─ 3.2 Create database             [⏸️ Not Started]
  ├─ 3.3 Save connection string      [⏸️ Not Started]
  └─ 3.4 Test connection             [⏸️ Not Started]

Step 4: Redis (30 min)
  ├─ 4.1 Create Redis instance       [⏸️ Not Started] ⏰ 15 min wait
  └─ 4.2 Save Redis URL              [⏸️ Not Started]

✅ Phase 1 Complete → Can connect to database and cache
```

---

## Step 0: Initial GCP Setup (20 minutes)

### 0.1 Install gcloud CLI (10 min)

**macOS:**
```bash
# Install via Homebrew
brew install --cask google-cloud-sdk

# Verify installation
gcloud version

# Expected output:
# Google Cloud SDK 455.0.0
# ...
```

**Linux:**
```bash
# Download and install
curl https://sdk.cloud.google.com | bash

# Restart shell
exec -l $SHELL

# Verify installation
gcloud version
```

**Windows:**
```powershell
# Download installer from:
# https://cloud.google.com/sdk/docs/install#windows

# After installation, open "Google Cloud SDK Shell"
gcloud version
```

**✅ Checkpoint:** `gcloud version` shows a version number

---

### 0.2 Create GCP Project (5 min)

```bash
# Authenticate with Google account
gcloud auth login
# → Opens browser, select your Google account

# Set project ID (must be globally unique)
export PROJECT_ID="tih-production-$(date +%s)"
echo "Your project ID: $PROJECT_ID"

# Create project
gcloud projects create $PROJECT_ID \
  --name="Threat Intelligence Hub"

# Set as default project
gcloud config set project $PROJECT_ID

# Verify
gcloud config get-value project
# Should output: tih-production-XXXXX
```

**Enable Billing:**
```bash
# List billing accounts
gcloud billing accounts list

# If no billing account, go to:
# https://console.cloud.google.com/billing
# Click "Add billing account" and link credit card

# Get billing account ID
export BILLING_ACCOUNT_ID="YOUR-BILLING-ACCOUNT-ID"

# Link billing to project
gcloud billing projects link $PROJECT_ID \
  --billing-account=$BILLING_ACCOUNT_ID
```

**✅ Checkpoint:** `gcloud config get-value project` shows your project ID

---

### 0.3 Enable Required APIs (5 min)

```bash
# Enable all 11 APIs at once (takes 2-3 minutes)
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

# Wait for "Operation finished successfully" message

# Verify APIs are enabled
gcloud services list --enabled | grep -E 'compute|sql|run|redis'
```

**✅ Checkpoint:** See "compute.googleapis.com", "sqladmin.googleapis.com" in list

---

### 0.4 Set Default Region (2 min)

```bash
# Choose region closest to your users
# Options:
# - us-central1 (Iowa, USA)
# - us-east1 (South Carolina, USA)
# - europe-west1 (Belgium, Europe)
# - asia-southeast1 (Singapore, Asia)

export REGION="us-central1"
export ZONE="${REGION}-a"

# Set defaults
gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE

# Verify
echo "Region: $(gcloud config get-value compute/region)"
echo "Zone: $(gcloud config get-value compute/zone)"
```

**✅ Checkpoint:** Region and zone are set

**⏱️ Step 0 Complete! Time: ~20 minutes**

---

## Step 1: VPC Network Setup (15 minutes)

**Why:** Create private network so database isn't exposed to internet.

### 1.1 Create VPC Network (5 min)

```bash
# Create custom VPC
gcloud compute networks create tih-vpc \
  --subnet-mode=custom \
  --description="TIH private network"

# Verify creation
gcloud compute networks list | grep tih-vpc
# Should show: tih-vpc | custom | ...
```

**✅ Checkpoint:** VPC "tih-vpc" appears in list

---

### 1.2 Create Private Subnet (5 min)

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
# Should show: tih-private-subnet | 10.0.0.0/24 | ...
```

**What this means:**
- IP range: 10.0.0.0 - 10.0.0.255 (254 usable IPs)
- Private: Not accessible from internet
- Google access: Can reach GCP APIs without public IP

**✅ Checkpoint:** Subnet "tih-private-subnet" exists

---

### 1.3 Create VPC Connector (10 min: 5 min setup + 5 min wait)

**Purpose:** Allows Cloud Run (serverless) to connect to VPC resources (Cloud SQL, Redis).

```bash
# Create connector
gcloud compute networks vpc-access connectors create tih-connector \
  --region=$REGION \
  --subnet=tih-private-subnet \
  --min-instances=2 \
  --max-instances=10 \
  --machine-type=e2-micro

# Check status (takes 3-5 minutes)
gcloud compute networks vpc-access connectors describe tih-connector \
  --region=$REGION \
  --format="value(state)"

# Keep running this until it shows: READY
# (Run every 30 seconds)
```

**While waiting (3-5 minutes):** ☕ Take a coffee break!

**✅ Checkpoint:** Connector state is "READY"

**⏱️ Step 1 Complete! Time: ~15 minutes**

---

## Step 2: Secret Manager Setup (20 minutes)

**Why:** Store API keys securely (not in code or env vars).

### 2.1 Create Secrets (10 min)

```bash
# OpenAI API key (replace with yours)
echo -n "sk-YOUR_OPENAI_API_KEY_HERE" | \
  gcloud secrets create openai-api-key --data-file=-

# SendGrid API key
echo -n "SG.YOUR_SENDGRID_API_KEY_HERE" | \
  gcloud secrets create sendgrid-api-key --data-file=-

# Generate strong database password
DB_PASSWORD=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-25)
echo -n "$DB_PASSWORD" | \
  gcloud secrets create db-password --data-file=-

# Save password locally (you'll need it)
echo "Database password: $DB_PASSWORD"
echo "$DB_PASSWORD" > ~/.tih-db-password.txt
echo "Saved to: ~/.tih-db-password.txt"

# Auth0 credentials (if using Auth0)
echo -n "YOUR_AUTH0_DOMAIN" | \
  gcloud secrets create auth0-domain --data-file=-
echo -n "YOUR_AUTH0_CLIENT_ID" | \
  gcloud secrets create auth0-client-id --data-file=-
echo -n "YOUR_AUTH0_CLIENT_SECRET" | \
  gcloud secrets create auth0-client-secret --data-file=-

# Verify secrets
gcloud secrets list
# Should show: openai-api-key, sendgrid-api-key, db-password, etc.
```

**Don't have API keys yet?**
```bash
# Create placeholder secrets (update later)
echo -n "PLACEHOLDER" | gcloud secrets create openai-api-key --data-file=-
echo -n "PLACEHOLDER" | gcloud secrets create sendgrid-api-key --data-file=-
```

**✅ Checkpoint:** 6 secrets created and listed

---

### 2.2 Create Service Account (5 min)

**What's a service account?** An identity for your Cloud Run app to access other services.

```bash
# Create service account
gcloud iam service-accounts create tih-api \
  --display-name="TIH API Service Account" \
  --description="Service account for Cloud Run backend"

# Get full email
SA_EMAIL="tih-api@${PROJECT_ID}.iam.gserviceaccount.com"
echo "Service account: $SA_EMAIL"

# Verify
gcloud iam service-accounts list | grep tih-api
```

**✅ Checkpoint:** Service account "tih-api" exists

---

### 2.3 Grant Secret Access (5 min)

**Why:** Cloud Run needs permission to read secrets.

```bash
# Grant access to all secrets
for SECRET in openai-api-key sendgrid-api-key db-password \
              auth0-domain auth0-client-id auth0-client-secret; do
  gcloud secrets add-iam-policy-binding $SECRET \
    --member="serviceAccount:${SA_EMAIL}" \
    --role="roles/secretmanager.secretAccessor"

  echo "✓ Granted access to $SECRET"
done

# Verify permissions on one secret
gcloud secrets get-iam-policy openai-api-key
# Should show: tih-api@... with role secretAccessor
```

**✅ Checkpoint:** Service account has access to all secrets

**⏱️ Step 2 Complete! Time: ~20 minutes**

---

## Step 3: Cloud SQL PostgreSQL (40 minutes)

**Most critical step!** Database is the heart of your application.

### 3.1 Create Cloud SQL Instance (20 min: 5 min setup + 15 min wait)

```bash
# Retrieve database password
DB_PASSWORD=$(gcloud secrets versions access latest --secret=db-password)

# Create PostgreSQL instance (this takes 10-15 minutes)
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

# Monitor progress (run this repeatedly)
gcloud sql operations list --instance=tih-db --limit=1
# Status will change: PENDING → RUNNING → DONE
```

**While waiting (10-15 minutes):** 🍕 This is a good lunch break!

**Configuration explained:**
- `db-custom-2-7680`: 2 vCPU, 7.68 GB RAM (good for 1000+ users)
- `availability-type=REGIONAL`: High availability with auto-failover
- `no-assign-ip`: No public IP (security)
- `backup-start-time=02:00`: Daily backups at 2am

**Cost optimization for MVP:**
```bash
# If $75/mo is too much, use smaller tier:
--tier=db-f1-micro  # 1 vCPU, 0.6GB RAM = $7/mo
--availability-type=ZONAL  # No HA = save 50%
```

**✅ Checkpoint:** `gcloud sql instances list` shows tih-db as "RUNNABLE"

---

### 3.2 Create Database and User (10 min)

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

# Get private IP address
DB_PRIVATE_IP=$(gcloud sql instances describe tih-db \
  --format="value(ipAddresses[0].ipAddress)")

echo "Database private IP: $DB_PRIVATE_IP"
```

**✅ Checkpoint:** Database "tih_db" and user "tih_user" created

---

### 3.3 Save Connection String (5 min)

```bash
# Build connection string
DATABASE_URL="postgresql://tih_user:${DB_PASSWORD}@${DB_PRIVATE_IP}:5432/tih_db"

# Save to Secret Manager
echo -n "$DATABASE_URL" | \
  gcloud secrets create database-url --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding database-url \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

echo "✓ Database URL saved to Secret Manager"
echo "Connection string: $DATABASE_URL"
```

**✅ Checkpoint:** Secret "database-url" created

---

### 3.4 Test Connection (5 min)

**Install psql client (if needed):**

```bash
# macOS
brew install postgresql

# Linux (Ubuntu/Debian)
sudo apt-get install postgresql-client

# Verify
psql --version
```

**Test connection via Cloud SQL Proxy:**

```bash
# Download Cloud SQL Proxy
# macOS (Intel)
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.darwin.amd64

# macOS (Apple Silicon)
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.darwin.arm64

# Linux
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64

chmod +x cloud-sql-proxy

# Get connection name
CONNECTION_NAME=$(gcloud sql instances describe tih-db \
  --format="value(connectionName)")

# Start proxy in background
./cloud-sql-proxy $CONNECTION_NAME &

# Wait 5 seconds
sleep 5

# Connect with psql
PGPASSWORD=$DB_PASSWORD psql -h 127.0.0.1 -U tih_user -d tih_db

# You should see:
# tih_db=>

# Test query
\dt
# Output: Did not find any relations (this is correct - no tables yet)

# List databases
\l

# Exit
\q

# Stop proxy
killall cloud-sql-proxy
```

**✅ Checkpoint:** Successfully connected to database and ran `\dt` command

**⏱️ Step 3 Complete! Time: ~40 minutes (including 15 min wait)**

---

## Step 4: Memorystore Redis (30 minutes)

**Purpose:** Cache GPT-4 translations, session storage, rate limiting.

### 4.1 Create Redis Instance (25 min: 5 min setup + 15 min wait)

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
gcloud redis instances describe tih-cache \
  --region=$REGION \
  --format="value(state)"

# Keep checking until: READY
```

**While waiting (10-15 minutes):** 🎮 Another break!

**Cost optimization for MVP:**
```bash
# Use BASIC tier (no HA) = $35/mo instead of $65/mo
--tier=BASIC
# Remove these flags:
# --read-replicas-mode=READ_REPLICAS_ENABLED
```

**✅ Checkpoint:** Redis state is "READY"

---

### 4.2 Save Redis URL (5 min)

```bash
# Get Redis host and port
REDIS_HOST=$(gcloud redis instances describe tih-cache \
  --region=$REGION --format="value(host)")
REDIS_PORT=$(gcloud redis instances describe tih-cache \
  --region=$REGION --format="value(port)")

REDIS_URL="redis://${REDIS_HOST}:${REDIS_PORT}"

# Save to Secret Manager
echo -n "$REDIS_URL" | \
  gcloud secrets create redis-url --data-file=-

# Grant Cloud Run access
gcloud secrets add-iam-policy-binding redis-url \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/secretmanager.secretAccessor"

echo "✓ Redis URL saved"
echo "Redis connection: $REDIS_URL"
```

**✅ Checkpoint:** Secret "redis-url" created

**⏱️ Step 4 Complete! Time: ~30 minutes (including 15 min wait)**

---

## Phase 1 Completion Checklist

Verify everything is working:

- [ ] VPC network "tih-vpc" exists: `gcloud compute networks list`
- [ ] VPC connector "tih-connector" is READY
- [ ] 6 secrets created: `gcloud secrets list`
- [ ] Service account "tih-api" exists
- [ ] Cloud SQL instance "tih-db" is RUNNABLE
- [ ] Database "tih_db" exists
- [ ] Can connect via psql and run `\dt` command
- [ ] Redis instance "tih-cache" is READY
- [ ] All connection strings saved to Secret Manager

**If all checked:** ✅ Phase 1 Complete!

---

## What You've Built

```
┌─────────────────────────────────────────┐
│          GCP Project: tih-production     │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │         VPC Network (Private)      │ │
│  │  ┌──────────────┐  ┌────────────┐ │ │
│  │  │  Cloud SQL   │  │   Redis    │ │ │
│  │  │ PostgreSQL   │  │   Cache    │ │ │
│  │  │  (tih-db)    │  │(tih-cache) │ │ │
│  │  │ HA enabled   │  │ HA enabled │ │ │
│  │  │ 2vCPU/7.6GB  │  │    1GB     │ │ │
│  │  └──────────────┘  └────────────┘ │ │
│  │           ↕                         │ │
│  │    VPC Connector                   │ │
│  │  (for Cloud Run access)            │ │
│  └────────────────────────────────────┘ │
│                                          │
│  ┌────────────────────────────────────┐ │
│  │       Secret Manager               │ │
│  │  • openai-api-key                  │ │
│  │  • sendgrid-api-key                │ │
│  │  • database-url                    │ │
│  │  • redis-url                       │ │
│  │  • auth0-domain                    │ │
│  │  • auth0-client-id                 │ │
│  └────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

**Cost:** $149/mo (SQL $75 + Redis $65 + VPC $9)

---

## Next Steps

**You're now ready for Phase 2: Deployment**

In Phase 2 you'll:
- Create storage buckets for backups/uploads
- Deploy your FastAPI app to Cloud Run
- Test the /health endpoint

**Before starting Phase 2:**
1. ✅ Verify all checkpoints above are complete
2. ✅ Test database connection works
3. ✅ Save all credentials securely

**When ready:** Proceed to `PHASE_2_DETAILED_GUIDE.md`

---

## Troubleshooting

### "API not enabled" error
```bash
ERROR: API [sqladmin.googleapis.com] not enabled
```
**Fix:**
```bash
gcloud services enable sqladmin.googleapis.com
# Wait 1-2 minutes, then retry
```

### "Billing not enabled" error
```bash
ERROR: Billing must be enabled
```
**Fix:** Go to https://console.cloud.google.com/billing and link credit card

### "Quota exceeded" error
```bash
ERROR: Quota 'CPUS' exceeded. Limit: 8.0
```
**Fix:** Request quota increase at https://console.cloud.google.com/iam-admin/quotas

### Can't connect to database
```bash
# Check Cloud SQL is running
gcloud sql instances describe tih-db --format="value(state)"
# Should be: RUNNABLE

# Check VPC connector
gcloud compute networks vpc-access connectors describe tih-connector \
  --region=$REGION --format="value(state)"
# Should be: READY
```

### Redis creation fails
```bash
ERROR: RESOURCE_EXHAUSTED
```
**Fix:** Redis might not be available in your region. Try different region or use BASIC tier.

---

## Save Your Progress

```bash
# Export environment variables for later
cat > ~/.tih-env.sh <<EOF
export PROJECT_ID="$PROJECT_ID"
export REGION="$REGION"
export ZONE="$ZONE"
export DB_PRIVATE_IP="$DB_PRIVATE_IP"
export SA_EMAIL="$SA_EMAIL"
export DATABASE_URL="$DATABASE_URL"
export REDIS_URL="$REDIS_URL"
EOF

echo "Environment saved to: ~/.tih-env.sh"
echo "Load it later with: source ~/.tih-env.sh"
```

---

**🎉 Congratulations! Phase 1 Complete!**

You now have:
- ✅ Secure private network
- ✅ Production-grade PostgreSQL database
- ✅ High-performance Redis cache
- ✅ All secrets stored securely
- ✅ Infrastructure ready for application deployment

**Total time:** 2-3 hours (including wait times)
**Total cost:** $149/mo

**Next:** Take a break, then move to Phase 2 when ready!
