# Threat Intelligence Hub - Implementation Guide

**Backend-First Approach | 6-Week MVP Build Plan**

---

## Why Backend-First?

**Decision Rationale:**
- **Data Structure Drives UI**: Threat data structure must be understood before designing UI
- **Core Logic is Complex**: CPE matching algorithm requires testing before visualization
- **Real Data Needed**: Cannot mock threat intelligence accurately - frontend needs actual API responses
- **API Stability**: Building backend first prevents 3-4x API redesigns that occur with reverse engineering
- **Testability**: Matching engine can be unit tested independently of frontend

**Key Risk Mitigated**: Reverse engineering would waste 2-3 weeks redesigning the API once real threat data complexity is discovered.

---

## 6-Week Build Plan

### **Week 1-2: Backend Foundation**

#### **Day 1-2: MISP/OpenCTI Deployment**
```bash
# Use Docker to avoid setup hell
docker pull misp/misp-docker:latest
docker run -d -p 80:80 -p 443:443 misp/misp-docker

# Or OpenCTI (lighter alternative)
git clone https://github.com/OpenCTI-Platform/docker
cd docker
docker-compose up -d
```

**Deliverable**: MISP running on localhost, admin panel accessible

#### **Day 3: Ingest First Feed (CISA KEV)**
```python
# Add CISA KEV feed in MISP
# Navigate to: Sync Actions > List Feeds > Add Feed
# URL: https://www.cisa.gov/sites/default/files/feeds/known_exploited_vulnerabilities.json
# Type: MITRE format
# Frequency: Daily

# Verify ingestion
curl http://localhost/events/index | jq '.[] | {id, info, date}'
```

**Deliverable**: 50+ threats visible in MISP dashboard

#### **Day 4-5: Verify Data Quality**
```python
import requests

# Check threat data structure
response = requests.get('http://localhost/events/restSearch',
    headers={'Authorization': 'YOUR_MISP_API_KEY'},
    json={'returnFormat': 'json', 'limit': 10}
)

for event in response.json()['response']:
    print(f"CVE: {event.get('Event', {}).get('info')}")
    print(f"Date: {event.get('Event', {}).get('date')}")
    print(f"Attributes: {len(event.get('Event', {}).get('Attribute', []))}")
    print("---")
```

**Deliverable**: Document CPE format, MITRE ATT&CK structure, severity levels

#### **Week 2: Database Schema + FastAPI Skeleton**

**Database Schema (PostgreSQL):**
```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    tier VARCHAR(20) DEFAULT 'free' CHECK (tier IN ('free', 'pro', 'smb', 'enterprise', 'ngo')),
    device_limit INTEGER DEFAULT 5,
    created_at TIMESTAMP DEFAULT NOW(),
    preferences JSONB DEFAULT '{}'::jsonb,
    last_digest_sent TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_tier ON users(tier);

-- Device catalog (100 curated devices)
CREATE TABLE device_catalog (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category VARCHAR(50) NOT NULL, -- 'mobile', 'computer', 'iot', 'isp', 'saas'
    vendor VARCHAR(100) NOT NULL,
    product_name VARCHAR(200) NOT NULL,
    model VARCHAR(100),
    cpe_patterns TEXT[] NOT NULL, -- Array of CPE strings
    popular BOOLEAN DEFAULT false,
    icon_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_device_catalog_category ON device_catalog(category);
CREATE INDEX idx_device_catalog_vendor ON device_catalog(vendor);
CREATE INDEX idx_device_catalog_popular ON device_catalog(popular);

-- Example device catalog entries
INSERT INTO device_catalog (category, vendor, product_name, model, cpe_patterns, popular) VALUES
('mobile', 'Apple', 'iPhone', 'iPhone 14 Pro', ARRAY['cpe:2.3:h:apple:iphone_14_pro:*:*:*:*:*:*:*:*'], true),
('computer', 'Microsoft', 'Windows', '11', ARRAY['cpe:2.3:o:microsoft:windows_11:*:*:*:*:*:*:*:*'], true),
('iot', 'Google', 'Nest Thermostat', 'Gen 3', ARRAY['cpe:2.3:h:google:nest_thermostat:3.0:*:*:*:*:*:*:*'], true),
('saas', 'Zoom', 'Zoom Client', 'Desktop', ARRAY['cpe:2.3:a:zoom:zoom:*:*:*:*:*:*:*:*'], true);

-- User devices (user's inventory)
CREATE TABLE user_devices (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    device_id UUID REFERENCES device_catalog(id),
    nickname VARCHAR(100), -- e.g., "Living room thermostat"
    added_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_devices_user ON user_devices(user_id);
CREATE INDEX idx_user_devices_device ON user_devices(device_id);

-- Threats (ingested from MISP)
CREATE TABLE threats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    misp_event_id VARCHAR(100) UNIQUE,
    cve_id VARCHAR(50),
    title TEXT NOT NULL,
    description TEXT,
    severity VARCHAR(20) CHECK (severity IN ('critical', 'high', 'medium', 'low')),
    cvss_score FLOAT,
    published_date DATE,
    mitre_attack JSONB, -- {"tactics": ["TA0001"], "techniques": ["T1078"]}
    stride_category VARCHAR(1) CHECK (stride_category IN ('S', 'T', 'R', 'I', 'D', 'E')),
    affected_cpes TEXT[], -- Array of CPE strings
    source VARCHAR(100), -- 'CISA KEV', 'NVD', 'AlienVault'
    raw_data JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_threats_cve ON threats(cve_id);
CREATE INDEX idx_threats_severity ON threats(severity);
CREATE INDEX idx_threats_published ON threats(published_date DESC);
CREATE INDEX idx_threats_stride ON threats(stride_category);

-- Threat-device matches (the matching engine output)
CREATE TABLE threat_device_matches (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    threat_id UUID REFERENCES threats(id) ON DELETE CASCADE,
    device_id UUID REFERENCES device_catalog(id) ON DELETE CASCADE,
    confidence FLOAT NOT NULL CHECK (confidence >= 0 AND confidence <= 1),
    match_type VARCHAR(50) CHECK (match_type IN ('exact_cpe', 'vendor_product', 'keyword')),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(threat_id, device_id)
);

CREATE INDEX idx_tdm_threat ON threat_device_matches(threat_id);
CREATE INDEX idx_tdm_device ON threat_device_matches(device_id);
CREATE INDEX idx_tdm_confidence ON threat_device_matches(confidence);

-- Alerts (threats sent to users)
CREATE TABLE alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    threat_id UUID REFERENCES threats(id),
    user_device_id UUID REFERENCES user_devices(id),
    status VARCHAR(20) DEFAULT 'sent' CHECK (status IN ('sent', 'read', 'acted', 'dismissed')),
    sent_at TIMESTAMP DEFAULT NOW(),
    read_at TIMESTAMP,
    acted_at TIMESTAMP
);

CREATE INDEX idx_alerts_user ON alerts(user_id);
CREATE INDEX idx_alerts_status ON alerts(status);
CREATE INDEX idx_alerts_sent ON alerts(sent_at DESC);

-- User actions (proof tracking)
CREATE TABLE user_actions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_id UUID REFERENCES alerts(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    action VARCHAR(50) NOT NULL, -- 'updated', 'patched', 'disabled', 'no_action'
    notes TEXT,
    verified BOOLEAN DEFAULT false,
    verification_method VARCHAR(50), -- 'screenshot', 'attestation', 'scan'
    verification_data JSONB,
    timestamp TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_actions_alert ON user_actions(alert_id);
CREATE INDEX idx_user_actions_user ON user_actions(user_id);
CREATE INDEX idx_user_actions_timestamp ON user_actions(timestamp DESC);

-- Translations cache (to reduce GPT-4 costs)
CREATE TABLE translations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    threat_id UUID REFERENCES threats(id) ON DELETE CASCADE UNIQUE,
    plain_english TEXT NOT NULL,
    what_summary TEXT,
    who_affected TEXT,
    why_matters TEXT,
    action_steps TEXT,
    comprehension_score FLOAT, -- User survey feedback
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_translations_threat ON translations(threat_id);
```

