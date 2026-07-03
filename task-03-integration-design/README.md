# Task 03 — Integration Design (HubSpot + Karix + Google Ads)

**Scenario:** when a patient submits the consultation form, three things must happen automatically — (1) contact created/updated in HubSpot, (2) WhatsApp confirmation within 2 minutes via Karix, (3) `consultation_form_submitted` conversion fires in Google Ads.

---

## Written answer (≈ 380 words)

**Architecture, end-to-end.** I would **not** post the form directly from the browser to HubSpot, and I would **not** use Zapier/Make as the primary path. I'd route the submit through a **single serverless webhook** (Cloudflare Workers or AWS Lambda behind API Gateway) that acts as an orchestrator, calling each system by **direct API**. Flow:

1. **Browser →** the form fires the `consultation_form_submitted` `dataLayer` push (Google Ads/GA4 conversion via GTM) *and* sends a `fetch()` POST of `{name, phone, clinic_preference}` to the webhook.
2. **Webhook → Google Ads:** the client-side gtag conversion already fired, so this is decoupled and reliable even if the backend is slow.
3. **Webhook → HubSpot:** a **direct CRM API upsert** (see dedup below).
4. **Webhook → Karix:** direct WhatsApp Business API call using an approved template.

**Why direct API over Zapier/Make/native embed/Forms API:** the native HubSpot embed can't collect a clinic preference cleanly or trigger Karix, and Forms API still leaves WhatsApp/dedup unsolved. Zapier/Make add polling latency (risking the 2-min SLA) and an extra vendor to fail. A serverless function gives full control over ordering, retries, and — critically — the phone-based dedup logic.

**The dedup trap.** HubSpot deduplicates on **email by default, not phone** — and here we collect **no email**. Left alone, every submit creates a *new* contact and the "update if exists" requirement silently breaks. Fix: define **phone as a unique-value property** in HubSpot (or, more robustly, call the **Search API by phone first**, then `create` or `update` on the result). Business rule for *same phone, different name*: because phone is the identity key, both submits resolve to **one contact** — I'd keep the latest name, preserve the original in a "known aliases" field, and flag it for the care team rather than silently overwrite, since in Indian healthcare a shared family phone is common.

**Biggest failure point + fallback.** The single point of failure is the **webhook→HubSpot/Karix leg**. I'd make the webhook write every raw submission to a **durable queue/store first** (e.g., SQS or a backup Google Sheet/DB) *before* calling downstream APIs, with retries and a dead-letter queue — so a HubSpot/Karix outage never loses a lead.

**WhatsApp 2-min SLA.** Breakers: Karix downtime, unapproved template, rate limits, or queue backlog. Monitor by comparing `submit_timestamp` against Karix's **delivery-status webhook**, alerting when the delta exceeds 2 minutes, plus a synthetic hourly test submission.

---

## Supporting diagram

```
                         ┌─────────────────────────────┐
                         │  Landing page (browser)      │
                         │  form submit                 │
                         └───────────┬─────────────────┘
                                     │
              ┌──────────────────────┼───────────────────────────┐
              │ (a) dataLayer push   │ (b) fetch() POST           │
              ▼                      ▼                            │
      ┌───────────────┐     ┌──────────────────────┐             │
      │ GTM → GA4 +   │     │  Serverless webhook   │             │
      │ Google Ads    │     │  (Cloudflare/Lambda)  │             │
      │ conversion    │     └──────────┬───────────-┘             │
      └───────────────┘                │                          │
                        1. write raw submission to durable store  │
                           (SQS / DB / backup sheet) ─────────────┘
                                       │
                        2. HubSpot: search-by-phone → upsert
                                       │
                        3. Karix: send WhatsApp template (≤2 min)
                                       │
                        ← Karix delivery webhook → SLA monitor
```

## Why the client-side conversion is decoupled from the backend

Firing the Google Ads conversion client-side (via GTM, on the `consultation_form_submitted` event) means bidding data is captured **even if HubSpot or Karix are down**. The lead is still preserved by the durable store in step 1. This is the core resilience idea: the three actions must not be a single fragile synchronous chain.

## HubSpot contact payload (reference)

```json
{
  "properties": {
    "firstname": "Rohan",
    "phone": "9876543210",
    "clinic_preference": "Indiranagar, Bengaluru",
    "hs_lead_status": "NEW",
    "lead_status_custom": "New Enquiry",
    "lead_source": "Google Ads - Consultation Landing Page"
  }
}
```

> `phone` must be configured as a **unique property** (Settings → Properties → *Contact information* → edit `phone` → "Require unique values"), **or** dedup is handled explicitly via the Search API in the webhook. Without one of these, "update if exists" does not work because HubSpot's native dedup is email-only.
