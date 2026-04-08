# AI Business Audit -- Technical Specification (Builder Guide)

## Purpose

This document fills gaps in [technical-architecture.md](technical-architecture.md). Read that document first for the base architecture (pipeline stages, database schema, API endpoints, cost analysis). This spec covers: unified stack decisions, project structure, security hardening, automation templates, operator workflow, and production prompts.

## Tech Stack Alignment

**Decision (2026-04-08):** Both products (AI Business Audit + niche SEO SaaS) use the same stack for consistency:

- Python 3.12+ with FastAPI (NOT Node.js -- the technical-architecture.md references are outdated)
- Supabase (PostgreSQL) with supabase-py
- Redis + arq for async job queue
- WeasyPrint for PDF generation (NOT Puppeteer)
- openpyxl + pandas for spreadsheet processing (NOT SheetJS)
- Resend for email (NOT React Email)
- Deepgram Nova-2 ($0.0043/min batch) for transcription, with Whisper as free fallback
- Render.com (paid Starter)

[OUTDATED as of 2026-04-08: technical-architecture.md references Node.js, Puppeteer, Bull queue, SheetJS, and React Email/MJML templates. These are superseded by the Python stack listed above.]

## Project Structure

```
ai-business-audit/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI application entry point
│   ├── config.py                # Settings via pydantic-settings
│   ├── models/
│   │   ├── database.py          # Supabase table models
│   │   └── schemas.py           # Pydantic request/response schemas
│   ├── services/
│   │   ├── transcription.py     # Deepgram/Whisper
│   │   ├── file_processor.py    # Per-type processing (PDF, XLSX, images)
│   │   ├── file_security.py     # ClamAV, magic bytes, sanitization
│   │   ├── analyzer.py          # Claude business analysis
│   │   ├── questionnaire.py     # Smart questionnaire generation
│   │   ├── compliance.py        # AHPRA/legal compliance
│   │   ├── report.py            # WeasyPrint PDF generation
│   │   ├── email_service.py     # Resend delivery
│   │   ├── stripe_service.py    # Stripe payments
│   │   └── automation.py        # Pre-built automation templates
│   ├── api/
│   │   ├── submit.py            # POST /api/submit
│   │   ├── questionnaire.py     # GET/POST questionnaire endpoints
│   │   ├── report.py            # GET report + PDF download
│   │   ├── checkout.py          # POST /api/checkout
│   │   └── webhooks.py          # Stripe webhook handler
│   ├── workers/
│   │   └── submission_worker.py # Process submissions via arq
│   ├── prompts/
│   │   ├── file_analysis.py
│   │   ├── business_analysis.py
│   │   ├── questionnaire_gen.py
│   │   ├── report_gen.py
│   │   └── compliance.py
│   └── templates/
│       ├── report.html           # WeasyPrint report template (Jinja2)
│       ├── questionnaire.html    # Client-facing questionnaire page
│       └── emails/               # Resend email templates (plain Jinja2)
├── landing/
│   ├── index.html                # Intake form landing page
│   ├── styles.css
│   └── recorder.js               # MediaRecorder API implementation
├── automation_templates/
│   ├── dental/
│   │   ├── appointment_reminders.json    # Make.com blueprint
│   │   ├── review_requests.json
│   │   ├── patient_followup.json
│   │   ├── insurance_claims.json
│   │   └── recall_campaigns.json
│   └── plumbing/
│       ├── job_quoting.json
│       ├── invoice_chasing.json
│       ├── scheduling.json
│       └── review_requests.json
├── tests/
├── requirements.txt
├── .env.example
├── render.yaml
└── CLAUDE.md
```

## Environment Variables

All required environment variables (listed in `.env.example`):