**FastAPI Skeleton:**
```python
# main.py
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
import os

DATABASE_URL = os.getenv('DATABASE_URL', 'postgresql://localhost/tih_db')

app = FastAPI(title="Threat Intelligence Hub API", version="1.0.0")
security = HTTPBearer()

# Database session
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Health check
@app.get("/health")
async def health():
    return {"status": "ok", "version": "1.0.0"}

# User endpoints
@app.post("/users/signup")
async def signup(email: str, db: Session = Depends(get_db)):
    # TODO: Implement Auth0 integration
    pass

@app.get("/users/me")
async def get_user(token: str = Depends(security), db: Session = Depends(get_db)):
    # TODO: Verify token, return user data
    pass

# Device endpoints
@app.get("/devices/catalog")
async def get_device_catalog(category: str = None, db: Session = Depends(get_db)):
    # TODO: Return 100 devices, filter by category
    pass

@app.post("/devices/add")
async def add_user_device(user_id: str, device_id: str, nickname: str = None, db: Session = Depends(get_db)):
    # TODO: Add device to user inventory
    pass

# Threat endpoints
@app.get("/threats")
async def get_threats(user_id: str, severity: str = None, db: Session = Depends(get_db)):
    # TODO: Get threats affecting user's devices
    pass

@app.get("/threats/{threat_id}")
async def get_threat_detail(threat_id: str, db: Session = Depends(get_db)):
    # TODO: Return threat with plain English translation
    pass

# Alert endpoints
@app.get("/alerts")
async def get_alerts(user_id: str, status: str = None, db: Session = Depends(get_db)):
    # TODO: Return user's alerts
    pass

@app.post("/alerts/{alert_id}/action")
async def mark_action(alert_id: str, action: str, notes: str = None, db: Session = Depends(get_db)):
    # TODO: Record user action (proof tracking)
    pass

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**Deliverable**: API skeleton running, database tables created, `/health` endpoint returns 200

---

### **Week 3-4: Core Logic** ⚠️ **HARDEST PART**

#### **Day 1-2: CPE Matching Algorithm**

```python
# matching_engine.py
from sqlalchemy.orm import Session
from typing import List, Dict, Tuple
import re

def match_threat_to_devices(threat: Dict, db: Session) -> List[Dict]:
    """
    Match a threat (with CPEs) to devices in catalog
    Returns list of {device_id, confidence, match_type}
    """
    matches = []
    threat_cpes = threat.get('affected_cpes', [])

    if not threat_cpes:
        # Fallback: keyword matching
        return keyword_match(threat, db)

    # Get all devices from catalog
    devices = db.query(DeviceCatalog).all()

    for device in devices:
        for device_cpe in device.cpe_patterns:
            for threat_cpe in threat_cpes:
                confidence = cpe_similarity(device_cpe, threat_cpe)

                if confidence > 0.7:  # Threshold for match
                    matches.append({
                        'device_id': device.id,
                        'confidence': confidence,
                        'match_type': 'exact_cpe' if confidence > 0.9 else 'vendor_product'
                    })

    # Deduplicate and sort by confidence
    matches = sorted(matches, key=lambda x: x['confidence'], reverse=True)
    return matches

