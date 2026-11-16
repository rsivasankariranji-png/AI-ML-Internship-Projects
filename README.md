# College Enquiry Assistant

**Project code:** G04
**Institution:** VET Institute of Arts and Science (Co-education) College, Thindal, Erode, Tamil Nadu

A luxury-themed, FAQ-based enquiry assistant. A visitor types a question in natural
language; the Python backend finds the closest FAQ using TF-IDF and cosine similarity
and shows the answer together with its FAQ ID, or an honest fallback when no reliable
answer exists.

> **Data source:** FAQ answers are taken from the college's own *VETIAS Handbook 2024-25*
> (https://vetias.ac.in/wp-content/uploads/2024/08/HANDBOOK-2024-25.pdf). Details the handbook
> does not publish (fees, eligibility, scholarships, admission dates, etc.) are answered honestly
> as "not published - contact the college office". Fill these in `data/college_faqs.json` once the
> college confirms them.

## Objectives

1. Load an approved FAQ dataset from JSON and validate it.
2. Accept natural-language questions, including paraphrases.
3. Return the best answer together with its unique FAQ ID.
4. Fall back when no sufficiently reliable answer exists; reject unrelated questions.
5. Keep the user interface separate from the core question-processing logic.
6. Allow the core system to be evaluated independently of Flask.

## Features

- Three core classes: `FAQEntry`, `QueryProcessor`, `EnquiryAssistant` (in `core/`, no Flask imports).
- TF-IDF (unigrams + bigrams) with cosine similarity, a configurable threshold and an ambiguity check.
- 64 FAQs (handbook-sourced) across 13 categories; 89 independently written evaluation questions.
- Pages: Home, Enquiry (chat), FAQ Reference (searchable), About, Evaluation (development).
- JSON API: `POST /api/enquire`.
- Repeatable evaluation script with accuracy, precision, recall, F1 and a per-type breakdown.
- 181 automated pytest tests that need no internet access.

## Technology stack

Python 3.10+, Flask, scikit-learn, NumPy, JSON files, HTML5, CSS3, vanilla JavaScript, pytest.
Fonts (Playfair Display, Inter) load from Google Fonts when online and fall back to
Georgia and system fonts offline. No API keys or cloud services are used.

## Folder structure

```
college-enquiry-assistant/
├── app.py                      Flask app factory, routes, JSON API
├── requirements.txt
├── pytest.ini
├── README.md
├── core/                       Flask-free retrieval logic
│   ├── config.py               threshold, messages, paths
│   ├── faq_entry.py            FAQEntry + safe JSON loading/validation
│   ├── query_processor.py      normalisation, TF-IDF, cosine similarity
│   └── enquiry_assistant.py    application-level facade and fallback
├── data/
│   ├── college_faqs.json       64 FAQs from the VETIAS Handbook 2024-25
│   └── evaluation_questions.json   89 synthetic evaluation questions
├── evaluation/
│   ├── evaluate.py             metrics + command line
│   └── results/                saved reports (--save)
├── templates/                  base, index, enquiry, about, faq, evaluation, error
├── static/
│   ├── css/style.css           luxury theme
│   ├── js/main.js              navigation, FAQ search
│   ├── js/enquiry.js           chat behaviour (fetch API)
│   └── images/README.md
└── tests/                      pytest suite
```

## Installation (Windows PowerShell)

```powershell
cd college-enquiry-assistant
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

If PowerShell refuses to run the activation script, allow it for this window only and retry:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.venv\Scripts\Activate.ps1
```

On macOS/Linux use `python3 -m venv .venv` and `source .venv/bin/activate`.

## Run the application

```powershell
python app.py
```

Then open **http://127.0.0.1:5000** in your browser. Press `Ctrl+C` in the terminal to stop.

Optional environment variables (PowerShell syntax, set before `python app.py`):

| Variable | Default | Purpose |
|---|---|---|
| `ENQUIRY_SIMILARITY_THRESHOLD` | `0.35` | Minimum score (0–1) for returning an FAQ answer |
| `ENQUIRY_ENABLE_EVALUATION` | `1` | Set to `0` to hide the Evaluation page and API |
| `PORT` / `HOST` | `5000` / `127.0.0.1` | Where the development server listens |
| `FLASK_DEBUG` | `0` | Set to `1` for the debugger (local development only) |

## Run the tests

```powershell
python -m pytest
```

## Run the evaluation

```powershell
python -m evaluation.evaluate                 # print the report
python -m evaluation.evaluate --verbose       # also list every incorrect case
python -m evaluation.evaluate --save          # also write JSON to evaluation\results\
python -m evaluation.evaluate --threshold 0.4 # try another threshold
```

The same figures appear live on the **Evaluation** page of the web app. The evaluation uses
`core/` only; no browser or web server is involved.

### Metric definitions

Each evaluation question has an expected outcome: a specific FAQ ID (`match`) or the fallback.

| Outcome | Meaning |
|---|---|
| correct_match | FAQ expected, correct FAQ returned |
| wrong_match | FAQ expected, a different FAQ returned |
| missed_match | FAQ expected, fallback returned (an *incorrect fallback*) |
| correct_fallback | fallback expected, fallback returned |
| false_match | fallback expected, an FAQ answer returned |

- **Accuracy** = (correct_match + correct_fallback) / all questions
- **Precision** = correct_match / (correct_match + wrong_match + false_match): of the FAQ answers given, how many were right
- **Recall** = correct_match / (correct_match + wrong_match + missed_match): of the questions that have an answer, how many got the right one
- **F1** = 2·P·R / (P + R)
- **Rejection precision / recall** measure the fallback decision the same way.
- A metric whose denominator is zero (for example precision on a test type that has no expected FAQ answers) is shown as **n/a**, never as 0.

"Incorrect retrievals" on the Evaluation page = wrong_match + false_match. "Unmatched" = every question that received the fallback.

### Results from the shipped data

Run on 2026-10-25 with threshold 0.35 (`python -m evaluation.evaluate`):

| | |
|---|---|
| Questions | 89 (64 expect an FAQ, 25 expect the fallback) |
| Accuracy | 0.854 |
| Precision / Recall / F1 | 0.871 / 0.844 / 0.857 |
| direct / paraphrase / conversational | 0.867 / 0.840 / 0.778 accuracy |
| ambiguous / out-of-scope | 0.800 / 0.933 accuracy |

**Read these numbers with care.** The evaluation questions were written by the same
author as the FAQ data. The threshold (0.35) was chosen after sweeping values on this same set, where
0.30–0.40 performed almost identically, so the figures are optimistic for real users. Results will change
if you change the FAQ data, the lexicon or the threshold; re-run the script and use your own numbers.

Known misses in this run include generic one-word-style questions ("What is the process?",
"Is there any facility?"), "How do I reset my email password?" (matched the contact FAQ because of the
word "email"), and several undergraduate/postgraduate questions matched to the general courses FAQ.

## How retrieval works

1. **Validate** input: must be text, not empty, at most 500 characters.
2. **Normalise**: lower-case, drop punctuation and stop words, light plural stemming, plus a small editable
   domain lexicon (`DEFAULT_SYNONYMS` / `DEFAULT_PHRASES` in `core/query_processor.py`, e.g. "programme" → "course").
3. **TF-IDF** vectors (unigrams and bigrams) are fitted on every FAQ question and approved paraphrase.
4. **Cosine similarity** between the question and each text; each FAQ keeps its best score.
5. **Coverage penalty**: the score is multiplied by `(known words / all words) ** 0.5`, so one shared word
   cannot carry an unrelated question. Set `DEFAULT_COVERAGE_EXPONENT = 0` in `core/config.py` for plain cosine.
6. **Decision**: the best FAQ is returned only if its score ≥ threshold **and** it beats the runner-up by at
   least `DEFAULT_MIN_MARGIN` (0.05). Otherwise the fallback is returned with `reason` = `below_threshold`,
   `ambiguous_match` or `no_known_terms`.

The match-confidence bar in the UI is this similarity score. It measures text overlap, **not** the probability
that the answer is correct.

## JSON API

`POST /api/enquire` with `Content-Type: application/json`:

```json
{ "question": "What courses are available?" }
```

Response (values illustrative):

```json
{
  "success": true, "matched": true, "faq_id": "FAQ007",
  "question": "What courses are available?", "answer": "…", "category": "Courses",
  "similarity": 1.0, "reason": "reliable_match",
  "message": "Here is the information available in our college FAQ.", "error": null
}
```

Every response has these same keys. A fallback is `success: true, matched: false` with HTTP 200.
Invalid input returns `success: false` with 400 (empty, not text, too long, missing field, bad JSON),
415 (not JSON), 405 (wrong method), 413 (body over 16 KB), 503 (FAQ data could not be loaded) or 500.
`GET /api/evaluation` returns the evaluation report as JSON.

## Add a new FAQ

1. Open `data/college_faqs.json` and append a record with the next unused ID:

   ```json
   {
     "faq_id": "FAQ065",
     "question": "Your canonical question?",
     "answer": "Verified answer text.",
     "category": "Admissions",
     "paraphrases": ["Another way to ask it", "And another"]
   }
   ```

2. IDs must look like `FAQ065` and be unique; questions must be unique; `question`, `answer` and `category`
   are required. The file is validated when the app starts, and a clear error is shown if it is wrong.
3. Restart the app, run `python -m pytest` (a few tests assume the shipped data) and re-run the evaluation.
   Add evaluation questions worded differently from your FAQ text.

## Change the similarity threshold

Higher = stricter (more fallbacks, fewer wrong answers); lower = more answers, more mistakes.

```powershell
$env:ENQUIRY_SIMILARITY_THRESHOLD = "0.45"
python app.py
```

Or change `DEFAULT_SIMILARITY_THRESHOLD` in `core/config.py`. Use `python -m evaluation.evaluate --threshold 0.45`
to see the effect first. Invalid values are rejected with a clear error.

## Security notes (what is actually implemented)

- Question and FAQ text is escaped by Jinja or inserted with `textContent`; the page never uses `innerHTML` for such text.
- Submitted questions are not echoed back by the API, not written to files or a database, and not logged by the app.
- Strict `Content-Security-Policy` (scripts and connections from the same origin only), `X-Content-Type-Options`,
  `X-Frame-Options` and `Referrer-Policy` headers; API responses are sent with `Cache-Control: no-store`.
- Request bodies over 16 KB are rejected; questions over 500 characters are rejected.
- There are **no** accounts, authentication, rate limiting or CSRF tokens (not needed for this stateless, read-only
  API, but add rate limiting before any public deployment). The built-in Flask server is for local use; use a
  production WSGI server for deployment.

## Known limitations

- Matching is word-based. Synonyms outside the small lexicon, misspellings and non-English questions are missed.
- Very short or generic questions can match the wrong FAQ; the ambiguity check catches only some of them.
- No conversation memory: each question is answered independently.
- The lexicon, stop-word list and threshold were tuned by hand on synthetic data.
- Fonts need internet access the first time; otherwise fallback fonts are used.
- No official logo is bundled; the header uses a typographic monogram.

## Ethical and data-quality considerations

- Every shipped answer cites the VETIAS Handbook 2024-25 or defers to the college office; re-check against the latest handbook each year.
- Keep one owner for the FAQ file, and review answers whenever fees, dates or rules change.
- Do not add personal data to FAQs. The assistant gives general information only and refers users to the college
  office when unsure.
- Report evaluation results honestly, including failures, and re-run them after any change.

## Future enhancements

- Replace the hand-built lexicon with sentence embeddings (still running locally) and compare against TF-IDF.
- Spelling correction; Tamil-language support.
- Admin screen for editing FAQs with validation.
- Rate limiting, production deployment guide, and an analytics-free feedback button ("was this helpful?").
- A larger evaluation set written by people who did not write the FAQs.

## Requirement checklist

| Requirement | Where |
|---|---|
| Load approved FAQ dataset from JSON | `core/faq_entry.py` `load_faq_entries`, `data/college_faqs.json` |
| Natural-language questions | `QueryProcessor.process`, `POST /api/enquire` |
| Best answer with unique FAQ ID | `process` result, `EnquiryAssistant.ask`, answer card + ID badge in `enquiry.js` |
| Fallback when unreliable | threshold + margin in `QueryProcessor`; `FALLBACK_MESSAGE` |
| Paraphrase support | paraphrases in dataset, normalisation, TF-IDF; `paraphrase` evaluation type |
| Reject unrelated questions | `no_known_terms` / `below_threshold`; `out_of_scope` evaluation type |
| UI separated from core logic | `core/` has no Flask import (tested) |
| Independent evaluation | `evaluation/evaluate.py` (uses `core/` only) |
| Classes `FAQEntry`, `QueryProcessor`, `EnquiryAssistant` | `core/` |
| ≥ 60 FAQs, 13 categories, unique IDs | `data/college_faqs.json` (64); tests |
| ≥ 80 independent evaluation questions | `data/evaluation_questions.json` (89); independence test |
| Home, Enquiry, About, FAQ Reference, Evaluation pages | `templates/`, routes in `app.py` |
| Luxury theme, palette, fonts, responsive | `static/css/style.css`, `templates/base.html` |
| Input validation, safe JSON, HTTP status codes, missing files | `app.py`, `core/faq_entry.py` |
| No hard-coded secrets, no stored questions | none used; questions are never persisted |
| pytest tests (12 listed areas) | `tests/` |
| README with PowerShell commands | this file |
