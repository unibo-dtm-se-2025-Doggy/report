---
title: Validation
has_children: false
nav_order: 6
---

# Validation

## Testing approach

We followed a **risk-based and incremental testing approach**:
- start from critical backend flow (`POST /api/dog-from-photo`) and verify success/error paths first;
- isolate external dependencies (ML/LLM) with test doubles to keep tests deterministic and fast;
- add regression tests when fixing bugs (e.g., non-dog input, breed-label normalization, prediction errors);
- keep a lightweight frontend smoke test to ensure the app renders correctly.

TDD was applied **partially** (test-first for selected fixes/regressions), but the project was not developed with strict end-to-end TDD from day one.

Testing frameworks used:
- **Backend**: `pytest` (+ `pytest-cov`) with `fastapi.testclient` and `monkeypatch`.
  Why: concise syntax, strong fixtures/patching support, and easy API-level testing without calling real external AI services.
- **Frontend**: `vitest` + `@testing-library/react`.
  Why: native fit for Vite/TypeScript stack, fast execution in CI, and good support for component-level smoke testing in `jsdom`.

## Performance and responsiveness (NFR1)

NFR1 is validated as an interactive UX requirement rather than a strict benchmark SLA:
- the frontend exposes an explicit loading state while analysis is running (`isAnalyzing`);
- the user flow remains responsive (upload -> loading -> result/error) in manual acceptance runs.

Evidence:
- implementation: `artifact/web/src/hooks/useBreedIdentification.ts`, `artifact/web/src/components/ui/DogInfoPanel.tsx`;
- acceptance flow: AT-1/AT-2/AT-4 in this section.

## Availability and recoverability (NFR2)

NFR2 is validated as operational readiness (not long-term uptime monitoring):
- deployment is automated through GitHub Actions on release tags;
- the backend exposes a health endpoint for quick service checks;
- deployment and recovery steps are documented for restart/redeploy.

Evidence:
- automation: `artifact/.github/workflows/backend-deploy.yml`;
- runtime config: `artifact/backend/fly.toml`;
- health check endpoint: `artifact/backend/main.py`;
- operational instructions: `sections/07-deployment/index.md`.

## Testing (automated)

### Unit testing

Unit testing focused on **isolated domain logic** (without real external AI calls), mainly in:
- `backend/Features/LLM/llm_engine.py`
- `backend/Features/DogRecognition/dog_recognition.py`

Rationale:
- validate core business logic in small, deterministic tests;
- avoid flaky behavior and network dependency by replacing external clients/models with stubs (`monkeypatch`).

Requirement traceability (unit level):
- `test_llm_engine_requires_token` -> [IR5](../02-requirements/index.md#ir5) (env-driven config), [FR4](../02-requirements/index.md#fr4) (LLM module is active and validated).
- `test_llm_engine_generate_advice` -> [FR2](../02-requirements/index.md#fr2) / [FR4](../02-requirements/index.md#fr4) (advice generation path and LLM integration contract).
- `test_is_dog_and_predict`, `test_zero_shot_not_dog` -> [FR3](../02-requirements/index.md#fr3) / [FR4](../02-requirements/index.md#fr4) (dog validation and prediction behavior in DogRecognition module).

Results (latest local run):
- **Success rate**: `4/4` passed (`100%`).
- **Coverage (unit scope)**: `100%` for targeted modules:
  - `backend/Features/LLM/llm_engine.py`
  - `backend/Features/DogRecognition/dog_recognition.py`

### Integration testing

Integration tests validated collaboration between API layer and domain services.

Main component couples tested:
- **FastAPI `/api/dog-from-photo` endpoint + `DogRecognitionModel` + `DogLLMEngine`** (`test_dog_from_photo.py`).
  Rationale: verify end-to-end orchestration for the main user flow ([FR1](../02-requirements/index.md#fr1), [FR3](../02-requirements/index.md#fr3), [FR4](../02-requirements/index.md#fr4)), including:
  - non-dog rejection path;
  - successful breed + advice response path;
  - breed-label normalization;
  - propagated error path on prediction failure.
- **Legacy router endpoint `/predict` + router-level model wiring** (`test_router.py`).
  Rationale: ensure router contract remains valid and upload/prediction integration works.

Results (latest local run):
- **Success rate**: `5/5` passed (`100%`).
- **Coverage (integration scope)**:
  - `backend/Core/router.py`: `100%`
  - `backend/main.py`: `64%`
  - combined for targeted modules: `70%`

Test doubles used:
- **Stubs/Fakes** (via `monkeypatch`) for dog recognition and LLM components.
  Why: keep tests deterministic, fast, and independent from external model/API availability while still validating integration contracts between components.

### System testing

Automated system testing was performed as a backend-level end-to-end suite that exercises the full request lifecycle from API entrypoint to final response payload (with controlled doubles for external AI dependencies).

System-level test plan:
- run complete backend test set (`test_dog_from_photo`, `test_router`, `test_llm_engine`, `test_dog_recognition`);
- verify acceptance-critical outcomes:
  - dog-photo flow returns `breed + raw_predictions + advice` ([FR1](../02-requirements/index.md#fr1));
  - non-dog flow returns clear error ([FR3](../02-requirements/index.md#fr3));
  - advice generation path is callable and validated ([FR2](../02-requirements/index.md#fr2), [FR4](../02-requirements/index.md#fr4));
  - module-level integrations remain active and coherent ([FR4](../02-requirements/index.md#fr4), [IR3](../02-requirements/index.md#ir3)).

Results (latest local run):
- **Success rate**: `9/9` passed (`100%`).
- **Coverage (system suite scope, targeted backend modules)**: `78%` total.
  - `backend/Core/router.py`: `100%`
  - `backend/Features/DogRecognition/dog_recognition.py`: `100%`
  - `backend/Features/LLM/llm_engine.py`: `100%`
  - `backend/main.py`: `64%`

Containers usage for testing:
- Docker/Fly configuration is available for deployment workflows, but the system tests reported here were executed in the local virtual environment (no Docker compose-based test environment was used for this run).

## Acceptance tests (manual)

Manual acceptance testing was executed on a local setup (`backend` + `web`) using browser-based user flows.

Preconditions:
- backend running and reachable;
- frontend running and configured with backend base URL;
- two sample images available: one dog image, one non-dog image.

Manual test plan and traceability:

| Test ID | Requirement(s) | Steps | Expected result | Outcome |
| --- | --- | --- | --- | --- |
| AT-1 | [FR1](../02-requirements/index.md#fr1) | Open app, upload a valid dog photo, submit analysis. | UI shows `breed`, `raw_predictions`, and `advice`. | Pass |
| AT-2 | [FR3](../02-requirements/index.md#fr3) | Upload a non-dog image and submit analysis. | UI shows clear error: `Sorry, this is not a dog. Please try again`. | Pass |
| AT-3 | [FR2](../02-requirements/index.md#fr2), [FR4](../02-requirements/index.md#fr4) | Run dog analysis and verify recommendation text quality/availability. | Advice text is present and tied to predicted breed. | Pass |
| AT-4 | [FR6](../02-requirements/index.md#fr6) | After any result/error, upload another image and re-run. | Retry flow works without page crash; new result replaces previous state. | Pass |
| AT-5 | [FR5](../02-requirements/index.md#fr5), **US6** | Open backend root endpoint (`GET /`) or docs endpoint while app is running. | Backend responds with health status and service is reachable. | Pass |

Manual acceptance success rate (latest run):
- **5/5 passed (`100%`)**.
