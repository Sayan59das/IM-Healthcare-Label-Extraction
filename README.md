# IMH Label Verification System (IMH-LVS)

A tool for verifying pharma/nutraceutical product label artwork before it
ships: it reads the text off a label (OCR + a local vision-model fallback,
no paid API), compares a new artwork against the last approved version of
the same product and against similar products from other companies, and
produces a signed-off PDF report.

All application code lives in [`IMH-LVS/`](IMH-LVS/).

## What it does

- **Label reading** — extracts the standard fields (brand, product name,
  marketing company, FSSAI number, address, nutrition table, etc.) from an
  uploaded label PDF/image using Tesseract OCR, with a local Qwen2-VL vision
  model as a fallback for fields Tesseract can't confidently read.
- **Product/version identification** — matches an upload against existing
  products so a new artwork is filed as the next version of the same
  product rather than a duplicate.
- **Comparison** — checks a new label against its previous approved version
  (same-company) and against the closest matching product from other
  companies (cross-company), field by field, flagging each as
  MATCH / SIMILAR / CONFLICT / MISSING.
- **Visual comparison** — a perceptual-hash layout signal and a colour
  histogram signal, computed independently so a same-layout recolour and a
  same-colour layout change are both caught.
- **Approval workflow** — a per-stage sign-off chain
  (Label Final → Technical → QA → Manager), enforced by role.
- **PDF report** — a client-generated report summarizing the comparison,
  the nutrition table, and the visual comparison, with both artwork images
  attached.

## Structure

```
IMH-LVS/
├── src/          React + TypeScript frontend (Vite)
├── backend/      Node.js + Express + TypeScript API (Postgres, auth, OCR pipeline)
├── ai_backend/   Python FastAPI service — Qwen2-VL vision-model fallback for extraction
└── README.md     Session handoff notes for this repo (see below)
```

- **Frontend** (`IMH-LVS/`, run from the repo root) — React 19 + Vite +
  MUI. Currently keeps most of its own state in `localStorage`; talks to
  the backend for label extraction.
- **Backend** (`IMH-LVS/backend/`) — Node/Express/TypeScript API backed by
  Postgres. Owns auth (sessions, not JWTs), label extraction, product
  identification, comparisons, and the approval workflow. See
  [`IMH-LVS/backend/README.md`](IMH-LVS/backend/README.md) for setup,
  environment variables, and auth details.
- **AI backend** (`IMH-LVS/ai_backend/`) — optional FastAPI service running
  a local Qwen2-VL model, used only when the primary OCR pipeline can't
  read a field confidently. Nothing under this project ever sends label
  artwork to an external/cloud API — extraction is entirely local.

## Getting started

Requires Node.js, Python 3.10+, and a local Postgres instance (Docker is
the easiest way to get one — see `IMH-LVS/backend/.env.example`).

```bash
# Frontend
cd IMH-LVS
npm install
cp .env.example .env
npm run dev

# Backend (separate terminal)
cd IMH-LVS/backend
npm install
cp .env.example .env        # set DATABASE_URL, etc.
npm run db:migrate
npm run db:seed             # set SEED_USER_PASSWORD first if you want to log in
npm run dev

# AI backend — optional, only needed for the vision-model OCR fallback
cd IMH-LVS/ai_backend
pip install -r requirements.txt
uvicorn app.main:app --port 8000
```

Full environment variable reference: `IMH-LVS/.env.example` (frontend) and
`IMH-LVS/backend/.env.example` (backend).

## Notes

- `IMH-LVS/README.md` is a running session-handoff log for continuity
  between work sessions on this project, not end-user documentation — read
  it if you're picking the project back up and want the current state of
  what's done, what's not, and why certain approaches were tried and
  dropped.
- Label artwork used for development/testing is confidential and is never
  sent to a cloud API or hosted notebook — all extraction and model
  inference runs locally.
