# Upwork Portfolio Entry (copy-paste ready, edit before posting)

**Title:** AI-Classified Maintenance Request Routing for Property Management (n8n, Airtable, OpenAI API)

**Cover image:** a screenshot of the Airtable record showing the full
result: category, urgency, assigned contractor, and status all filled in
automatically, next to the original tenant description that triggered it.

**Links to include in the listing:** [README.md](README.md)

**Description:**

Property managers lose time and take on real risk in one specific spot:
sorting incoming maintenance requests. A flooding pipe and a dripping
faucet arrive through the same channels and get read in whatever order
they land, not the order they matter. Built a system that removes that
sorting step entirely.

A tenant submits through a form. n8n picks up the new request, sends the
description to OpenAI for classification into a category (Plumbing,
Electrical, General) and an urgency level (Emergency or Routine), matches
it against a contractor roster filtered by category and availability,
writes the result back to the record, and emails both the tenant and the
contractor with the details. No step is manual after the tenant hits
submit.

The AI call goes straight to OpenAI's Chat Completions API through an HTTP
Request node, not through a pre-built connector. That's the part worth
pointing at directly: it shows working with an LLM API at the
request/response level, handling the auth header and the response shape
myself, rather than only through a no-code wrapper. The response is
requested as strict JSON so parsing is reliable, and the parsing step still
falls back to safe defaults if the model ever returns something
unexpected, so a bad AI response degrades the routing instead of breaking
the workflow.

**Skills tags:** n8n, Airtable, OpenAI API, LLM Integration, API
Integration, Workflow Automation, Property Management Software,
Classification, Process Automation

**Before posting, confirm:**
- [ ] Airtable base has fake/demo contractor and tenant data, nothing that
      looks like a real client's information
- [ ] Loom recorded, walking through one Emergency-classified submission
      and one Routine one, showing the record fill in and both
      notification emails land
- [ ] Repo is public and includes the latest workflow.json
- [ ] Case study description above has no leftover placeholder text
      before copy-pasting into the Upwork listing
