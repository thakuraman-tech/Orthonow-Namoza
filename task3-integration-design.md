# Task 03 — Integration Design

## Data Flow Architecture

The data pipeline connects the landing page submission to CRM storage, automated messaging, and campaign attribution:

```
[Landing Page] 
   └── (HTTPS Webhook POST) 
        └── [Make.com]
             ├── 1. HubSpot Search API (Find Contact by Phone)
             ├── 2. HubSpot Upsert Contact & Create Deal
             ├── 3. Karix WhatsApp API (Send Template Trigger)
             └── 4. Google Ads API (Upload Offline Conversion)
```

### Make.com Middleware Justification
Using Make.com (or Zapier) as an integration middleware is selected over direct frontend-to-API calls or native form embeds to:
1. **Secure API Credentials**: Prevent exposing HubSpot, Karix, and Google Ads bearer tokens to the browser.
2. **Decouple Architecture**: Ensure the landing page loads instantly without heavy SDK scripts.
3. **Queue Resilience**: Buffer webhook failures and rate-limiting errors natively.

---

## CRM Deduplication & Edge Case Strategy

HubSpot natively deduplicates contacts using the `email` property. Because this campaign collects only `name` and `phone`, the standard deduplication is bypassed. 

### Search API Deduplication Logic
Make.com queries the HubSpot CRM Search API:
* **Endpoint**: `POST https://api.hubapi.com/crm/v3/objects/contacts/search`
* **Payload**:
  ```json
  {
    "filterGroups": [{
      "filters": [{
        "propertyName": "phone",
        "operator": "EQ",
        "value": "+918040005000"
      }]
    }]
  }
  ```

### Edge Case: Same Phone, Different Names (e.g., Family Members)
Overwriting the existing contact name corrupts historic database integrity, while rejecting the submission loses new patients. 
* **Our Resolution**:
  1. If the contact exists, **retain the original name** on the primary record.
  2. Map the new name to a custom auditing text field `latest_submitted_name` and check a boolean flag `name_discrepancy_detected`.
  3. Create a **new Deal** associated with that Contact ID, titled `[New Name] - Consultation Request`.
  4. This surfaces a discrepancy flag for the triage team to confirm details over the phone, maintaining database integrity.

---

## Failure Points & SLA Mitigation

### Biggest Failure Point: Karix WhatsApp API Timeout
If the Karix API is unresponsive or rate-limited, the 2-minute SLA for automated patient outreach will be breached.

### Fallback Design (Retry Queue)
Make.com captures the payload in a native datastore queue first. If the Karix API call fails, the scenario activates an **exponential backoff retry loop** (3 attempts at 10s, 30s, and 60s). If the error persists, the payload falls back to an **alternative SMS gateway** (e.g., Twilio) within 90 seconds.

### SLA Monitoring & Alerting
We configure an automated checkpoint filter. If `now() - submission_time > 90 seconds` and WhatsApp delivery status is not `delivered`, Make.com triggers an instant webhook to Slack and the clinic dashboard, alerting the team to place a manual call immediately.