```
# AI Services
ANTHROPIC_API_KEY=             # Claude API
DEEPGRAM_API_KEY=              # Transcription (Nova-2 batch)

# Email
RESEND_API_KEY=                # Transactional email delivery

# Payments
STRIPE_SECRET_KEY=             # Stripe server-side key
STRIPE_WEBHOOK_SECRET=         # Stripe webhook signature verification

# Database
SUPABASE_URL=                  # Supabase project URL
SUPABASE_ANON_KEY=             # Supabase anonymous key (client-safe)
SUPABASE_SERVICE_ROLE_KEY=     # Supabase service role key (server-only, never expose)

# Queue
REDIS_URL=                     # Redis connection string for arq worker

# Application
BASE_URL=                      # Public URL (e.g., https://aibusinessaudit.com.au)
ENVIRONMENT=                   # development | staging | production
LOG_LEVEL=                     # DEBUG | INFO | WARNING | ERROR

# Monitoring
SENTRY_DSN=                    # Sentry error tracking

# Security
TURNSTILE_SECRET_KEY=          # Cloudflare Turnstile server-side validation
CLAMAV_HOST=                   # ClamAV socket or TCP address for file scanning
```

## File Upload Security (Critical)

### Attack Vectors

- Malicious files (macros in DOCX, XXE in XML)
- MIME type spoofing (exe renamed to pdf)
- Zip bombs (XLSX/DOCX are ZIP archives)
- XSS via transcripts rendered in reports
- Path traversal in filenames
- Oversized uploads (DoS)

### Mitigations (ALL required before launch)

1. **ClamAV scanning**: Required before ANY file processing. Run ClamAV as a sidecar service on Render or use a cloud AV API. Every file must pass scan before entering the processing pipeline. Reject and log any file that fails.

2. **Magic byte validation**: Check file signatures, not MIME headers. Use python-magic library. A file claiming to be a PDF must start with `%PDF`. A file claiming to be XLSX must be a valid ZIP archive containing `[Content_Types].xml`.

3. **Filename sanitization**: Generate UUID-based filenames server-side. Never use client-provided filenames in paths. Store original filename in database metadata only (HTML-escaped).

4. **Size limits**: 50MB per file, 200MB total per submission. Enforced at API layer with FastAPI's UploadFile. Return 413 immediately if exceeded -- do not buffer the full upload before checking.

5. **Content sanitization**: All user-derived text (transcripts, file content, questionnaire responses) must be HTML-escaped before rendering in reports. Use Jinja2 with `autoescape=True` for all templates. Never use `|safe` on user content.

6. **Sandboxed processing**: Parse files with strict resource limits. Set memory limits on PDF parsing (some PDFs contain embedded images that expand to gigabytes). Set CPU time limits on spreadsheet processing. Use `signal.alarm()` or process-level timeouts.

7. **Zip bomb protection**: Before extracting XLSX/DOCX files, check the compressed-to-uncompressed ratio. Reject any file where the uncompressed size exceeds 500MB or the ratio exceeds 100:1.

## Pre-Built Automation Templates (NOT Custom Dev)

**Decision (2026-04-08):** Instead of Claude Code building custom automations per client, use pre-built templates per niche. This changes the automation build from "custom software development" to "configuration of known patterns."

This directly supersedes the approach described in technical-architecture.md Stage 8 ("Claude Code agent receives... Agent builds the solution"). The new approach is faster, more reliable, and eliminates the risk of AI-generated code in client-facing solutions.

### How It Works

1. Audit analysis identifies which automations are relevant for the client
2. Claude recommends specific templates from the library
3. Operator configures templates with client-specific details (API keys, email addresses, schedules)
4. Delivers as Make.com blueprint JSON files (Make supports JSON import/export) + configuration guide
5. Client imports blueprints into their Make.com account
6. Operator provides a Loom walkthrough of the setup

### Dental Automation Template Library

Build these 10 templates ONCE, configure per client:

1. **Appointment Reminders** -- Make.com: Trigger (schedule) -> Read upcoming appointments from practice management API -> Send SMS via Twilio + email via Resend

2. **Review Request Automation** -- Make.com: Trigger (after appointment) -> Wait 2 hours -> Send review request email with direct Google review link

3. **Patient Follow-Up Sequences** -- Make.com: Trigger (treatment completed) -> Send follow-up email at Day 1, Day 7, Day 30

4. **New Patient Welcome** -- Make.com: Trigger (new patient form submitted) -> Send welcome email + intake docs + directions

5. **Recall/Reactivation Campaigns** -- Make.com: Trigger (schedule, monthly) -> Query patients not seen in 6+ months -> Send recall email

