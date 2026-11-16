# Taskora AI

An AI workforce for document-heavy professional work. Specialised agents (Legal, CA & Accounting, Insurance, HR, Business, plus custom agents you define) read uploaded documents, extract key details, flag risks and missing items, and draft follow-ups. A person approves every result.

## Quick start

```bash
npm install
cp .env.example .env.local     # fill in DATABASE_URL, GROQ_API_KEY (and Razorpay keys)
npm run dev                    # http://localhost:3000
```

Production: `npm run build && npm run start`.

The **first account you create becomes the platform admin** and can open `/admin`. To add more admins, set `ADMIN_EMAILS`.

Requires **Node 22.13+** and a **Postgres** database (a free [Neon](https://neon.tech) database works; set `DATABASE_URL`). Tables are created automatically on first request.

## Features

| Area | What works |
|---|---|
| Auth | Sign up, log in, log out, password reset (email or server console), invite links, httpOnly session cookies, bcrypt hashes, rate limiting |
| AI Advisor (RAG) | Pick your industry, describe a client matter or paste transactions (attach files, link processed documents). The advisor retrieves relevant law/tax notes, your firm's knowledge base and your documents, then returns the position, applicable rules with citations, quantified solutions, dated next steps, risks, clarifying questions and a draft. Follow-ups keep context. |
| Knowledge base | 44 built-in Indian legal, GST/income-tax, insurance, HR and business reference notes; upload your own precedents, SOPs and circulars; every processed document is indexed automatically; "Test retrieval" shows what the AI will read |
| Documents | Upload PDF/DOCX/TXT/MD/CSV (≤4 MB) or paste text → AI analysis (summary, key fields, findings with severity, missing items, actions, draft) |
| Solutions | Every document analysis now includes practical solutions and the reference knowledge it was grounded in |
| Review | Approve, re-open, **request changes** (agent re-runs with your feedback → new version), wrong-agent detection with one-click re-run, retry on failure, version history |
| Q&A | Chat with any document; answers cite the source |
| Exports | Print/PDF report, Markdown, CSV of key fields, JSON, original file |
| Agents | Enable/disable built-in agents (plan-limited); build custom agents from plain-English instructions (Professional+) |
| Team | Invite members/admins, change roles, remove members (Professional: up to 25) |
| Billing | Razorpay checkout (UPI/cards/net banking), server-side signature verification, webhook, receipts, renewals stack, plans expire back to Free |
| Admin | Website leads (status, CSV export), workspaces (manual plan assignment for Enterprise), payments, platform stats |
| Compliance | Full audit log, configurable data-retention auto-deletion |
| Website | All landing-page forms save to the database; Login/Get Started/Pricing go to the real flows |

## Plans (edit `src/lib/plans.ts`)

| Plan | Price (incl. GST) | Docs / month | Agents | Members |
|---|---|---|---|---|
| Free | ₹0 | 10 | All 5 | 1 |
| Starter | ₹999/mo or ₹9,990/yr | 150 | 1 | 1 |
| Professional | ₹2,999/mo or ₹29,990/yr | 1,000 | All 5 + custom | 25 |
| Enterprise | Custom (set in `/admin`) | Unlimited | All + custom | Unlimited |

## Configuration

See `.env.example`. Key settings:

- `GROQ_API_KEY`: required for AI. Models: `openai/gpt-oss-120b`, with `openai/gpt-oss-20b` as fallback.
- `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` / `RAZORPAY_WEBHOOK_SECRET`: payments. Add the webhook at `{APP_URL}/api/billing/webhook` with events `payment.captured`, `order.paid`, `payment.failed`.
- `SMTP_*`: optional. Without it, reset and invite emails are printed to the server console, and invite links are shown in the UI.

### Groq rate limits
Groq's free tier allows **8,000 tokens per minute per model**. The app retries, switches to the fallback model when one is busy, and analyses only the first part of documents that are too long, with a visible warning. For real workloads, upgrade the Groq account (Dev tier) so long documents are analysed in full.

## How the RAG works

1. **Index**: built-in notes (`src/lib/knowledge/*.ts`), firm uploads and processed documents are split into ~1,400-character passages and stored in `kb_chunks` with a GIN-indexed Postgres `tsvector` (headings weighted higher). No embedding provider is needed.
2. **Rewrite**: the question is turned into statute/concept search terms by the small Groq model (e.g. "cheque bounced" → "dishonour", "section 138").
3. **Retrieve**: passages are ranked BM25-style (IDF-weighted per-term rank), boosted for the chosen industry, max 2 per source.
4. **Generate**: the top passages go to the model as numbered `[S1]…` references; answers cite them and the UI links each citation to its source.

Edit the built-in knowledge in `src/lib/knowledge/` and bump `KNOWLEDGE_VERSION` in `types.ts`; deployed databases re-seed automatically.

## Architecture

```
src/
  proxy.ts                  redirects signed-out users away from /app and /admin
  lib/
    plans.ts                plan catalogue & limits (shared client/server)
    industries.ts           industries, advisor examples, Advice type
    knowledge/              built-in reference knowledge per industry
    analysis.ts             AI result type & status labels
    server/
      db.ts                 Postgres schema + helpers (uploaded files stored as BYTEA)
      auth.ts               sessions, users, workspaces, current-user helpers
      ai.ts                 Groq client, agent + advisor prompts, query rewriting, rate-limit handling
      rag.ts                chunking, indexing, retrieval (Postgres full-text search)
      advisor.ts            consultations: quota, attachments, answer pipeline
      documents.ts          quota checks, processing pipeline, retention
      extract.ts            PDF / DOCX / text extraction
      razorpay.ts           orders, signature checks, plan fulfilment
      mail.ts, http.ts      email + API helpers (errors, rate limit, audit log)
  app/
    (auth)/                 login, signup, forgot/reset password, invite
    app/                    the workspace (overview, documents, agents, team, billing, settings, activity)
    admin/                  platform admin
    api/                    route handlers
  components/
    app/                    workspace UI
    sections/, providers/   marketing site
```

## Deployment (Vercel)

1. `vercel link`, then in the Vercel dashboard → project → **Storage** → create a **Neon** Postgres database and connect it (sets `DATABASE_URL`).
2. Add `GROQ_API_KEY`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` (and optionally `RAZORPAY_WEBHOOK_SECRET`, `APP_URL`, `SMTP_*`) in **Settings → Environment Variables**.
3. `vercel --prod`.

Notes:
- Uploads are limited to 4 MB (Vercel's request body limit is 4.5 MB). Files are stored in Postgres.
- Vercel's free Hobby plan is for non-commercial use; move to Pro before taking real payments.
- Scanned (image-only) PDFs have no text layer; run OCR on them first.
- The legal pages (`/privacy`, `/terms`) are templates. Have them reviewed before launch.
# taskora-ai
