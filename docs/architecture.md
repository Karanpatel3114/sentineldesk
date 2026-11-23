# SentinelDesk: Architecture and Design Decisions

**Goal:** a live console where anyone can paste an unseen ticket and see the triage, the escalation decision, cited precedents and a grounded draft, all backed by honest holdout metrics.

## 1. Requirements and where they live

| Requirement (from the [brief](assignment.md)) | Where |
|---|---|
| Multi-task triage (`type`, `queue`, `priority` + confidence) | `ml/` |
| Holdout Macro-F1 + High-Priority Incident Recall | `eval/` |
| `/triage` HTTP service + latency timing | `api/` |
| Escalation and safety gate with explicit rules | `agent/gate.py`, `agent/config.py` |
| Top-k retrieval of past tickets from the KB split only | `agent/retrieve.py`, `agent/build_index.py` |
| Grounded draft in en/de that cites ticket IDs and refuses to fabricate | `agent/draft.py`, `agent/prompts.py`, `agent/citations.py` |
| Live console: predictions, confidence, latency, badge, precedents, draft | `web/` |
| README, `requirements.txt`, eval script, demo | root, `eval/`, `docs/presentation/` |
| Bonus: cross-lingual retrieval | `agent/retrieve.py` (multilingual embeddings) |
| Bonus: burst / major-incident banner | `agent/burst.py` (logic), `web/` (banner) |
| Bonus: PII masking before the LLM call | `agent/pii.py` |

## 2. Architecture

```
            ┌──────────────── web/ ─────────────────────────┐
            │ Vite + React console  ── POST /triage ──┐     │
            └─────────────────────────────────────────┼─────┘
                                                      ▼
┌──────────────────────────── api/ ────────────────────────────────────────┐
│ 1 build_text(subject, body)              shared/text.py                  │
│ 2 mask_pii(text)                          agent/pii.py                   │
│ 3 detect_language(masked)                 agent/lang.py                  │
│ 4 TriageClassifier.predict(masked)        ml/predict.py                  │
│ 5 AgentPipeline.run(masked, lang, preds)  agent/pipeline.py              │
│     ├─ Retriever.embed + search (top-5)   agent/retrieve.py              │
│     ├─ BurstDetector.observe              agent/burst.py                 │
│     ├─ pre_draft_gate                     agent/gate.py                  │
│     ├─ generate_draft → Gemini API        agent/draft.py                 │
│     ├─ validate_citations                 agent/citations.py             │
│     └─ finalize_escalation                agent/gate.py                  │
│ 6 assemble response + latency_ms          api/service.py                 │
└──────────────────────────────────────────────────────────────────────────┘
   artifacts/clf.joblib         artifacts/kb.faiss + kb_meta.parquet
   eval/run_eval.py → eval/results/metrics.{json,md} → GET /metrics → console KPI strip
```

Every request takes the same path in the same order: build the text, mask PII, detect the language, classify, then run the agent pipeline. `eval/run_eval.py` calls `api.service.triage()` in-process, so the evaluation measures exactly what the demo runs. If a component or its artifact is missing, `api/stubs.py` stands in with contract-shaped data, and `/health` reports which parts are real.

## 3. Stack decisions

| Concern | Decision | Why |
|---|---|---|
| Language | Python 3.12+ backend, TypeScript frontend | Fast to build, strong ML ecosystem |
| Classifier | TF-IDF (word 1–2 grams + char 3–5 grams) → 3 logistic regression heads | Trains in seconds, strong baseline, good probabilities |
| Embeddings | `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (384-dim, normalised) | Fast on CPU, en↔de cross-lingual retrieval included |
| Vector store | FAISS `IndexFlatIP` (exact cosine) | About 23k vectors: exact search takes milliseconds, no tuning |
| Language detection | `lingua-language-detector` | Accurate on short text |
| LLM | Google Gemini via the `google-genai` Python SDK. Default `gemini-3.5-flash-lite`, set by `SENTINEL_LLM_MODEL` | Free tier, fast, structured JSON output (`response_schema`). If grounding or German quality is weak, switch the env var to `gemini-3.8-flash` |
| API | FastAPI + Uvicorn on port 8000 | Typed, quick |
| Frontend | Vite + React + TypeScript + Tailwind on port 5173 | A richer operator console than Streamlit allows |

## 4. Risks and fallbacks

| Risk | Fallback |
|---|---|
| LLM latency > 10 s or the LLM API is down | Already on the fastest free model (`gemini-3.5-flash-lite`). Draft calls back off and retry (2 s, 4 s, 8 s). If the API is down, the gate returns `LLM_UNAVAILABLE` and the console still shows triage + precedents. |
| The backend is unavailable during a demo | The console's **Demo data** switch loads built-in sample tickets and a local heuristic, with a banner saying so. |
| Index build too slow on a laptop | Build it once and reuse `artifacts/` (never commit it). `run.sh` only rebuilds missing artifacts. |
| Classifier priority F1 is low (noisy labels) | Expected. Report it honestly. Recall on high-priority incidents comes from the gate's probability thresholds, not from argmax. |
| macOS segfault when torch and faiss both load OpenMP | Import `sentence_transformers` before `faiss`, and keep faiss single-threaded ([agent/NOTES.md](../agent/NOTES.md)). |

## 5. Decision log

Add one line per decision: the decision, then why.

- **Frontend stack:** Vite 8 + React 19 + TypeScript + Tailwind v4 (`@tailwindcss/vite`), lucide-react icons, oxlint. Not Streamlit.
- **Classifier `C=10` for all heads:** best val macro-F1 and lowest ECE on every head. Sigmoid calibration lowered macro-F1 on every head, so it was rejected ([ml/EDA.md](../ml/EDA.md)).
- **`BURST_SIM` 0.75 → 0.65, plus a shared rare term:** 0% false banners with 10 random tickets in the window, and demo-12 (the third burst ticket) raises the banner. Cosine alone at 0.65 gave 21% false banners ([agent/NOTES.md](../agent/NOTES.md)).
- **Gate thresholds tuned on KB val** ([eval/results/threshold_sweep.md](../eval/results/threshold_sweep.md)): `T_HIGH_PRIORITY_P` 0.35→0.20, `T_INCIDENT_P` 0.40→0.30, `T_CONF_TYPE`/`T_CONF_PRIORITY` 0.45→0.50; `T_CONF_QUEUE` 0.25 and `T_SIM` 0.55 unchanged. Val: HPI escalation recall 0.799→0.901, escalation rate 0.338→0.478, `NO_PRECEDENT` 0.000. The sweep's top row (`T_SIM` 0.851 = val p10) was not taken: it adds only ~2 points of recall, through ~10% `NO_PRECEDENT` escalations on tickets that do have close precedents.
- **`T_SIM` 0.55 → 0.50:** on hand-written in-domain tickets, wrongly blocked tickets fell from 25% to 12.5%, and 93% of off-topic tickets are still caught. demo-08 stays far below at 0.24. An off-topic ticket above the threshold escalated through `LLM_INSUFFICIENT_EVIDENCE` in the live check.
- **faiss single-threaded on every request thread (macOS):** `omp_set_num_threads` only applies to the calling thread, and the API serves each request on a worker thread. `Retriever.search` now sets it on every call, so concurrent requests no longer segfault.