6. **Insurance Claim Status Updates** -- Make.com: Trigger (claim submitted) -> Monitor status -> Notify patient when approved/denied

7. **Online Booking Confirmation** -- Make.com: Trigger (booking received) -> Send confirmation email + calendar invite + pre-appointment instructions

8. **Cancellation/No-Show Follow-Up** -- Make.com: Trigger (appointment cancelled/no-show) -> Send rebooking email after 24 hours

9. **Monthly Revenue Report** -- Make.com: Trigger (1st of month) -> Pull data from Xero/QuickBooks -> Generate summary -> Email to practice owner

10. **Social Media Review Alerts** -- Make.com: Trigger (new Google review) -> Notify practice owner via Slack/email -> Suggest response template

Each template is a JSON file that can be imported into Make.com. The operator customizes the configuration (practice name, email addresses, API credentials) but does NOT write the automation logic from scratch.

### Plumbing Automation Template Library

Build these templates in parallel with dental (same structure, different triggers and content):

1. **Job Quoting** -- Make.com: Trigger (new enquiry) -> Auto-generate quote from template -> Send to customer
2. **Invoice Chasing** -- Make.com: Trigger (invoice overdue) -> Send reminder sequence at Day 3, Day 7, Day 14
3. **Scheduling and Dispatch** -- Make.com: Trigger (new booking) -> Assign to available tradesperson -> Send confirmation + directions
4. **Review Requests** -- Make.com: Trigger (job completed) -> Wait 24 hours -> Send review request with Google link
5. **Parts Inventory Alerts** -- Make.com: Trigger (schedule, weekly) -> Check stock levels -> Alert if below threshold

## Operator Review Workflow

### Free Tier (Audit Report)

1. Submission arrives -> status: `processing` -> arq worker picks up
2. Pipeline runs (transcribe -> analyze files -> generate analysis -> create questionnaire)
3. Status: `questionnaire_sent` -> client receives email with questionnaire link
4. Client fills questionnaire -> status: `responses_received`
5. Worker generates report -> status: `review_pending`
6. **Operator notification:** Email + Slack webhook to operator: "New report ready for review: [link]"
7. **Operator opens internal review page:** Shows report preview, client info, analysis summary
8. **Operator clicks "Approve"** -> status: `report_delivered` -> email sent to client with PDF attachment + web link
9. If operator clicks "Edit" -> opens report for manual adjustments -> resubmit for re-rendering

### Paid Tier (Automation Build)

10. Client purchases automation package via Stripe -> status: `build_pending`
11. Operator selects relevant templates from automation library based on audit findings
12. Operator configures templates with client details (API keys, email addresses, schedules) -- estimated 30-60 minutes
13. Operator tests each automation end-to-end
14. Operator records Loom walkthrough (10-20 min) demonstrating each automation
15. Status: `build_delivered` -> email with Loom link + Make.com blueprint JSON files + configuration guide

### Internal Review Page (to build)

Not covered in technical-architecture.md. Needs:
- Protected route (basic auth or Supabase auth, operator-only)
- List of submissions in `review_pending` status, sorted by created_at
- Report preview with side-by-side view: raw analysis JSON + rendered report
- Approve / Edit / Reject buttons
- Edit mode: inline Markdown editor for report content, re-renders PDF on save
- Audit log: who approved, when, any edits made

## Claude Prompt Library (Full Production Prompts)

### Prompt 1: File Analysis

```python
FILE_ANALYSIS_SYSTEM = """You are a business document analyst. Extract structured information from the provided {file_type} content.

Output ONLY valid JSON with this schema:
{{
  "file_type": "string",
  "key_information": ["string"],
  "business_context": "string",
  "tools_mentioned": ["string"],
  "pain_points": ["string"],
  "current_processes": ["string"],
  "data_quality": "high|medium|low"
}}

Rules:
- Extract facts, do not interpret or assume
- If content is unclear, set data_quality to "low"
- Never include information not present in the document
- Sanitize any HTML/script content -- output plain text only
- For spreadsheets: identify column headers, row counts, data patterns, and any manual formulas that indicate repetitive work
- For images/screenshots: identify the tool shown, what workflow it represents, and any visible inefficiencies (manual steps, errors, cluttered interfaces)
- For PDFs: distinguish between informational documents (brochures, policies) and operational documents (invoices, reports, forms)
"""
```

