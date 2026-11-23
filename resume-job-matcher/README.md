# Intelligent Resume-Job Matching System

A focused Flask dashboard that reads PDF / DOCX / TXT resumes, extracts structured information with NER, ranks candidates with sentence embeddings + TF-IDF cosine similarity, analyses skill gaps, and generates **external job-search links** for any job title and location.

Sentiment analysis and LDA topic modelling are **separate executed Jupyter notebooks** (`notebooks/`). Gemini is an **optional card inside the matching results page** - local NLP works fully without an API key, and nothing is sent to Google without explicit consent.

**Stack:** Python 3.10+ · Flask · spaCy · Sentence Transformers (`all-MiniLM-L6-v2`) · scikit-learn · VADER · pandas / matplotlib / seaborn · google-genai · pypdf · python-docx · vanilla HTML/CSS/JS

---

## Architecture at a glance

| Surface | Purpose | Key files |
|---|---|---|
| **One dashboard** | Job inputs → resume upload → extraction/NER → embeddings + TF-IDF → skill gap → ranking → report → job links → optional Gemini | `templates/index.html`, `static/js/main.js`, `static/css/style.css`, `app.py` |
| **Reusable NLP backend** | Parsing, preprocessing, NER, skill taxonomy, embeddings, matching, skill gap, pipeline demo, IE benchmark | `nlp/*.py`, `data/*.json` |
| **Sentiment notebook** | Independently executed recruitment-feedback sentiment exercise with metrics and charts | `notebooks/sentiment_analysis.ipynb` |
| **Topic notebook** | Independently executed LDA exercise on resume/job corpus with keywords and charts | `notebooks/topic_modelling.ipynb` |
| **Optional cloud AI** | Four consent-gated Gemini tasks inside the results page; never changes scores | `nlp/gemini_service.py`, `nlp/prompt_templates.py` |

The previous five-tab navigation was removed from the UI. The backend modules and routes for extraction, sentiment and topics are **preserved** for tests and API use. The notebooks embed the same small classroom datasets inline so they execute independently.

---

## Main dashboard features

### 1. Role and resume inputs
- Type **any job title** (AI Engineer, Python Developer, Data Scientist, Web Developer, etc.) or start from 5 sample roles.
- Paste/edit a custom job description (at least 10 words) or use the sample text.
- Enter **any preferred location**. Quick chips: Pune, Mumbai, Bengaluru, Remote and Across India.
- Drag-and-drop or browse for up to 10 PDF/DOCX/TXT resumes (5 MB each).
- **Load sample resume** adds a real in-memory TXT file for an instant demo.

### 2. Local NLP pipeline (runs on this machine)
- Text extraction and Unicode cleaning (`nlp/resume_parser.py`, `nlp/preprocessing.py`).
- spaCy `en_core_web_sm` NER plus regex extraction for names, emails, phones, URLs, education, dates and years of experience (`nlp/ner_extractor.py`).
- 116-skill taxonomy with aliases and categories (`data/skills.json`) so `ReactJS`/`react.js`/`React` normalise to **React**.
- `all-MiniLM-L6-v2` sentence embeddings with chunking and averaging, cosine similarity, and a TF-IDF (1-2-gram) baseline (`nlp/embeddings.py`, `nlp/semantic_matcher.py`).
- Skill-gap analysis: matched, missing and extra skills with priority and category coverage (`nlp/skill_gap.py`).
- Transparent scoring: raw cosines are shown beside calibrated 0-100 scores; semantic backend and NER availability are stated explicitly.

### 3. Matching report
For every resume the report shows:
- rank, verdict and overall score;
- MiniLM semantic score and TF-IDF baseline with raw cosines;
- matched, missing (high-priority flagged) and extra skills;
- category coverage bars and grounded suggestions;
- **NER and extracted candidate information** (name, contacts, education, experience, organisations, locations, dates) directly inside the result card;
- a preview of the actual extracted resume text;
- score weights, job requirements and clear fallback warnings.

