# Intelligent Student Assignment Evaluation Assistant – SRS & Execution Plan

Nov 19, 2026&#32;

## 1. SRS (Software Requirements Specification)

The system reads one answer-sheet image, grades it against the faculty rubric, and shows a **preliminary** report that faculty must review before marks are final.

**Users**

- **Faculty** – uploads images, reviews the report, overrides marks.
- **Coordinator** – maintains `questions.csv` and `marking_rubric.csv`.

**Inputs and outputs**

| Type | Item | Source |
| --- | --- | --- |
| Input | Answer-sheet image (JPG/PNG) | Upload |
| Input | Question, reference answer, max marks | `questions.csv` |
| Input | Concepts and marks per question | `marking_rubric.csv` |
| Validation only | Typed answers, expected marks | `student_answers.csv` |
| Output | Evaluation report: marks, concepts covered/missing, feedback | Screen + CSV download |

**Functional requirements**

| ID | Requirement |
| --- | --- |
| FR1 | Upload and display the answer-sheet image |
| FR2 | OCR (EasyOCR/Tesseract) extracts Student ID, question number, question and answer; show raw text |
| FR3 | Normalise question ID (Q1, Q.1, Question 1 → Q1) and fetch reference answer + max marks |
| FR4 | Preprocess: lowercase, strip punctuation, tokenize, remove stop words (keep negations), lemmatize |
| FR5 | Show TF-IDF cosine and semantic similarity scores |
| FR6 | Check each rubric concept (keyword + synonym + semantic match) and award its marks |
| FR7 | Generate feedback: covered, missing, one improvement suggestion |
| FR8 | Show full report, allow download, and an "Evaluate another" button |
| FR9 | Faculty can correct OCR text and override marks before saving |
| FR10 | Batch validation mode: run all images, compare with `student_answers.csv` |

**Non-functional requirements**

- **Explainable** – every mark points to the concept and the answer phrase that earned it.
- **Fast** – target under 10 s per image on a laptop CPU after models load.
- **Offline and private** – runs locally; no student data leaves the machine.
- **Safe default** – every screen labels marks as preliminary.

**Tech stack:** Python 3.10+, Streamlit, EasyOCR (or pytesseract), OpenCV, pandas, NLTK/spaCy, scikit-learn (TF-IDF), sentence-transformers (`all-MiniLM-L6-v2`), rapidfuzz.

## 2. Existing systems and their gaps

No common tool goes from an answer-sheet photo to rubric marks with explained feedback, and that is the gap this project fills.

| Existing approach | What it does | Gap for this problem |
| --- | --- | --- |
| Manual evaluation | Faculty read and mark each sheet | Slow, tiring, marks vary between evaluators |
| LMS quizzes (Moodle, Google Forms) | Auto-grade MCQ and exact short answers | Cannot read images or judge descriptive answers |
| Gradescope | Scans sheets, groups similar answers, faculty apply a rubric | Paid; the marking itself is still done by humans |
| Essay scorers (e.g. ETS e-rater) | Score writing quality and grammar | Do not check subject concepts against a rubric |
| Plagiarism checkers (Turnitin) | Find copied text | Measure copying, not correctness |
| Research ASAG models (BERT short-answer grading) | Predict a score from typed text | No OCR, no concept-level feedback, not a ready tool |

**Our improvement:** image → OCR → rubric concept check → explainable marks + feedback, with faculty approval at the end.

## 3. Effective execution process

Build the pipeline as small, separately tested modules first, and add the Streamlit UI last.

1. **Study the dataset** – open all 4 files; note CSV column names, the question-ID format, and how the images are laid out (printed or handwritten, labels used).
2. **OCR module** (`ocr.py`)
   - Preprocess with OpenCV: grayscale, upscale 2x, denoise, threshold, deskew.
   - Run EasyOCR line by line, then split fields on the labels "Student ID", "Question", "Answer" using regex.
   - Test on every image right away against `student_answers.csv`.
