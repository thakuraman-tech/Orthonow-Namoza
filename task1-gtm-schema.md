# Task 01 — GTM Event Schema

## GTM Event Measurement Schema

This schema maps user interactions across the OrthoNow web properties to Google Tag Manager (GTM) triggers and Google Analytics 4 (GA4) parameters and reports.

| Event Name | Trigger Type | Key Parameters (min 3) | Target GA4 Report / Audience |
| :--- | :--- | :--- | :--- |
| `call_now_click` | **Link Click** (Just Links)<br>• Click URL starts with `tel:` | 1. `click_url` (e.g. `tel:+9180...`) <br>2. `link_text` (e.g. "Call Now") <br>3. `page_location` (Current URL) | **Report**: *Engagement > Events*<br>**Audience**: *Click-to-Call Prospects* (High local mobile intent) |
| `whatsapp_click` | **Link Click** (Just Links)<br>• Click URL contains `wa.me` or `api.whatsapp.com` | 1. `click_url`<br>2. `whatsapp_number`<br>3. `page_location` | **Report**: *Engagement > Events*<br>**Audience**: *Chat Initiators* (Active inquiry/conversational intent) |
| `pdf_lead_submission` | **Custom Event** (fired on form success)<br>• Event name equals `pdf_lead_submission` | 1. `form_id` (e.g. `knee_exercises_pdf`) <br>2. `pdf_title` (e.g. "9 Knee Exercises for Back Pain") <br>3. `page_location` | **Report**: *Engagement > Conversions*<br>**Audience**: *Information Seekers* (Top-of-Funnel nurturing pool) |
| `page_view` | **Page View** (DOM Ready)<br>• Page Path matches `/locations/*` | 1. `page_type` (value: `clinic_location`) <br>2. `clinic_name` (e.g. `Indiranagar`, `Jayanagar`) <br>3. `clinic_city` (value: `Bengaluru`) | **Report**: *Engagement > Pages and screens* (Filtered by `page_type`) <br>**Audience**: *Local High-Intent Visitors* |
| `blog_scroll_progress` | **Scroll Depth**<br>• Vertical scroll percentages: `10, 25, 50, 75, 90`<br>• Page Path matches `/blog/*` | 1. `blog_title` (extracted from `h1`) <br>2. `scroll_depth_percent` (e.g. `75`) <br>3. `page_location` | **Report**: *Engagement > Events* (Analyze content retention)<br>**Audience**: *Engaged Blog Readers* (Scroll depth &ge; 75%) |
| `booking_step_1_complete` | **Custom Event**<br>• Event name equals `booking_step_1_complete` | 1. `form_id` (e.g. `orthonow_booking_funnel`) <br>2. `step_name` (value: `personal_details`) <br>3. `user_status` (e.g. `new_patient`) | **Report**: *Explore > Funnel Exploration* (Step 1)<br>**Audience**: *Booking Funnel Initiators* |
| `booking_step_2_complete` | **Custom Event**<br>• Event name equals `booking_step_2_complete` | 1. `form_id` (e.g. `orthonow_booking_funnel`) <br>2. `step_name` (value: `clinic_and_specialty`) <br>3. `selected_clinic` (e.g. `Indiranagar`) | **Report**: *Explore > Funnel Exploration* (Step 2)<br>**Audience**: *Mid-Funnel Warm Leads* |
| `booking_complete` | **Custom Event**<br>• Event name equals `booking_complete` | 1. `form_id` (e.g. `orthonow_booking_funnel`) <br>2. `step_name` (value: `confirmation`) <br>3. `booking_id` (e.g. `BK-984210`) | **Report**: *Engagement > Conversions* & *Explore > Funnel Exploration*<br>**Audience**: *Booked Patients* (Exclude from acquisition ads) |

---

## 3-Step Booking Form dataLayer.push Payloads

To ensure accurate tracking, the front-end application must push structured data directly into the `dataLayer` at the successful validation and transition of each step.

### Step 1: Personal Details Completion
*Fired when the user enters their name, contact details, and clicks "Next".*
```json
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  "event": "booking_step_1_complete",
  "form_id": "orthonow_booking_funnel",
  "step_name": "personal_details",
  "user_status": "new_patient"
});
```

