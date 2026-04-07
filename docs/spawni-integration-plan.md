# Spawni Integration Plan

## Current State

AI Business Audit starts as a **standalone product** with its own domain, landing page, and infrastructure. This is intentional -- it allows the concept to be validated quickly without coupling it to Spawni's development timeline or technical stack.

**Standalone setup:**
- Separate domain (TBD -- e.g., aibusinessaudit.com or getaudit.ai)
- Separate codebase and deployment
- Own Stripe account (or separate product in shared Stripe)
- Own Supabase project (or separate schema in shared Supabase)
- No dependency on Spawni being live or functional

## Why Start Standalone

1. **Speed to market.** Spawni is still in development. The audit product can launch in 2-3 weeks with a simple landing page and backend pipeline. Waiting for Spawni delays validation.

2. **Clean validation.** If the audit product gets paying clients on its own, we know the concept works independent of Spawni's brand, audience, or feature set. The data is unambiguous.

3. **Separate identity.** The audit service targets business owners who need consulting. Spawni targets developers and businesses who want to host AI agents. Different audiences, different messaging, different trust signals. Combining them too early could confuse both audiences.

4. **Reduced risk.** If the audit product fails, it doesn't affect Spawni. If Spawni pivots, it doesn't affect the audit product. Independence = resilience.

## Integration Trigger

**Merge into Spawni when ALL of these are true:**
- 10+ paying clients have been successfully delivered through the standalone product
- Client satisfaction is high (NPS 50+ or equivalent qualitative signal)
- Spawni is live and has its own traction (active users, content presence)
- The technical integration makes sense (shared auth, shared database, shared billing)

**Do NOT merge if:**
- The audit product is working well standalone and merging would add complexity without clear benefit
- Spawni's audience doesn't overlap with the audit product's audience
- The merge would require significant re-architecture of either product

## How the Audit Fits Into Spawni

The AI Business Audit is fundamentally **a Spawni agent doing consulting work.** This is the conceptual bridge.

### Product Framing Within Spawni

- **Spawni = platform for hosting AI agents**
- **The audit agent = a specific agent that lives on Spawni**
- **The audit service = a productized offering using the audit agent**

This means:
- The audit agent could be one of Spawni's flagship demo agents
- "Look at what a Spawni agent can do -- it replaced a $200/hr consultant"
- Every audit client becomes a potential Spawni customer ("want to build your own AI agent? Try Spawni")
- Every Spawni user becomes a potential audit client ("need an AI to audit your business? We built one")

### URL Structure (Post-Merge)

```
spawni.ai/                    -- Spawni main platform
spawni.ai/audit               -- AI Business Audit landing page
spawni.ai/audit/submit        -- Intake form
spawni.ai/audit/q/{id}        -- Questionnaire page
spawni.ai/audit/report/{id}   -- Report viewing page
spawni.ai/audit/pricing       -- Paid tier pricing
```

The audit lives as a first-class product within Spawni, not a hidden feature. It has its own landing page, its own navigation entry, and its own marketing.

## Migration Path

### Phase 1: Cross-Promotion (No Code Changes)

Before any technical integration, start cross-promoting:
- Standalone audit site links to Spawni: "Built with Spawni AI agents"
- Spawni site links to audit: "Try our AI Business Audit -- free"
- Shared social media content
- Shared email list (with consent)

**Effort: 1-2 hours. No technical work.**

### Phase 2: Shared Branding

Update the standalone audit site to use Spawni branding:
- Spawni logo in the header
- "Powered by Spawni" badge
- Consistent color scheme, typography, and design language
- Spawni mascot appears on the audit pages

**Effort: 4-8 hours. Design changes only.**

### Phase 3: Shared Infrastructure

Migrate the audit backend to share Spawni's infrastructure:
- **Database:** Move audit tables into Spawni's Supabase project (or create a dedicated schema)
- **Auth:** If Spawni has user accounts by this point, allow audit clients to optionally create Spawni accounts to track their audits and projects
- **Stripe:** Move audit products into Spawni's Stripe account. Shared customer records mean a Spawni subscriber who buys an audit is recognized as the same customer.
- **Email:** Use Spawni's email sending infrastructure and templates

**Effort: 1-2 weeks. Backend migration.**

### Phase 4: Full Integration

