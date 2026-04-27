# ForgeDetectAI Backend (MVP)

Production-style FastAPI backend for AI-powered media authenticity verification.

## Architecture

```text
backend/
├── main.py                       # App bootstrap, middleware, routers, error handling
├── routes/
│   ├── auth.py                   # Firebase token verification endpoint
│   ├── upload.py                 # Secure upload endpoint
│   ├── analyze.py                # Forensic pipeline orchestration
│   └── reports.py                # Results/history/report APIs
├── services/
│   ├── metadata_detector.py      # ExifTool-driven metadata forensics
│   ├── watermark_detector.py     # Visible watermark + frequency synthetic checks
│   ├── feature_match_detector.py # ORB matching + anomaly heuristics
│   ├── deepfake_api_client.py    # Hugging Face inference integration
│   ├── trust_score_engine.py     # Weighted fusion engine (30/30/40)
│   └── gemini_explainer.py       # Gemini explainable report generation
├── utils/
│   ├── preprocessing.py          # Input/file validation
│   ├── file_handler.py           # Firebase Storage + Firestore helpers
│   └── report_generator.py       # Report document serializer
└── models/
    ├── user_model.py             # User schema
    └── report_model.py           # Analysis report schema
```

## Core Pipeline

1. `POST /upload` validates file type/size and stores media in Firebase Storage.
2. `POST /analyze` downloads media to temp file and runs:
   - Metadata forensics (`metadata_detector`)
   - Watermark/synthetic checks (`watermark_detector`)
   - Feature anomaly checks (`feature_match_detector`)
   - Hugging Face deepfake inference (`deepfake_api_client`)
3. `trust_score_engine` fuses scores:
   - 30% watermark
   - 30% metadata
   - 40% deepfake/feature
4. `gemini_explainer` generates structured forensic reasoning JSON.
5. Full result is persisted in Firestore (`scans` collection).

## Setup

### 1) Install dependencies

```bash
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 2) Configure environment

```bash
cp .env.example .env
```

Fill in:
- Firebase service account and storage bucket
- `HUGGINGFACE_API_TOKEN`
- `GEMINI_API_KEY`

### 3) ExifTool installation

Install ExifTool on host/container so metadata module can run full checks.

### 4) Run locally

```bash
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

## API Documentation

Interactive OpenAPI docs are available at:
- `http://localhost:8000/docs`
- `http://localhost:8000/redoc`

### Headers (required for protected routes)

```http
Authorization: Bearer <FIREBASE_ID_TOKEN>
```

### `POST /upload`
Multipart upload endpoint.

Form field:
- `media`: image/video file

Sample response:

```json
{
  "scan_id": "scan_9f1fd34d2f4a45cf8d40b19a936f0861",
  "file_name": "suspect.jpg",
  "storage_path": "uploads/scan_9f1fd34d2f4a45cf8d40b19a936f0861/suspect.jpg",
  "content_type": "image/jpeg",
  "uploaded_at": "2026-04-26T12:10:10.101010+00:00",
  "size_bytes": 298204,
  "status": "uploaded"
}
```

### `POST /analyze`

Body:

```json
{
  "scan_id": "scan_9f1fd34d2f4a45cf8d40b19a936f0861",
  "storage_path": "uploads/scan_9f1fd34d2f4a45cf8d40b19a936f0861/suspect.jpg",
  "reference_storage_path": null
}
```

Sample response (trimmed):

```json
{
  "scan_id": "scan_9f1fd34d2f4a45cf8d40b19a936f0861",
  "metadata_analysis": {
    "metadata_risk_score": 38.0,
    "anomalies": ["CreateDate and ModifyDate mismatch"]
  },
  "watermark_analysis": {
    "synthetic_likelihood_score": 64.7
  },
  "feature_analysis": {
    "feature_risk_score": 59.2
  },
  "deepfake_analysis": {
    "deepfake_risk_score": 73.1
  },
  "trust_score": {
    "authenticity_score": 35.48,
    "risk_level": "high",
    "confidence_score": 67.26,
    "final_verdict": "likely_manipulated"
  },
  "ai_explanation": {
    "summary": "Signals indicate probable synthetic manipulation.",
    "risk_reasoning": ["Metadata mismatch", "High deepfake confidence"],
    "evidence": ["Editing software traces", "Frequency artifacts"],
    "verdict_explanation": "Weighted engine prioritizes deepfake and watermark risks."
  }
}
```

### `GET /results/{scan_id}`
Returns stored analysis for one scan ID.

### `GET /history?limit=20`
Returns recent scan analyses.

### `POST /generate-report`

Body:

```json
{
  "scan_id": "scan_9f1fd34d2f4a45cf8d40b19a936f0861",
  "include_verbose": true
}
```