def cpe_similarity(cpe1: str, cpe2: str) -> float:
    """
    Compare two CPE strings
    CPE format: cpe:2.3:part:vendor:product:version:update:edition:language:...

    Example:
    cpe1 = "cpe:2.3:a:microsoft:edge:*:*:*:*:*:*:*:*"
    cpe2 = "cpe:2.3:a:microsoft:edge:110.0.1587.50:*:*:*:*:*:*:*"
    Returns: 0.8 (vendor+product match, version wildcard)
    """
    try:
        parts1 = cpe1.split(':')
        parts2 = cpe2.split(':')

        if len(parts1) < 6 or len(parts2) < 6:
            return 0.0

        # Must match: part (a/h/o), vendor, product
        if parts1[2] != parts2[2]:  # part
            return 0.0
        if parts1[3].lower() != parts2[3].lower():  # vendor
            return 0.0
        if parts1[4].lower() != parts2[4].lower():  # product
            return 0.0

        # Base match (vendor + product)
        base_score = 0.7

        # Version matching (parts[5])
        version1, version2 = parts1[5], parts2[5]

        if version1 == '*' or version2 == '*':
            # Wildcard match - affects all versions
            return 0.8

        if version1 == version2:
            # Exact version match
            return 1.0

        # Partial version match (e.g., "110.0" matches "110.0.1587.50")
        if version1.startswith(version2.rstrip('.*')) or version2.startswith(version1.rstrip('.*')):
            return 0.9

        # Vendor + product match only
        return base_score

    except Exception as e:
        print(f"CPE parsing error: {e}")
        return 0.0

def keyword_match(threat: Dict, db: Session) -> List[Dict]:
    """
    Fallback matching for threats without CPE data
    Uses keywords in threat title/description
    """
    matches = []
    keywords = extract_keywords(threat)

    devices = db.query(DeviceCatalog).all()

    for device in devices:
        device_text = f"{device.vendor} {device.product_name} {device.model}".lower()

        for keyword in keywords:
            if keyword.lower() in device_text:
                matches.append({
                    'device_id': device.id,
                    'confidence': 0.6,  # Lower confidence for keyword match
                    'match_type': 'keyword'
                })
                break

    return matches

def extract_keywords(threat: Dict) -> List[str]:
    """
    Extract vendor/product keywords from threat text
    """
    text = f"{threat.get('title', '')} {threat.get('description', '')}".lower()

    # Common vendor/product patterns
    vendors = ['apple', 'microsoft', 'google', 'cisco', 'adobe', 'oracle', 'vmware',
               'zoom', 'slack', 'amazon', 'samsung', 'dell', 'hp', 'lenovo']

    products = ['windows', 'macos', 'ios', 'android', 'chrome', 'firefox', 'edge',
                'office', 'outlook', 'teams', 'zoom', 'nest', 'alexa']

    keywords = []
    for vendor in vendors:
        if vendor in text:
            keywords.append(vendor)

    for product in products:
        if product in text:
            keywords.append(product)

    return keywords

# Test the matcher
def test_matcher():
    # Test case 1: Exact CPE match
    device_cpe = "cpe:2.3:a:zoom:zoom:5.13.0:*:*:*:*:*:*:*"
    threat_cpe = "cpe:2.3:a:zoom:zoom:*:*:*:*:*:*:*:*"

    score = cpe_similarity(device_cpe, threat_cpe)
    print(f"Test 1 - Wildcard version: {score}")  # Expected: 0.8

    # Test case 2: Version match
    device_cpe = "cpe:2.3:o:microsoft:windows_11:22h2:*:*:*:*:*:*:*"
    threat_cpe = "cpe:2.3:o:microsoft:windows_11:22h2:*:*:*:*:*:*:*"

    score = cpe_similarity(device_cpe, threat_cpe)
    print(f"Test 2 - Exact match: {score}")  # Expected: 1.0

    # Test case 3: Different vendor
    device_cpe = "cpe:2.3:a:google:chrome:*:*:*:*:*:*:*:*"
    threat_cpe = "cpe:2.3:a:mozilla:firefox:*:*:*:*:*:*:*:*"

    score = cpe_similarity(device_cpe, threat_cpe)
    print(f"Test 3 - No match: {score}")  # Expected: 0.0

if __name__ == "__main__":
    test_matcher()
```

**Expected Accuracy**: 70% for MVP (acceptable - iterate based on user feedback)

**Deliverable**: Matching engine returns confidence scores for 50+ test threats

#### **Day 3-4: MITRE ATT&CK + STRIDE Integration**

```python
# threat_categorization.py
from typing import Dict, List

# MITRE ATT&CK -> STRIDE mapping
ATTACK_TO_STRIDE = {
    # Spoofing (S)
    'T1078': 'S',  # Valid Accounts
    'T1134': 'S',  # Access Token Manipulation
    'T1550': 'S',  # Use Alternate Authentication Material

    # Tampering (T)
    'T1565': 'T',  # Data Manipulation
    'T1547': 'T',  # Boot or Logon Autostart Execution
    'T1556': 'T',  # Modify Authentication Process

    # Repudiation (R)
    'T1070': 'R',  # Indicator Removal on Host
    'T1562': 'R',  # Impair Defenses

    # Information Disclosure (I)
    'T1005': 'I',  # Data from Local System
    'T1039': 'I',  # Data from Network Shared Drive
    'T1552': 'I',  # Unsecured Credentials
    'T1087': 'I',  # Account Discovery

    # Denial of Service (D)
    'T1498': 'D',  # Network Denial of Service
    'T1499': 'D',  # Endpoint Denial of Service
    'T1529': 'D',  # System Shutdown/Reboot

    # Elevation of Privilege (E)
    'T1068': 'E',  # Exploitation for Privilege Escalation
    'T1548': 'E',  # Abuse Elevation Control Mechanism
    'T1055': 'E',  # Process Injection
}

def categorize_threat(threat: Dict) -> str:
    """
    Assign STRIDE category based on MITRE ATT&CK techniques
    Returns: 'S', 'T', 'R', 'I', 'D', or 'E'
    """
    mitre_attack = threat.get('mitre_attack', {})
    techniques = mitre_attack.get('techniques', [])

    if not techniques:
        # Fallback: keyword-based categorization
        return keyword_categorize(threat)

    # Count STRIDE votes
    stride_votes = {'S': 0, 'T': 0, 'R': 0, 'I': 0, 'D': 0, 'E': 0}

    for technique in techniques:
        stride_category = ATTACK_TO_STRIDE.get(technique)
        if stride_category:
            stride_votes[stride_category] += 1

    # Return category with most votes
    max_category = max(stride_votes, key=stride_votes.get)

    # If no votes, default to Information Disclosure (most common)
    if stride_votes[max_category] == 0:
        return 'I'

    return max_category