Merge the audit frontend into the Spawni application:
- Audit pages become routes within the Spawni Next.js app
- Shared navigation (Spawni header/footer on audit pages)
- Unified dashboard for clients who have both Spawni agents and audit projects
- Audit agent appears in the Spawni agent marketplace
- Redirect standalone domain to spawni.ai/audit (301 redirect, maintain all existing links)

**Effort: 2-3 weeks. Full frontend and backend integration.**

### Phase 5: Agent Marketplace Listing

The audit agent becomes a product in Spawni's agent marketplace:
- Other Spawni users can see how the audit agent works
- Template: "Business Consulting Agent" that other users can fork and customize for their own consulting practice
- This creates a new revenue stream: Spawni users paying to access the audit agent template

**Effort: 1 week. Marketplace listing + template creation.**

## Data Migration

When merging databases, the following tables move from the standalone Supabase project to the Spawni project:

```
submissions        -> spawni.audit_submissions
questionnaires     -> spawni.audit_questionnaires
questionnaire_responses -> spawni.audit_questionnaire_responses
reports            -> spawni.audit_reports
projects           -> spawni.audit_projects
automations        -> spawni.audit_automations
retainers          -> spawni.audit_retainers
```

File storage migration:
- Audio files, uploaded documents, generated PDFs, and solution artifacts move from the standalone Supabase Storage to Spawni's storage
- Signed URLs will need to be regenerated
- Old URLs should redirect to new locations for 6 months

Stripe migration:
- Create the same products/prices in Spawni's Stripe account
- Migrate active subscriptions (retainers) using Stripe's subscription migration tools
- Customer data can be linked via email address

## Cross-Promotion Opportunities

### Audit -> Spawni

Every audit client is introduced to Spawni:
- Audit report footer: "This report was generated by a Spawni AI agent. Want to build your own? Visit spawni.ai"
- Delivery email: "PS: The AI that audited your business runs on Spawni. If you want AI agents for your own customers, check out spawni.ai"
- Retainer clients get a free Spawni account (basic tier) so they can see their agent in action

### Spawni -> Audit

Every Spawni user is introduced to the audit:
- Spawni dashboard: "New: Get a free AI audit of your business operations"
- Spawni blog: Case studies from audit clients
- Spawni newsletter: Monthly highlights from audit engagements
- Spawni agent marketplace: The audit agent is a featured example of what Spawni agents can do

### Shared Content

Content that benefits both products:
- "How AI Agents Are Replacing Consultants" (promotes both audit + Spawni)
- "We Built an AI Business Consultant on Spawni -- Here's What Happened" (behind-the-scenes)
- Client case studies that show the audit results AND explain the Spawni tech behind it

## Revenue Attribution

Post-merge, track revenue attribution:
- **Audit-originated revenue:** Clients who first interacted with the audit, then became Spawni subscribers
- **Spawni-originated revenue:** Spawni users who discovered and purchased the audit service
- **Blended:** Clients who use both products

This data informs future product and marketing decisions. If audit clients convert to Spawni at a high rate, the audit becomes Spawni's primary acquisition channel. If the conversion is low, they remain complementary but independent products.

## Timeline Estimate

| Phase | Trigger | Effort | Calendar Time |
|-------|---------|--------|---------------|
| Standalone launch | Now | 2-3 weeks | Weeks 1-3 |
| Cross-promotion | Standalone has 5+ audits | 1-2 hours | Week 4+ |
| Shared branding | 10+ audits completed | 4-8 hours | Month 2+ |
| Shared infrastructure | 10+ paying clients + Spawni live | 1-2 weeks | Month 3-4 |
| Full integration | Both products validated | 2-3 weeks | Month 4-6 |
| Marketplace listing | Full integration complete | 1 week | Month 5-7 |

**Key principle: Don't merge prematurely.** The standalone product needs to prove itself first. Merging too early couples two unvalidated products and doubles the surface area for problems.

## Related Repositories

- [spawni](https://github.com/monikaalmasy/spawni) -- The Spawni platform (future integration target)
- [business-idea-automation](https://github.com/monikaalmasy/business-idea-automation) -- Research and idea validation hub where this concept was developed
- [spawni-growth](https://github.com/monikaalmasy/spawni-growth) -- Content and marketing engine that will power cross-promotion
