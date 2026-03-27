---
title: Requirements
has_children: false
nav_order: 3
---

# Requirements

## User Stories

Main persona: a user who wants to identify dog breeds fast from a camera/photo, in a web browser (desktop or mobile).  
The app must manage image upload, breed inference, and text advice generation.

- **US1**: As a user, I want to upload a dog photo so I can get a breed prediction.
- **US2**: As a user, I want to see a detected breed and AI recommendations for care.
- **US3**: As a user, I want to retry with another photo if the result is wrong.
- **US4**: As a user, I want to see a clear error when the uploaded image is not a dog.
- **US5**: As an admin, I want the backend to support a breed catalog update API.
- **US6**: As a developer, I want health checks and a stable API.
- **US7**: As a developer, I want a simple dev workflow (`pytest`, `vitest`, `ruff`, Docker/Fly`).

---

## Requirements Analysis

The system uses a FastAPI backend and React frontend with ML inference.

### Functional Requirements (FR)

| ID | Description | Acceptance Criteria | Done / Evidence |
|----|-------------|---------------------|-----------------|
| <a id="fr1"></a>**FR1** | `POST /api/dog-from-photo` accepts file and returns result. | JSON contains `breed`, `raw_predictions`, `advice`; or `error` for non-dog input. | Done. See `artifact/backend/Core/router.py` and `artifact/backend/tests/test_dog_from_photo.py`. |
| <a id="fr2"></a>**FR2** | `POST /api/dog-advice?breed=<name>` returns breed advice. | JSON contains `advice`. | Done. See `artifact/backend/Core/router.py` and `artifact/openapi/openapi.yaml`. |
| <a id="fr3"></a>**FR3** | Non-dog input gets a clear error. | JSON contains `error`: "Sorry, this is not a dog. Please try again". | Done. See `artifact/backend/tests/test_dog_from_photo.py` (`test_non_dog_returns_error`). |
| <a id="fr4"></a>**FR4** | Backend uses DogRecognition and LLM modules. | `backend/Features/DogRecognition/dog_recognition.py` and `backend/Features/LLM/llm_engine.py` are active. | Done. See `artifact/backend/Features/DogRecognition/dog_recognition.py`, `artifact/backend/Features/LLM/llm_engine.py`, and `artifact/backend/Core/router.py`. |
| <a id="fr5"></a>**FR5** | Health endpoint available. | `GET /` returns `{"status":"ok","message":"backend is running"}`. | Done. See `artifact/backend/main.py` and AT-5 in `sections/05-validation/index.md`. |
| <a id="fr6"></a>**FR6** | User can retry upload flows. | UI allows multiple upload attempts. | Done. See `artifact/web/src/hooks/useBreedIdentification.ts` and AT-4 in `sections/05-validation/index.md`. |

### Non-Functional Requirements (NFR)

| ID | Description | Acceptance Criteria | Done / Evidence |
|----|-------------|---------------------|-----------------|
| <a id="nfr1"></a>**NFR1** | Fast response times. | Response time is acceptable for interactive use under normal conditions, and the UI shows a clear loading state while analysis is running. | Done. Loading/interactive behavior is covered by UI flow and manual validation evidence in `sections/05-validation/index.md`; loading state is implemented in `artifact/web/src/hooks/useBreedIdentification.ts` and related UI components. |
| <a id="nfr2"></a>**NFR2** | Service availability and recoverability. | The service has automated deployment, a health endpoint, and documented restart/redeploy steps to recover from failures. | Done. Evidence is provided in `artifact/.github/workflows/backend-deploy.yml`, `artifact/backend/fly.toml`, `artifact/backend/main.py` (health endpoint), and `sections/07-deployment/index.md`. |
| <a id="nfr3"></a>**NFR3** | Security baseline for web API usage. | CORS is explicitly configured, API input is validated by FastAPI schemas, and sensitive runtime credentials are provided through environment variables. | Done. See `artifact/backend/main.py` (CORS), `artifact/backend/Core/router.py` (`UploadFile = File(...)` and request handling), and `artifact/backend/Features/LLM/llm_engine.py` (`HF_TOKEN` from env). |
| <a id="nfr4"></a>**NFR4** | Responsive and usable UI. | Main user flow (upload -> analyze -> result/error) works on desktop and mobile viewports, with visible loading and error states. | Done. See `artifact/web/src/pages/Index.tsx` (responsive classes), `artifact/web/src/hooks/useBreedIdentification.ts`, `artifact/web/src/components/ui/DogInfoPanel.tsx`, and manual acceptance tests in `sections/05-validation/index.md`. |
| **NFR5** | No user data persist. | No personal data stored; files are temporary. | Done. Data handling rationale is documented in `sections/03-design/index.md`; backend uses temporary files in `artifact/backend/Core/router.py`. |

### Implementation Requirements (IR)

| ID | Description | Justification | Acceptance Criteria | Done / Evidence |
|----|-------------|--------------|---------------------|-----------------|
| <a id="ir1"></a>**IR1** | Python 3 + FastAPI backend. | Existing codebase language and framework. | `python -m uvicorn backend.main:app` runs. | Done. See `artifact/backend/main.py` and `artifact/backend/README.md`. |
| <a id="ir2"></a>**IR2** | React + Vite + TypeScript frontend. | Existing web project stack. | `npm run build` passes. | Done. See `artifact/web/package.json` and `.github/workflows/web-ci.yml`. |
| <a id="ir3"></a>**IR3** | Test tooling `pytest`, `vitest`, `ruff`. | CI standard. | all tests pass. | Done. See `.github/workflows/backend-ci.yml` and `.github/workflows/web-ci.yml`. |
| <a id="ir4"></a>**IR4** | Docker and Fly deployment config exist. | Provided project files. | container builds and starts. | Done. See `artifact/backend/Dockerfile`, `artifact/backend/fly.toml`, `.github/workflows/backend-deploy.yml`. |
| <a id="ir5"></a>**IR5** | Env var configuration. | security in deployments. | settings driven via env. | Done. See `artifact/backend/Features/LLM/llm_engine.py` and `sections/07-deployment/index.md`. |

---

## Glossary

- **Breed**: predicted dog type from model.
- **raw_predictions**: top-3 model outputs (`label`, `score`).
- **advice**: text from `DogLLMEngine.generate_advice`.
- **inference**: model pipeline running on image.
- **health check**: `GET /` endpoint.
- **API**: FastAPI endpoints in `backend/main.py`.
- **rate-limit**: deployment request throttling.

---

## Acceptance Criteria

- [FR1](#fr1): `/api/dog-from-photo` returns breed/advice (or error response). 
- [FR2](#fr2): `/api/dog-advice` returns advice.
- [FR3](#fr3): non-dog returns `error` message.
- [NFR1](#nfr1): response time is acceptable for interactive use and loading state is visible during analysis.
- [NFR2](#nfr2): automated deploy, health endpoint, and documented recovery steps are in place.
- [NFR3](#nfr3): security baseline controls are implemented (CORS, validation, env-based credentials).
- [NFR4](#nfr4): responsive flow works with explicit loading/error states.
- [IR2](#ir2): `npm run build` passes.
- [IR3](#ir3): `pytest` + `vitest` + `ruff` pass.