def keyword_categorize(threat: Dict) -> str:
    """
    Fallback STRIDE categorization using keywords
    """
    text = f"{threat.get('title', '')} {threat.get('description', '')}".lower()

    if any(word in text for word in ['spoof', 'phish', 'imperson', 'fake', 'credential']):
        return 'S'
    if any(word in text for word in ['tamper', 'modify', 'alter', 'corrupt']):
        return 'T'
    if any(word in text for word in ['log', 'audit', 'trace', 'evidence']):
        return 'R'
    if any(word in text for word in ['disclosure', 'leak', 'expose', 'steal', 'exfiltrat']):
        return 'I'
    if any(word in text for word in ['denial', 'dos', 'ddos', 'crash', 'hang']):
        return 'D'
    if any(word in text for word in ['privilege', 'escalat', 'admin', 'root', 'elevat']):
        return 'E'

    return 'I'  # Default

def get_stride_description(category: str) -> Dict[str, str]:
    """
    Return human-friendly STRIDE descriptions
    """
    descriptions = {
        'S': {
            'name': 'Spoofing',
            'description': 'Attacker pretends to be someone else',
            'icon': '🎭'
        },
        'T': {
            'name': 'Tampering',
            'description': 'Attacker modifies data or code',
            'icon': '🔧'
        },
        'R': {
            'name': 'Repudiation',
            'description': 'Attacker covers their tracks',
            'icon': '🕵️'
        },
        'I': {
            'name': 'Information Disclosure',
            'description': 'Attacker steals sensitive data',
            'icon': '📄'
        },
        'D': {
            'name': 'Denial of Service',
            'description': 'Attacker disrupts service availability',
            'icon': '🚫'
        },
        'E': {
            'name': 'Elevation of Privilege',
            'description': 'Attacker gains higher access level',
            'icon': '👑'
        }
    }
    return descriptions.get(category, {})
```

**Deliverable**: All threats in database have STRIDE category assigned

#### **Day 5: Plain English Translation Engine**

```python
# translation_engine.py
import openai
from sqlalchemy.orm import Session
from typing import Dict
import os

openai.api_key = os.getenv('OPENAI_API_KEY')

def translate_to_plain_english(threat: Dict, db: Session) -> Dict[str, str]:
    """
    Convert CVE jargon to plain English
    Returns: {what, who, why, do}
    """

    # Check cache first
    cached = db.query(Translation).filter(
        Translation.threat_id == threat['id']
    ).first()

    if cached:
        return {
            'what': cached.what_summary,
            'who': cached.who_affected,
            'why': cached.why_matters,
            'do': cached.action_steps,
            'plain_english': cached.plain_english
        }

    # Check template library
    template = get_template(threat.get('cve_id'), db)
    if template:
        translation = template.format(**threat)
    else:
        # Use GPT-4 for novel threats
        translation = gpt4_translate(threat)

    # Cache translation
    save_translation(threat['id'], translation, db)

    return translation

def gpt4_translate(threat: Dict) -> Dict[str, str]:
    """
    Use GPT-4 to translate technical jargon
    """

    stride_info = get_stride_description(threat.get('stride_category', 'I'))

    prompt = f"""
Translate this cybersecurity threat to plain English for non-technical users:

CVE: {threat.get('cve_id', 'Unknown')}
Title: {threat.get('title', '')}
Description: {threat.get('description', '')}
Severity: {threat.get('severity', 'medium').upper()}
Threat Type: {stride_info['name']} - {stride_info['description']}
Affected: {', '.join(threat.get('affected_cpes', [])[:3])}

Format your response as JSON with these keys:
{{
  "what": "One sentence explaining the threat (max 20 words)",
  "who": "Who is affected (specific devices/software)",
  "why": "Why it matters and urgency level",
  "do": "Specific action steps (numbered list, max 3 steps)"
}}

Rules:
- Use plain English (8th grade reading level)
- No jargon (avoid: vulnerability, exploit, CVE, patch)
- Be specific about devices
- Create urgency for critical/high severity
- Action steps must be achievable by non-technical users

Example:
{{
  "what": "A bug in Zoom lets attackers spy on your camera without permission",
  "who": "Anyone using Zoom on Windows or Mac computers",
  "why": "Your camera could be accessed without you knowing - act today",
  "do": "1. Open Zoom\\n2. Click Settings > Update\\n3. Restart Zoom when done"
}}
"""

    try:
        response = openai.ChatCompletion.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "You are a cybersecurity expert who explains threats to non-technical audiences."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=300,
            temperature=0.3,  # Lower temperature for consistent output
            response_format={"type": "json_object"}
        )

        import json
        translation = json.loads(response.choices[0].message.content)

        # Combine into full plain English summary
        translation['plain_english'] = f"{translation['what']}\n\n**Who:** {translation['who']}\n\n**Why:** {translation['why']}\n\n**What to do:**\n{translation['do']}"

        return translation

    except Exception as e:
        print(f"GPT-4 translation error: {e}")
        # Fallback to template
        return {
            'what': threat.get('title', 'Security update available'),
            'who': 'Users of affected devices',
            'why': f"{threat.get('severity', 'Medium')} severity security issue",
            'do': "1. Check for software updates\n2. Install any available updates\n3. Restart your device",
            'plain_english': threat.get('description', '')
        }

def get_template(cve_id: str, db: Session) -> str:
    """
    Get pre-written template for common CVEs
    Reduces GPT-4 costs by 80%
    """

    # Template library (build over time)
    templates = {
        'CVE-2023-XXXXX': "...",  # Add common CVE templates here
    }

    return templates.get(cve_id)

