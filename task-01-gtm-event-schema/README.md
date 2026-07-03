# Task 01 — GTM Event Schema for OrthoNow

**Goal:** Implement complete event tracking in Google Tag Manager (GTM) so GA4 captures every meaningful interaction *before* paid campaigns go live. Right now GA4 only has pageviews — the marketing team is flying blind.

**Naming convention used:** all events are `snake_case`, verb-led where possible, and every event carries a consistent set of context parameters (`page_type`, `clinic_location` where relevant) so segmentation is uniform across GA4.

---

## 1. Complete event schema

> `Trigger Type` = the GTM trigger that fires the tag.
> `Key Parameters` = event parameters sent to GA4 (registered as **custom dimensions** in GA4 Admin so they are queryable). Minimum 3 per event.

| # | Event Name | Trigger Type (GTM) | Key Parameters (min 3) | Feeds into (GA4 report / audience / conversion) |
|---|------------|--------------------|------------------------|--------------------------------------------------|
| 1 | `booking_step_view` | Custom Event — `booking_step_view` (dataLayer push from front-end) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (step entrances) |
| 2 | `booking_step_complete` | Custom Event — `booking_step_complete` (dataLayer push from front-end) | `step_number`, `step_name`, `clinic_location`, `specialty` | Funnel Exploration (step completions / drop-off) |
| 3 | `booking_confirmed` | Custom Event — `booking_confirmed` (dataLayer push on step 3 success) | `booking_id`, `clinic_location`, `specialty`, `preferred_date` | **Key Event / Conversion** → imported to Google Ads. "Booked patients" audience |
| 4 | `call_now_click` | Click — *Just Links* / *All Elements*, filtered on `tel:` href or `.btn-call` class | `click_location` (homepage/clinic/landing), `clinic_location`, `phone_number` | Key Event (secondary). "High-intent callers" audience |
| 5 | `whatsapp_click` | Click — *Just Links*, filtered on `Click URL contains wa.me` | `click_location`, `clinic_location`, `link_url` | Engagement report. "WhatsApp enquirers" audience |
| 6 | `guide_form_submit` | Custom Event — `guide_form_submit` (dataLayer push on gated form submit) | `form_name` (`patient_guide`), `clinic_location`, `guide_topic` | Key Event (lead). "Guide leads" audience |
| 7 | `patient_guide_download` | Custom Event — `file_download` / GA4 enhanced measurement (PDF link) | `file_name`, `file_extension` (`pdf`), `guide_topic` | Engagement report (content interest) |
| 8 | `clinic_page_view` | Page View — trigger fires on URL path `/clinics/` (or History Change for SPA nav) | `clinic_location`, `clinic_city`, `page_type` (`clinic_location`) | Landing-page / location report. Per-clinic geo audiences for local campaigns |
| 9 | `blog_article_read` | Scroll Depth — vertical, thresholds 25/50/75/90% on `/blog/` paths | `article_title`, `scroll_percentage`, `page_type` (`blog_article`) | Content engagement report. "Engaged readers (75%+)" remarketing audience |
| 10 | `consultation_form_submitted` | Custom Event — `consultation_form_submitted` (dataLayer push from landing page, see Task 2) | `clinic_preference`, `form_location` (`consultation_landing`), `lead_source` | **Key Event / Conversion** — landing-page campaign optimisation |

**Notes on why these parameter choices matter**

- `clinic_location` is attached to almost every event because OrthoNow runs local campaigns per city — the marketing team must be able to slice every interaction by clinic.
- `click_location` on `call_now_click` distinguishes a call from the homepage vs a call from the paid landing page — those are very different intent signals for bidding.
- Parameters are only useful in GA4 if registered as **custom dimensions** (Admin → Custom definitions). All non-standard params above must be registered, or they won't be reportable.

---

## 2. Booking form funnel — step-level drop-off tracking

### The critical point (and the honest answer to "who writes the dataLayer push")

**GTM cannot natively detect steps inside a JavaScript multi-step form.** A 3-step booking form usually renders all steps in the same page/DOM and swaps them with JS — there is no page navigation and often no unique submit event GTM can hang a trigger on. GTM only *reads* the `dataLayer`; it does not *know* a user moved from step 1 to step 2.