The extraction and pipeline APIs remain available at `POST /api/extract` and `POST /api/nlp/pipeline`, and the labelled extraction benchmark remains at `GET /api/extract/evaluation`.

### 4. Explore Job Opportunities
After the report, `GET /api/job-links` builds **real external search URLs** from the entered title and location:

- LinkedIn Jobs - `jobs/search/?keywords=...&location=...`
- Indeed India - `in.indeed.com/jobs?q=...&l=...`
- Naukri.com - slug plus `query`/`city` parameters
- Glassdoor India - `sc.keyword` plus `locName`
- Wellfound - `jobs?q=...&l=...`

Links open in a new tab (`target="_blank" rel="noopener noreferrer"`). The UI labels them **External job search** and states that the app does **not verify or invent individual vacancies**. Changing the title or location refreshes the links without rerunning matching.

### 5. Optional Gemini Career Assistant (inside results)
Four tasks:
1. **Match explanation** - explains local strengths, gaps and score interpretation.
2. **Resume improvement** - priority actions based on the computed gap.
3. **Bullet suggestions** - rewrites *existing* bullets without inventing numbers or experience.
4. **Interview preparation** - technical and HR questions with answer hints.

Safety: consent checkbox per request, redaction of emails/phones/URLs/names, length-limited excerpt, untrusted text wrapping, strict JSON schemas, task-specific prompts, classified errors, and `advisory_only: true` / `does_not_change_scores: true`. Local matching completes before any cloud call and works with no key.

---

## Project layout

```
resume-job-matcher/
├── app.py                      Flask routes (15 endpoints) + single-dashboard renderer
├── requirements.txt
├── .env.example / .gitignore
├── README.md
├── RUBRIC_MAPPING.md           Four-criterion evidence map
├── DEMO_GUIDE.md               Presentation walkthrough
├── BUILD_STATUS.md             Verification log
├── IMPLEMENTATION_REPORT.md    Fully / partially / missing status
├── nlp/
│   ├── resume_parser.py        PDF / DOCX / TXT extraction
│   ├── preprocessing.py        Cleaning, spaCy pipeline, lemmatisation
│   ├── ner_extractor.py        NER + regex + 116-skill normalisation
│   ├── embeddings.py           MiniLM wrapper (lazy, chunked, shared)
│   ├── semantic_matcher.py     Semantic/TF-IDF scores, ranking, report payload
│   ├── skill_gap.py            Matched / missing / extra skills
│   ├── pipeline_demo.py        Transparent NLP pipeline
│   ├── sentiment_analyzer.py   VADER + labelled evaluation (API/notebook support)
│   ├── topic_modeling.py       LDA pipeline (API/notebook support)
│   ├── ie_evaluation.py        Extraction benchmark
│   ├── prompt_templates.py     Four Gemini prompts + JSON validation
│   └── gemini_service.py       SDK client, redaction, fallbacks, errors
├── data/
│   ├── skills.json             116-skill taxonomy with aliases
│   ├── sample_jobs.json        5 sample roles
│   ├── sentiment_dataset.json  36 balanced labelled sentences
│   ├── topic_corpus.json       15-document anonymised corpus
│   └── ie_eval_cases.json      10 annotated extraction cases
├── notebooks/
│   ├── sentiment_analysis.ipynb   Executed VADER notebook (outputs + charts saved)
│   └── topic_modelling.ipynb      Executed LDA notebook (outputs + charts saved)
├── templates/index.html        One dashboard, no navigation tabs
├── static/css/style.css        Blue/white responsive theme
├── static/js/main.js           Single-dashboard workflow + AI + job links
└── tests/                      7 pytest modules (164 tests)
```

---

## Installation (Windows, PowerShell)

