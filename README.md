# Maintenance Request Routing (Property Management)

## The problem this solves

A property manager handling more than a handful of units gets maintenance
requests through whatever channel a tenant happens to use: a phone call, a
text, an email, a paper note under the door. Every one of those has to be
read, sorted into a category, judged for how urgent it actually is, and
handed to the right contractor. That triage step is manual and it happens
under time pressure, which is exactly when mistakes happen. A tenant who
types "water is coming through the ceiling" and a tenant who types "the
faucet drips a little" can end up sitting in the same queue, read in the
order they arrived instead of the order they matter. A flooding call that
sits for six hours because it landed behind five routine ones is a real
liability and repair-cost problem, not just an inconvenience.

The second failure mode is routing to a contractor who isn't actually
available, or forgetting to route at all, so the ticket goes stale and the
tenant has to chase someone for an update.

This system removes both failure points. Every request gets triaged the
moment it comes in, sorted by real urgency, and handed straight to a
contractor who's marked available for that category, with both sides
notified automatically.

## How it works

1. **Intake.** Tenants submit through an Airtable Form, which writes
   directly into the Maintenance Requests table. No separate sync step,
   since Airtable owns the form natively.
2. **Trigger.** n8n's Airtable Trigger polls the table for new records.
3. **AI classification.** The request description is sent to OpenAI's Chat
   Completions API, which returns a category (Plumbing, Electrical, or
   General) and an urgency level (Emergency or Routine). Emergency covers
   flooding, no heat, no water, gas smell, or an active electrical hazard.
   Everything else is Routine.
4. **Contractor match.** n8n searches the Contractor Roster table for
   someone in that category who's marked available.
5. **Record update.** The original request gets written back with the
   classification, the assigned contractor, a status of "Assigned," and a
   timestamp of exactly when it was processed.
6. **Notifications.** The tenant gets an email confirming who's coming and
   for what. The contractor gets an email with the job details and the
   tenant's contact info. Both fire in parallel, not one after the other.

## Why this calls the OpenAI API directly

This build uses n8n's HTTP Request node to call OpenAI's Chat Completions
endpoint directly, rather than n8n's built-in OpenAI node. That's a
deliberate choice, not an oversight: it proves the ability to work with an
LLM API at the request/response level, handling auth headers and parsing
the raw response, instead of only through a pre-built wrapper. For a client
who just wants the simplest possible working system, the built-in node is
arguably the better call. This project exists specifically to show the
other skill.

The response is requested as strict JSON (`response_format: json_object`)
so it parses reliably, and the parsing step still wraps in a try/catch that
falls back to General/Routine if the model ever returns something
unexpected. A bad AI response degrades the routing, it never breaks the
workflow.

## Setup

1. Create an Airtable base with two tables:
   - **Maintenance Requests**: Property/Unit, Requester Name, Requester
     Contact, Description, Category (single select), Urgency (single
     select), Assigned Contractor, Status (single select), Submitted At
     (Created time field type), Assigned At (date field with time).
   - **Contractor Roster**: Name, Category (single select), Email, Phone,
     Available (checkbox).
2. Add a Form view on Maintenance Requests for tenant intake.
3. Seed the Contractor Roster with your real contractors and their current
   availability.
4. Import `workflow.json` into n8n.
5. Reconnect credentials on every node: Airtable Personal Access Token
   (needs `data.records:read`, `data.records:write`, `schema.bases:read`
   scopes, and the base explicitly added under the token's Access
   section), OpenAI API key as a Header Auth credential (`Authorization:
   Bearer sk-...`), and SMTP for both email nodes.
6. Activate the workflow, submit a test request through the form, and
   confirm the record fills in correctly within one poll cycle.

## What would change for a real client

- **Contractor availability would sync from somewhere real**, not get
  toggled manually in Airtable. Most property managers either run this
  through the contractor's own calendar or a simple form the contractor
  fills in themselves when they take on or finish a job.
- **Emergency-tier requests would escalate to SMS**, not just email, for
  both the tenant confirmation and the contractor alert. Email alone isn't
  fast enough for a flooding call at 2am.
- **Multi-property portfolios** would add a Property lookup table instead
  of a free-text field, so routing can also account for which contractors
  actually cover which building or region.
- If the client already runs dedicated property management software
  (Buildium, AppFolio, etc.), the Airtable base would get replaced by that
  system's own API as the source of truth, with this same
  classify-and-route logic sitting on top of it.

## Troubleshooting notes from building this

- **Airtable 401 on the Trigger node**, even with a credential selected:
  the Personal Access Token needs the base explicitly listed under its
  Access section, and the actual secret value only shows once, at
  creation or regeneration. If you're not sure the right value made it
  into n8n, regenerate the token and re-paste it.
- **"No columns found in Airtable" on an action node**: usually the same
  auth problem as above, since n8n needs a working credential to fetch
  the base's schema at all.
- **Airtable node has two separate auth types**, API Key (legacy) and
  Access Token. Every Airtable node in the workflow needs Authentication
  set to Access Token, using the same credential, or they won't share a
  connection even if one of them works.
- **Workflow won't activate: "Your request is invalid or could not be
  processed by the service."** This is Airtable rejecting the Trigger
  node's underlying query, almost always because the Trigger Field
  parameter doesn't match a real field name in the table.
- **Trigger activates but never fires on new form submissions**: check
  that the field you set as Trigger Field is actually Airtable's
  **Created time** field type, not a plain manual date field. A plain
  date field is never auto-filled, so it stays empty on every form
  submission and the trigger has nothing to compare against.
