# Product Specification

## Overview

AI Business Audit is an async, AI-powered consulting service that replaces traditional discovery meetings with voice intake, automated analysis, and delivered automation solutions. The entire client experience -- from initial contact to delivered solution -- happens without a single meeting.

## Complete User Journey

### Stage 1: Intake

The business owner lands on the website. The hero section has one clear CTA: "Record a voice note about your biggest bottleneck."

**Voice Recording**
- Browser-based recording using the MediaRecorder API (works on desktop and mobile, no app install needed)
- No time limit -- clients can record 2 minutes or 45 minutes. More context = better analysis.
- Visual waveform feedback so the client knows it's recording
- Option to re-record before submitting
- Fallback: text input field for clients who prefer typing (but voice is encouraged because people share more when talking than typing)

**File Uploads**
- Drag-and-drop zone supporting: images (PNG, JPG, HEIC), PDFs, spreadsheets (XLSX, CSV, Google Sheets export), documents (DOCX, TXT), screenshots
- No file size limit displayed to the user (backend limit: 50MB per file, 200MB total per submission)
- Upload progress indicators
- Clients can upload screenshots of their current tools, spreadsheets they maintain manually, email threads showing their workflow chaos, anything relevant

**Basic Info Fields**
- Business name (text)
- Industry (dropdown: Professional Services, Healthcare, E-commerce, Real Estate, Legal, Accounting, Trades, Marketing Agency, Other)
- Team size (dropdown: Just me, 2-5, 6-15, 16-50, 50+)
- Main pain point (dropdown: Too much manual data entry, Slow client communication, Messy scheduling/booking, Invoice and payment follow-ups, Social media is a time sink, Reporting takes forever, Other)
- Email address (for delivery)
- Name (first and last)

**What the client does NOT need to do:**
- Create an account
- Schedule a meeting
- Fill out a 30-question intake form
- Download an app
- Join a video call

### Stage 2: AI Analysis

The backend processes the submission immediately upon receipt.

**Transcription**
- Voice note is transcribed using Whisper (local, free for self-hosted) or Deepgram API ($0.0043/min, ~$0.04 for a 10-minute voice note)
- Speaker diarization not needed (single speaker)
- Transcription stored in database alongside original audio

**Multimodal File Processing**
- Claude reads all uploaded files using multimodal capabilities:
  - Screenshots and images: Vision API reads text, identifies UI elements, understands tool interfaces
  - PDFs: Extracted text + visual layout analysis
  - Spreadsheets: Parsed into structured data, column headers identified, data patterns analyzed
  - Documents: Full text extraction and analysis
- Each file is summarized and tagged with relevant business context

**Business Analysis Generation**
- Claude receives: full transcription + all file summaries + basic info fields
- System prompt instructs Claude to act as a senior business consultant identifying:
  - Pain points (ranked by severity and frequency of mention)
  - Current workflow bottlenecks (mapped from voice description + uploaded evidence)
  - Automation opportunities (specific, actionable, with estimated time savings)
  - Quick wins vs. strategic improvements
  - Estimated monthly time savings per automation (in hours)
  - Estimated monthly cost savings (time saved x reasonable hourly rate for the role)
- Output: Structured JSON containing the complete analysis
- Estimated API cost: $0.15-$0.40 depending on input size

### Stage 3: Smart Questionnaire

This is the innovation that replaces the scoping meeting. Instead of generic questions, Claude generates a TARGETED questionnaire based on the specific analysis.

**How it works:**
- Claude reads the Business Analysis JSON and generates 5-10 specific decision questions
- Questions are multiple-choice or short-answer, designed to:
  - Prioritize which problems to solve first
  - Clarify ambiguous points from the voice note
  - Understand constraints (budget, timeline, team capabilities)
  - Confirm assumptions made during analysis

**Example questions (generated, not templated):**
- "You mentioned 3 main bottlenecks: appointment scheduling, invoice follow-ups, and social media posting. Which should we solve first? (A) Appointment scheduling -- you said this takes 2 hours daily (B) Invoice follow-ups -- you mentioned $15K in overdue invoices (C) Social media -- you said you haven't posted in 3 weeks"
- "In the screenshot of your spreadsheet, we see you're tracking client appointments in Excel. Are you open to switching to a dedicated scheduling tool like Calendly, or do you need the solution to work within your existing spreadsheet workflow?"
- "You mentioned your team of 4 each handles their own client emails. Would a shared inbox (like Front or Help Scout) work for your team, or does each person need their own communication channel?"

**Delivery:**
- Questionnaire rendered as a clean web form (unique URL per client)
- Link emailed to the client with a note: "We've analyzed your submission. Please answer these 5 questions so we can finalize your audit report. Takes about 3 minutes."
- Responses stored in database, linked to the original submission

### Stage 4: Audit Report (Free Tier)

Once the client completes the questionnaire, Claude generates the final Audit Report.