So the `dataLayer` pushes below **must be written by the front-end developer** inside the form's step-transition logic. My job as the martech/dev owner is to (a) define the exact schema, (b) write the pushes if I own the front-end, and (c) hand the front-end team a precise brief if they own it. GTM then listens with a **Custom Event trigger** per step. Anyone who assumes GTM auto-tracks this has never wired a funnel from scratch.

### dataLayer pushes (actual JSON, one per step)

**Step 1 — clinic location + specialty selected** (push when the user advances past step 1):

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint"
}
```

**Step 2 — name / phone / preferred date entered** (push when the user advances past step 2):

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-10"
}
```

**Step 3 — booking confirmed** (push on the success response after final submit):

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "booking_id": "ON-2026-004821",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-10"
}
```

> Note: no PII (name/phone) is pushed to the `dataLayer` — GA4 must not receive personal data. The CRM captures identity server-side (see Task 3).

### GTM triggers at each step

| Step | GTM Trigger | Fires GA4 event |
|------|-------------|-----------------|
| 1 | Custom Event trigger, Event name = `booking_step_complete`, condition `step_number equals 1` | `booking_step_complete` |
| 2 | Custom Event trigger, Event name = `booking_step_complete`, condition `step_number equals 2` | `booking_step_complete` |
| 3 | Custom Event trigger, Event name = `booking_confirmed` | `booking_confirmed` |

One GA4 event tag can serve all three, reading `step_number` / `step_name` from Data Layer Variables — cleaner than three separate tags.

### Surfacing drop-off in GA4 Funnel Exploration

1. Register `step_number` and `step_name` as **custom dimensions** in GA4.
2. In **Explore → Funnel exploration**, build an **open funnel** with three ordered steps:
   - Step 1: event `booking_step_complete` where `step_number = 1`
   - Step 2: event `booking_step_complete` where `step_number = 2`
   - Step 3: event `booking_confirmed`
3. Add `clinic_location` and `specialty` as **breakdowns** to see which clinics/specialties leak the most.
4. Enable **"Show elapsed time"** to spot where users stall, and set the funnel to **trended** to watch drop-off improve after landing-page/form changes.

The step-to-step completion percentages GA4 shows are the drop-off. Example read: if 100 hit step 1, 62 complete step 2, 44 confirm → the biggest leak is step 1→2 (the details form), which points the fix at reducing form friction.

### Brief I would hand the front-end dev for Step 2

> "On the step-2 'Continue' handler, **after** client-side validation passes and **before** you render step 3, call:
> `window.dataLayer = window.dataLayer || []; window.dataLayer.push({event:'booking_step_complete', step_number:2, step_name:'patient_details_entered', clinic_location: <selected clinic string>, specialty: <selected specialty>, preferred_date: <YYYY-MM-DD>});`
> Do **not** include the patient's name or phone. Fire it exactly once per step advance (guard against double-fires on re-click). `clinic_location` and `specialty` come from the state selected in step 1 — carry them forward."

---

## 3. Conversion action to import into Google Ads

**Import `booking_confirmed`** as the primary Google Ads conversion.

**Why this one over the others:**

- It is the **only bottom-of-funnel event that represents a real, committed appointment** — the actual business outcome OrthoNow is paying for. `consultation_form_submitted`, `call_now_click`, `whatsapp_click` and `guide_form_submit` are all *leads* (intent), not booked patients. Optimising bids toward a mid-funnel lead teaches Smart Bidding to chase volume of enquiries, many of which never convert to a booking.
- It is **deduplicated and unambiguous** — it carries a unique `booking_id`, so it won't double-count. Phone-call clicks (`call_now_click`) can't confirm an actual conversation happened, and can be inflated by mis-clicks.
- It gives Smart Bidding (tCPA / tROAS) the **cleanest optimisation signal** tied to revenue.

**Caveat I'd raise with the team:** `booking_confirmed` volume may initially be too low for Smart Bidding to learn (needs ~15–30 conversions/month). If so, I'd import `consultation_form_submitted` as a **secondary** conversion for early-stage optimisation, then switch the primary to `booking_confirmed` once volume is sufficient — and use conversion values to weight bookings above raw leads.
