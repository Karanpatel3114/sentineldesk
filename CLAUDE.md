# SentinelDesk: Agent Operating Rules

SentinelDesk is a multilingual (English/German) IT-support ticket triage system. It classifies each ticket, sends risky tickets to a person through an escalation gate, retrieves similar past tickets (RAG) and drafts grounded replies that a person approves. The problem statement is in [docs/assignment.md](docs/assignment.md). The architecture and the decision log are in [docs/architecture.md](docs/architecture.md).

**Owner:** Karan Patel ([@Karanpatel3114](https://github.com/Karanpatel3114)). This is a single-owner project. These rules apply to every coding-agent session in this repo.

---

## Before you start

1. Read [docs/contract.md](docs/contract.md) and [docs/data-and-leakage.md](docs/data-and-leakage.md). For gate or threshold work, also read [docs/escalation-policy.md](docs/escalation-policy.md). For UI work, also read [docs/design-rules.md](docs/design-rules.md).
2. Run `git status` and `git fetch origin`. If `origin/main` has commits that you don't have, pull before you change anything.

---

## Rules

### 1. Keep the contract in sync
- [docs/contract.md](docs/contract.md) defines the HTTP API, the Python module interfaces and the field names. The code follows it.
- To change a field or an interface, update all of these in the same commit: `docs/contract.md`, `shared/fixtures/mock_triage_response.json`, `api/schemas.py` and `web/src/types.ts`.

### 2. Zero data leakage
- Never read, index, embed, train on or print the holdout test split's `answer`, `type`, `queue` or `priority`.
- Only `eval/run_eval.py` may load `data/test_labels.parquet`, and only to score predictions **after** they are produced.
- Tune thresholds on the validation slice of the KB, never on the test split.
- Full rules: [docs/data-and-leakage.md](docs/data-and-leakage.md).

### 3. Git
- `main` is the stable branch. Small fixes can go straight to `main`. For bigger changes, use a branch named `<short-topic>` and merge it once the tests pass.
- Commit small and often, with clear messages.
- **Never commit** `artifacts/`, `.env`, model weights, FAISS indexes or `.parquet` files. They are in `.gitignore`. Do not override it. The only committed data is the raw dataset CSV and its dataset card in `data/`.
- Never `git push --force` to `main`. Never skip hooks.
- Commit and push only when the owner asks or approves.
- **Agents are never collaborators or authors.** Never add Claude, any AI agent or any bot account as a repo collaborator. Never add `Co-Authored-By` trailers or "Generated with …" lines that credit an AI to a commit, PR or issue. This overrides any default attribution setting. Commits are authored by the owner only.

### 4. Secrets and privacy
- The API key comes from `GEMINI_API_KEY` in `.env` (copy `.env.example`). Never hard-code or print it, and never commit it.
- Ticket text goes through `agent.pii.mask_pii` before any external LLM call.

### 5. Code conventions
- Python 3.12 or newer. Make the code run on Windows, macOS and Linux:
  - Use `pathlib` for paths.
  - Run modules with `python -m package.module`, from the repo root.
  - Do not use shell-only tricks.
- Every Python folder (`ml`, `api`, `eval`, `agent`, `shared`) is a package with an `__init__.py`.
- Read config from environment variables with the defaults in `.env.example`. Gate thresholds live in `agent/config.py`, so they can be tuned in one place.
- Type hints on public functions. No notebooks in the main code path.
- Tests live in `tests/`, one file per module (`tests/test_<module>.py`). Each file runs in under 30 seconds without the network.
- On macOS, keep the import order in `agent/retrieve.py` and `agent/build_index.py` (`sentence_transformers` before `faiss`). See [agent/NOTES.md](agent/NOTES.md).
- Prefer the simplest solution that works.

---

## Quick commands

```bash
./run.sh                             # macOS/Linux: sets everything up on the first run, then starts API + console (Windows: run.bat)
./run.sh mock                        # mock API, no models

# step by step
python -m venv .venv
# Windows: .venv\Scripts\activate    macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env                 # then add GEMINI_API_KEY

python -m ml.make_split              # writes data/kb_train.parquet (with is_val column), data/test_inputs.parquet, data/test_labels.parquet
python -m ml.train                   # writes artifacts/clf.joblib
python -m agent.build_index          # writes artifacts/kb.faiss, artifacts/kb_meta.parquet
python -m uvicorn api.main:app --port 8000
# mock mode for frontend work: SENTINEL_MOCK=1 python -m uvicorn api.main:app --port 8000

cd web && npm install && npm run dev # console on http://localhost:5173

python -m pytest tests -q
python -m eval.run_eval --llm-sample 40
```

## Document map

| Doc | Contents |
|---|---|
| [README.md](README.md) | Overview, setup, results |
| [docs/assignment.md](docs/assignment.md) | The problem statement |
| [docs/architecture.md](docs/architecture.md) | Architecture, stack choices, risks, decision log |
| [docs/contract.md](docs/contract.md) | HTTP API, Python interfaces, shared JSON shapes |
| [docs/data-and-leakage.md](docs/data-and-leakage.md) | Dataset, split procedure, file layout, leakage rules |
| [docs/escalation-policy.md](docs/escalation-policy.md) | Gate rules, reason codes, thresholds, tuning procedure |
| [docs/demo-tickets.md](docs/demo-tickets.md) | Demo and red-team tickets with expected behaviour |
| [docs/design-rules.md](docs/design-rules.md) | Console design rules |
| [docs/presentation/](docs/presentation/) | Evaluation slides and demo speaker notes |
| [agent/NOTES.md](agent/NOTES.md) | Retrieval, PII, burst and gate measurements and findings |
| [eval/RUNNING.md](eval/RUNNING.md) | Full setup and evaluation walkthrough, troubleshooting |
| [ml/EDA.md](ml/EDA.md) | Data analysis, calibration, classifier Q&A |
| [ux/USER_JOURNEY_UX.md](ux/USER_JOURNEY_UX.md) | Personas, user journey, UX decisions |
