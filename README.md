# Screening Question Generator

Paste a job description, get 6–10 structured screening questions for a first phone call — each one citing the fragment of the JD that justifies it, with the citation verified against the pasted text.

No framework, no build step, no dependencies. Three files.

---

## Try it without an API key

Open `index.html` in a browser and click **"Ver ejemplo"**. That renders a bundled sample — no key, no account, no server.

Two of the six sample citations are wrong **on purpose**: one paraphrases the JD (`Barcelona` where the JD says `Madrid`), one invents a requirement outright. So the demo shows the check *catching* something rather than displaying six green ticks.

The sample response is fixed, but the verification is not — the badges are computed live against the sample JD by the same code path a real response goes through. Faking them would have hollowed out the one guarantee this project exists to demonstrate.

---

## What it does

1. You paste a job description.
2. The page sends it to a local endpoint, which calls the Anthropic API with a JSON schema.
3. You get back questions, each with the evidence that justifies it.
4. Every citation is checked against your pasted text and marked on screen.

Each question carries eight fields:

| Field | What it holds |
|---|---|
| `question` | The literal text to read to the candidate. |
| `category` | One of five: technical skill, experience depth, domain knowledge, role logistics, collaboration. |
| `jd_evidence` | The quoted fragment of the JD that justifies the question. |
| `probes` | What the question is trying to find out. |
| `strong_answer` | What a good answer looks like. |
| `red_flag` | What should worry you. |
| `time_minutes` | 2, 5, or 10 — for budgeting the call. |
| `id` | Ordering. |

**Card face vs. scoring guide.** The question, its badges, and its citation are always visible — that answers *"what do I ask, and is it grounded?"* The scoring guide (`probes`, `strong_answer`, `red_flag`) collapses into a `<details>` — that answers *"how do I grade the answer?"* Those are two different moments: scanning the list beforehand, and being on the call.

---

## Why citations are verified

`jd_evidence` is the anti-hallucination anchor. If the model can't quote the JD to support a question, the question shouldn't exist. That turns "generate some questions" into something checkable: you can confirm every citation appears in the text you pasted.

The check runs **in the app**, at render time — not as a manual review step afterward. The results header shows the tally ("4 de 6 citas verificadas"), and each card shows its own state.

Three states, not two:

| State | When | Shown as |
|---|---|---|
| Verified | The citation appears literally in the JD after normalization. | Green |
| Partial | A contiguous run of ≥60% of the citation matches, but not the whole thing. | Amber, with the percentage |
| Not found | No meaningful match. | Red, plus a red bar down the card |

**Why normalize instead of comparing raw text.** A plain `includes()` would produce constant false negatives — curly vs. straight quotes, a line break inside the quoted span, a period the model appended, capitalization. A check that flags things that *were* actually there trains you to ignore the indicator, and then the whole feature is worthless. So both sides are lowercased, quotes and dashes are unified, whitespace is collapsed, and edge punctuation is trimmed.

**Ellipses are legitimate.** A citation joining two spans with `...` is split into segments, which must appear **in order**. Out-of-order segments do not pass.

**Why a third state.** "Paraphrased closely" and "invented from nothing" are different failures. The first is often recoverable — the question may still be fine. The second means the question shouldn't exist. Collapsing them into one red "failed" throws away the distinction that actually matters.

---

## Setup for real generation

Requires **Node 20+** (for `fetch` and `--env-file`). No `npm install` — there are no dependencies.

1. Get an API key from [console.anthropic.com](https://console.anthropic.com) → Settings → API keys → Create Key. It is shown once; copy it then. Note that API billing is separate from any Claude subscription — add credit under Billing or the first call will fail.

2. Create your `.env`:

   ```bash
   cp .env.example .env
   ```

   Then edit it to hold your real key. `.env` is gitignored.

3. Start the server:

   ```bash
   node --env-file=.env server.js
   ```

4. Open **http://localhost:3000**.

Opening `index.html` directly still works for the demo, but "Generar preguntas" stays disabled there — see below.

---

## Architecture: why there is a server

The obvious design is a single HTML file. It doesn't work, for two reasons:

- A page opened over `file://` **cannot read a `.env`**. Dotenv files are a server-side process convention, not a browser mechanism.
- Calling the API directly from browser JavaScript ships the key to the client, where it's visible in devtools and the network log.

So the UI stays a single dependency-free `index.html`, and a ~50-line `server.js` serves it and proxies `/api/generate`, reading the key from `process.env`. Node 20+ covers both needs natively.

The page detects at load whether it's being served over HTTP. If not, it disables generation and says why, pointing you at the demo — rather than letting you paste 1,300 characters and fail at the end with a generic network error. The check is "not served over HTTP" rather than "protocol is `file:`", which also covers `data:` and other viewers.

### Files

| File | Contents |
|---|---|
| `index.html` | Entire UI — markup, CSS, JS inline. No framework, no build. |
| `server.js` | Node, no dependencies. Static serving + API proxy + error mapping. |
| `.env.example` | Template. Copy to `.env`. |
| `PLAN.md` | Design document written before any code. Scope, contracts, trade-offs. |

---

## How the API is called

`POST /v1/messages` with `claude-opus-5`, `effort: "medium"`, and **structured outputs** (`output_config.format` with a JSON schema) — not a "please return JSON" instruction in the prompt. The API guarantees the response validates, which removes an entire class of parsing errors.

Two consequences worth knowing:

- Structured outputs don't support numeric or length constraints, so `time_minutes` is an `enum` rather than a range, and the 6–10 question count is requested in the prompt and enforced client-side.
- Instructions live in `system`; the pasted JD goes into `messages` **unmodified**. A JD is untrusted input — if someone pastes one containing "ignore your instructions", the structural boundary helps. The system prompt also states the user block is data to analyze, never instructions.

Cost is roughly $0.06 per generation at typical JD length.

---

## Error handling

Checked before spending tokens: empty input, under 200 characters (warns, allows), over 15,000 characters (blocks).

Mapped from the API to actionable messages: missing key, 401, 400 (the API's own message is passed through with the request ID), 413, 429 (one automatic retry honoring `retry-after`), 500/529 (one retry with backoff), network failure and timeout.

Checked in the response before parsing: `stop_reason: "refusal"` surfaces the explanation, and `stop_reason: "max_tokens"` reports truncation instead of failing on incomplete JSON. No attempt is made to repair partial JSON.

---

## Out of scope

Deliberately not built: persistence or history, editing individual questions, scoring candidate answers, export to PDF/CSV, ATS integration, file upload, accounts, deployment, streaming, model selection, dark mode.

`PLAN.md` records the full list and the reasoning.

---

## Status

The demo path, the citation verification, the presentation hierarchy, and every client-side and server-side error path have been exercised and verified — including a test confirming the badges track the pasted text rather than the response.

The live API call has **not** been run: no personal API key was available at build time. The request shape follows the documented structured-outputs contract, but the first real round trip is unverified.
