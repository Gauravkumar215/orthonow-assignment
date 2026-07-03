# OrthoNow — Developer Assignment (Web Dev & Martech)

Submission for **Namoza · Developer Position 1 (Client Web + Martech)**.
**Candidate:** Gaurav Kumar

**Client:** OrthoNow — a chain of 9 orthopaedic clinics across Bengaluru, Hyderabad and Chennai.

This repository contains all three deliverables. Each task lives in its own folder with its own README so the work is easy to review in isolation.

---

## Repository structure

```
.
├── README.md                          ← you are here (overview + Loom guide)
├── task-01-gtm-event-schema/
│   └── README.md                      ← Task 1: full GTM event schema (table + JSON)
├── task-02-landing-page/
│   ├── index.html                     ← Task 2: single self-contained landing page
│   └── README.md                      ← how to run + PageSpeed screenshot slot
└── task-03-integration-design/
    └── README.md                      ← Task 3: HubSpot + Karix + Google Ads writeup
```

---

## Deliverables at a glance

| # | Task | Deliverable | Location |
|---|------|-------------|----------|
| 01 | GTM Event Schema | Full event schema table, funnel tracking design, `dataLayer` JSON, Google Ads conversion pick | [`task-01-gtm-event-schema/`](./task-01-gtm-event-schema/README.md) |
| 02 | Landing Page Build | Single self-contained HTML file + PageSpeed screenshot | [`task-02-landing-page/`](./task-02-landing-page/README.md) |
| 03 | Integration Design | 300–400 word end-to-end architecture writeup | [`task-03-integration-design/`](./task-03-integration-design/README.md) |

---

## How to review Task 2 quickly

1. Open `task-02-landing-page/index.html` in any browser — no server required.
2. Open DevTools → **Console**.
3. Fill in Name + Phone and submit.
4. Watch `window.dataLayer` receive the `consultation_form_submitted` push (the console logs it on submit, **not** on page load).

```js
// Run this in the console after submitting to inspect the push:
window.dataLayer.filter(e => e.event === 'consultation_form_submitted')
```

---

## Loom walkthrough plan (max 8 min)

A ready-to-record script is in [`LOOM-SCRIPT.md`](./LOOM-SCRIPT.md):
- **0:00–2:00** — GTM schema decisions (Task 1)
- **2:00–5:00** — Live demo of the landing page firing the `dataLayer` push in the console (Task 2)
- **5:00–8:00** — Integration architecture answer (Task 3)

---

## Submission checklist

- [ ] Repo made public **or** shared with `himanshu@namoza.com`
- [ ] PageSpeed Insights (Mobile) screenshot added to `task-02-landing-page/` (must be 90+)
- [ ] Loom recorded (max 8 min, screen share of the actual repo + live page — no slides)
- [ ] Repo link + Loom link emailed to `naman@namoza.com`
- [ ] Subject line: `Developer Assignment - Gaurav Kumar`
