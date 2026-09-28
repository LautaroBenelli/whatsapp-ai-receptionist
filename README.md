# WhatsApp AI Receptionist (n8n)

A WhatsApp assistant that books, reschedules and cancels appointments for small service businesses. It started as a project for a barbershop and was later adapted for a dental clinic and an aesthetics center in Uruguay.

Customers write (or send a voice note) as they would to a person. The agent checks the real calendar before offering times, keeps a small customer record in Google Sheets, and hands the conversation to the owner when it can't help.

**In production:** 3 businesses, around 30 messages per day each. Admin time spent on WhatsApp scheduling dropped by roughly 80% after deployment.

> The workflow in this repo is anonymized. Business names, contact data, calendar and sheet IDs were replaced with placeholders.

## How a message is handled

1. Evolution API forwards the incoming WhatsApp message to an n8n webhook.
2. Group chats and messages sent by the business itself are ignored.
3. Voice notes are downloaded and transcribed with Whisper, so the rest of the flow only deals with text.
4. A guardrails step rejects NSFW content before it reaches the agent.
5. The agent (GPT-4o) answers using the conversation history stored in Postgres (last 10 messages, keyed by phone number) and a set of tools:
   - Google Calendar: check availability, create, update and delete events
   - Google Sheets: find or register the customer, save the appointment and update its status
   - Gmail: escalate to a human
6. The reply goes back through Evolution API. If the agent confirmed, changed or cancelled an appointment, the owner gets a WhatsApp notification too.

## Decisions worth mentioning

**Availability comes from the calendar, never from the model.** The prompt forces the agent to call the availability tool before proposing any time, and it always queries the full day in Montevideo time (UTC-3). Early versions sometimes read an empty result as "no availability"; the tool description now says explicitly that returned events are the *busy* slots.

**Calendar and Sheets stay linked.** Every appointment saved in Sheets stores the Google Calendar event ID, so rescheduling or cancelling updates the right event instead of creating duplicates.

**Escalation is explicit.** When a request is out of scope, the agent adds a marker to its answer. n8n detects it, strips it from the text the customer sees, tells the customer someone will contact them, and alerts the owner by WhatsApp and email.

**Business rules live in the prompt, side effects live in tools.** Prices, opening hours and how to interpret things like "a la tarde" or "mediodía" are in the system prompt. Anything that changes data goes through a tool call.

## Stack

n8n (self-hosted) · OpenAI GPT-4o and GPT-4o-mini · Whisper · Evolution API (WhatsApp) · PostgreSQL · Google Calendar, Sheets and Gmail APIs

## Running it

1. Import `workflow/whatsapp-receptionist.json` into n8n.
2. Create the credentials the nodes ask for: Evolution API, OpenAI, Postgres, Google Calendar, Google Sheets and Gmail.
3. Point every Sheets node to your spreadsheet (`YOUR_GOOGLE_SHEET_ID`). It needs two tabs: customers and appointments.
4. Set the calendar in the Calendar nodes (`primary` by default) and the owner's email in the escalation node.
5. Configure the Evolution API instance to send `messages.upsert` events to the webhook URL.
6. Edit the business section of the agent prompt (name, services, prices, hours).

## What I would change next

- Move prices and opening hours out of the prompt into a sheet or table, so the business can edit them without touching the workflow.
- Replace Sheets with Postgres for appointments once volume grows.
- Add automated test conversations for the booking flow. I did this later in a TypeScript version of the same idea.