def save_translation(threat_id: str, translation: Dict, db: Session):
    """
    Cache translation to reduce API costs
    """
    trans = Translation(
        threat_id=threat_id,
        plain_english=translation['plain_english'],
        what_summary=translation['what'],
        who_affected=translation['who'],
        why_matters=translation['why'],
        action_steps=translation['do']
    )
    db.add(trans)
    db.commit()

# Cost calculation
def calculate_translation_cost(num_threats: int, users: int) -> float:
    """
    GPT-4 pricing: $0.03/1K input tokens, $0.06/1K output tokens
    Average: 500 input tokens, 200 output tokens per threat
    Cost per translation: $0.027

    With caching (80% hit rate):
    - 1,000 users, 10 new threats/day = 10 translations/day
    - Monthly cost: 10 * 30 * $0.027 = $8.10

    Without caching:
    - 1,000 users, 10 new threats/day = 10 translations/day (same)
    - Monthly cost: 10 * 30 * $0.027 = $8.10

    NOTE: Cost is per unique threat, not per user!
    """
    cost_per_translation = 0.027
    cache_hit_rate = 0.80  # 80% of threats are duplicates

    unique_threats = num_threats * (1 - cache_hit_rate)
    monthly_cost = unique_threats * 30 * cost_per_translation

    return monthly_cost
```

**Cost Optimization**:
- Without caching: ~$50/mo for 1,000 users
- With caching (80% hit rate): ~$10/mo for 1,000 users
- 5x cost reduction

**Deliverable**: 50 threats translated to plain English, cached in database

#### **Week 4: User Flow + Daily Digest**

```python
# digest_generator.py
from celery import Celery
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
import os

# Celery config
celery = Celery('tih_tasks', broker=os.getenv('REDIS_URL', 'redis://localhost:6379'))

@celery.task
def generate_daily_digest():
    """
    Run at 8am daily for all users
    Scheduled via: celery -A digest_generator beat
    """
    db = SessionLocal()

    try:
        users = db.query(User).all()

        for user in users:
            process_user_digest(user.id, db)

        db.commit()

    except Exception as e:
        print(f"Digest generation error: {e}")
        db.rollback()
    finally:
        db.close()

def process_user_digest(user_id: str, db: Session):
    """
    Generate digest for single user
    """
    user = db.query(User).filter(User.id == user_id).first()

    # Get user's devices
    user_devices = db.query(UserDevice).filter(
        UserDevice.user_id == user_id
    ).all()

    if not user_devices:
        # No devices configured - skip
        return

    device_ids = [ud.device_id for ud in user_devices]

    # Find threats affecting user's devices (last 24 hours)
    yesterday = datetime.now() - timedelta(days=1)

    threats = db.query(Threat).join(ThreatDeviceMatch).filter(
        ThreatDeviceMatch.device_id.in_(device_ids),
        Threat.published_date >= yesterday,
        ThreatDeviceMatch.confidence > 0.7  # Only high-confidence matches
    ).order_by(
        Threat.severity.desc(),  # Critical first
        ThreatDeviceMatch.confidence.desc()
    ).limit(5).all()  # Max 5 alerts per email

    if not threats:
        # All safe - send positive email
        send_all_safe_email(user, db)
        return

    # Translate threats to plain English
    translated_threats = []
    for threat in threats:
        translation = translate_to_plain_english(threat, db)

        # Find which user device is affected
        affected_device = db.query(UserDevice).join(ThreatDeviceMatch).filter(
            ThreatDeviceMatch.threat_id == threat.id,
            UserDevice.user_id == user_id
        ).first()

        translated_threats.append({
            'threat': threat,
            'translation': translation,
            'device': affected_device,
            'stride': get_stride_description(threat.stride_category)
        })

    # Generate email HTML
    email_html = render_digest_email(user, translated_threats)

    # Send via SendGrid
    send_email(
        to=user.email,
        subject=f"🔴 {len(threats)} Action{'s' if len(threats) > 1 else ''} Needed - Threat Intelligence Hub",
        html=email_html
    )

    # Log alerts
    for threat_data in translated_threats:
        alert = Alert(
            user_id=user.id,
            threat_id=threat_data['threat'].id,
            user_device_id=threat_data['device'].id,
            status='sent',
            sent_at=datetime.now()
        )
        db.add(alert)

    # Update user's last digest timestamp
    user.last_digest_sent = datetime.now()
    db.commit()