3. **Question lookup** (`lookup.py`) – normalise the ID with regex; if it fails, fuzzy-match the question text against `questions.csv`.
4. **Preprocessing** (`preprocess.py`) – lowercase, clean, tokenize, remove stop words but keep "not/no", lemmatize.
5. **Scoring engine** (`scorer.py`)
   - Overall: TF-IDF cosine + sentence-embedding similarity (display only).
   - Per concept: exact/lemma match → synonym list → embedding similarity of the concept vs each answer sentence.
   - Award full marks above a tuned threshold, optional half marks in a middle band, then sum and cap at max marks.
6. **Feedback** (`feedback.py`) – template text built from covered and missing concepts.
7. **Streamlit app** (`app.py`) – one section per brief step (1–8), editable OCR text, mark override, CSV download, "Evaluate another".
8. **Validate and tune** (`validate.py`) – batch-run all images; report OCR match % and marks difference vs expected; tune thresholds.
9. **Document and demo** – README with setup, approach, results table, known limits.

```text
project/
├── app.py
├── modules/  ocr.py  lookup.py  preprocess.py  scorer.py  feedback.py
├── data/     answer_images/  questions.csv  marking_rubric.csv  student_answers.csv
├── validate.py
└── requirements.txt
```

**Team split (4 people):** OCR · lookup + preprocessing · scoring + feedback · UI + validation.

## 4. Loopholes to concentrate on

Most wrong marks will come from OCR and field parsing, not from the NLP, so fix the top four rows first.

| # | Loophole | What goes wrong | Fix |
| --- | --- | --- | --- |
| 1 | OCR misreads (blur, skew, handwriting) | Wrong words → wrong marks | OpenCV preprocessing; show OCR confidence; let faculty edit the text |
| 2 | Question text leaks into the answer | Student earns marks for words in the question itself | Cut the answer strictly after the "Answer" label; remove question words |
| 3 | Question ID misread (Ql, QI, Q.1, Question 1) | Wrong or no reference answer | Normalise l/I → 1; fall back to fuzzy match on question text |
| 4 | Using `student_answers.csv` as input | Breaks the main rule of the brief | Use it only in `validate.py` |
| 5 | Stop-word removal deletes "not" | "NLP is not part of AI" still gets the AI mark | Keep negation words; check for negation near a matched concept |
| 6 | Synonyms and abbreviations missed | "machines", "comprehend", "AI" vs "artificial intelligence" get 0 | Synonym/abbreviation map + embedding match |
| 7 | Keyword stuffing | A list of keywords gets full marks | Match on whole sentences; flag very short answers with many concepts |
| 8 | Semantic threshold too loose | Every on-topic answer gets all concepts | Tune per concept on the validation set |
| 9 | Similarity score used as the mark | Brief says similarity alone must not decide marks | Marks come only from the rubric; similarity is shown for info |
| 10 | Overlapping concepts (understand vs interpret) | One phrase earns two marks | Require separate evidence, or let faculty decide |
| 11 | CSV mismatch (column names, IDs, rubric total ≠ max marks) | Crash or wrong total | Validate files on load and show a clear error |
| 12 | Blank, off-topic or wrong-question answer | Random partial marks | Minimum similarity gate → 0 marks + "needs review" flag |
| 13 | Models reload on every Streamlit rerun | App feels very slow | `@st.cache_resource` for EasyOCR and the embedding model |
| 14 | No faculty control | Marks look final | "Preliminary" label, override field, reviewer name before saving |

## 5. Demo checklist

- [ ] Upload an image and show the extracted Student ID, question and answer
- [ ] Show the matched question, reference answer and max marks
- [ ] Show the cleaned answer and similarity scores
- [ ] Show the concept-wise marks table (covered / missing) and the total
- [ ] Show the feedback and the "Evaluate another" button
- [ ] Show the validation table: OCR match % and marks vs expected
- [ ] Show one faculty override
- [ ] Show one known failure case and how it is flagged
