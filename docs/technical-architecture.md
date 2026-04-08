# Technical Architecture

> **Note (2026-04-08):** This document has been updated to align with the unified Python stack decision. See `technical-spec.md` for the builder-focused specification covering gaps in this document (security hardening, automation templates, operator workflow, production prompts).

## System Overview

The AI Business Audit platform is a pipeline-oriented system. Data flows in one direction: client submission -> AI processing -> deliverable output. There is no complex user-facing application state. The frontend is a form. The backend is a series of processing stages. The output is a report.

```
Client Browser                Backend (Render.com)              External Services
--------------                --------------------              -----------------
                                                                
[Voice Recorder] -----> [Audio Upload] -----> [Whisper/Deepgram]
[File Upload]    -----> [File Storage] -----> [Claude Multimodal]
[Info Fields]    -----> [Supabase]     -----> [Claude Analysis]
                                       -----> [Claude Questionnaire]
                                       -----> [Claude Report Gen]
                                       -----> [WeasyPrint PDF]
                                       -----> [Resend Email]
                                       -----> [Stripe Checkout]
```

## Frontend

### Tech Stack
- **Next.js** (App Router) or simple **HTML/JS** -- the frontend is intentionally minimal
- The entire client-facing surface is: one landing page, one intake form, one questionnaire page, one report viewing page
- No authentication, no user accounts, no dashboards for the free tier

### Voice Recording
```
Browser MediaRecorder API
  -> Records audio as webm/opus (best compression) or wav (fallback)
  -> Blob stored in memory during recording
  -> On submit: uploaded as multipart/form-data to backend
  -> Visual waveform using Web Audio API (AnalyserNode)
  -> Max recording length: 60 minutes (enforced client-side)
```

### File Upload
```
Drag-and-drop zone (react-dropzone or native HTML5 drag events)
  -> Accepted types: .png, .jpg, .jpeg, .heic, .pdf, .xlsx, .xls, .csv, .docx, .doc, .txt
  -> Per-file limit: 50MB
  -> Total submission limit: 200MB
  -> Upload progress via XMLHttpRequest or fetch with ReadableStream
  -> Files uploaded to Supabase Storage (or S3) with signed URLs
```

### Form Fields
```
Standard HTML form with validation
  -> Business name: text, required
  -> Industry: select dropdown, required
  -> Team size: select dropdown, required
  -> Main pain point: select dropdown, required
  -> Email: email input, required, validated
  -> Name: text, required
```

### Questionnaire Page
```
Dynamic form rendered from JSON schema
  -> Each question has a type: multiple_choice, short_answer, scale (1-5)
  -> Unique URL per client: /questionnaire/{submission_id}?token={auth_token}
  -> No login required -- token in URL provides access
  -> Responses submitted back to backend API
```

### Report Page
```
Server-rendered HTML page
  -> Unique URL per client: /report/{submission_id}?token={auth_token}
  -> Responsive design, printable
  -> Download as PDF button (links to pre-generated PDF in storage)
  -> "Build This For Me" CTA button -> Stripe checkout
```

## Backend Pipeline

### Stage 1: Submission Ingestion

**Endpoint:** `POST /api/submit`

```
Request:
  - audio: File (webm/wav)
  - files[]: File[] (multiple)
  - businessName: string
  - industry: string
  - teamSize: string
  - painPoint: string
  - email: string
  - name: string

Processing:
  1. Validate all fields
  2. Generate unique submission_id (UUID v4)
  3. Upload audio to Supabase Storage: /submissions/{id}/audio.webm
  4. Upload each file to Supabase Storage: /submissions/{id}/files/{filename}
  5. Create submission record in Supabase:
     - id, business_name, industry, team_size, pain_point, email, name
     - status: "processing"
     - created_at, updated_at
  6. Enqueue processing job (or process inline for MVP)

Response: { submissionId, status: "processing", message: "We'll email your questionnaire within 2 hours." }
```

### Stage 2: Transcription

**Trigger:** Submission created

```
Option A: Whisper (self-hosted, free)
  - whisper.cpp or faster-whisper running on backend server
  - Model: whisper-large-v3 (best accuracy)
  - Convert webm to wav if needed (ffmpeg)
  - Output: timestamped text transcription
  - Processing time: ~1-3 minutes for a 10-minute recording
  - Cost: $0 (compute only)

Option B: Deepgram API (hosted, fast)
  - POST audio to Deepgram Nova-2 model
  - $0.0043/min (~$0.04 for 10 minutes)
  - Output: JSON with transcription, confidence scores, paragraphs
  - Processing time: ~10-30 seconds for a 10-minute recording
  - Advantage: faster, more reliable, no server GPU needed

**Pricing note (verified 2026-04-07):** Deepgram $0.0043/min applies to Nova-2 batch (pre-recorded) audio only. Nova-3 model costs $0.0077/min. Alternative: Whisper (free, runs locally) for zero-cost transcription at the expense of slightly lower accuracy.

Decision: Start with Deepgram for speed and reliability. Switch to self-hosted Whisper if volume makes the cost meaningful (it won't for a long time -- even 1,000 submissions/month at 10 min average = $43/month).
```