def render_digest_email(user: User, threats: List[Dict]) -> str:
    """
    Generate HTML email with plain English threats
    """

    severity_colors = {
        'critical': '#DC2626',  # Red
        'high': '#EA580C',      # Orange
        'medium': '#F59E0B',    # Yellow
        'low': '#6B7280'        # Gray
    }

    html = f"""
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <style>
        body {{ font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; line-height: 1.6; color: #1F2937; margin: 0; padding: 0; background-color: #F3F4F6; }}
        .container {{ max-width: 600px; margin: 20px auto; background: white; border-radius: 8px; overflow: hidden; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }}
        .header {{ background: linear-gradient(135deg, #667EEA 0%, #764BA2 100%); color: white; padding: 30px 20px; text-align: center; }}
        .header h1 {{ margin: 0; font-size: 24px; }}
        .content {{ padding: 30px 20px; }}
        .threat {{ background: #F9FAFB; border-left: 4px solid #667EEA; padding: 20px; margin: 20px 0; border-radius: 4px; }}
        .threat.critical {{ border-left-color: #DC2626; }}
        .threat.high {{ border-left-color: #EA580C; }}
        .threat.medium {{ border-left-color: #F59E0B; }}
        .severity {{ display: inline-block; padding: 4px 12px; border-radius: 12px; font-size: 12px; font-weight: 600; text-transform: uppercase; margin-bottom: 10px; }}
        .severity.critical {{ background: #FEE2E2; color: #DC2626; }}
        .severity.high {{ background: #FFEDD5; color: #EA580C; }}
        .severity.medium {{ background: #FEF3C7; color: #F59E0B; }}
        .device {{ font-size: 14px; color: #6B7280; margin: 10px 0; }}
        .action-steps {{ background: white; padding: 15px; border-radius: 4px; margin: 15px 0; border: 1px solid #E5E7EB; }}
        .action-steps ol {{ margin: 10px 0; padding-left: 20px; }}
        .cta {{ text-align: center; margin: 30px 0; }}
        .button {{ display: inline-block; padding: 12px 30px; background: #667EEA; color: white; text-decoration: none; border-radius: 6px; font-weight: 600; }}
        .footer {{ background: #F9FAFB; padding: 20px; text-align: center; font-size: 14px; color: #6B7280; border-top: 1px solid #E5E7EB; }}
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🔴 Action Needed</h1>
            <p style="margin: 5px 0 0 0; opacity: 0.9;">Your daily security update</p>
        </div>

        <div class="content">
            <p>Hi {user.email.split('@')[0]},</p>
            <p>We found <strong>{len(threats)} security issue{'s' if len(threats) > 1 else ''}</strong> affecting your devices. Here's what you need to know:</p>
"""

    for i, threat_data in enumerate(threats, 1):
        threat = threat_data['threat']
        translation = threat_data['translation']
        device = threat_data['device']
        stride = threat_data['stride']

        html += f"""
            <div class="threat {threat.severity}">
                <div>
                    <span class="severity {threat.severity}">{threat.severity}</span>
                    <span style="font-size: 20px; margin-left: 10px;">{stride['icon']}</span>
                </div>

                <h3 style="margin: 10px 0;">{translation['what']}</h3>

                <div class="device">
                    📱 Affects your: <strong>{device.nickname or device.device.product_name}</strong>
                </div>

                <p><strong>Who:</strong> {translation['who']}</p>
                <p><strong>Why it matters:</strong> {translation['why']}</p>

                <div class="action-steps">
                    <strong>What to do:</strong>
                    <ol>
"""

        # Parse action steps
        steps = translation['do'].split('\n')
        for step in steps:
            if step.strip():
                # Remove numbering if present
                step_text = step.lstrip('0123456789. ')
                html += f"                        <li>{step_text}</li>\n"

        html += f"""
                    </ol>
                </div>
            </div>
"""

    html += f"""
            <div class="cta">
                <a href="https://yourdomain.com/dashboard?user_id={user.id}" class="button">
                    View Full Report & Mark Actions
                </a>
            </div>

            <p style="font-size: 14px; color: #6B7280; margin-top: 30px;">
                <strong>💡 Pro tip:</strong> Mark your actions in the dashboard to track your security posture over time.
            </p>
        </div>

        <div class="footer">
            <p>You're receiving this because you signed up for Threat Intelligence Hub</p>
            <p><a href="https://yourdomain.com/settings" style="color: #667EEA;">Manage preferences</a> | <a href="https://yourdomain.com/unsubscribe" style="color: #667EEA;">Unsubscribe</a></p>
        </div>
    </div>
</body>
</html>
"""

    return html

def send_all_safe_email(user: User, db: Session):
    """
    Positive email when no threats found
    """
    html = """
<!DOCTYPE html>
<html>
<body style="font-family: sans-serif; line-height: 1.6;">
    <div style="max-width: 600px; margin: 20px auto; padding: 30px; background: white; border-radius: 8px;">
        <h2 style="color: #10B981;">✅ All Clear!</h2>
        <p>Good news! No new security threats affecting your devices in the past 24 hours.</p>
        <p style="color: #6B7280; font-size: 14px;">We're keeping watch. You'll hear from us if anything changes.</p>
    </div>
</body>
</html>
"""

    send_email(
        to=user.email,
        subject="✅ All Clear - Threat Intelligence Hub",
        html=html
    )

def send_email(to: str, subject: str, html: str):
    """
    Send email via SendGrid
    """
    import sendgrid
    from sendgrid.helpers.mail import Mail, Email, To, Content

    sg = sendgrid.SendGridAPIClient(api_key=os.getenv('SENDGRID_API_KEY'))

    mail = Mail(
        from_email=Email("alerts@threatintelhub.com", "Threat Intelligence Hub"),
        to_emails=To(to),
        subject=subject,
        html_content=Content("text/html", html)
    )

    try:
        response = sg.send(mail)
        print(f"Email sent: {response.status_code}")
    except Exception as e:
        print(f"Email error: {e}")

# Celery schedule configuration
celery.conf.beat_schedule = {
    'daily-digest': {
        'task': 'digest_generator.generate_daily_digest',
        'schedule': crontab(hour=8, minute=0),  # 8am daily
    },
    'follow-up-actions': {
        'task': 'digest_generator.send_follow_up_reminders',
        'schedule': crontab(hour=20, minute=0),  # 8pm daily
    },
}

@celery.task
def send_follow_up_reminders():
    """
    48-hour follow-up for unacted alerts (proof tracking)
    """
    db = SessionLocal()

    cutoff_time = datetime.now() - timedelta(hours=48)

    alerts = db.query(Alert).filter(
        Alert.status == 'sent',
        Alert.sent_at <= cutoff_time
    ).all()

    for alert in alerts:
        user = db.query(User).filter(User.id == alert.user_id).first()
        threat = db.query(Threat).filter(Threat.id == alert.threat_id).first()

        email_html = f"""
<html>
<body style="font-family: sans-serif;">
    <h3>⏰ Reminder: Security Action Needed</h3>
    <p>Hi {user.email.split('@')[0]},</p>
    <p>You haven't marked this security action as complete yet:</p>
    <p><strong>{threat.title}</strong></p>
    <p><a href="https://yourdomain.com/alerts/{alert.id}">Mark as complete</a></p>
</body>
</html>
"""

        send_email(
            to=user.email,
            subject="⏰ Security Reminder",
            html=email_html
        )

    db.close()