### Step 2: Clinic & Specialty Selection
*Fired when the user selects their preferred clinic location (from the 9 locations) and medical specialty, then clicks "Next".*
```json
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  "event": "booking_step_2_complete",
  "form_id": "orthonow_booking_funnel",
  "step_name": "clinic_and_specialty",
  "selected_clinic": "Indiranagar",
  "selected_specialty": "Knee Specialist"
});
```

### Step 3: Booking Confirmation
*Fired on final submission when the appointment is successfully saved in the backend and the confirmation screen appears.*
```json
window.dataLayer = window.dataLayer || [];
window.dataLayer.push({
  "event": "booking_complete",
  "form_id": "orthonow_booking_funnel",
  "step_name": "confirmation",
  "booking_id": "BK-984210",
  "selected_clinic": "Indiranagar",
  "selected_specialty": "Knee Specialist",
  "appointment_type": "In-Person Consultation"
});
```

---

## Technical Integration & GA4 Funnel Analysis

### Critical Technical Note for Developers
> [!IMPORTANT]
> **GTM cannot natively detect multi-step form progression.** Traditional GTM form triggers (like standard Form Submission triggers) only listen to the browser's DOM `submit` event, which usually fires once at the very end of the process or fails entirely in client-side routed forms.
>
> Therefore, **the front-end developer must explicitly invoke each step's `dataLayer.push` via custom JavaScript logic on successful client-side validation of that specific step.** Do not attempt to infer steps via element visibility or next button clicks, as this leads to false conversion data when validation fails.

### Analyzing Funnel Drop-off in GA4
To analyze user drop-off across this booking journey, navigate to **Explore > Funnel Exploration** in GA4.

1. **Funnel Steps Configuration**:
   - **Step 1**: `booking_step_1_complete`
   - **Step 2**: `booking_step_2_complete`
   - **Step 3**: `booking_complete`
2. **Open vs. Closed Funnels**:
   - **Closed Funnel (Recommended)**: A user must enter the funnel at Step 1 (`booking_step_1_complete`) to be included in subsequent step metrics. This is the correct configuration for this booking form because the steps are sequential and mandatory; you cannot choose a clinic (Step 2) or get a confirmation (Step 3) without providing basic contact details first.
   - **Open Funnel**: Users can enter at any stage. This would show skewed metrics if users could bookmark or jump straight to Step 2, which is not possible in this application architecture.
3. **Breakdown Dimension**:
   - Apply a breakdown dimension like **"Session source/medium"** to identify which ad campaigns, referral sites, or organic keywords drive the highest drop-off at Step 1 or Step 2.
   - Apply **"selected_clinic"** (configured as a custom dimension in GA4) as a breakdown to see if specific clinic branches experience higher completion friction or lower booking interest.

---

## Google Ads Conversion Strategy

### Recommended Conversion Action to Import
The event chosen for Google Ads optimization is **`booking_complete`** (Step 3 confirmation).

### Strategic Justification
1. **Strongest Bottom-Funnel Signal**:
   - Unlike top-of-funnel signals (such as `pdf_lead_submission` which represents a user searching for home-based exercises or reading material), or mid-funnel interactions (such as `booking_step_1_complete` which can have high drop-offs due to reluctance to proceed), `booking_complete` represents a completed booking with specific location details, and a generated booking ID.
2. **Revenue-Aligning Smart Bidding**:
   - Feeding `booking_complete` into Google Ads Smart Bidding (e.g., Target CPA or Maximize Conversions) forces Google's bidding algorithm to optimize for actual patients scheduled in clinics, rather than accidental phone clicks or casual window-shoppers.
3. **Comparison Against Call/WhatsApp Leads**:
   - `call_now_click` and `whatsapp_click` represent strong intent, but they are *intermediated* steps. A significant portion of callers drop off due to busy lines, wrong numbers, simple direction requests, or price checks. 
   - Optimizing bids for clicks on these widgets would dilute bid optimization toward non-converting traffic. `booking_complete` provides a clean, concrete database-validated lead that translates directly into clinic footfall.