**Storage:** Transcription stored in Supabase as JSON field on submission record.

### Stage 3: File Analysis

**Trigger:** Transcription complete

```
For each uploaded file:
  1. Determine file type
  2. Process accordingly:
     - Images/Screenshots: Send directly to Claude Vision API
     - PDFs: Extract text (pdf-parse), also send pages as images to Claude Vision
     - Spreadsheets: Parse with openpyxl + pandas, convert to structured JSON
     - Documents: Extract text (mammoth for docx, raw read for txt)
  3. Claude analyzes each file with context from the transcription:
     System prompt: "You are analyzing files uploaded by a business owner as part of a business audit. The business owner described their situation as follows: {transcription_excerpt}. Analyze this file and identify: what tool/process it represents, what inefficiencies are visible, what data patterns exist, and what automation opportunities it suggests."
  4. Store per-file analysis in Supabase

Estimated cost: $0.05-$0.20 per file (depending on size and type)
```

### Stage 4: Business Analysis Generation

**Trigger:** All files analyzed

```
Claude API call:
  Model: claude-sonnet-4-20250514 (best balance of quality and cost for this task)
  
  System prompt: Senior business consultant role, structured output format
  
  User prompt includes:
    - Full transcription
    - All file analyses
    - Basic info (industry, team size, pain point)
  
  Output: Structured JSON
  {
    "executive_summary": "...",
    "pain_points": [
      {
        "title": "Manual invoice follow-ups",
        "severity": "high",
        "description": "...",
        "client_quote": "I spend 2 hours every morning chasing invoices",
        "estimated_hours_per_month": 40,
        "automation_potential": "high"
      }
    ],
    "automation_opportunities": [
      {
        "title": "Automated invoice reminders",
        "problem_it_solves": "Manual invoice follow-ups",
        "proposed_solution": "...",
        "tools_needed": ["Xero API", "n8n", "custom email templates"],
        "estimated_time_savings_hours": 35,
        "estimated_cost_savings_monthly": 1750,
        "implementation_complexity": "low",
        "priority": 1
      }
    ],
    "quick_wins": [...],
    "total_monthly_time_savings_hours": 67,
    "total_monthly_cost_savings": 3350,
    "annual_savings_projection": 40200
  }

Estimated cost: $0.10-$0.30
```

### Stage 5: Questionnaire Generation

**Trigger:** Business analysis complete

```
Claude API call:
  Input: Business analysis JSON
  
  Output: Questionnaire JSON
  {
    "questions": [
      {
        "id": "q1",
        "type": "multiple_choice",
        "text": "You mentioned 3 main bottlenecks...",
        "options": ["A) Appointment scheduling", "B) Invoice follow-ups", "C) Social media"],
        "purpose": "priority_selection"
      },
      {
        "id": "q2",
        "type": "short_answer",
        "text": "What accounting software are you currently using?",
        "purpose": "tool_clarification"
      }
    ]
  }

Questionnaire stored in Supabase.
Unique URL generated: /questionnaire/{submission_id}?token={random_token}
Email sent to client via Resend with the questionnaire link.

Estimated cost: $0.05-$0.10
```

### Stage 6: Report Generation

**Trigger:** Client completes questionnaire

```
Claude API call:
  Input: Business analysis JSON + questionnaire responses
  
  Output: Full report content (Markdown or structured JSON)
  
  Report sections:
    1. Executive Summary
    2. Current State Assessment
    3. Automation Opportunities (detailed, with ROI per item)
    4. Quick Wins
    5. 12-Month ROI Projection
    6. Recommended Next Steps + CTA

Report rendering:
  - Markdown -> HTML using a branded template (CSS styling, company logo, professional layout)
  - HTML -> PDF using WeasyPrint (Python HTML-to-PDF library)
  - PDF stored in Supabase Storage: /submissions/{id}/report.pdf
  - HTML version served at /report/{submission_id}?token={token}

Email sent to client:
  Subject: "Your AI Business Audit Report is Ready"
  Body: 2-3 highlight stats + "View Full Report" button
  
Estimated cost: $0.15-$0.30
```

### Stage 7: Payment Processing

**Trigger:** Client clicks "Build This For Me" in report