```

**Deliverable**: Celery workers running, daily digests sent to test users

---

### **Week 5-6: Frontend Integration**

#### **Week 5: Connect to Working API**

Now that backend is complete with real threat data, build frontend:

```javascript
// Example: React dashboard component
import React, { useState, useEffect } from 'react';
import axios from 'axios';

function Dashboard({ userId }) {
  const [threats, setThreats] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    axios.get(`/api/threats?user_id=${userId}`)
      .then(response => {
        setThreats(response.data);
        setLoading(false);
      });
  }, [userId]);

  if (loading) return <div>Loading...</div>;

  return (
    <div className="dashboard">
      <h1>Your Security Status</h1>

      {threats.length === 0 ? (
        <div className="all-clear">
          <h2>✅ All Clear!</h2>
          <p>No active threats affecting your devices</p>
        </div>
      ) : (
        <div className="threats">
          {threats.map(threat => (
            <ThreatCard key={threat.id} threat={threat} />
          ))}
        </div>
      )}
    </div>
  );
}

function ThreatCard({ threat }) {
  const [actionMarked, setActionMarked] = useState(false);

  const markAction = async (action) => {
    await axios.post(`/api/alerts/${threat.alert_id}/action`, {
      action: action,
      notes: ''
    });
    setActionMarked(true);
  };

  return (
    <div className={`threat-card ${threat.severity}`}>
      <span className="severity-badge">{threat.severity}</span>
      <h3>{threat.translation.what}</h3>

      <div className="device">
        📱 Affects: {threat.device_name}
      </div>

      <p><strong>Who:</strong> {threat.translation.who}</p>
      <p><strong>Why:</strong> {threat.translation.why}</p>

      <div className="actions">
        <h4>What to do:</h4>
        <ol>
          {threat.translation.do.split('\n').map((step, i) => (
            <li key={i}>{step}</li>
          ))}
        </ol>
      </div>

      {!actionMarked ? (
        <div className="action-buttons">
          <button onClick={() => markAction('patched')}>✅ I fixed this</button>
          <button onClick={() => markAction('no_action')}>Not applicable</button>
        </div>
      ) : (
        <div className="action-marked">✅ Action recorded</div>
      )}
    </div>
  );
}
```

**Key Integration Points**:
1. `/api/threats` - Get user's threats (filtered by devices)
2. `/api/alerts/{id}/action` - Mark actions (proof tracking)
3. `/api/devices/catalog` - Device selector
4. Authentication via Auth0

#### **Week 6: Deploy**

```bash
# Backend deployment (GCP Cloud Run)
gcloud run deploy tih-api \
  --source . \
  --region us-central1 \
  --allow-unauthenticated

# Frontend deployment (existing GCP setup)
# Connect frontend to Cloud Run API URL

# Set environment variables
gcloud run services update tih-api \
  --set-env-vars DATABASE_URL=$DB_URL \
  --set-env-vars OPENAI_API_KEY=$OPENAI_KEY \
  --set-env-vars SENDGRID_API_KEY=$SENDGRID_KEY
