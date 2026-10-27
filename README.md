[README.md](https://github.com/user-attachments/files/33999346/README.md)
# Moderna mRNA Vaccine Clinical Safety & Dosage Intelligence Platform

> **Student engineering project - not real clinical guidance.** Every generated report carries this disclaimer.

End-to-end pipeline: **patient data -> ML risk classifier -> mRNA self-attention -> guideline RAG -> grounded, schema-validated report -> dashboard.**

## Quick start
```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt   # pinned, tested versions
python run_pipeline.py                      # Modules 1-5, ~10 s; add --provider anthropic|openai for a hosted LLM
python -m unittest discover -s tests -v     # 28 tests, one or more per functional requirement (FR-1.1 ... FR-5.3)
```
For a hosted LLM: `cp .env.example .env`, fill in a key (never commit `.env`). With no key the **offline rule-based generator** runs.

## Layout
| Path | Purpose |
|---|---|
| `src/data_pipeline.py`, `risk_model.py` | M1: validation, imputation, LR vs RF, model selection |
| `src/attention.py` | M2: from-scratch scaled dot-product (+ multi-head) attention, heatmap |
| `src/rag_index.py`, `rag_eval.py` | M3: chunking, embeddings, persistent index, retrieval + latency log |
| `src/schemas.py`, `guideline_rules.py`, `llm_clients.py`, `report_generator.py` | M4: prompt, LLM call, one-retry validation, guardrails, JSON+MD output |
| `src/dashboard.py`, `eda.py` | M5 + EDA figures |
| `outputs/` | model, metrics, figures (`dashboard.png`), `reports/` (5 JSON+MD pairs + `sample_report`), RAG index/eval |
| `REPORT.md` | Portfolio report: results, written explanations, limitations |

## Reproducing each week's output (run from the repo root)
| Week | Command | Produces |
|---|---|---|
| 1 | `python -m src.risk_model` and `python -m src.eda` | `data/patient_records_clean.csv`, `outputs/risk_model.joblib`, `outputs/model_metrics.json`, confusion matrices + classification reports + EDA figures in `outputs/figures/` |
| 2 | `python -m src.attention` | `outputs/figures/attention_heatmap.png`, `outputs/attention_result.npz` |
| 3 | `python -m src.rag_index` then `python -m src.rag_eval` | `outputs/chroma_store/`, `outputs/rag_retrieval_eval.md` |
| 4 | `python -m src.report_generator` (one sample) / `python run_pipeline.py` (5 samples + dashboard) | `outputs/reports/*.json|md`, `outputs/figures/dashboard.png` |

Modules import each other as `src.*`, so run them with `python -m src.<module>` (not `python src/<module>.py`).
Week 4 needs Weeks 1 and 3 first (it loads the saved model and index); `run_pipeline.py` runs them all in order.

## Data sourcing
`data/patient_records.csv` (500 synthetic patients) and the guideline text (converted from the provided
`Moderna_mRNA_Safety_Guidelines.docx`) were supplied with the course PRD. Nothing is scraped. The data are synthetic,
so all results are for learning only.

## Environment variables (`.env`, copy from `.env.example`)
`LLM_PROVIDER` (`auto`/`anthropic`/`openai`/`offline`), `ANTHROPIC_API_KEY`, `ANTHROPIC_MODEL`, `OPENAI_API_KEY`, `OPENAI_MODEL`. No key -> offline generator.

## Where this deviates from the PRD (and why)
Built and verified in an **offline sandbox** (no internet, no API key), so:
1. **Embeddings/vector store:** the code uses ChromaDB + `all-MiniLM-L6-v2` automatically when installed; otherwise a persisted **TF-IDF** index. *This run used TF-IDF*; the Chroma path is written but **not executed**.
2. **LLM:** *this run used the deterministic offline generator, not an LLM.* Anthropic/OpenAI clients are written (stdlib HTTP) but **not executed**. Guardrails run on any provider's output.
3. **Pydantic:** real Pydantic is used if installed; this run used an equivalent strict fallback (same field names / `ValidationError`).
4. **LangChain / PyTorch not used** (attention is NumPy, which the PRD allows; chunking and prompting are plain Python).
5. **Attention:** weight scale 0.5 (template 0.1 gives near-uniform weights, 0.063-0.070 vs uniform 0.067) and sinusoidal positional encoding added. FR-2.1 lists length 5-50 but its example `encode_sequence('ACGU')` has length 4, so length is enforced in `analyze()` (the pipeline entry point), characters always.
6. **Chunking** is token-based (regex token proxy), target ~350 tokens, 50 overlap; the PRD's template counts characters, which contradicts FR-3.1.
7. **Units:** the schema field is `Recommended_Dose_mg`; data is in mcg, so values are converted (1 mg = 1000 mcg).
8. **Pandas trap:** `"None"` in `Prior_Reaction_History` is a real category; pandas>=2 reads it as NaN. The loader disables default NA parsing (otherwise 291 rows are corrupted).