```
Stripe Checkout Session:
  - Product: "AI Business Audit - Automation Build"
  - Price: $3,000 / $4,000 / $5,000 (based on complexity tier)
  - Metadata: submission_id, selected_automations[]
  
Stripe webhook (checkout.session.completed):
  1. Update submission status to "paid"
  2. Create project record in Supabase:
     - project_id, submission_id, tier, amount_paid
     - status: "queued"
     - selected_automations (from audit report)
  3. Send confirmation email to client
  4. Notify operator (Slack webhook or email)
```

### Stage 8: Solution Building

**Trigger:** Payment confirmed

```
For each selected automation:
  1. Claude Code agent receives:
     - Automation specification from audit report
     - Client's current tools and constraints (from questionnaire)
     - Implementation requirements
  2. Agent builds the solution:
     - n8n workflows: exported as shareable JSON files (self-hosted on Render.com)
     - Custom scripts: GitHub repo with documentation
     - API integrations: deployed and configured
     - Dashboards: built and shared
  3. Solution artifacts stored in Supabase Storage
  4. Operator notified for review

Operator review (~20 min per automation):
  - Test the solution
  - Verify it matches the spec
  - Record Loom walkthrough
  - Upload Loom link to project record
```

### Stage 9: Delivery

**Trigger:** Operator approves all solutions

```
Email to client:
  Subject: "Your Automation Solutions Are Ready"
  Body:
    - Summary of what was built
    - Loom video link for each automation
    - Documentation links
    - "Your 30-day support window starts today"
    - Slack Connect invitation (or email support instructions)

Update project status to "delivered"
```

## Infrastructure

### Hosting
- **Application server:** Render.com (Web Service, $25-$50/month)
  - Python 3.12+ with FastAPI runtime
  - Auto-scaling if needed (but unlikely at early stage)
  - Health checks and auto-restart
- **Background jobs:** Render.com Background Worker (same deployment) or arq with Redis

### Database
- **Supabase** (PostgreSQL)
  - Tables: submissions, questionnaires, questionnaire_responses, reports, projects, automations
  - Row Level Security (RLS) policies for data isolation
  - Realtime subscriptions for status updates (optional)

### File Storage
- **Supabase Storage** (S3-compatible)
  - Buckets: audio, uploads, reports, solutions
  - Signed URLs for secure access (24-hour expiry for reports)
  - Storage policies: authenticated upload, public read with token

### Payments
- **Stripe**
  - Checkout Sessions for one-time payments
  - Customer Portal for receipt access
  - Webhooks for payment confirmation
  - Products/Prices configured for each tier

### Email
- **Resend** (preferred) or **SendGrid**
  - Transactional emails: questionnaire delivery, report delivery, solution delivery
  - Email templates stored in codebase (Jinja2 HTML templates + Resend SDK)
  - Custom domain for sender reputation

### Video Delivery
- **Loom** (manual recording by operator)
  - Each automation walkthrough: 5-10 minute Loom video
  - Links embedded in delivery email
  - Loom Business account for custom branding (optional)

### Client Communication
- **Slack Connect** (preferred for retainer clients)
  - Dedicated channel per client
  - Async Q&A within the 30-day support window
- **Email** (fallback for all tiers)

## Database Schema

