![Screenshot of workflow](Screenshot%20of%20Workflow%20-%20Nicholas%20Hennell-Foley.png)
**To see the full resolution, right click on the image and choose "Open image in new tab", or download it**.


# Download n8n Workflow
[https://github.com/nicholashennellfoley/automation-workflow-example/blob/main/n8n%20Blueprint%20-%20Nicholas%20Hennell-Foley.json](https://github.com/nicholashennellfoley/automation-workflow-example/blob/main/n8n%20Blueprint%20-%20Nicholas%20Hennell-Foley.json)

Import to a blank n8n canvas.

# What
Hire decision → CRM update → Candidate comms → IT provisioning. With a reversible cancel window + AI error-handler.

# Purpose
Centralise hiring outcomes coming from any system (for example a form or CRM), then—based on the decision—update the CRM, email the candidate, and (on full acceptance) kick off IT onboarding. The flow is resilient (retry logic, reversible “cancel” window) and observable (AI triage on errors), which maps well to cross-functional ops where accuracy and speed matter.

# How
1. Intake + reversible window
A Webhook node (Receive hire decision from any source) accepts a POST with decision, itemId, firstName, lastName, email, etc. It immediately returns an HTML confirmation including a Cancel button. Internally I add a cancelUrl with the current $execution.id, then a Wait node gives stakeholders a grace period. If a second webhook (cancel-action) arrives with that executionId, the n8n API stops the execution and replies “request cancelled”; if processing already started, it replies accordingly. This creates a safe, human-in-the-loop “Wait + Cancel” pattern.

2. Fetch latest CRM state
After the window, an HTTP Request to SharePoint/Graph (Query CRM for most up to date data) pulls canonical applicant data before branching—important for idempotency and correctness.

3. Branch by outcome (Switch node)
- Interview:
  - SharePoint REST updates the application to “Interview Stage”.
  - An HTML email template is assembled (“Invitation to Interview”) with a booking link; Gmail sends it.

- Fully accept (offer accepted):
  - Crypto generates a temporary password; Microsoft Entra creates the user disabled (force password change on first sign-in).
  - Graph API lists SharePoint onboarding pages → I filter for specific webparts (file embeds), Set/Aggregate to build a progress string, then POST a row to an “Onboarding Progress” list and PATCH a “processed” flag in the CRM.
  - A branded welcome/onboarding HTML email is sent via Gmail.
  - If the user already exists in our Microsoft tenant, an If node notifies IT to pick a unique address (graceful duplicate handling).

- Decline:
  - SharePoint REST moves the record to “Declined”.
  - A branded HTML decline email is sent via Gmail.

4. Robust error handling (separate, reusable sub-flow)
- Specific failures fans out to an AI Error Handler. That handler uses:
  - Error Trigger → n8n API → Code to collect and sort runData for a clear execution timeline.
  - OpenAI Chat + MCP client to query API docs and an HTTP Request tool to investigate root causes and potentially remediate (the blueprint pins shows a Microsoft example).
  - A single HTML triage email is generated to IT—either with clear recommendations, or, if write access was decided to be granted to the agent, resolution steps taken—so issues are actionable, not just logged.

6. Good n8n hygiene
OAuth2 creds for Microsoft/SharePoint/Gmail, retryOnFail on external calls, defensive Filter/If checks, and modular sections (noted in sticky comments) so these blocks can be promoted into sub-workflows for reuse across teams.

# Why
Hopefully it demonstrates some of what you're looking for!

It shows connecting APIs, routing/cleaning data, human-safe control (cancel window), and production-grade resilience (idempotency, retries, AI-assisted triage). The same patterns could port cleanly to any other APIs (Airtable etc) as required.
My background is specifically in building these kinds of production systems, with code nodes where required, webhooks, OAuth, and robust documentation.