```

**Deliverable**: Fully functional MVP accessible at production URL

---

## Anticipated Issues & Mitigations

### **1. CPE Matching Accuracy (~70%)**

**Problem**:
- CPE strings are inconsistent between NVD and MISP
- ~30% of threats have no CPE data
- Version wildcards are complex to parse

**Impact**: 2 weeks of debugging if not anticipated

**Mitigation**:
- Start with exact matches only (accept 70% accuracy for MVP)
- Build comprehensive test suite with 50+ known CVE-device pairs
- Iterate based on user feedback: "Did you get alerted about threats that don't affect you?"
- Manual curation for top 20 most popular devices

**Success Metric**: <10% false positive rate

---

### **2. Translation Quality Inconsistency**

**Problem**:
- GPT-4 outputs vary in tone and technical level
- Some explanations still too technical
- Action steps sometimes generic

**Impact**: Users confused, low engagement

**Mitigation**:
- Human review first 100 translations
- Build template library for common CVEs (80% coverage over 3 months)
- User survey after each email: "Was this easy to understand? Yes/No"
- Iterate prompts based on comprehension feedback

**Success Metric**: 95%+ comprehension rate

---

### **3. MISP Setup Complexity**

**Problem**:
- MISP documentation is outdated
- Feeds don't ingest properly
- API returns empty results

**Impact**: 3-5 days lost troubleshooting

**Mitigation**:
- Use Docker image (not manual install)
- Join MISP community Slack for quick answers
- Start with CISA KEV only (most reliable feed)
- Test data ingestion within first 2 days - if broken, pivot to OpenCTI

**Contingency**: Use NVD API directly (no MISP) for MVP

---

### **4. Email Deliverability**

**Problem**:
- Emails go to spam
- SendGrid rejects templates
- Low open rates

**Impact**: 2-3 days fixing

**Mitigation**:
- Warm up SendGrid IP (send 50/day for 1 week before launch)
- Implement SPF, DKIM, DMARC records
- Test with Gmail, Outlook, Yahoo before launch
- Use plain text + HTML multipart emails
- Include clear unsubscribe link (CAN-SPAM compliance)

**Success Metric**: >40% open rate

---

### **5. GPT-4 API Cost**

**Problem**:
- At scale: 1,000 users × 10 threats/day × $0.027 = $270/day = $8,100/month
- Actually: Threats are deduplicated, so cost is per unique threat, not per user
- Realistic: 10 new unique threats/day × $0.027 = $0.27/day = $8/month

**Mitigation**:
- Cache ALL translations in database (implemented in schema)
- Build template library (80% coverage within 3 months)
- Use GPT-3.5-turbo for MVP ($0.002/translation = 15x cheaper)
- Switch to GPT-4 only for complex/novel threats

**Cost at Scale** (1,000 users with caching):
- 10 unique threats/day × 30 days × $0.027 = $8.10/month
- Revenue: 50 paying users × $4.99 = $249.50/month
- Margin: 97%

---

### **6. Database Performance**

**Problem**:
- Matching engine queries become slow at scale
- Daily digest generation times out

**Impact**: Digests delayed, poor UX

**Mitigation**:
- Add indices on `threat_device_matches.device_id` and `alerts.user_id` (done in schema)
- Batch digest generation (100 users per Celery task)
- Use Redis for caching threat translations
- Monitor query performance with `EXPLAIN ANALYZE`

**Success Metric**: Digest generation <5 minutes for 1,000 users

---

## Week 7-8: Launch Preparation

### Beta Testing
- Invite 20 users (friends, family, Twitter followers)
- Send daily digests for 1 week
- Collect feedback via Typeform survey
- Iterate on translation quality and action steps

### Metrics to Track
- Email open rate (target: 40%+)
- Click-through rate (target: 20%+)
- Action marked rate (target: 10%+)
- Unsubscribe rate (target: <5%)
- False positive reports

### Launch Checklist
- [ ] Domain purchased and configured
- [ ] SSL certificate installed
- [ ] Auth0 tenant configured
- [ ] SendGrid domain verified (SPF/DKIM)
- [ ] 100 devices in catalog
- [ ] MISP ingesting 4+ feeds
- [ ] Celery workers running
- [ ] Backup/restore procedures tested
- [ ] Privacy policy + Terms of Service published
- [ ] Product Hunt post drafted
- [ ] Landing page with signup form

---

## Week 9-12: Public Launch & Iteration

### Product Hunt Launch (Week 9)
- Post on Tuesday-Thursday (highest traffic)
- Include demo video (<90 seconds)
- Highlight: "Cyber threats in plain English. For the 97%."
- Goal: 500+ upvotes, 50+ signups

### Growth Tactics (Week 10-12)
- Reddit: r/cybersecurity, r/homelab, r/sysadmin (share as helpful resource)
- Twitter: Daily thread with real CVE translated to plain English
- Hacker News: "Show HN: Threat Intelligence for Non-Technical Users"
- NGO outreach: 10 cold emails/week

### Monetization (Week 11+)
- Add Stripe integration
- Pro tier gating (>5 devices)
- Track conversion rate (target: 1%+)
- First paid customer = validation

### Iteration Priorities
1. Fix false positives (based on user reports)
2. Add top 20 most-requested devices
3. Improve translation quality (template library)
4. Add proof tracking verification (screenshot upload)
5. Weekly summary email option

---

## Success Criteria (90-Day Validation)

**Signals to Continue**:
- ✅ 500+ signups
- ✅ 40%+ email open rate
- ✅ 5+ paying users ($25+ MRR)
- ✅ <10% unsubscribe rate
- ✅ Positive user feedback (NPS >30)

**Signals to Pivot**:
- ❌ <100 signups after Product Hunt launch
- ❌ <20% email open rate (users don't care)
- ❌ >30% unsubscribe rate (annoying, not valuable)
- ❌ 0 paying users after 3 months

**Signals to Kill**:
- ❌ No engagement after 2 growth attempts
- ❌ Consistently negative feedback
- ❌ Technical infeasibility (CPE matching <50% accuracy)

---

## Tech Stack Summary

**Backend**:
- FastAPI (Python async web framework)
- PostgreSQL (primary database)
- Redis (caching, job queues)
- Celery (background jobs)
- MISP/OpenCTI (threat intelligence platform)

**APIs**:
- OpenAI GPT-4 (translation)
- SendGrid (email delivery)
- Auth0 (authentication)

**Frontend** (existing):
- React (UI)
- TailwindCSS (styling)
- Hosted on GCP

**Infrastructure**:
- GCP Cloud Run (serverless backend)
- GCP Cloud SQL (PostgreSQL)
- GCP Memorystore (Redis)
- Docker (MISP deployment)

**Monitoring**:
- Sentry (error tracking)
- Google Analytics (user behavior)
- Custom dashboard (email metrics)

---

## Skills Required (1 = Easy, 5 = Expert)

| Skill | Difficulty | Can Learn? |
|-------|-----------|-----------|
| FastAPI basics | 2/5 | Yes - 1 week |
| PostgreSQL schema design | 3/5 | Yes - 1 week |
| MISP setup/configuration | 5/5 | Hard - join community |
| CPE matching algorithm | 5/5 | Hard - requires testing |
| GPT-4 prompt engineering | 3/5 | Yes - iterate |
| Celery/Redis | 4/5 | Moderate - 2 weeks |
| Email deliverability | 4/5 | Moderate - test early |
| React integration | 2/5 | Yes - if you have frontend |

**Recommendation**: Partner with a backend engineer for Weeks 3-4 (core logic) if you're not confident in CPE matching and MISP integration.

---

## Alternative: Hybrid Approach

If backend-first feels too complex, consider this hybrid:

**Phase 1 (2 weeks)**: Manual MVP
- Curate 10 threats/day manually from CISA KEV
- Translate to plain English yourself
- Send via Mailchimp (no code)
- 100 users × 10 devices = test matching logic manually

**Phase 2 (4 weeks)**: Automate backend
- Build matching engine once you understand patterns
- Add GPT-4 translation
- Deploy backend

**Benefit**: Validates demand before technical investment

---

## Questions to Resolve Before Starting

1. **Do you have a backend engineer?** If no, budget 2-3x timeline
2. **Which threat platform?** MISP (complex, powerful) vs OpenCTI (simpler) vs NVD API (simplest)
3. **Device catalog**: Will you curate 100 devices yourself? (8-10 hours of research)
4. **Email infrastructure**: Do you have SendGrid account? (needs 1 week warmup)
5. **Launch timeline**: 6 weeks (aggressive) or 12 weeks (comfortable)?

---

This implementation guide provides the complete technical roadmap for building TIH backend-first. Save this document as your single source of truth for the next 6-12 weeks.

**Next Step**: Start Day 1 - Deploy MISP with Docker.
