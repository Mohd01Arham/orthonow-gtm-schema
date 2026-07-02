# OrthoNow — Integration Design (Task 03)

## How I'd build the flow

When someone submits the consultation form, it should send the data straight to our own backend — not a plain HubSpot form embed, and not a tool like Zapier or Make.

Why not those? A native HubSpot form only knows how to talk to HubSpot. It has no way to call Karix's WhatsApp API or fire a Google Ads conversion on its own. Zapier or Make could technically link all three steps together, but they only give you the basic, out-of-the-box version of each integration — you can't easily build the custom logic this setup actually needs (more on that below). Writing this on our own backend means we control exactly what happens, in what order, and how we handle it if something fails.

So here's the order of operations once the backend gets the form data:

1. **Call the HubSpot Contacts API** to create or update the patient's record — Name, Phone, Clinic Preference, Source ("Google Ads - Consultation Landing Page"), and Lead Status ("New Enquiry"). Here's the catch: HubSpot normally matches contacts by email, and this form doesn't even ask for an email. If we leave that as-is, two patients who submit with the *same phone number* but *different names* would end up as two separate contacts instead of one — HubSpot has no way to know they're the same person. The fix is simple: mark phone number as a unique property in HubSpot and tell the API to match on that instead of email.

2. **Call Karix's WhatsApp API** to send the confirmation message to that same phone number.

3. **Fire the Google Ads conversion** for `consultation_form_submitted`, so the ad campaign learns from real bookings.

## Where this is most likely to break

The weakest link is Karix — it's the one outside service in this chain with an actual time limit attached (2 minutes), so if it's slow or down, the whole thing falls apart right there.

To guard against that, I wouldn't send the WhatsApp message directly and just hope it works — I'd put it in a small queue that retries a few times if the first attempt fails. If it still hasn't gone through by the time we're close to missing the SLA, fall back to sending an SMS through a different provider, and log it so someone on the team can follow up by hand if needed. That way one bad moment from Karix doesn't mean the patient hears nothing.

## Keeping the 2-minute promise

The most realistic way this breaks is Karix getting slow or hitting rate limits during a busy stretch — say, right after a new ad campaign goes live and a bunch of people submit the form at once.

To catch this early, I'd log the exact time the form was submitted and the exact time the WhatsApp message actually went out, and set up an alert if that gap gets close to 90 seconds — before it actually breaches the 2-minute mark. I'd also keep an eye on Karix's success/failure rate as its own number to watch, so we notice it slowing down before it turns into a full outage nobody caught in time.