**Report Contents:**
1. **Executive Summary** -- 2-3 paragraph overview of the business's automation potential
2. **Current State Assessment** -- What the business is doing now, mapped from voice note + files + questionnaire
3. **Automation Opportunities** -- Ranked list of 3-8 specific automations, each with:
   - Problem description (in the client's own words, quoted from transcription)
   - Proposed solution (specific tool or custom build)
   - Estimated time savings (hours/month)
   - Estimated cost savings ($/month)
   - Implementation complexity (Low / Medium / High)
   - Recommended priority (1-5 scale)
4. **Quick Wins** -- Things the client can do TODAY without hiring anyone (tool recommendations, settings changes, template suggestions)
5. **ROI Projection** -- Total estimated savings over 12 months if all recommended automations are implemented
6. **Recommended Next Steps** -- Clear path forward, ending with the CTA for the paid tier

**Report Format:**
- Generated as a branded PDF (using Puppeteer to render HTML template + dynamic content)
- Also available as a web page (unique URL, shareable)
- Clean, professional design -- this IS the sales pitch, it needs to look premium

**Delivery:**
- Emailed to the client with the subject line: "Your AI Business Audit Report is Ready"
- Email includes 2-3 highlight stats from the report (e.g., "We identified 47 hours/month in automation potential across 5 areas")
- CTA button in email: "View Full Report"

**Cost to generate:** $0.50-$1.00 total (transcription + analysis + questionnaire + report generation)

### Stage 5: Solution Build (Paid Tier)

The audit report includes a "Build This For Me" button linking to a Stripe checkout page.

**What the client buys:**
- 3-5 automations from their audit report, built, tested, and delivered
- Each automation includes:
  - Working solution (could be a Zapier/Make workflow, a custom script, an API integration, a chatbot, a dashboard, etc.)
  - Loom video walkthrough (5-10 minutes) showing the solution in action
  - Written documentation for the client's team
  - 30-day support window for questions and tweaks

**Build Process:**
1. Client completes Stripe checkout ($3,000-$5,000 depending on complexity tier selected)
2. Stripe webhook triggers project creation in Supabase
3. AI agents (Claude Code) begin building the automation solutions based on the audit report specifications
4. Operator reviews each solution (~20 minutes per automation) -- this is the quality gate
5. Operator records Loom walkthrough demonstrating each solution
6. Solutions + Loom videos delivered to client via email
7. Client has 30-day window for adjustments

**Timeline:** 5-7 business days from payment to delivery

### Stage 6: Ongoing Optimization (Retainer)

For clients who want continuous improvement, a monthly retainer is offered.

**What the retainer includes:**
- Monthly review of automation performance (are they still working? Saving the expected time?)
- Adjustments and fixes as business needs change
- 2-3 new automations per month based on emerging bottlenecks
- Priority Slack channel for async Q&A
- Monthly summary report showing ROI of all automations

**Retainer structure:**
- $1,500/month: Monitoring + maintenance + 1 new automation/month
- $2,000/month: Monitoring + maintenance + 2 new automations/month
- $2,500/month: Monitoring + maintenance + 3 new automations/month + priority support

## Automation Level

**Target: 85-90% automated.**

What the AI does:
- Transcription (100% automated)
- File analysis (100% automated)
- Business analysis generation (100% automated)
- Questionnaire generation (100% automated)
- Report generation (100% automated)
- Solution building (80-90% automated via Claude Code agents)

What the operator does (10-15%):
- Reviews each audit report before sending (~10 minutes)
- Reviews each built solution before delivery (~20 minutes per automation)
- Records Loom walkthroughs (~10 minutes per automation)
- Handles edge cases and client questions that the AI can't resolve
- Quality assurance on all deliverables

**Operator time per client:**
- Free tier (audit only): ~15 minutes
- Paid tier (3-5 automations): ~2-3 hours total
- Retainer: ~3-5 hours/month

## Customer-Facing Language

**Decision (2026-04-07):** The service presents as "a team of experts" — not "AI-powered automation." This is commercially accurate (there IS a human quality gate) and appropriate for the target market (small business owners who are more comfortable paying humans).

Example language:
- "Our team of specialists will analyze your submission"
- "Expert review of your business workflows"
- "Personalized recommendations from our automation team"

## Brand Structure

**Decision (2026-04-07):** This product operates under ONE company brand alongside the niche SEO SaaS product. Anonymous company — no personal name or photo. The company name is TBD (open question).

## Optional Intake Methods

**Consideration (2026-04-07):** In addition to voice notes and file uploads:
- **Loom screen recording**: Client records a walkthrough of their workflow. Frames extracted and fed to Claude Vision for analysis. Not primary intake method but available as an option.
- **Zoom onboarding call**: Deferred for now. May be introduced as premium add-on once operator gains confidence. Screen share would be recorded and processed by AI.

## Non-Functional Requirements

- **Turnaround time:** Audit report delivered within 24 hours of questionnaire completion
- **Paid deliverables:** 5-7 business days from payment
- **Uptime:** 99.5% for the intake form and report viewing pages
- **Data privacy:** All client data encrypted at rest and in transit. Client data deleted upon request. No client data used for AI training.
- **Scalability:** System should handle 50 concurrent submissions without degradation