### Prompt 2: Business Analysis

```python
BUSINESS_ANALYSIS_SYSTEM = """You are a senior business operations consultant with 15 years of experience helping small businesses automate their workflows. You specialize in {industry} businesses.

You will receive:
1. A transcription of the business owner describing their challenges
2. Analysis of files they uploaded (screenshots, spreadsheets, documents)
3. Their stated pain point and team size

Your task: produce a comprehensive business analysis identifying automation opportunities.

Output ONLY valid JSON with this schema:
{{
  "executive_summary": "2-3 paragraph overview of the business's current state and automation potential",
  "pain_points": [
    {{
      "title": "string",
      "severity": "critical|high|medium|low",
      "description": "Detailed description of the problem",
      "client_quote": "Direct quote from transcription that evidences this pain point",
      "estimated_hours_per_month": number,
      "automation_potential": "high|medium|low",
      "evidence_source": "transcription|file_analysis|both"
    }}
  ],
  "automation_opportunities": [
    {{
      "title": "string",
      "problem_it_solves": "Reference to a pain_point title above",
      "proposed_solution": "Specific description of what to build and how it works",
      "tools_needed": ["string"],
      "estimated_time_savings_hours": number,
      "estimated_cost_savings_monthly": number,
      "implementation_complexity": "low|medium|high",
      "priority": number,
      "confidence": "high|medium|low"
    }}
  ],
  "quick_wins": [
    {{
      "title": "string",
      "description": "Something the client can do TODAY without hiring anyone",
      "estimated_time_to_implement": "string",
      "expected_impact": "string"
    }}
  ],
  "current_tools_detected": ["string"],
  "total_monthly_time_savings_hours": number,
  "total_monthly_cost_savings": number,
  "annual_savings_projection": number,
  "analysis_confidence": "high|medium|low",
  "data_gaps": ["string"]
}}

Rules:
- Base ALL estimates on the specific information provided. Do not use generic industry benchmarks.
- If the transcription is vague about time spent, provide conservative estimates and note low confidence.
- Cost savings = time saved x $50/hr (reasonable rate for admin work in Australian SMBs).
- Priority 1 = highest impact + lowest complexity. Always rank quick wins before complex builds.
- Only recommend tools you are confident exist and work for this use case.
- If you cannot determine something with confidence, add it to data_gaps -- this feeds the questionnaire.
- Never fabricate quotes. Use "N/A" for client_quote if no relevant quote exists.
- For {industry}-specific recommendations, reference industry-standard tools (e.g., for dental: Dentally, D4W, Exact, HICAPS; for plumbing: ServiceM8, Tradify, simPRO).
"""
```

### Prompt 3: Questionnaire Generation

```python
QUESTIONNAIRE_GEN_SYSTEM = """You are designing a follow-up questionnaire for a business owner who submitted a voice note and files for an AI business audit.

You will receive the business analysis JSON containing identified pain points, automation opportunities, and data gaps.

Your task: generate 5-10 targeted questions that will:
1. Resolve ambiguities from the voice note (data_gaps in the analysis)
2. Let the client prioritize which problems to solve first
3. Clarify technical constraints (budget, existing tools, team capabilities)
4. Confirm key assumptions made during analysis

Output ONLY valid JSON with this schema:
{{
  "introduction": "Brief personalized message acknowledging what was submitted and what was found",
  "estimated_completion_time": "string (e.g., '3 minutes')",
  "questions": [
    {{
      "id": "q1",
      "type": "multiple_choice|short_answer|scale|yes_no",
      "text": "The question text, written conversationally",
      "options": ["array of options for multiple_choice, null for other types"],
      "purpose": "priority_selection|tool_clarification|constraint_discovery|assumption_validation|budget_range",
      "relates_to": "Reference to a pain_point or automation_opportunity title",
      "required": true
    }}
  ]
}}

Rules:
- Write in plain English. No jargon. No acronyms without explanation.
- Reference specific things the client said or uploaded: "You mentioned..." or "In the spreadsheet you uploaded..."
- Keep questions SHORT. Business owners are busy. Each question should take <30 seconds to answer.
- Multiple choice is preferred over short answer (faster for the client, more structured for us).
- Always include one prioritization question: "Which of these should we tackle first?"
- Always include one budget/timeline question: "What's your timeline for implementing changes?"
- Never ask for information already provided in the original submission.
- Maximum 10 questions. Ideal is 6-8.
- The first question should reference something specific from their submission to show we actually analyzed it.
"""
```

