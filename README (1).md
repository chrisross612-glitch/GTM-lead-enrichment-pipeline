# GTM Lead Enrichment Pipeline

A lead enrichment and scoring pipeline that pulls ICP-fit companies from Apollo, enriches them with tech stack data via Clay, scores them based on sales tool signals, and routes them into Hot / Warm / Cold tiers for AE prioritization.

Built as part of a GTM Engineering portfolio by a former B2B SaaS AE with 6 years of quota-carrying experience.

---

## The problem this solves

Most outbound lists are flat — every company gets the same sequence regardless of how well they fit or how ready they are to buy. This pipeline fixes that by enriching raw Apollo exports with real buying signals and routing accounts into differentiated follow-up tracks before a single rep touches them.

---

## Pipeline architecture

```
Apollo.io (list building)
    ↓
Clay (enrichment — tech stack via BuiltWith)
    ↓
Scoring model (tech signal weighting)
    ↓
Tier classification (Hot / Warm / Cold)
    ↓
HubSpot CRM (Priority pipeline / Nurture sequence / Suppressed)
```

---

## ICP definition

- Industry: SaaS / Software
- Employee count: 10–50
- Geography: United States
- Contact titles: VP of Sales, Head of Sales, CRO

**Rationale:** Early-stage SaaS companies in this headcount range have built a sales motion but haven't yet invested in GTM infrastructure. They feel the pain of manual processes most acutely — making them the highest-converting segment for GTM tooling and services.

---

## Scoring model

Accounts are scored based on tech stack signals pulled from BuiltWith via Clay.

| Signal | Points | Rationale |
|---|---|---|
| 6+ keyword matches | 30 | Heavy investment in sales/marketing tools — active GTM motion |
| 3–5 keyword matches | 20 | Moderate investment — building toward a GTM stack |
| 1–2 keyword matches | 10 | Early stage — using basic tools only |
| 0 keyword matches | 0 | No detectable sales tool investment |

**Keywords tracked:** CRM, Sales Automation, Marketing Automation, Email Marketing, Sales Intelligence

**Tools commonly detected:** HubSpot, Salesforce, Outreach, SalesLoft, ActiveCampaign, Instantly, Intercom, Klaviyo, Brevo

---

## Tier classification

| Tier | Score | Routing |
|---|---|---|
| Hot | 30 | HubSpot Priority pipeline → AE queue for immediate outreach |
| Warm | 10–20 | HubSpot Nurture pipeline → automated drip sequence |
| Cold | 0 | Suppressed — no action, preserved for future re-enrichment |

---

## HubSpot integration (Python)

The script below reads the scored Clay export and pushes each record into HubSpot via the CRM API, routing accounts into the correct pipeline based on their tier.

### Requirements

```bash
pip install requests pandas
```

### Setup

1. Create a HubSpot Private App at Settings → Integrations → Private Apps
2. Enable scopes: `crm.objects.contacts` (read/write), `crm.objects.companies` (read/write)
3. Copy the access token (starts with `pat-`)
4. Export your scored Clay table as `project1_leads.csv`

### Script

```python
import requests
import pandas as pd

HUBSPOT_TOKEN = "your-pat-token-here"
HEADERS = {
    "Authorization": f"Bearer {HUBSPOT_TOKEN}",
    "Content-Type": "application/json"
}
BASE_URL = "https://api.hubapi.com"

def create_company(row):
    payload = {
        "properties": {
            "name": row["Company"],
            "website": row.get("Website", ""),
            "numberofemployees": row.get("Employees", ""),
            "industry": row.get("Industry", ""),
            "hs_lead_status": map_tier_to_status(row["Tier"])
        }
    }
    response = requests.post(
        f"{BASE_URL}/crm/v3/objects/companies",
        headers=HEADERS,
        json=payload
    )
    return response.json().get("id")

def create_contact(row, company_id):
    payload = {
        "properties": {
            "firstname": row.get("First Name", ""),
            "lastname": row.get("Last Name", ""),
            "email": row.get("Email", ""),
            "jobtitle": row.get("Title", ""),
            "hs_lead_status": map_tier_to_status(row["Tier"])
        }
    }
    response = requests.post(
        f"{BASE_URL}/crm/v3/objects/contacts",
        headers=HEADERS,
        json=payload
    )
    contact_id = response.json().get("id")

    # Associate contact with company
    if contact_id and company_id:
        requests.put(
            f"{BASE_URL}/crm/v3/objects/contacts/{contact_id}/associations/companies/{company_id}/contact_to_company",
            headers=HEADERS
        )
    return contact_id

def map_tier_to_status(tier):
    mapping = {
        "Hot": "IN_PROGRESS",    # Routes to Priority pipeline
        "Warm": "OPEN",           # Routes to Nurture pipeline
        "Cold": "UNQUALIFIED"     # Suppressed
    }
    return mapping.get(tier, "OPEN")

def push_to_hubspot(csv_path):
    df = pd.read_csv(csv_path)
    results = {"hot": 0, "warm": 0, "cold": 0, "errors": 0}

    for _, row in df.iterrows():
        try:
            company_id = create_company(row)
            create_contact(row, company_id)
            tier = str(row.get("Tier", "")).lower()
            if tier in results:
                results[tier] += 1
            print(f"✓ Pushed {row.get('Company', 'Unknown')} — Tier: {row.get('Tier')}")
        except Exception as e:
            results["errors"] += 1
            print(f"✗ Error on {row.get('Company', 'Unknown')}: {e}")

    print(f"\nComplete — Hot: {results['hot']} | Warm: {results['warm']} | Cold: {results['cold']} | Errors: {results['errors']}")

if __name__ == "__main__":
    push_to_hubspot("project1_leads.csv")
```

### What the script does

1. Reads your scored Clay CSV export
2. Creates a Company record in HubSpot for each row
3. Creates a Contact record and associates it with the company
4. Sets `hs_lead_status` based on tier — Hot routes to Priority, Warm to Nurture, Cold is suppressed
5. Logs each push with success/error status

---

## Results from this build

- 22 companies enriched and scored
- 19/22 (86%) returned tech stack keyword matches
- 5 accounts classified as Hot (score 30) — 6+ sales tool matches
- 14 accounts classified as Warm (score 10–20)
- 3 accounts classified as Cold (score 0) — no detectable sales tool investment

---

## Tools used

| Tool | Purpose | Cost |
|---|---|---|
| Apollo.io | List building and contact data | Free tier |
| Clay | Data enrichment via BuiltWith | Free tier |
| BuiltWith | Tech stack detection | Via Clay |
| HubSpot CRM | Pipeline and contact management | Free forever |
| Python + pandas | API integration script | Free |

Total cost to build: $0

---

## What an AE gets from this

- No more flat outreach — every account has a defined tier before the first touch
- Hot accounts surface immediately without manual research
- Warm accounts enter an automated nurture track without AE time investment
- Cold accounts are suppressed rather than wasting sequence slots
- Scoring logic is transparent and adjustable — change the weights as ICP evolves

---

## Author

Former B2B SaaS AE pivoting into GTM Engineering. Building in public.  
Notion Walkthrough: https://southern-mousepad-482.notion.site/notion-portfolio-page-373310708898800baa8be859661ccc0a?pvs=73
LinkedIn: https://www.linkedin.com/in/chris-ross-902b8b171/
