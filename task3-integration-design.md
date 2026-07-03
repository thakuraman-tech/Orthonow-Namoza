# Task 03 — Integration Design

## Integration Architecture End-to-End

To achieve a robust and scalable integration, I would architect this data flow using Make.com as the middleware connecting the custom frontend, HubSpot, and Karix.

* **Data Capture (Frontend)**: Upon form submission, the vanilla JavaScript prevents the default reload and triggers an asynchronous fetch request payload to a Make.com Webhook. Simultaneously, the frontend triggers the `window.dataLayer.push`. GTM listens for this event and immediately fires the Google Ads `consultation_form_submitted` conversion tag from the client side.
* **CRM Routing & The Deduplication Trap**: HubSpot natively deduplicates contacts using email addresses, not phone numbers. Since this lead-gen form only collects Name and Phone, relying on a standard HubSpot Form embed or a basic Zapier integration will either create massive duplicates or fail entirely. To solve this, Make.com will first execute a call to the HubSpot CRM Search API using the submitted phone number. If a match exists, Make.com updates the existing contact's details; if not, it uses the HubSpot Create API to generate a new contact, mapping the custom properties (Clinic Preference, Source = 'Google Ads - Consultation Landing Page', Lead Status = 'New Enquiry').
* **WhatsApp Dispatch**: Once the HubSpot module returns a success status, Make.com triggers an HTTP POST request directly to the Karix WhatsApp Business API to dispatch the confirmation template.

---

## Biggest Failure Point & Fallback

The single biggest point of failure is the middleware webhook dropping the payload due to network instability on the user's mobile device during submission. To build a strict fallback, I would implement `localStorage` caching in the frontend JavaScript. The script will save the form payload locally and continually retry the webhook fetch request until it receives a `200 OK` response, ensuring zero lead leakage due to transient drops.

---

## 2-Minute WhatsApp SLA: Risks & Monitoring

The Karix API dispatch could breach the 2-minute SLA due to Karix server latency, Make.com execution queue delays during high traffic, or invalid phone number formats causing immediate API rejections. To monitor this, I would implement an error-handler route within Make.com. If the Karix HTTP module times out or throws a `4XX`/`5XX` error, Make.com will instantly route an alert containing the failed payload to a dedicated Slack channel via webhook. This ensures the operations team is instantly aware of SLA breaches and can initiate a manual follow-up.