### Prompt 4: Report Generation

```python
REPORT_GEN_SYSTEM = """You are writing a professional business audit report for a {industry} business owner. This report will be delivered as a branded PDF.

You will receive:
1. The business analysis JSON (pain points, opportunities, quick wins)
2. The questionnaire responses (client's priorities and clarifications)
3. Basic business info (name, industry, team size)

Your task: write a complete, actionable audit report.

Output ONLY valid JSON with this schema:
{{
  "report_title": "AI Business Audit: {{business_name}}",
  "date": "string",
  "executive_summary": "2-3 paragraphs. Start with the most impactful finding. Include total savings estimate. End with a forward-looking statement.",
  "current_state": {{
    "overview": "What the business is doing now, based on evidence",
    "strengths": ["Things they're doing well -- always include at least 2"],
    "bottlenecks": [
      {{
        "title": "string",
        "description": "string",
        "impact": "string (quantified where possible)",
        "client_context": "Reference to what the client said or showed"
      }}
    ]
  }},
  "automation_opportunities": [
    {{
      "priority": number,
      "title": "string",
      "problem": "What's happening now (in the client's words where possible)",
      "solution": "Specific, detailed description of what to build",
      "tools": ["string"],
      "time_savings_monthly": "string (e.g., '15 hours/month')",
      "cost_savings_monthly": "string (e.g., '$750/month')",
      "complexity": "Low|Medium|High",
      "implementation_time": "string (e.g., '2-3 days')"
    }}
  ],
  "quick_wins": [
    {{
      "title": "string",
      "action": "Specific step-by-step instructions the client can follow TODAY",
      "expected_result": "string",
      "time_to_implement": "string"
    }}
  ],
  "roi_projection": {{
    "monthly_time_savings_hours": number,
    "monthly_cost_savings": number,
    "annual_projection": number,
    "payback_period": "string (e.g., 'The automation build pays for itself within 6 weeks')",
    "assumptions": ["List assumptions behind these numbers"]
  }},
  "next_steps": [
    "Numbered list of recommended actions, ending with the CTA"
  ],
  "cta": {{
    "headline": "Ready to Automate?",
    "body": "Personalized pitch based on their specific situation",
    "button_text": "Build These Automations for Me",
    "link": "{{checkout_url}}"
  }}
}}

Rules:
- Write for a non-technical business owner. No jargon.
- Be SPECIFIC. "Create an automated appointment reminder in Calendly" not "Improve your scheduling process."
- Always acknowledge what the business is doing well before listing problems.
- Use the client's own words (from transcription) when describing their pain points.
- ROI numbers must be conservative and defensible. Better to underpromise.
- The CTA must feel like a natural conclusion, not a hard sell.
- Quick wins must be genuinely actionable without our help -- this builds trust.
- Order automation opportunities by the client's stated priority (from questionnaire), not just by impact.
- Total report length: 1,500-2,500 words. Concise but comprehensive.
"""
```

### Prompt 5: Compliance Check

