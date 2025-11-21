# leads_bakersfield.py
import requests
import json
import time
import csv
from datetime import datetime, timedelta
import os
from typing import List, Dict

# 1. Get new businesses from California SOS
def get_new_california_businesses(days_back=180):
    url = "https://bizfileonline.sos.ca.gov/api/report/businesssearch"
    end_date = datetime.now()
    start_date = end_date - timedelta(days=days_back)
    
    payload = {
        "searchType": "ENTITY_NAME",
        "searchSubType": "GENERAL",
        "filingDateFrom": start_date.strftime("%Y-%m-%d"),
        "filingDateTo": end_date.strftime("%Y-%m-%d"),
        "jurisdiction": ["CA"],
        "entityType": []  # all
    }
    
    # This is a public endpoint; some people reverse-engineered it
    # Alternative: use https://opencorporates.com with reconcile
    # I'll use OpenCorporates (free & legal)
    pass  # we'll switch to OpenCorporates below

# Proper OpenCorporates version (recommended)
def search_opencorporates(city="Bakersfield", state="CA", days=365):
    base = "https://api.opencorporates.com/v0.4/companies/search"
    leads = []
    page = 1
    
    while True:
        url = f"{base}?q=&jurisdiction_code=us_ca&per_page=100&page={page}&registered_address=*Bakersfield*"
        r = requests.get(url, timeout=20)
        data = r.json()
        companies = data['results']['companies']
        if not companies:
            break
            
        for c in companies:
            company = c['company']
            inception = company.get('incorporation_date') or company.get('creation_date')
            if inception:
                inc_date = datetime.strptime(inception, "%Y-%m-%d")
                age_days = (datetime.now() - inc_date).days
                if age_days > days:
                    continue
                    
            leads.append({
                "name": company['name'],
                "number": company.get('company_number'),
                "address": company.get('registered_address_in_full', ''),
                "status": company.get('status'),
                "age_days": age_days if 'age_days' in locals() else None,
                "source": "OpenCorporates"
            })
        page += 1
        time.sleep(1)  # be nice
    return leads

# 2. Enrich with Google Places (website + GBP check)
def check_google_place(business_name, address):
    # You need a Google Places API key
    API_KEY = os.getenv("GOOGLE_PLACES_KEY")
    if not API_KEY:
        return {"has_website": None, "gbp_claimed": None}
        
    url = "https://maps.googleapis.com/maps/api/place/findplacefromtext/json"
    params = {
        "input": f"{business_name} {address} Bakersfield CA",
        "inputtype": "textquery",
        "fields": "website,formatted_address,place_id,user_ratings_total,photos,rating,business_status",
        "key": API_KEY
    }
    r = requests.get(url, params=params)
    data = r.json()
    candidates = data.get("candidates", [])
    if candidates:
        cand = candidates[0]
        return {
            "has_website": bool(cand.get("website")),
            "gbp_photos": len(cand.get("photos", [])) if cand.get("photos") else 0,
            "reviews": cand.get("user_ratings_total", 0)
        }
    return {"has_website": False, "gbp_claimed": False, "reviews": 0}

# 3. Scoring engine
def score_lead(lead: Dict) -> int:
    score = 0
    
    age = lead.get("age_days", 999)
    if age <= 90:
        score += 30
    elif age <= 365:
        score += 15
        
    high_need_keywords = ["restaurant", "cafe", "contractor", "plumbing", "hvac", "dentist", "attorney", "salon", "auto repair", "cleaning"]
    name_lower = lead['name'].lower()
    if any(k in name_lower for k in high_need_keywords):
        score += 20
        
    if lead.get("has_website") == False:
        score += 25
    elif lead.get("has_website") == True:
        score -= 15
        
    if lead.get("gbp_photos", 0) < 5:
        score += 20
    if lead.get("reviews", 0) < 10:
        score += 10
        
    return min(100, score)

# Main runner
def main():
    raw_leads = search_opencorporates(days=180)
    enriched = []
    
    for lead in raw_leads[:50]:  # limit for testing
        print(f"Checking {lead['name']}")
        google = check_google_place(lead['name'], lead['address'])
        lead.update(google)
        lead['score'] = score_lead(lead)
        enriched.append(lead)
        time.sleep(2)  # stay under Google rate limit
    
    # Sort by score
    enriched.sort(key=lambda x: x['score'], reverse=True)
    
    # Save
    with open(f"bakersfield_leads_{datetime.now().strftime('%Y%m%d')}.csv", "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=["name","address","age_days","has_website","gbp_photos","reviews","score","number"])
        writer.writeheader()
        writer.writerows(enriched)
        
    print("Done! Hot leads:")
    for l in enriched[:10]:
        print(f"{l['score']} pts – {l['name']} ({l.get('age_days','?')} days old)")

if __name__ == "__main__":
    main()