Returns generated human-readable report text and persists to `reports` collection.

## Gemini Integration Details

`backend/services/gemini_explainer.py` builds and sends a forensic reasoning prompt to Gemini and enforces JSON output parsing.

Internal prompt template (simplified):

```text
You are a digital media forensic analyst. Analyze detector outputs and produce strict JSON only.
scan_id: ...
metadata_result: ...
watermark_result: ...
feature_result: ...
deepfake_result: ...
trust_result: ...
Return JSON with summary, risk_reasoning, evidence, verdict_explanation.
```

## Security Notes

- Firebase Auth ID token enforcement via middleware (`Authorization: Bearer ...`).
- Input validation for MIME type, extension, and file size.
- Rate limiting configured in app middleware (per-IP per-minute window).
- Global exception handling and structured logging.
- Temp-file based processing pipeline for reduced memory pressure.
- Secrets kept in `.env` (never commit real secrets).

### Suggested Firebase Security Rules (MVP baseline)

Firestore rules (tighten for production IAM):

```text
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /scans/{scanId} {
      allow read, write: if request.auth != null;
    }
    match /reports/{scanId} {
      allow read, write: if request.auth != null;
    }
  }
}
```

Storage rules:

```text
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /uploads/{scanId}/{fileName} {
      allow read, write: if request.auth != null;
    }
  }
}
```

## Deploy to Google Cloud Run

1. Build Docker image with ExifTool installed.
2. Push to Artifact Registry.
3. Deploy Cloud Run service with environment variables and service account.

Example commands:

```bash
gcloud builds submit --tag gcr.io/<PROJECT_ID>/forgedetectai-backend

gcloud run deploy forgedetectai-backend \
  --image gcr.io/<PROJECT_ID>/forgedetectai-backend \
  --platform managed \
  --region <REGION> \
  --allow-unauthenticated \
  --set-env-vars FIREBASE_STORAGE_BUCKET=...,HUGGINGFACE_API_TOKEN=...,GEMINI_API_KEY=...
```

Use Secret Manager for API keys in production and restrict ingress/auth policies.

```
ForgeDetectAI
├─ .env
├─ .env.example
├─ backend
│  ├─ main.py
│  ├─ models
│  │  ├─ report_model.py
│  │  ├─ user_model.py
│  │  ├─ __init__.py
│  │  └─ __pycache__
│  │     ├─ report_model.cpython-312.pyc
│  │     ├─ user_model.cpython-312.pyc
│  │     └─ __init__.cpython-312.pyc
│  ├─ routes
│  │  ├─ analyze.py
│  │  ├─ auth.py
│  │  ├─ reports.py
│  │  ├─ upload.py
│  │  ├─ __init__.py
│  │  └─ __pycache__
│  │     ├─ analyze.cpython-312.pyc
│  │     ├─ auth.cpython-312.pyc
│  │     ├─ reports.cpython-312.pyc
│  │     ├─ upload.cpython-312.pyc
│  │     └─ __init__.cpython-312.pyc
│  ├─ services
│  │  ├─ deepfake_api_client.py
│  │  ├─ feature_match_detector.py
│  │  ├─ gemini_explainer.py
│  │  ├─ metadata_detector.py
│  │  ├─ trust_score_engine.py
│  │  ├─ watermark_detector.py
│  │  ├─ __init__.py
│  │  └─ __pycache__
│  │     ├─ deepfake_api_client.cpython-312.pyc
│  │     ├─ feature_match_detector.cpython-312.pyc
│  │     ├─ gemini_explainer.cpython-312.pyc
│  │     ├─ metadata_detector.cpython-312.pyc
│  │     ├─ trust_score_engine.cpython-312.pyc
│  │     ├─ watermark_detector.cpython-312.pyc
│  │     └─ __init__.cpython-312.pyc
│  ├─ utils
│  │  ├─ auth_middleware.py
│  │  ├─ file_handler.py
│  │  ├─ preprocessing.py
│  │  ├─ report_generator.py
│  │  ├─ __init__.py
│  │  └─ __pycache__
│  │     ├─ auth_middleware.cpython-312.pyc
│  │     ├─ file_handler.cpython-312.pyc
│  │     ├─ preprocessing.cpython-312.pyc
│  │     ├─ report_generator.cpython-312.pyc
│  │     └─ __init__.cpython-312.pyc
│  ├─ __init__.py
│  └─ __pycache__
│     ├─ main.cpython-312.pyc
│     └─ __init__.cpython-312.pyc
├─ Dockerfile
├─ firebase-service-account.json
├─ README.md
└─ requirements.txt

```#   F o r g e D e t e c t A I  
 