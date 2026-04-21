# SAR Draft Automation
### Agentic Suspicious Activity Report Generator
**ajatau AML · Second Line of Defence (2LoD)**

---

## What This Is

A 6-step web application that assists AML compliance officers in drafting, reviewing, and exporting Suspicious Activity Reports (SAR/STR). It combines a structured data-entry workflow with AI-generated narrative drafting, a mandatory human validation checkpoint, and a real PDF export.

**This tool does not file reports. It drafts them.**
Every output requires a human compliance officer to review, edit, and approve before any regulatory action is taken. The AI generates a starting point — the officer owns the final document.

---

## The Human-in-the-Loop Principle

This is the most important design decision in the system.

```
TM Alert Triggered
       │
       ▼
[Step 1–3]  Officer enters subject info + transactions manually
       │      ← Human provides all factual inputs
       ▼
[Step 4]    Claude generates a draft narrative
       │      ← AI generates, based on officer's inputs
       ▼
[Step 5]    Officer reviews, edits, and validates     ◄── CRITICAL GATE
       │      ← Human reads every sentence
       │      ← Human confirms 4 regulatory checkboxes
       │      ← Export is BLOCKED until all 4 are ticked
       ▼
[Step 6]    PDF downloaded for internal sign-off / FIU submission
              ← Human MLRO reviews and countersigns
```

The 2LoD checklist in Step 5 is not decorative. Until all four items are checked by a human officer, the export button remains disabled. This enforces the regulatory requirement that SAR filings are human-authorised documents, not automated outputs.

---

## File Structure

```
SAR-Draft-Automation/
│
├── index.html          # Single-page React app (all UI, logic, PDF export)
│                       # No build step — loads React and jsPDF from CDN
│
├── api/
│   └── generate.js     # Vercel serverless function
│                       # Proxies browser → Anthropic API
│                       # Reads ANTHROPIC_API_KEY server-side (never exposed to browser)
│
└── vercel.json         # Routing config
                        # /api/* → serverless functions
                        # /*     → index.html (SPA fallback)
```

---

## Why a Proxy Function?

Browsers cannot call `api.anthropic.com` directly because:
1. **CORS** — Anthropic does not include browser CORS headers
2. **Key exposure** — API keys in frontend JS are publicly readable

`api/generate.js` solves both:

```
Browser (index.html)
    │
    │  POST /api/generate
    │  { model, max_tokens, messages }
    │  ← same origin, always allowed
    ▼
api/generate.js  (Vercel serverless, Node.js)
    │
    │  reads process.env.ANTHROPIC_API_KEY  ← server-side only, never in browser
    │
    │  POST https://api.anthropic.com/v1/messages
    │  { ...body, headers: { x-api-key: KEY } }
    ▼
Anthropic API
    │
    │  { content: [{ type: "text", text: "..." }] }
    ▼
api/generate.js streams response back to browser
    │
    ▼
index.html receives narrative text, sets React state
```

---

## The 6-Step Workflow

### Step 1 — Triage
Officer selects which TM alert to work on. Two sample alerts are pre-loaded:
- `CUST-NORDIC-772` — Crypto mixer exposure, Swedish adverse media
- `BP-9921-LT` — Shell company / crypto-to-fiat, Nordic adverse media

The alert panel shows pre-populated risk flags (Mixer, Adverse Media, High Risk) and the OSINT summary from the monitoring system.

**Test data button:** Fills all subject fields and the full transaction table with a realistic dummy case. Use this for demos and training. The dummy data is different for each alert — it includes wallets, IBANs, counterparty names, and transaction notes consistent with the alert's typology.

### Step 2 — Subject Information
Officer manually enters:
- Full name, date of birth, address
- Phone / email
- ID or passport number
- Officer's own email (appears in the PDF filing details)

This data is passed directly into the Claude prompt and into the PDF. It is never stored anywhere.

### Step 3 — Transactions
Officer enters the suspicious and related transactions. Each row has:
- Date
- Amount
- Currency (EUR / USD / GBP / BTC / ETH / USDT)
- Sender (wallet address, name, or IBAN)
- Receiver (wallet address, name, or IBAN)
- Payment method (Crypto Transfer / SEPA / SWIFT / Card / Cash / Internal)
- Details / notes

Rows can be added and removed freely. The transaction data is:
1. Passed into the Claude prompt so the narrative references real amounts, dates, and counterparties
2. Written into Section 4 of the exported PDF as a formatted table

### Step 4 — AI Narrative Generation

The prompt sent to Claude is constructed from the officer's inputs:

```javascript
const prompt = `You are a Senior AML Compliance Officer writing a formal SAR
Statement of Facts for FIU submission.

Alert: ${alert.id} | ${alert.txType} | Risk: ${alert.risk}
Subject: ${sub.name}, DOB: ${sub.dob}, Address: ${sub.address}
Mixer: ${alert.mixer} | Adverse Media: ${alert.adverseMedia} |
SoW Verified: ${alert.sow} | OFAC: ${alert.ofac}
OSINT: ${alert.osint}

Transactions:
${txStr}   // ← each row the officer entered, formatted as key:value

Write exactly 3 professional paragraphs:
1. STATEMENT OF FACTS: Chronological. Reference actual dates/amounts.
   Explain why mixer flag indicates fund obfuscation.
2. INVESTIGATIVE FINDINGS: Cross-reference OSINT with AMLD6/MiCA.
   Analyse SoW failure. Cite specific regulatory articles.
3. COMPLIANCE DECISION: Recommend or not. Cite OFAC, PEP, AMLD6 Art. 33.
   Be definitive.`
```

