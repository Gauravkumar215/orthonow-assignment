# Loom Walkthrough Script (max 8 minutes)

No slides. Screen-share your actual repo and the live page. Keep it conversational — they will ask follow-ups, so speak to *why* you made each decision, not just *what* it is.

---

## Part 1 — GTM schema decisions (0:00 – 2:00)

- Open `task-01-gtm-event-schema/README.md`. Scroll the event table.
- Say: "I built one consistent schema — `snake_case` events, and I attach `clinic_location` to almost everything because OrthoNow runs local campaigns per clinic."
- **Hit the key point they're testing:** "The important one is the 3-step booking funnel. GTM cannot natively see steps inside a JavaScript multi-step form — it only reads the dataLayer. So the front-end dev has to push these events. I wrote the exact JSON and the dev brief for step 2 here." → scroll to the JSON blocks + the dev brief.
- "For Google Ads I'd import `booking_confirmed`, not the form-submit lead — it's the only event that means a real booked appointment, and it's deduplicated by `booking_id`."

## Part 2 — Live landing page + dataLayer push (2:00 – 5:00)

- Open `task-02-landing-page/index.html` in the browser. Open DevTools → **Console**.
- "Notice the console is quiet on load — nothing fires on page load."
- Type a name and a 10-digit number. Submit.
- Point at the console: "There's the push — `consultation_form_submitted` with clinic_preference, lead_source, campaign. And the page swapped to the thank-you state with no reload."
- Run in console: `window.dataLayer.filter(e => e.event === 'consultation_form_submitted')` → show the object.
- "It fires only on a valid submit — try an invalid phone and nothing pushes." (Demo it.)
- Mention performance: "Single file, no web fonts, no images, so it hits 90+ on PageSpeed Mobile — screenshot's in the repo."

## Part 3 — Integration architecture (5:00 – 8:00)

- Open `task-03-integration-design/README.md`, show the diagram.
- "Form submit does two things: fires the Google Ads conversion client-side, and POSTs to a serverless webhook. I picked a direct API webhook over Zapier/Make because polling risks the 2-minute WhatsApp SLA and adds a vendor to fail."
- **Hit the trap they're testing:** "The catch is HubSpot dedupes on *email* by default — and we collect no email. So I either make `phone` a unique property or search-by-phone before upsert. If two patients share a phone with different names, they resolve to one contact — I keep the latest name, store the alias, and flag it, because shared family phones are common in Indian healthcare."
- "Biggest failure point is the webhook→HubSpot/Karix leg, so I write every raw lead to a durable queue first — a HubSpot outage never loses a lead. I monitor the WhatsApp SLA by comparing submit time to Karix's delivery webhook and alerting past 2 minutes."

---

### Timing tip
Practice once. If you run long, trim Part 1 — the two things they most want to see live are the **dataLayer push firing in the console** (Part 2) and your **HubSpot phone-dedup reasoning** (Part 3).