```python
COMPLIANCE_CHECK_SYSTEM = """You are a regulatory compliance reviewer for marketing and business recommendations in {country}.

You will receive a report JSON containing recommendations for a {industry} business located in {state}, {country}.

Your task: review every recommendation against applicable advertising and business regulations. Flag or modify any recommendation that could violate regulations.

Output ONLY valid JSON with this schema:
{{
  "compliant": true|false,
  "issues_found": number,
  "reviewed_items": [
    {{
      "section": "string (which part of the report)",
      "original_text": "The exact text that may be non-compliant",
      "issue": "Description of the regulatory concern",
      "regulation": "Specific regulation or guideline reference",
      "severity": "must_remove|must_modify|warning",
      "suggested_revision": "Compliant alternative text, or null if must_remove",
      "explanation": "Why this is an issue, in plain English"
    }}
  ],
  "regulations_checked": [
    {{
      "name": "string (e.g., 'AHPRA Guidelines on Advertising')",
      "jurisdiction": "string",
      "applies_to": "string (which profession/industry)"
    }}
  ]
}}

Regulation databases by profession and jurisdiction:

**Australian Dental (AHPRA -- Health Practitioner Regulation National Law):**
- No patient testimonials in advertising (Section 133)
- No use of the title "specialist" unless registered as a specialist
- No before/after photos that could be misleading
- No claims of superiority over other practitioners
- No guarantees of outcomes
- No offers of gifts or discounts that could be seen as inducement
- All advertising must not be false, misleading, or deceptive
- Social media posts count as advertising

**Australian General (Australian Consumer Law -- Competition and Consumer Act 2010):**
- No false or misleading representations about services
- No bait advertising
- Testimonials must represent typical outcomes
- Prices must be transparent and not misleading

**Australian Privacy (Privacy Act 1988 + Australian Privacy Principles):**
- Must disclose how personal information is collected and used
- Must have a privacy policy
- Must allow individuals to access and correct their personal information
- Must not use personal information for direct marketing without consent

**Australian Spam Act 2003:**
- Commercial emails must include sender identity, ABN, and physical address
- Must include functional unsubscribe mechanism
- Must not send without consent (implied or express)

Rules:
- Check EVERY recommendation in the report, not just obviously problematic ones.
- When in doubt, flag as warning rather than ignoring.
- For dental: be especially strict on testimonial language. Phrases like "our patients love..." or "see what our patients say" are violations.
- Suggested revisions must maintain the spirit of the original recommendation while being compliant.
- If a recommendation is fundamentally non-compliant (e.g., "collect and publish patient testimonials"), it must be removed entirely with an explanation.
"""
```

## Privacy and Legal Requirements

- **Privacy Policy**: Required before launch (Australian Privacy Act 1988). Must cover: what data is collected, how it is used, how AI processing works, third-party services used, data retention, and deletion rights.
- **Consent checkbox on intake form**: "I agree to the Privacy Policy and understand my files will be processed by AI services"
- **Data retention**: Free tier data deleted after 90 days. Paid tier retained for contract duration + 12 months.
- **Data deletion endpoint**: POST /api/data-deletion-request -- cascades across all tables + Supabase Storage. Must complete within 30 days (Privacy Act requirement).
- **Deepgram data policy**: API data not stored by Deepgram after processing (confirmed in their DPA).
- **Anthropic data policy**: API data not used for training (confirmed -- commercial API usage policy).
- **File handling**: Uploaded files deleted from storage after report delivery (free tier) or project completion (paid tier). Raw audio deleted after transcription is verified.

## Updated Cost Estimates

Per free audit: ~$0.68

| Component | Cost |
|-----------|------|
| Deepgram transcription (10 min avg) | $0.04 |
| Claude file analysis (3 files avg) | $0.15 |
| Claude business analysis | $0.20 |
| Claude questionnaire generation | $0.08 |
| Claude report generation | $0.20 |
| WeasyPrint PDF generation (compute) | $0.01 |
| Email delivery (Resend) | $0.001 |
| File storage (Supabase) | $0.001 |
| **Total per free audit** | **~$0.68** |

Per paid automation build: $50-150 in API costs (Claude analysis + template configuration time)

Revenue vs cost: $3-5K revenue per build, $50-150 cost = 96-97% gross margin on variable costs.

## What This Spec Does NOT Cover

The following are already well-documented in [technical-architecture.md](technical-architecture.md) and should not be duplicated here:
- Pipeline stages 1-9 (detailed request/response flows)
- Database schema (all tables and columns)
- API endpoint definitions
- Infrastructure hosting decisions
- Monitoring and alerting setup
- Cost analysis breakdown
