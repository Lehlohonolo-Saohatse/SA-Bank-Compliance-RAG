# SA Bank Compliance RAG

Grounded regulatory Q&A for South African banks. Retrieves from official SARB/Prudential Authority, FSCA, FIC and
Information Regulator sources, and returns a verdict (aligned / conditionally aligned / non-aligned / cannot determine),
gap analysis and remediation steps. **Every claim is tied to a verbatim, machine-verified quote.**

> Decision support only. Not legal advice and **not personal financial advice** (a regulated activity under FAIS).

## Why it is "strict"
| Control | Where | What it does |
|---|---|---|
| Input guard | `sacrag/guard.py` | Declines personal-advice requests, prompt injection, and queries containing SA ID / card numbers (Luhn-validated) |
| Status filter | `sacrag/index.py` | Drafts and bills are excluded unless the user opts in, and are always labelled "not law" |
| Abstention gate | `sacrag/index.py::is_relevant` | Weak retrieval → "cannot determine", no LLM call |
| Constrained prompt | `sacrag/agent.py` | Sources-only, JSON-only, mandatory quote per claim |
| Citation validator | `sacrag/agent.py` | Drops any claim whose `chunk_id`/quote is not found verbatim in the retrieved text; downgrades the verdict if nothing survives |
| Audit trail | `logs/audit.jsonl` | Hashed query, chunk IDs, prompt version, model, corpus date |

## Corpus: official sources
Defined in [`corpus/manifest.yaml`](corpus/manifest.yaml). `documents:` are fetchable; `to_locate:` are confirmed-to-exist
instruments whose exact official URL still needs verifying (promote them once located).

| Layer | Official site |
|---|---|
| Prudential (banks) | https://www.resbank.co.za → Prudential Authority → Deposit-takers (Regulations, Directives, Guidance Notes, Circulars) |
| Joint Standards / Communications | https://www.resbank.co.za and https://www.fsca.co.za |
| AML/CFT | https://www.fic.gov.za (Act, Guidance Notes, PCCs, Directives) |
| Privacy | https://inforegulator.org.za and https://www.gov.za/documents/protection-personal-information-act |
| Statutes | https://www.gov.za, https://www.justice.gov.za, https://lawlibrary.org.za (CC BY) |
| Credit | https://www.ncr.org.za |
| Gazettes | https://www.gpwonline.co.za |
| International | https://www.bis.org/basel_framework/ and https://www.bis.org/bcbs/publ/d424.htm, https://www.fatf-gafi.org |

Use law-firm and news sites (e.g. acts.co.za news, moonstone.co.za) **only as a change radar**, never as corpus.

### Keeping it current
- PA publishes yearly status notices (e.g. *Guidance Note 1 of 2026*, *Circular C1 of 2026*) listing which notes and
  directives remain effective. Re-ingest those first each year and retire anything not listed.
- `refresh.yml` re-fetches weekly, hashes files, rebuilds the index and opens an issue when a source changes.
- When something changes, update `legal_status` in the manifest (`in_force`, `guidance`, `draft`, `bill`, `superseded`).

## Layout
```
corpus/      manifest.yaml, state.json (hashes), retrieval_log.jsonl, raw/ (git-ignored PDFs), manual/
sacrag/      config, manifest, parsing, index, guard, agent, schema
scripts/     fetch_corpus.py, build_index.py
app/         streamlit_app.py
eval/        golden_set.yaml, run_eval.py
tests/       guard, parsing, retrieval + agent (fake LLM), manifest, fetch
```

## Deploy on Streamlit Community Cloud
1. Commit `index/` (chunks + embeddings + meta) so the app does not need raw PDFs. Confirm redistribution terms
   for each source first (official legislation is generally fine; check regulator guidance terms).
2. Create the app from `app/streamlit_app.py`; add `ANTHROPIC_API_KEY` under *Secrets*.
3. Never send customer personal data to the app. It blocks ID and card numbers, but that is a backstop, not a policy.

## Known limitations
- Golden-set references are **provisional** until reviewed by a compliance professional.
- Scanned PDFs need OCR before ingestion; the parser reads text layers only.
- Section detection is regex-based; check `index/chunks.jsonl` for any instrument with odd structure.
- Basel Framework (web-based) is represented by BCBS d424; ingest the consolidated framework separately if needed.
- Not a substitute for Compliance/Legal sign-off. Regulations change; check the "corpus as of" date.
