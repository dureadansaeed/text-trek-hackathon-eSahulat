# Multilingual E-Sahulat AI

Evidence-grounded question answering for Pakistan's passport services (TextTrek NLP challenge).
Ask in **Urdu script, Roman Urdu, English or a mix**. The system finds the most relevant official passage, **quotes it, and cites it** (passage id, document id, source link).
When the evidence is weak or a safety rule applies, it **abstains or asks for clarification** instead of guessing.

Everything runs locally with open-source models. **No hosted LLM or API is used.**

## Features
- Cross-script search: Urdu, Roman Urdu and English questions reach the same passage
- Five retrieval methods compared: BM25, BM25 + normalization, semantic, semantic + normalization, hybrid
- Extractive, cited answers in the language the user picks (Urdu, Roman Urdu, English)
- Abstention and clarification with a reason code
- Streamlit app: confidence meter, status cards, normalization view, highlighted evidence, per-passage score bars, evaluation tab, embedding map, query history
- Optional microphone input using a local speech model (faster-whisper)

## Repository layout
| File | Purpose |
|---|---|
| `app.py` | Streamlit interface |
| `eSahulatCore.py` | Engine: normalization, BM25, embeddings, hybrid fusion, guards, answer building |
| `buildData.py` | Turns `data/hackathonDataset.txt` into `passages.json`, `queries.json`, `devNegatives.json` |
| `evaluate.py` | Tunes on dev only, then evaluates dev and test; writes `results/` |
| `evalData.py` | Builds the data behind the Evaluation tab |
| `makeValidationPack.py` | Builds `validation_pack.zip` (per-question results, comparison, failures, settings) |
| `charts.py` | Charts drawn as SVG/HTML, no plotting library |
| `voice.py` | Local speech-to-text for the microphone button |
| `running_nb.ipynb` | Colab notebook that runs everything in order |
| `data/` | Organisers' dataset and our derived JSON files |

## Quick start

### Option A: Google Colab (recommended)
Open `running_nb.ipynb` and run the cells in order: install, get code, data and evaluation, validation pack, launch the app (prints a public link).

### Option B: your own machine
```bash
pip install -r requirements.txt
python buildData.py          # dataset file -> data/*.json
python evaluate.py           # downloads intfloat/multilingual-e5-small once, tunes on dev, writes results/
python makeValidationPack.py # writes validation_pack.zip
streamlit run app.py         # opens the app
```
Offline fallback for debugging only (weaker semantic part): `python evaluate.py --embedder hashing` and `ESAHULAT_EMBEDDER=hashing streamlit run app.py`.

## How it works
```
question -> normalize (Unicode, Urdu letters, Roman Urdu skeleton, concept thesaurus)
         -> safety guards (prompt injection, personal status, appointments, outdated fees, unsafe, ambiguity)
         -> retrieve: BM25 + e5 semantic search -> hybrid fusion
         -> confidence check (keyword coverage + semantic score, thresholds tuned on dev only)
         -> answer = sentences quoted from the top passage + passage id + document id + source URL
            or abstain / clarify in the selected language
```
Main ideas (all in `eSahulatCore.py`):
1. **Cross-script concept thesaurus**: the same idea in English, Roman Urdu and Urdu script maps to one token, so Urdu questions match English evidence.
2. **Phonetic skeleton** for Roman Urdu spelling variation (`pasport`, `paasport`, `passport` all match).
3. **Hybrid retrieval**: score fusion with the keyword weight tuned on dev data (0.6).
4. **Abstention**: rule guards plus two thresholds (keyword coverage 0.25, semantic similarity 0.8162), tuned on dev data only.

## Results
Embedding model `multilingual-e5-small`. Test = the organisers' 60 queries (39 answerable, 21 unanswerable). Dev = 90 answerable `train_dev` queries plus 26 team-written negatives. Test data was never used for tuning.

**Retrieval comparison (test, 39 answerable questions)**

| Method | Recall@1 % | Recall@5 % | MRR@5 | Dev MRR@5 |
|---|---|---|---|---|
| BM25 (plain) | 69.2 | 82.1 | 0.738 | 0.698 |
| BM25 + normalization | 79.5 | 97.4 | 0.860 | 0.908 |
| Semantic (plain) | 66.7 | 84.6 | 0.741 | 0.746 |
| Semantic + normalization | 84.6 | 92.3 | 0.869 | 0.829 |
| **Hybrid** | **84.6** | **97.4** | **0.906** | **0.928** |

**End-to-end (hybrid)**

| Measure | Test | Dev |
|---|---|---|
| Answerable questions answered | 94.9% (37/39) | 98.9% (89/90) |
| Unanswerable held back | 100% (21/21) | 96.2% (25/26) |
| Decision accuracy | 96.7% | 98.3% |
| Citation correctness | 83.8% | 88.8% |
| Latency mean / p95 | 92.8 / 132.0 ms | 75.0 / 86.1 ms |

Cross-script consistency (dev, 30 groups): hybrid keeps all variants in the top 5 for 96.7% of groups and agrees on the first result for 90.0%. Plain BM25 manages 73.3% and 46.7%.
The test set has only 3 multi-script groups and just 3 English and 1 Urdu answerable questions, so per-language test numbers for those two are not reliable.

Re-run `python evaluate.py` to reproduce these figures; they are written to `results/evaluationReport.md`. Exact values can shift slightly with library versions.

## Validation pack
`python makeValidationPack.py` writes `validation_pack.zip` containing `per_question_results_test.csv` and `_dev.csv`, `retrieval_comparison.csv`, `failures.csv`, `settings_used.json`, the evaluation report and the data files.

## Microphone (optional)
The "Speak your question" box records in the browser and transcribes locally with `faster-whisper` (default size `small`; change with `ESAHULAT_WHISPER=base`). No audio leaves the machine and no hosted speech service is used.
Urdu speech comes back in Urdu script and English in English (not Roman Urdu). The transcript goes into the question box, where you can edit it before searching.
Speech accuracy has **not** been measured and is not part of the evaluation.

## Models, data and compliance
- Embeddings: `intfloat/multilingual-e5-small`, loaded locally with sentence-transformers.
- Speech (optional): `faster-whisper`, loaded locally.
- Keyword search: BM25 written from scratch.
- Data: 30 official excerpts (EN + UR) from dgip.gov.pk and 150 labelled queries (90 `train_dev`, 60 `test`), provided by the organisers. Provenance per passage is in `data/passages.json`. Source freshness should be re-checked before real use.
- Extra dev negatives: `data/devNegatives.json` (26 queries written by the team).
- Roman Urdu passage text is our own unofficial rendering and has not been reviewed by a native speaker.

## Limitations
- The knowledge base is only 30 passages, so retrieval is easier than over whole websites.
- One passage is cited per answer; questions needing two passages get one.
- The safety rules were written after we saw the structure of the test file, so test abstention results are likely optimistic.
- The sources contain no fee amounts, so fee questions ask for more detail instead of answering.
- Known confusions: adult/minor age rules, and neighbouring passages (F19/F20, F08/F30, F15/F27, F18/F17).
- Speech recognition accuracy is not measured.

## Team
[Team name] - [Member names]

Built for the TextTrek NLP challenge, 16 October 2026.