The API call is non-streaming (`stream: false`). The response is a standard JSON object:

```javascript
// API response shape
{
  content: [
    { type: "text", text: "On 10 November 2025, the transaction monitoring..." }
  ]
}

// Extraction
const text = data.content
  .filter(b => b.type === 'text')
  .map(b => b.text)
  .join('');
```

### Step 5 — Human Review (the critical gate)

The generated narrative appears in a textarea. The officer must:

1. **Read it** — every sentence
2. **Edit it** — correct any inaccuracies, add context Claude couldn't know
3. **Tick all 4 checkboxes:**

| Checkbox | Regulatory basis | What the officer is confirming |
|----------|-----------------|-------------------------------|
| Proof of Funds / Wealth verified | AML Art. 8(3) | SoF/SoW documentation has been reviewed |
| KYC confirmed | AMLD6 §13 | Identity and residency documents checked |
| MiCA crypto exposure reviewed | MiCA Reg. 2023/1114 | Crypto asset exposure assessed under MiCA |
| OFAC / Sanctions re-confirmed | OFAC 31 CFR | Sanctions screening re-run and clear |

The export button is **programmatically disabled** until all four are checked:

```javascript
const allOk = chk.pof && chk.kyc && chk.mica && chk.ofac;

// Button rendered as disabled if allOk is false
h('button', {
  disabled: !allOk,
  onClick: () => { exportPDF(...); setPhase('exported'); }
}, allOk ? 'Download SAR PDF →' : 'Complete checklist to unlock')
```

This is not a UI suggestion. It is an enforced gate.

### Step 6 — PDF Export

`jsPDF` (loaded from cdnjs CDN, no install needed) generates a multi-page A4 PDF in the browser. The PDF contains 10 sections:

| Section | Contents |
|---------|----------|
| 1 | Case reference, filing date, institution, contact |
| 2 | Subject name, DOB, address, ID, account number |
| 3 | The AI-generated narrative (as edited by the officer) |
| 4 | Transaction table (all rows the officer entered) |
| 5 | Red flags list (auto-generated from alert data) |
| 6 | Typologies (ML stage analysis — layering, integration, etc.) |
| 7 | Additional info — evidence list, related accounts, adverse media |
| 8 | Escalation status — account hold, EDD initiated, FIU referral pending |
| 9 | Recommendation — RECOMMEND SAR FILING (with regulatory basis) |
| 10 | Signature block — officer name + MLRO countersignature line |

The PDF is saved as `SAR_[ALERT_ID]_[DATE].pdf` directly to the officer's Downloads folder.

---

## What Claude Does vs What the Officer Does

| Action | Who does it |
|--------|-------------|
| Select alert | Officer |
| Enter subject name, DOB, address, ID | Officer |
| Enter transaction dates, amounts, counterparties | Officer |
| Generate narrative draft | Claude (AI) |
| Read and verify every sentence of the narrative | **Officer** |
| Correct inaccuracies in the narrative | **Officer** |
| Confirm PoF/W verification | **Officer** |
| Confirm KYC validation | **Officer** |
| Confirm MiCA crypto review | **Officer** |
| Confirm OFAC re-screening | **Officer** |
| Approve export | **Officer** |
| MLRO countersignature | **Senior Officer** |
| Submit to FIU | **Officer / MLRO** |

Claude contributes one thing: a structured first draft of the narrative. Every factual input, every verification checkbox, every signature, and every submission decision is a human action.

---

## Environment Variables

| Variable | Where | Value |
|----------|-------|-------|
| `ANTHROPIC_API_KEY` | Vercel → Project Settings → Environment Variables | `sk-ant-...` |

Never put the API key in `index.html` or any client-side file. It must only exist in Vercel's server-side environment, where `api/generate.js` can read it via `process.env.ANTHROPIC_API_KEY`.

---

## Regulatory Context

The tool is designed against the following frameworks:
- **EU AMLD (4th–6th Directives)** — SAR/STR reporting obligations, EDD triggers
- **EU MiCA (Reg. 2023/1114)** — Crypto-asset service provider obligations, Travel Rule
- **OFAC 31 CFR** — US sanctions screening requirements
- **FATF 40 Recommendations** — Risk-based approach, beneficial ownership
- **Nordic FIU requirements** — Reporting thresholds and FIU submission formats (FI, DK, SE)

The checklist in Step 5 maps directly to specific articles in these frameworks. Each checkbox is a regulatory obligation, not a UX pattern.

---

## Limitations

1. **No real database** — alerts and dummy data are hardcoded. In production, these would pull from a live TM system.
2. **No audit trail** — the app does not log who reviewed what or when. Production use requires an audit log.
3. **No FIU submission** — the PDF is downloaded for manual review and submission. Direct API submission to FIUs is not implemented.
4. **Claude can hallucinate** — the narrative may contain plausible-sounding but incorrect details. The officer's review in Step 5 exists specifically to catch this. Never submit a Claude-generated narrative without reading it.
5. **Two sample alerts only** — real deployment would integrate with actual TM alert feeds.

---

## Local Development

No build step required. Open `index.html` directly in a browser. The API call will fail without the proxy, so for local testing either:
- Run a local Express proxy on `localhost:3000/api/generate`, or
- Use `vercel dev` (requires Vercel CLI) which emulates the serverless function locally

---

*Built by Gintarė Jatautytė — AML & FinCrime Compliance · ajatauaml.com*
*Second Line of Defence · Not a substitute for qualified legal or compliance advice*
