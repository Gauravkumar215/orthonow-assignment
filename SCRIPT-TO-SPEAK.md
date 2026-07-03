# 🎤 Loom Script — Word for Word (Gaurav Kumar)

> Read this naturally, don't rush. Short pauses are fine. Total target: under 8 minutes.
> Keep 3 tabs open: (1) GitHub repo, (2) Live landing page, (3) Task 1 README.

---

## 🎬 INTRO (0:00 – 0:20)

"Hi, I'm Gaurav Kumar. This is my walkthrough for the Developer assignment for OrthoNow.
I'll cover three things — the GTM event schema, a live demo of the landing page firing the dataLayer push, and my integration design. Let's start."

---

## 🟦 TASK 1 — GTM Event Schema (0:20 – 2:15)

*(Open the GitHub repo → Task 1 README → show the event table.)*

"This is Task 1 — the full GTM event schema for OrthoNow.
For every key interaction on the site, I've defined the event name, the trigger type, at least three parameters, and which GA4 report or audience it feeds into.

You can see events for the booking form, the Call Now buttons, the WhatsApp widget, the patient guide download, clinic page views, and blog scroll depth.

One design choice — I attached `clinic_location` to almost every event. That's because OrthoNow runs separate local campaigns per clinic, so the marketing team needs to slice every interaction by clinic."

*(Scroll down to the booking funnel JSON section.)*

"Now the most important part — the 3-step booking form.
Here's a key point a lot of people get wrong: GTM cannot natively track steps inside a JavaScript multi-step form. GTM only reads the dataLayer — it doesn't know when a user moves from step 1 to step 2.

So the front-end developer has to push a custom dataLayer event at each step. I've written the exact JSON for each step right here, and even a short brief for the dev team on how to implement step 2.

GTM then listens with a Custom Event trigger per step, and in GA4 I build a Funnel Exploration using step_number to see exactly where users drop off."

*(Scroll to the Google Ads part.)*

"And for Google Ads, I'd import `booking_confirmed` as the conversion — because that's a real, confirmed appointment, not just a lead. It's also deduplicated by a booking ID, so it gives Smart Bidding the cleanest signal."

---

## 🟩 TASK 2 — Live Landing Page + dataLayer (2:15 – 5:15)  ⭐ MOST IMPORTANT

*(Switch to the live landing page tab.)*

"This is Task 2 — the replacement landing page. It's a single self-contained HTML file, no frameworks, and it scored 99 on PageSpeed Mobile.

The copy is written for the real audience — working professionals in Bengaluru with knee or back pain — so no generic health-clinic lines. It has one job: get the form filled."

*(Press F12 → click the Console tab.)*

"Let me open the browser console. Notice — on page load, the console is empty. Nothing fires yet. That's important: the conversion must fire on submit, not on load."

*(Fill Name + a valid 10-digit number → click the button.)*

"Now I'll fill in a name and a mobile number, and submit.

There it is — you can see `consultation_form_submitted` pushed to the dataLayer, with the parameters: clinic_preference, lead_source, and campaign. And the page switched to a thank-you state without any reload."

*(Type this in the console and press Enter.)*

"Let me confirm it's actually stored in the dataLayer."

`window.dataLayer.filter(e => e.event === 'consultation_form_submitted')`

"There's the object — so the push is real and captured.

And one more thing — validation runs first. If I enter an invalid number, the push does not fire at all."

*(Optional: quickly show an invalid number not firing.)*

"So it's fully wired up — not just visually there."

---

## 🟨 TASK 3 — Integration Design (5:15 – 7:40)

*(Switch to GitHub repo → Task 3 README → show the diagram.)*

"Task 3 is the integration design — connecting the form to HubSpot, Karix WhatsApp, and Google Ads.

Here's the flow: when the form is submitted, two things happen. The Google Ads conversion fires client-side, and the form data is sent to a serverless webhook. That webhook then talks to HubSpot and Karix by direct API.

I chose a direct API webhook over Zapier or Make, because those add polling delay — and that risks the 2-minute WhatsApp SLA. The webhook gives me full control over ordering and retries."

*(Pause — this is the trap they test.)*

"Now the important catch. HubSpot deduplicates contacts on email by default — not phone. And in this case, we don't collect email at all, which is common in Indian healthcare lead gen.

So if I left it default, every submit would create a new contact and the 'update if exists' rule would silently break.

My fix: make phone a unique property in HubSpot, or search by phone first and then create or update. And if two people submit the same phone with different names, they resolve to one contact — I keep the latest name, store the old one as an alias, and flag it for the team instead of overwriting.

The biggest failure point is the webhook-to-HubSpot-and-Karix step. So I write every raw lead to a durable queue first — that way, even if HubSpot or Karix is down, no lead is ever lost.

For the 2-minute WhatsApp SLA, I monitor it by comparing the submit time against Karix's delivery webhook, and alert if the gap crosses two minutes."

---

## 🎬 CLOSING (7:40 – 8:00)

"So that's all three tasks — the GTM schema, the working landing page with the dataLayer push, and the integration design. Everything is in the GitHub repo. Thanks for watching, and I'm happy to answer any follow-up questions."

---

### 💡 Quick tips
- Practice once fully before recording — it removes nervousness.
- If you run short on time, trim Task 1 slightly. Never trim the console demo or the HubSpot phone-dedup point — those are what they're really testing.
- Speak slowly and clearly. It's okay to pause and breathe.
