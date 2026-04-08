# AI Business Audit — TODO

Single source of truth for all tasks related to the AI business audit product.

Build priority: This product is Phase 3 (after the SEO SaaS pipeline is live). See [niche-seo-saas/TODO.md](https://github.com/monikaalmasy/niche-seo-saas/blob/main/TODO.md) for the SEO pipeline tasks.

---

## Phase 3: Core Build (Weeks 2-4, parallel with SEO pipeline)

- [ ] Voice intake — browser-based recording (MediaRecorder API)
- [ ] File upload with security (ClamAV, magic bytes, filename sanitization)
- [ ] Transcription — Deepgram Nova-2 ($0.0043/min) with Whisper fallback
- [ ] Multimodal file processing — Claude Vision for screenshots, PDFs, spreadsheets
- [ ] Business analysis generation — Claude structured JSON output
- [ ] Smart questionnaire generation — targeted questions based on analysis
- [ ] Audit report generation — branded PDF (WeasyPrint)
- [ ] Email delivery of audit report (Resend)
- [ ] Stripe checkout for paid tier ("Build This For Me")
- [ ] Pre-built n8n automation templates (10 dental, 5 plumbing)
- [ ] Operator review workflow (notification → review → approve → deliver)
- [ ] Loom walkthrough recording process
- [ ] Landing page with voice recorder

---

## Post-Launch: Profession-Specific AI Guides (PDF Products)

**Concept:** Create highly specialized PDF guides on how specific professions should use AI to stay competitive. Each PDF is targeted at ONE profession and covers exactly how AI applies to THEIR daily work — not generic "AI for business" content.

### The Product
- Claude generates profession-specific AI guides (20-40 pages each)
- Extremely targeted: "How to Use AI as a Dental Receptionist" is different from "How to Use AI as a Project Manager"
- Practical, actionable — not theory. Step-by-step workflows, tool recommendations, prompt templates
- Every guide recommends Spawni as the implementation tool (subtle upsell, not hard sell)
- Sold as digital products ($19-$49 per guide, or bundle pricing)

### Target Audience
- **NOT business owners** (they're the SEO/audit customers) — this is for **working professionals**
- Employees who want to stay relevant, get promoted, or work more efficiently
- People who feel AI pressure in their role and want a specific playbook

### Example Guides to Create
- [ ] "AI for Dental Receptionists: Automate Bookings, Follow-ups & Patient Communication"
- [ ] "AI for Project Managers: Sprint Planning, Status Reports & Risk Assessment on Autopilot"
- [ ] "AI for Test Analysts: Automated Test Cases, Bug Reports & Regression Testing with AI"
- [ ] "AI for Real Estate Agents: Listings, Follow-ups & Market Reports in Minutes"
- [ ] "AI for Accountants: Tax Prep, Client Communication & Report Generation"
- [ ] "AI for Plumbers/Trades: Quoting, Scheduling & Invoice Management"
- [ ] "AI for Legal Assistants: Document Review, Research & Client Intake"
- [ ] "AI for HR Managers: Recruitment Screening, Onboarding & Policy Writing"
- [ ] "AI for Marketing Coordinators: Content Calendar, Copywriting & Analytics"
- [ ] "AI for Executive Assistants: Email Management, Meeting Prep & Travel Booking"

### Production Pipeline (Fully Automatable)
1. Claude researches the profession — daily tasks, pain points, tools used, common workflows
2. Claude generates the guide — structured chapters, practical steps, tool recommendations, prompt templates
3. WeasyPrint renders the PDF — branded, professional design, consistent across all guides
4. Upload to Gumroad / Lemon Squeezy / own Stripe checkout
5. Promote via targeted channels (see Distribution below)

### Distribution — How to Reach People by Profession
- [ ] **LinkedIn Ads** — Target by job title (e.g., "Dental Receptionist", "Project Manager"). LinkedIn is the only platform with reliable profession targeting.
- [ ] **LinkedIn organic content** — Post AI tips for specific professions, link to guide in comments
- [ ] **Reddit** — Post in profession-specific subreddits (r/projectmanagement, r/dentistry, r/accounting, etc.) with genuine value, link to guide
- [ ] **Google Ads** — Target "AI for [profession]" keywords (e.g., "how to use AI as a receptionist")
- [ ] **TikTok/Instagram** — Short-form content: "3 AI tricks every [profession] should know" → link in bio to guide
- [ ] **Email cross-sell** — Existing SEO/audit clients receive relevant guides for their staff (e.g., dental practice owner gets "AI for Dental Receptionists" to share with their team)
- [ ] **Affiliate/referral** — Give away guides free to profession-specific influencers for promotion
- [ ] **Bundle with SEO/audit services** — "Subscribe to Growth+ and get free AI guides for your entire team"

### Pricing Options (TBD)
- [ ] Individual guide: $19-$49
- [ ] Profession bundle (all guides for one industry): $99-$149
- [ ] All-access pass: $199-$299
- [ ] Free with Growth+ subscription (as value-add / retention tool)
- [ ] Lead magnet: give away ONE guide free → upsell to paid guides + newsletter

### AI Newsletter for Professionals
- [ ] **"AI at Work" newsletter** — Weekly email with AI tips for working professionals (not business owners — different audience from Scale & Polish newsletter)
- [ ] Free tier: weekly email with 1-2 actionable AI tips per profession
- [ ] Monetization: guide sales, Spawni referrals, affiliate links for AI tools, sponsors
- [ ] Lead magnet: free guide download → subscribe to newsletter → upsell to paid guides
- [ ] Claude writes the newsletter (15 min/week review)
- [ ] Distribution: LinkedIn + profession-specific channels

### Revenue Potential
- At $29 average guide price × 100 sales/month = $2,900/month passive income
- Scales with number of guides created (each new guide = new addressable profession)
- Cross-sell to Spawni: if 5% of guide buyers try Spawni at $29/month = additional recurring revenue
- Newsletter builds audience for all future products

### To Research Before Building
- [ ] What AI guide products already exist? Any profession-specific ones?
- [ ] What price points work for digital PDF products in 2026?
- [ ] Gumroad vs Lemon Squeezy vs own Stripe checkout — which platform?
- [ ] Can Claude produce genuinely useful, non-generic profession-specific content?
- [ ] What's the realistic production time per guide? (Estimate: 2-4 hours with Claude + design)
- [ ] Legal: any restrictions on profession-specific AI advice? (e.g., can't give legal advice in the "AI for Legal Assistants" guide)

---

## Post-Launch: Spawni Integration

- [ ] Cross-promote: audit clients who need ongoing AI agent work → recommend Spawni
- [ ] Feature Spawni in all PDF guides as the recommended implementation tool
- [ ] When Spawni is live + audit has 10+ clients → evaluate merging into spawni.ai/audit
- [ ] See [spawni-integration-plan.md](docs/spawni-integration-plan.md) for migration path

---

## Open Questions

- [ ] Spawni content engine timing — defer or run in background?
- [ ] When to hire a VA for support (~$5K/month revenue threshold)
- [ ] Call playbook — prepared response for "Can I jump on a call?" (needed before any outreach)
