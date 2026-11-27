# Security and Compliance Audit

**Date:** 2026-11-27
**Scope:** the full repository and its git history (commit `c52a940`), the built web console, the running API on `localhost`, all Python and npm dependencies, licenses, and every external service the project uses.

## Summary

| Area | Result |
|---|---|
| 1. Secrets in code and git history | ✅ Pass. No secrets found. No key needs rotating. |
| 2. Keys reaching the browser | ✅ Pass. The built JavaScript contains no keys. Gemini is called only from the backend. |
| 3. API attack surface | ✅ Pass for local use. Hardening is listed below in case the app is ever hosted. |
| 4. Dependencies and CI | ✅ Pass. 0 known vulnerabilities. No CI workflows exist. |
| 5. Licenses | ✅ Fixed in `7f0b5ee`. Added an MIT `LICENSE` and full dataset credit with a non-commercial note. |
| 6. Confidential or personal data | ✅ Pass. No employer, client or real personal data. |
| 7. Wording and disclaimers | ✅ Fixed in `7f0b5ee`. All metrics match. Narrowed one claim and added a trademark disclaimer. |
| 8. External services and costs | ✅ Pass. No hosting, database or paid service is used. Gemini is free unless billing is linked. |
| 9. Future-proofing | ✅ Pass. The model is current, failures fall back to human review, and a health endpoint exists. |

---

## 1. Secrets, now and in the past

**Checked**
- `gitleaks git` over the full history (1 commit, 27 MB): **no leaks found**.
- `trufflehog git` over the full history: 2 unverified `Smartsheets` matches in `data/aa_dataset-tickets-multi-lang-5-2-50-version.csv` (commit `c52a940`, lines 6307 and 7616). Both are ordinary words in synthetic ticket text that mention the Smartsheet product. **False positives.**
- Every file ever committed: no `.env`, `kaggle.json`, `*.pem`, `*.key` or credential file. Only `.env.example` and `web/.env.example` were committed.
- `.env.example` and `web/.env.example` contain only empty values, defaults and `localhost` URLs.
- `.gitignore` covers `.env`, `.env.*` (except `.env.example`), `web/.env.local`, `artifacts/`, `*.parquet`, `*.faiss`, `*.joblib` and `.claude/settings.local.json`.

**Keys to rotate:** none. No real key has ever been in the repository, and the local `.env` has an empty `GEMINI_API_KEY`.

**Fixed (`7f0b5ee`):** `.gitignore` now also covers `kaggle.json`, `credentials*.json`, `*.pem` and `*.key`.

## 2. Keys can't leak from the browser

**Checked**
- `npm run build` was run, and `web/dist` was searched for `AIza`, `sk-…`, `api_key`, `apiKey`, `secret`, `Bearer`, `GEMINI` and `token`: **0 matches**.
- The frontend reads only `VITE_API_URL`, `VITE_AGENT_NAME` and `VITE_AGENT_ROLE`. None of them is sensitive.
- The only Gemini client is `agent/draft.py` (backend). The web code calls only the SentinelDesk API (`web/src/api.ts`).
- Ticket text passes through `agent.pii.mask_pii` before the Gemini call.

## 3. Attackers and abuse

**Checked against the running API**
- **Network exposure:** the API listens on `127.0.0.1:8000` and the console on `[::1]:5173`. Neither is reachable from other machines.
- **CORS:** a preflight from `https://evil.example` is rejected with `400` and no `Access-Control-Allow-Origin` header. Only `localhost` origins are allowed.
- **Debug mode:** off. `run.sh` has no `--reload`, and FastAPI `debug` is the default `False`. No stack traces appear in responses.
- **Admin, debug or MCP endpoints:** none exist (`/admin`, `/debug` and `/mcp` return `404`). The endpoints are `/health`, `/config`, `/metrics`, `/triage` and `/burst/reset`.
- **Prompt injection:** three layers. Regex patterns in `agent/config.py` block drafting (`PROMPT_INJECTION_SUSPECTED`). Ticket and precedent text is HTML-escaped and wrapped in tags in `agent/prompts.py`. The system prompt tells the model to treat those tags as data. The red-team and gate tests pass (37 tests in `tests/test_redteam.py` and `tests/test_gate.py`), and the full suite passes (206 tests).
- **SQL guard:** not applicable. The project has no database and runs no SQL.
- **File uploads:** not applicable. There is no upload endpoint or file input.
- **Data retention:** tickets are kept only in memory (the burst window) and in the browser session. Nothing is written to disk.

**Issues found.** These matter only if the API is ever hosted publicly. None has been applied.

