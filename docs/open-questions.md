# Open Questions and Future Considerations

Items discussed but NOT finalized. These need further exploration before becoming decisions.

## Customer-Facing Communication

- **"Team of experts" positioning**: If standalone (not Spawni-branded), we can position the service as "a team of experts will evaluate your uploads" rather than "AI processes everything." This is commercially advantageous — many SMB owners are more comfortable paying humans. Not deceptive (there IS a human quality gate), just not leading with AI.
- **If merged into Spawni later**: The AI angle becomes a feature, not something to downplay. Different positioning for different audiences.
- **Decision needed**: Define the exact customer-facing language for the standalone brand.

**Decision (2026-04-07):** Standalone brand confirmed. Anonymous company (no personal name/photo). "Team of experts" positioning. Part of ONE company that also offers the SEO SaaS product.

## Onboarding Calls

- **Option A: No calls at all** (current plan) — Voice note + file upload only. Best information quality, fully async, most automated.
- **Option B: Optional onboarding Zoom** — Offer as an add-on. Record the session, extract insights. Only if client specifically wants it.
- **Option C: Hire a salesperson/BD person** — Goes to meetings, does discovery. Concern: loses credibility when they can't answer deep questions. Could work as "intake specialist" who says "let me take this back to the team."
- **Current leaning**: Option A primary, Option B available as premium add-on. Option C deferred — too much overhead for now.
- **Decision needed**: Whether to offer Zoom option at all, or keep it purely async.

**Decision (2026-04-07):** Start with 100% automated async approach. User willing to invest 10+ hours/week reviewing deliverables. Optional Zoom calls may be introduced later as a premium add-on once user gains confidence from reviewing backend logs and learning the consulting process.

## Screen Recording Intake

- **Feasibility**: Confirmed — Claude Vision can interpret screen recordings (extract frames, read UI elements, understand workflows)
- **Options**:
  - Client records a Loom walkthrough of their workflow (async, no live call)
  - Live Zoom screen share (recorded, frames extracted)
  - Client uploads screenshots directly (simplest, already supported)
- **Assessment**: Voice note + file upload likely produces better information than screen shares. Screen recording is a nice-to-have option, not primary.
- **Decision needed**: Whether to add Loom/screen recording as an intake option alongside voice notes.

## Authentication and System Access

- **Key insight**: Modern tools support OAuth/API keys — clients never share passwords.
- **Approach**: Intake questionnaire identifies client's tool stack. System generates personalized "connection guide" with OAuth authorization links. Client clicks Authorize, grants API access.
- **Edge cases**: Legacy systems without OAuth → scope out or recommend migration path.
- **Supported integrations (initial)**: Focus on the 80% — Stripe, HubSpot, Mailchimp, Xero, QuickBooks, Slack, Google Workspace, Shopify, Calendly.
- **Decision needed**: Which integrations to support at launch. Start narrow (5-10 most common SMB tools).

## Hiring a Salesperson / BD Person

- **Pros**: Handles client-facing work entirely, frees operator time
- **Cons**: Trust problem — can't answer deep technical questions. "Let me take this back to the team" works but feels junior. Salary adds fixed costs before revenue is proven.
- **Assessment**: Premature for launch. Revisit after 10+ paying clients when the model is validated and there's revenue to support a hire.
- **Decision needed**: Deferred until post-validation.

## Target Market

**Decision (2026-04-07):** First target is small businesses with 5-20 employees (dental offices, law firms, accounting practices, real estate agencies, trades). Decision-maker is the owner. $2-3K/month ticket size. Need ~10 clients for $25K/month revenue target.

Rationale: Owner decides alone (no committees), real budgets, simple systems (5-10 tools), fits async model, "team of experts" positioning appropriate at this tier.

## MVP Scope

**Decision (2026-04-07):** Full automated pipeline before taking clients. Voice transcription, AI analysis, auto-generated questionnaire, auto-generated report — all working end-to-end. Estimated 2-4 weeks build time.

Rationale: User wants polished experience from day one. 10+ hours/week time budget supports thorough build. Full automation aligns with $25K/month scaling target.

## Revenue Target

**Decision (2026-04-07):** $25K+ per month within 6 months, replacing a full salary. This is the primary business, not a side project.

## Spawni Integration Timeline

- **Current decision**: Start standalone, merge into Spawni later
- **Trigger for merge**: When Spawni is fully launched AND the audit service has 10+ paying clients
- **See**: [spawni-integration-plan.md](spawni-integration-plan.md) for the migration path
