# EduGrade-AI

[![Watch the video](https://img.shields.io/badge/Demo-Video-red?style=for-the-badge&logo=youtube)](https://www.youtube.com/watch?v=JWd_7I_x4Fc)
[![Flask](https://img.shields.io/badge/Backend-Flask-black?style=for-the-badge&logo=flask)](https://flask.palletsprojects.com/)
[![Gemini](https://img.shields.io/badge/LLM-Gemini-blue?style=for-the-badge&logo=googlegemini)](https://ai.google.dev/)

A virtual classroom platform that automates assignment evaluation. Teachers upload assignments, students submit scanned/PDF answers, and the system extracts the text, checks it for plagiarism (text-level and handwriting-level), and generates an LLM-based grading report with feedback — all inside a class + submission workflow like Google Classroom.

---

## Features

- Google Classroom–style flow: create/join classes via class codes, post announcements, create and submit assignments
- Automatic OCR + text extraction from submitted PDFs
- Two-layer plagiarism detection: text-similarity scoring, escalating to handwriting comparison when text overlap is very high
- LLM-based auto-grading with a written evaluation and improvement feedback, benchmarked against a teacher-provided ideal answer
- In-place PDF annotation: spelling mistakes highlighted red, grammar issues yellow, key terms green
- Role-based views for teachers vs. students, with secure auth

---

## 1. Text Extraction — Google Document AI

Every submitted PDF is sent to a Google Cloud **Document AI** processor rather than a plain OCR library, since scanned/handwritten student submissions are noisier than typical documents. The file is read as raw bytes and wrapped in a `RawDocument`, then sent to a `DocumentProcessorServiceClient` pointed at a region-specific Document AI endpoint. Document AI returns a structured `Document` object containing the recognized text plus layout information (bounding boxes per token/paragraph), which is what makes the later handwriting-comparison step possible — we're not just getting plain text back, we're getting *where* each word sits on the page.

## 2. Plagiarism Detection

**Text-based check** — pairwise, for every submission against every other submission on the same assignment:
- **Sequence similarity** — `difflib.SequenceMatcher` ratio between the two texts, plus `find_exact_matches` to pull out actual overlapping phrases (5+ word runs)
- **Cosine similarity** — texts vectorized and compared for cosine distance
- **Trigram overlap** — text broken into 3-word n-grams; overlap = intersection / max(set sizes)

These three scores are averaged into a single `plagiarism_likelihood` percentage, and every result (scores + the matched text spans) is stored in a `PlagiarismResult` table so results are viewable per-submission later.

**Visual (handwriting) check** — triggered only when the text similarity crosses a **90% threshold**, since that's the point where copied *content* becomes a real question of whether it was copied by *hand*, not just typed similarly:
1. Pick a matching word that both PDFs share
2. Use Document AI's bounding-box data to locate that exact word on each page
3. Crop just that word out of the page image using its bbox
4. Grayscale both crops to strip out scanner/lighting noise
5. Compare the two crops to judge whether the handwriting itself looks like a match

This two-stage design means cheap text comparison runs on every pair, and the more expensive image-based check only runs when there's already strong reason to suspect copying.

## 3. Database — Virtual Classroom Backend

Built with **SQLAlchemy** (Flask-SQLAlchemy) modeling the classroom domain as related tables: `User`, `Class`, `ClassEnrollment` (join table carrying the `is_teacher` flag), `Announcement`, `Assignment`, `Submission`, `AIEvaluation`, and `PlagiarismResult` — all linked with foreign keys so a submission traces back through its assignment, class, and student.

- **Security**: passwords are never stored in plaintext — `werkzeug.security`'s `generate_password_hash` / `check_password_hash` handle hashing and verification; class access is enforced by checking `ClassEnrollment` + the `is_teacher` flag before returning any submission, evaluation, or plagiarism data (so a student can't view another student's results by guessing a URL)
- **Scalability**: normalized relational schema (enrollments, submissions, and evaluations as separate tables rather than nested JSON blobs) so queries scale per-class/per-assignment instead of scanning whole tables, and each expensive artifact (AI evaluation, plagiarism result) is cached in its own row instead of being recomputed on every page load

## 4. Backend & Frontend

- **Backend**: Flask, handling auth, class/assignment CRUD, file uploads, and orchestrating the extraction → plagiarism → grading pipeline
- **Frontend**: server-rendered HTML templates styled with Bootstrap CSS, with JS for interactive bits (PDF viewers, dynamic forms)

## 5. LLM-Based Grading — Gemini

Uses Google's free-tier **Gemini** model (`google-generativeai`) in three modes:
- **Summary** — condenses a submission into a short summary for the teacher
- **Evaluation** — grades a submission on its own merits
- **Evaluation with reference** — grades a submission *against* the teacher's uploaded ideal answer, when one exists, for a more calibrated score

The response is parsed out of markdown into clean structured feedback and stored in an `AIEvaluation` row linked to the submission, so it only needs to be generated once and is then just displayed on demand.

---

## Tech Stack

| Layer | Tech |
|---|---|
| Backend | Flask |
| Database | SQLAlchemy (SQLite) |
| Auth | Werkzeug security (password hashing) |
| OCR / Extraction | Google Cloud Document AI |
| Grading LLM | Google Gemini |
| Frontend | HTML, Bootstrap CSS, JavaScript |

## Getting Started

### Prerequisites
- Python 3.x
- A Google Cloud Document AI processor + API credentials
- A Google Gemini API key

### Installation
```sh
git clone https://github.com/Aarya0706/EduGrade-AI.git
cd EduGrade-AI
pip install -r requirements.txt
```

Set your credentials (Document AI project/location/processor ID and Gemini API key) as environment variables or in a config section at the top of `app.py`, then run:

```sh
python app.py
```

The app will be available at `http://localhost:5000`.

## Demo

A walkthrough video is linked at the top of this README. *(No public live deployment currently — the app runs locally per the steps above.)*

## Team

Group project — text extraction, plagiarism detection (text + visual), and database/backend design by me.