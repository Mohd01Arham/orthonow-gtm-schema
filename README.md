# OrthoNow — GTM Event Schema & Landing Page (Task 01)
- **`index.html`** — self-contained landing page (HTML/CSS/JS, no build step) implementing:
  - Hero + booking CTA
  - 9 clinic cards (stand-in for 9 real clinic pages)
  - 3-step booking form (clinic/specialty → contact details → confirm)
  - Call Now links (header, hero, per-clinic)
  - Floating WhatsApp widget
  - "Download Patient Guide" gated behind a name + phone form
  - A blog post with scroll-depth tracking
  - **All 9 GTM events wired in via `dataLayer.push()`**, with inline comments explaining which events GTM can capture natively vs which require a manual push
  - A live on-page `dataLayer` console (bottom-left toggle) so events can be seen firing in real time during a walkthrough, without needing a GTM Preview session

- **`TASK_01_GTM_SCHEMA.md`** — the full event schema table, the booking funnel dataLayer JSON, the funnel drop-off explanation, and the Google Ads conversion pick.

## Running it

No build step — open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```

Click through the booking form, click Call Now / the WhatsApp widget / download the guide, and scroll the blog post — each action logs a `dataLayer.push()` visible in the on-page console (bottom-left "dataLayer log" button) and in the browser devtools console.

# Key design decision

The booking form's 3 steps render via JS state change only , so GTM's built-in triggers cannot detect step transitions on their own. Each step push is written by the front-end developer at the exact moment a step completes, and GTM listens for it via a Custom Event trigger. Full reasoning in `TASK_01_GTM_SCHEMA.md`.