| Severity | Issue | Suggested fix |
|---|---|---|
| Low (Medium if hosted) | No rate limit on `/triage`. Each call can trigger one Gemini request. | Add `slowapi` with a limit such as 10/minute per IP on `/triage`. |
| Low (Medium if hosted) | No request-size limit. A 2 MB body was accepted (it is truncated to 8,000 characters after parsing). | Add `max_length=8000` to `subject` and a `max_length` (for example 100,000) to `body` in `api/schemas.py`, and cap the body size at the proxy. |
| Low | Error responses return the exception text (`detail=str(exc)`), which can reveal internal file paths. | Return a generic message such as "Internal error" and keep the details in the server log. |
| Low (if hosted) | `/burst/reset` has no authentication. | Protect it with a shared token, or remove it from a hosted build. |
| Low (if hosted) | `/docs`, `/redoc` and `/openapi.json` are public. | Pass `docs_url=None, redoc_url=None` when hosted. |
| Low (if hosted) | The Vite dev server sends no security headers. | Serve the built `dist/` with `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `Referrer-Policy` and `frame-ancestors 'none'`. |
| Nit | CORS still allows unused `:8501` origins and `allow_credentials=True`, which no request needs. | Remove both. |
| Nit | `run.bat` starts uvicorn with `--reload`. | Drop `--reload` for parity with `run.sh`. |

## 4. Dependencies and GitHub Actions

**Checked**
- `pip-audit` on the 77 installed Python packages (exact versions from the project `.venv`): **no known vulnerabilities**.
- `npm audit` on all web packages, including dev dependencies: **0 vulnerabilities**.
- **GitHub Actions:** the repository has no `.github/workflows`, so nothing can print secrets in CI logs and no action versions need pinning.

**GitHub settings to enable** (Settings → Code security, and Settings → Branches):
1. Dependabot alerts: **On**.
2. Dependabot security updates: **On**.
3. Secret scanning and push protection: **On** (free for public repositories).
4. Branch protection for `main`: block force pushes and deletions. Currently, `main` is **not protected** (checked through the public GitHub API).

The audit could not read the Dependabot and secret-scanning settings because they require the owner's login. Check them by hand.

## 5. Licenses

**Project license:** at audit time there was no `LICENSE` file. **Fixed (`7f0b5ee`):** added an MIT `LICENSE` with `Copyright (c) 2026 Karan Patel`. The README states that the dataset in `data/` is excluded and stays under CC BY-NC 4.0. Also added a License section to the README and `"license": "MIT"` to `web/package.json`.

**Python dependencies** (`pip-licenses`, 77 packages): MIT, BSD, Apache-2.0, ISC, PSF and MPL-2.0. **No GPL, AGPL or LGPL.** The MPL-2.0 packages (`certifi`, part of `tqdm`) are file-level copyleft and are fine to use unmodified. One package reports its license as just "License". It is a permissive package whose metadata is incomplete.

**npm dependencies** (`license-checker`): MIT, ISC, Apache-2.0, BSD-3-Clause, 0BSD, BlueOak-1.0.0, OFL-1.1 (IBM Plex fonts) and MPL-2.0 (`lightningcss`, build-time only). **No GPL or AGPL.** The one `UNLICENSED` entry was this project's own `web/package.json`, which is now fixed.

**Datasets and models**

| Asset | License | Credited in README? |
|---|---|---|
| Tobi-Bueck/customer-support-tickets (Hugging Face) | CC BY-NC 4.0 | ✅ Fixed (`7f0b5ee`): author, license link, the note about the train/test split and "non-commercial only". |
| `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` | Apache-2.0 | ✅ Fixed (`7f0b5ee`): link and license added. |
| `lingua-language-detector` | Apache-2.0 | ✅ Covered as a dependency. |
| Google Gemini API (`gemini-3.5-flash-lite`) | Gemini API Terms of Service | ✅ Named. |

**Gemini outputs are never used as training data.** `ml/train.py` and `ml/calibrate.py` read only `data/kb_train.parquet`. `agent/build_index.py` reads only `kb_train.parquet` and the test ticket IDs, which it excludes from the index. LLM drafts are written only to `eval/results/llm_samples.jsonl`, which is git-ignored and which no training code reads.

## 6. Confidential or personal data

**Checked:** all tracked files, the full git history, the dataset CSV (as counts only), docs and image metadata.
- Former employer, client and university names: **0 matches**. Three raw substring hits are random characters inside base64 font data (`docs/presentation/slides.html`) and an npm checksum (`web/package-lock.json`).
- No local paths, usernames, or personal email addresses or phone numbers.
- Every demo and test email uses a placeholder domain. The phone and card numbers in `agent/pii.py` and `tests/test_pii.py` are standard test values (`555-…`, `4111 1111 1111 1111`).
- The screenshots in `web/screenshots/` carry no author, GPS or path metadata.
- The README does not mention any previous employer.

**Nit:** a few fake addresses use real country domains (`max.mustermann@firma.de`, `@example.de`, `@example.it`, `@example.fr`, `@example.co.uk`). Changing them to the reserved `.example` domain removes any chance of hitting a real inbox.

## 7. Wording and disclaimers

**Metrics.** Every figure in the README, the slides, the speaker notes and the LinkedIn post matches `eval/results/metrics.md` and `metrics.json`. That includes 0.907 escalation recall, 0.692 classifier recall, 0.878, 0.594 and 0.630 macro-F1, a 0.469 escalation rate, 40 ms and 60 ms latency without the LLM, 1,116 ms and 14,697 ms with it, 39 of 39 citations valid, 88.3% language agreement, and 22,869 indexed tickets.

**Wording fixes (all three applied in `7f0b5ee`)**
1. README "Grounded drafts" row. Current: *"Every AI reply cites past resolved tickets."* Holding replies (`INSUFFICIENT_EVIDENCE`, 2 of 40) have no citations. Change it to: *"Every supported AI draft cites past resolved tickets. When the evidence is thin, it writes a short holding reply with no claims."*
2. Add a disclaimer at the end of the README: *"SentinelDesk is an independent project. It is not affiliated with or endorsed by Google. Product names in sample tickets (such as SAP S/4HANA and Amazon Alexa) are trademarks of their owners."*
3. Dataset section: *"[Customer Support Tickets](https://huggingface.co/datasets/Tobi-Bueck/customer-support-tickets) by Tobias Bueck, licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/). The data was split into a knowledge base and a holdout test set, and is used here for non-commercial research and demonstration only. The MIT license does not apply to the data."*

## 8. External services and what happens at the free limit

| Service | Used for | Setting | At the free limit |
|---|---|---|---|
| Google Gemini API | Drafting replies | `GEMINI_API_KEY`, `SENTINEL_LLM_MODEL` | **No charge** while the Google Cloud project behind the key has no billing account. Calls return HTTP 429, and the app routes the ticket to human review (`LLM_UNAVAILABLE`). If billing is ever linked, all usage is charged. |
| Hugging Face Hub | Downloading the dataset (only if the committed CSV is missing) and the embedding model (first run, then cached) | `SENTINEL_EMBED_MODEL` | Free with no account. No charge is possible. |
| Google Fonts | The Inter font in `web/index.html` | none | Free with no account. Without internet, the system font is used. |
| shields.io, github.com | README badges and avatar | none | Free, display only. |

Not used: hosting (Render, Vercel, Hugging Face Spaces), Postgres, Redis, tracing, cloud storage, Docker, cron jobs and keep-alive pings. No `Dockerfile`, `render.yaml` or `vercel.json` exists, so nothing expires after 30 days and nothing burns free hours.

**Privacy note:** on Gemini's free tier, Google may use prompts and responses to improve its products. SentinelDesk sends only synthetic dataset tickets, and only after PII masking.

## 9. Future-proofing

- **Gemini model:** `gemini-3.5-flash-lite` (set in `agent/config.py`, overridable with `SENTINEL_LLM_MODEL`). It is generally available with no shutdown date announced, and Google's Vertex AI docs list retirement as July 2027 or later. If the model is retired or the quota runs out, the call fails, `agent/draft.py` retries 429 and 5xx errors (2 s, 4 s, 8 s), and the ticket falls back to human review with triage and precedents intact. To switch models, change one environment variable. The fallback keeps the same model, and nothing switches to a paid model automatically.
- **Rebuilding data:** `run.sh` and `run.bat` rebuild the split, the classifier and the FAISS index whenever they are missing, using the committed CSV (or Hugging Face if that is missing too).
- **Health endpoint:** `GET /health` reports whether the classifier and index are loaded and whether the LLM is configured.
- **Uptime monitor:** not needed while the app runs only on `localhost`. If it is hosted later, use UptimeRobot's free plan with an HTTP check on `/health` every 5 minutes. Don't ping a free host that sleeps when idle, because the pings keep it awake and use up its free hours.

## 10. Fixes applied

| Commit | Change |
|---|---|
| `7f0b5ee`, plus a follow-up | Added an MIT `LICENSE` (the dataset note lives in the README, so GitHub detects plain MIT), full dataset and model credits, a README License section, a not-affiliated disclaimer, the narrower "grounded drafts" wording, extra `.gitignore` guards, and the `license` field in `web/package.json`. |

Not applied, by choice: the hardening items in section 3. They matter only if the API is hosted publicly, and the app currently runs on `localhost` only. The fake-email domain nit in section 6 was also left as is.

## Manual checks for the owner

1. GitHub → Settings → Code security: turn on Dependabot alerts, Dependabot security updates, secret scanning and push protection.
2. GitHub → Settings → Branches: add a rule for `main` that blocks force pushes and deletions.
3. Create the Gemini key in Google AI Studio in a project with **no billing account**. Confirm this at console.cloud.google.com/billing. Never paste the key anywhere except `.env`.
4. If billing is ever enabled, set a budget alert of $1 first.