```sql
-- Core submission from the intake form
CREATE TABLE submissions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  business_name TEXT NOT NULL,
  industry TEXT NOT NULL,
  team_size TEXT NOT NULL,
  pain_point TEXT NOT NULL,
  email TEXT NOT NULL,
  name TEXT NOT NULL,
  audio_url TEXT,
  transcription JSONB,
  file_analyses JSONB,
  business_analysis JSONB,
  status TEXT DEFAULT 'processing',
  -- status values: processing, questionnaire_sent, questionnaire_completed,
  --                report_generated, report_delivered, paid, building, delivered
  access_token TEXT NOT NULL DEFAULT gen_random_uuid()::text,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- AI-generated questionnaire
CREATE TABLE questionnaires (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  submission_id UUID REFERENCES submissions(id),
  questions JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Client's responses to the questionnaire
CREATE TABLE questionnaire_responses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  questionnaire_id UUID REFERENCES questionnaires(id),
  submission_id UUID REFERENCES submissions(id),
  responses JSONB NOT NULL,
  completed_at TIMESTAMPTZ DEFAULT now()
);

-- Generated audit report
CREATE TABLE reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  submission_id UUID REFERENCES submissions(id),
  content JSONB NOT NULL,
  pdf_url TEXT,
  html_url TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Paid project (after Stripe checkout)
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  submission_id UUID REFERENCES submissions(id),
  report_id UUID REFERENCES reports(id),
  stripe_session_id TEXT,
  stripe_customer_id TEXT,
  tier TEXT NOT NULL,
  amount_paid INTEGER NOT NULL, -- cents
  selected_automations JSONB,
  status TEXT DEFAULT 'queued',
  -- status values: queued, building, review, delivered
  delivered_at TIMESTAMPTZ,
  support_ends_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Individual automation within a project
CREATE TABLE automations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id),
  title TEXT NOT NULL,
  description TEXT,
  specification JSONB,
  solution_artifacts JSONB, -- URLs to files, repos, etc.
  loom_url TEXT,
  documentation_url TEXT,
  status TEXT DEFAULT 'pending',
  -- status values: pending, building, review, approved, delivered
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- Retainer subscriptions
CREATE TABLE retainers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  project_id UUID REFERENCES projects(id),
  submission_id UUID REFERENCES submissions(id),
  stripe_subscription_id TEXT,
  tier TEXT NOT NULL,
  monthly_amount INTEGER NOT NULL, -- cents
  automations_per_month INTEGER NOT NULL,
  status TEXT DEFAULT 'active',
  slack_channel_id TEXT,
  started_at TIMESTAMPTZ DEFAULT now(),
  cancelled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

## API Endpoints

```
POST   /api/submit                    -- Submit intake form (voice + files + info)
GET    /api/submission/:id/status     -- Check submission processing status
GET    /api/questionnaire/:id         -- Get questionnaire for a submission
POST   /api/questionnaire/:id/respond -- Submit questionnaire responses
GET    /api/report/:id                -- Get report (HTML view)
GET    /api/report/:id/pdf            -- Download report PDF
POST   /api/checkout                  -- Create Stripe checkout session
POST   /api/webhooks/stripe           -- Stripe webhook handler
GET    /api/project/:id/status        -- Check project build status (for paid clients)
```

## Cost Analysis Per Submission

| Component | Cost |
|-----------|------|
| Deepgram transcription (10 min avg) | $0.04 |
| Claude file analysis (3 files avg) | $0.15 |
| Claude business analysis | $0.20 |
| Claude questionnaire generation | $0.08 |
| Claude report generation | $0.20 |
| PDF generation (WeasyPrint compute) | $0.01 |
| Email delivery (Resend) | $0.001 |
| File storage (Supabase) | $0.001 |
| **Total per free audit** | **$0.68** |

At 100 free audits/month: $68/month in variable costs.
At 10% conversion to paid tier ($4,000 avg): $40,000 revenue, $68 cost = 99.8% gross margin on variable costs.

## Security Considerations

- All data encrypted in transit (HTTPS/TLS)
- Supabase handles encryption at rest
- Access tokens in URLs (not ideal, but acceptable for MVP -- no user accounts)
- Rate limiting on submission endpoint (5 per IP per hour)
- File upload validation (MIME type checking, virus scanning via ClamAV optional)
- No client data used for AI model training (Anthropic API data usage policy confirms this)
- GDPR-compliant data deletion on request
- Stripe handles all payment data (PCI compliant, no card data touches our servers)

## Monitoring and Alerting

- **Render.com** built-in metrics: CPU, memory, response times
- **Sentry** for error tracking (frontend and backend)
- **Supabase Dashboard** for database monitoring
- **Stripe Dashboard** for payment monitoring
- **Custom Slack alerts** for: new submissions, completed reports, new payments, errors

## Error Handling Strategy

| Service | Failure Mode | Response |
|---------|-------------|----------|
| Deepgram | API timeout/error | Retry 2x with exponential backoff. Fallback to Whisper (local). |
| Claude API | Rate limit or error | Retry 2x. If persistent, queue for manual processing. |
| File processing | Malformed/corrupted file | Skip file, note in analysis: "File [name] could not be processed." |
| ClamAV | Virus detected | Reject submission. Notify operator. Do NOT process file. |
| Stripe | Webhook failure | Return 200 to Stripe immediately. Log error. Alert operator. |
| Supabase | Connection failure | Retry 3x. If persistent, return 503 to client. |
| Resend | Email delivery failure | Log to email_events table. Retry after 1 hour, max 3 retries. |

## Environment Variables

See `.env.example` in the repo root for the complete list. Key variables:

- `ANTHROPIC_API_KEY` — Claude API access
- `DEEPGRAM_API_KEY` — Speech-to-text ($0.0043/min Nova-2 batch)
- `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` — Database
- `REDIS_URL` — Job queue (arq)
- `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET` — Payments
- `RESEND_API_KEY` — Email delivery
- `SENTRY_DSN` — Error tracking
- `TURNSTILE_SECRET_KEY` — CAPTCHA on intake form
- `CLAMAV_HOST` — Virus scanning service