```powershell
cd resume-job-matcher

python -m venv .venv
.\.venv\Scripts\Activate.ps1
# If activation is blocked: Set-ExecutionPolicy -Scope CurrentUser RemoteSigned

python -m pip install --upgrade pip
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

`requirements.txt` includes notebook dependencies (`pandas`, `matplotlib`, `seaborn`, `nbformat`, `nbclient`, `ipykernel`).

> First run needs internet to download `all-MiniLM-L6-v2` (~90 MB), cached in `%USERPROFILE%\.cache\huggingface`. Afterwards local NLP works offline. If the download fails, the app explicitly falls back to TF-IDF.

### Optional Gemini key

```powershell
Copy-Item .env.example .env
notepad .env        # set GEMINI_API_KEY=your-key-here
```

Or for one session: `$env:GEMINI_API_KEY = "your-key-here"`. The key is read server-side only and is never logged or returned to the browser. `.env.example` contains only a placeholder; never commit a real key. Default model is `gemini-3.8-flash`; if it is overloaded, rate-limited, unavailable or times out, the app automatically tries `gemini-3.6-flash`, `gemini-3.5-flash`, `gemini-3.1-flash-lite`, `gemini-2.5-flash` and `gemini-2.0-flash`. Before any cloud call, both the resume excerpt and the job title/description are redacted and untrusted text is neutralised in prompts.

---

## Run and use

```powershell
python app.py
# open http://127.0.0.1:5000
```

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | `5000` | Port |
| `HOST` | `127.0.0.1` | Bind address |
| `PRELOAD_MODEL` | `1` | `0` loads MiniLM on first analysis |
| `GEMINI_API_KEY` | empty | Enables optional cloud AI |
| `GEMINI_MODEL` | `gemini-3.8-flash` | Primary model |
| `GEMINI_FALLBACK_MODELS` | `gemini-3.6-flash,gemini-3.5-flash,gemini-3.1-flash-lite,gemini-2.5-flash,gemini-2.0-flash` | Tried in order when the primary model is unavailable or overloaded |

### Dashboard workflow
1. Enter a job title, edit/paste a description and set a location (or click a sample role/location chip).
2. Add one or more resumes (or click **Load sample resume**).
3. Click **Analyze & rank**.
4. Read ranking, scores, NER/profile, skill gap and extracted-text preview.
5. Use **Explore Job Opportunities** to open real external searches for the title/location.
6. Optionally tick the consent checkbox and run a Gemini task in the **Gemini AI Career Assistant** card.

Header chips report spaCy, embedding and Gemini status from `/api/health`.

---

## API

| Endpoint | Method | Description |
|---|---|---|
| `GET /` | GET | Single matching dashboard |
| `GET /api/health` | GET | Model/backend status and feature availability |
| `GET /api/jobs` | GET | 5 sample roles |
| `GET /api/job-links` | GET | `title`, `location` to 5 encoded external search URLs plus disclaimer |
| `POST /api/match` | POST | `job_title`, `job_description`, 1-10 `resumes` to ranked report with text preview/stats |
| `POST /api/extract` | POST | Structured entities, skills, optional skill gap |
| `POST /api/nlp/pipeline` | POST | Tokens, lemmas, vectors, TF-IDF, embeddings, cosine |
| `GET /api/extract/evaluation` | GET | Labelled extraction benchmark |
| `POST /api/sentiment` / `GET /api/sentiment/evaluation` | POST / GET | VADER analysis / metrics (notebook support) |
| `GET /api/topics/sample` / `POST /api/topics` | GET / POST | LDA corpus / modelling (notebook support) |
| `GET /api/ai/status` / `POST /api/ai/test` | GET / POST | Gemini status / connection test |
| `POST /api/ai/insights` | POST | Consent-gated task; resume file or `resume_text`, job context |

```powershell
curl.exe "http://127.0.0.1:5000/api/job-links?title=AI%20Engineer&location=Pune"
curl.exe -X POST http://127.0.0.1:5000/api/match -F job_title="AI Engineer" -F "job_description=..." -F "resumes=@C:\path\cv.pdf"
```

`/api/ai/insights` returns `400 consent_required` without consent; `/api/ai/test` returns `503 not_configured` without a key. Neither calls Gemini in those cases.

---

## How scoring works

All scores are 0-100; raw cosines are shown beside them.

| Component | Weight | Definition |
|---|---|---|
| Semantic | 50% | MiniLM embedding cosine, rescaled (0.15 to 0, 0.70 to 100) |
| Skill coverage | 35% | % of recognised job skills found in the resume |
| TF-IDF baseline | 15% | Lemmatised 1-2-gram TF-IDF cosine, rescaled (0.50 to 100) |

If the job has no recognised skills, the skill weight is removed and the rest renormalised. Verdicts: 75+ Excellent, 60+ Good, 40+ Moderate, else Weak. Constants live in `nlp/semantic_matcher.py`. Sentiment and Gemini **never** affect these scores.

---

## Jupyter notebooks

Both notebooks are saved **with executed outputs and charts**. Run them from the project root:

```powershell
.\.venv\Scripts\python.exe -m jupyter nbconvert --to notebook --execute --inplace notebooks\sentiment_analysis.ipynb
.\.venv\Scripts\python.exe -m jupyter nbconvert --to notebook --execute --inplace notebooks\topic_modelling.ipynb
```

If the `jupyter` CLI is not installed, `nbclient` is in `requirements.txt` and the notebooks can also be run with *Kernel -> Restart & Run All* in JupyterLab/VS Code.

| Notebook | Dataset | Calculated results saved in outputs |
|---|---|---|
| `sentiment_analysis.ipynb` | 36 hand-labelled recruitment/job-review texts embedded inline (12 per class) | VADER scores, class visualisations, accuracy **0.917**, macro-F1 **0.916**, per-class precision/recall/F1, 3x3 confusion matrix, misclassified examples, interpretation |
| `topic_modelling.ipynb` | 15 anonymised resume/job/project documents embedded inline | CountVectorizer matrix (15x800), 4-topic LDA, perplexity **1063.19**, top keywords/weights, document-topic heatmap, dominant-topic counts, interpretation |

The notebooks are teaching exercises; sentiment is not candidate personality scoring and LDA topics are exploratory, not ground truth.

---

## Testing and verification

```powershell
python -m compileall -q app.py nlp tests
node --check static/js/main.js
python -m pytest -q
```

Latest full suite in the documented `.venv`: **164 passed**. Tests use a deterministic fake embedder and a fake Gemini service; no key or network call is required. A bare system Python is unsupported because it lacks `vaderSentiment` and `google-genai`. A separate live Gemini check was performed with the user-provided key in the local `.env` file (ignored by Git). Requests use a 45-second per-model timeout and automatically fall back on timeouts, overloaded models or per-model rate limits. See `BUILD_STATUS.md` and `IMPLEMENTATION_REPORT.md` for the executed verification log.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| spaCy chip says blank pipeline | `python -m spacy download en_core_web_sm`, restart |
| Embeddings chip says TF-IDF fallback | No MiniLM download; check internet/proxy and restart. Matching still works |
| Gemini card says not configured | Expected without key. Local matching is unaffected; set `GEMINI_API_KEY` to enable it |
| Gemini task returns consent error | Tick the consent checkbox in the results card and retry |
| Job links look wrong | Check the title/location fields; links are generated from those exact values and are external searches, not verified vacancies |
| `ModuleNotFoundError: nlp` in tests | Run `python -m pytest` from `D:\resume-job-matcher` |
| PDF rejected | Scanned/image-only PDFs are unsupported (no OCR); export a text-based PDF or use DOCX/TXT |

---

## Notes and limitations

- Built-in Flask server is for local use; use a WSGI server for deployment.
- Extraction and sentiment benchmarks are small classroom datasets, not production accuracy claims.
- Skill extraction is dictionary-based; add aliases in `data/skills.json`.
- Experience is taken from explicit "N years" statements or estimated from date ranges and labelled as an estimate.
- Gemini output is advisory and must be reviewed by a human.
