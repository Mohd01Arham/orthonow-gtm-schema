# OrthoNow — GTM Event Schema (Task 01)

## Full Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience |
|---|---|---|---|
| `call_click` | Click – Just Links (`href` starts with `tel:`) | `page_location`, `click_url`, `clinic_name` | Events report broken down by `clinic_name` → identifies highest/lowest call-volume clinics |
| `whatsapp_click` | Click – Just Links (`href` contains `wa.me`) | `page_location`, `percent_scrolled`, `page_type` | Engagement report segmenting chat-intent by content type → feeds "warm lead" retargeting audience |
| `guide_form_submit` | Form Submission (scoped to guide form ID) | `form_id`, `lead_type`, `form_success` | Conversions report → "Guide Download Leads" audience for nurture campaigns |
| `guide_pdf_download` | Click – Just Links (Click URL contains `.pdf`) | `file_name`, `link_url`, `page_location` | Confirms actual asset delivery vs form-only submits → lead funnel data quality check |
| `clinic_page_view` | Page View | `page_path`, `city`, `clinic_name` | Views-per-clinic report → identifies which clinics get traffic vs which are ignored |
| `blog_scroll` | Scroll Depth (built-in: 25/50/75/90%) | `scroll_depth_threshold`, `article_title`, `page_path` | Engagement report → identifies high-retention content; feeds "engaged reader" audience |
| `booking_step_complete` (step 1) | Custom Event (dataLayer push — no page reload) | `step_number`, `step_name`, `clinic_location`, `specialty` | GA4 Funnel Exploration → Step 1 of booking funnel |
| `booking_step_complete` (step 2) | Custom Event (dataLayer push) | `step_number`, `step_name`, `preferred_date_selected`, `contact_info_provided` | GA4 Funnel Exploration → Step 2 of booking funnel |
| `booking_step_complete` (step 3) | Custom Event (dataLayer push) | `step_number`, `step_name`, `clinic_location`, `specialty`, `booking_id` | GA4 Funnel Exploration → Step 3 (final conversion) → **imported into Google Ads as a conversion action** |

A note on PII: `guide_form_submit` and booking step 2 never carry the patient's actual name or phone number. GA4/GTM (as a Google product) prohibits sending PII into the platform, and this is healthcare data, so it warrants extra care regardless. Only boolean success flags (`form_success`, `contact_info_provided`) are pushed. The real name/phone would be posted straight to the CRM/backend by the form's normal submit handler — a separate code path from the dataLayer push, not shown in this front-end demo since it depends on the actual CRM integration.

---

## Booking Funnel — Why It Needs Manual dataLayer Pushes

The 3-step booking form does not reload the page or submit a native `<form>` between steps — moving from step 1 to step 2 is just JavaScript swapping which panel is visible. GTM's built-in triggers (Page View, Click, Form Submission) only fire on real browser-level events, so none of them can detect a step change on their own.

The only way to track it: **the front-end developer manually pushes an object to `window.dataLayer`** at the exact moment each step completes, and GTM listens for it via a **Custom Event trigger** (`event equals booking_step_complete`).

### Step 1 — Clinic + Specialty Selected
```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Whitefield",
  "specialty": "Orthopedics"
}
```

### Step 2 — Contact Details Entered
```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "preferred_date_selected": true,
  "contact_info_provided": true
}
```

### Step 3 — Booking Confirmed
```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Whitefield",
  "specialty": "Orthopedics",
  "booking_id": "generated-by-backend"
}
```

### Surfacing Drop-off in GA4 Funnel Exploration

Each push fires a separate GTM Custom Event trigger, filtered by `step_number`, sending a GA4 event tag each time (all under the same GA4 event name so they form one funnel). In **GA4 Funnel Exploration**, define a 3-step funnel:
- Step 1 = `booking_step_complete` where `step_number = 1`
- Step 2 = same event where `step_number = 2`
- Step 3 = same event where `step_number = 3`

GA4 then shows the percentage of users who completed step 1 but never reached step 2, and step 2 but never reached step 3 — the drop-off view the performance marketing team needs.

---

## Google Ads Conversion Action

**`booking_step_complete` where `step_number = 3`** (booking confirmed) is the one event to import as a Google Ads conversion action — not step 1 or the guide download.

Google Ads' bidding algorithms (Target CPA, Maximize Conversions) optimize spend toward whatever is marked as a conversion. Importing a low-intent signal like "started the form" or "downloaded a guide" would train the algorithm to chase clicks that don't turn into revenue. Importing the confirmed booking (step 3) ensures Ads optimizes toward the action that actually matters commercially — a booked consultation.

---

## Implementation Notes (from `index.html`)

- `call_click`, `whatsapp_click`, and `clinic_page_view` are events GTM can capture **natively** with its built-in Click and Page View triggers — no custom dataLayer code is required for these in a real deployment. They are reproduced with manual JS in this single-file demo only because there's no live GTM container attached to fire against; in production, these three would need zero front-end dev involvement beyond adding `data-clinic`/`tel:`/`wa.me` markup that's already on the page.
- `guide_form_submit`, `guide_pdf_download`, and all three `booking_step_complete` events **do** require a developer to write the dataLayer push, because they depend on application state (form validity, which step is active) that only front-end JS has access to.
- A live `dataLayer` console is rendered at the bottom of the page (toggle button, bottom-left) purely for demo/walkthrough purposes, so the Loom recording can show events firing in real time without needing GTM Preview mode connected.
