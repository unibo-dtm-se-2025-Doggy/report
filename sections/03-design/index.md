---
title: Design
has_children: false
nav_order: 4
---

# Design

This chapter explains the strategies used to meet the requirements identified in the analysis. 
## Architecture 

### Architectural style

We adopt a **layered client-server architecture**.

Why this style:
- clear separation of concerns between UI, API orchestration, and AI logic
- straightforward local and cloud deployment
- good testability at both API and component level
- independent frontend/backend evolution

Why not alternatives (for current scope):
- not **event-based**: the workflow is synchronous request/response, with no queue or event stream
- not **shared-dataspace**: there is no shared persistent data core that coordinates components
- not **microservice/object-based distributed decomposition**: current project size does not justify added operational complexity
- not **hexagonal architecture**: for this project scope, a layered client-server design provides sufficient separation of concerns with lower implementation overhead

### Chosen structure

We use a **3-tier N-tier structure**:
- **Tier 1 (Presentation)**: React + Vite web client for user interaction, image upload, and result rendering
- **Tier 2 (Application/API)**: FastAPI backend for input validation, orchestration, and error handling
- **Tier 3 (AI Services)**: dog recognition model and LLM advice engine invoked by the backend

This structure keeps the runtime simple while preserving clear boundaries for future extensions.

### High-level overview

The user interacts with the web client, which sends HTTP requests to the backend.
The backend coordinates dog recognition and advice generation, then returns one aggregated response (`breed + advice + raw_predictions`) to the client.

![UML Component Diagram](../../pictures/architecture-components.png)

### Component responsibilities

- **User**: provides input image and consumes results
- **Web Client (React + Vite)**: collects user input, sends API requests, renders results and errors
- **Backend API (FastAPI)**: exposes endpoints, validates requests, orchestrates analysis flow, aggregates final response
- **Dog Recognition Module (in backend runtime)**: checks if the image contains a dog and predicts breed labels
- **LLM Advice Service (external API via backend)**: generates textual recommendations for the predicted breed


## Infrastructure

Doggy uses a **client-server** setup.

### Used infrastructure components

- **Clients (web browsers) - Frontend side**  
  Interaction: user works in browser; browser loads UI and sends API requests.

- **Servers - Frontend and Backend**  
  Interaction: Vite serves frontend locally in development; FastAPI receives `/api/*` requests and calls external AI APIs.

- **External AI service (LLM provider API) - Backend outbound dependency**  
  Interaction: FastAPI sends HTTPS inference requests and receives generated recommendations.

- **Load balancer / ingress (Fly.io edge) - Backend entry point**  
  Interaction: receives public HTTPS traffic and routes requests to the FastAPI app instance.

- **Proxy (Fly.io edge reverse proxy) - Between Frontend and Backend**  
  Interaction: forwards request/response traffic between browser and FastAPI.

- **Firewalls (platform-managed) - Backend perimeter**  
  Interaction: Fly.io network controls filter internet traffic before it reaches the app service.

- **DNS - Name resolution service**  
  Interaction: browser resolves public hostnames before sending requests.

- **Backend endpoint configuration - Frontend to Backend binding**  
  Interaction: frontend calls backend URL from `VITE_API_BASE_URL` (development) or public URL (production).

### Network interaction process

1. The user uploads an image in the browser.
2. The browser resolves the API host via DNS.
3. DNS returns the target endpoint.
4. The browser sends an HTTPS request to the platform firewall boundary.
5. The firewall layer allows and passes traffic to Fly.io edge.
6. Fly.io edge proxies the request to the FastAPI backend.
7. FastAPI invokes the in-process dog-recognition module to verify that the image contains a dog.
8. The dog-recognition module returns the validation result.
9. FastAPI invokes the same module for breed prediction.
10. The dog-recognition module returns breed probabilities (`raw_predictions`).
11. FastAPI sends breed context to the external LLM provider API.
12. LLM provider API returns generated recommendations.
13. FastAPI builds the final JSON response (`breed`, `advice`, `raw_predictions`).
14. Fly.io edge forwards the response back through the platform boundary.
15. The browser receives and renders the final result.

Development note: in local mode, the frontend uses `VITE_API_BASE_URL` to call the backend directly (without the public production ingress path).

### Request flow (UML Sequence)

![UML Infrastructure Sequence Diagram](../../pictures/architecture-infra-sequence.png)

## Modelling

### Domain driven design (DDD) modelling

#### Bounded contexts and domain concepts

- **User Interaction Context (Frontend)**: image upload, request trigger, result/error rendering.  
  - value objects `SelectedImage`, `RequestState`, `ApiErrorMessage`
  - service `useBreedIdentification`
  - relevant events `ImageUploaded`, `AnalysisRequested`, `ResultDisplayed`, `ErrorDisplayed`

- **Analysis Orchestration Context (Backend API)**: request lifecycle orchestration and response aggregation.  
  - ephemeral aggregate `PhotoAnalysis`
  - value objects `BreedLabel`, `RawPrediction`, `AdviceText`, `AnalysisResult`
  - application service `/api/dog-from-photo`
  - factories/initialization `_get_dog_model()`, `_get_llm()`
  - repositories: not used (no persistent domain storage)
  - relevant events `PhotoReceived`, `DogCheckPassed` / `DogCheckFailed`, `BreedPredicted`, `AdviceRequested`, `AdviceReceived`, `AnalysisCompleted`

- **Dog Recognition Context (ML module)**: dog/non-dog validation and breed prediction.  
  - value objects `DogValidationResult`, `RawPrediction`
  - service `DogRecognitionModel` (`is_dog()`, `predict()`)
  - relevant events `DogValidated`, `BreedPredictionsComputed`

- **Advice Generation Context (LLM integration)**: breed-specific recommendation generation.  
  - value objects `AdvicePrompt`, `AdviceText`
  - service `DogLLMEngine.generate_advice()`
  - relevant events `AdviceGenerationStarted`, `AdviceGenerationCompleted`, `AdviceGenerationFailed`

Repositories/factories note: repositories are not used in the current scope (no persistent domain storage), and no dedicated domain factories are required beyond service initialization (`_get_dog_model()`, `_get_llm()`).

![DDD Context Map Diagram](../../pictures/ddd-context-map.png)

### Object-oriented modelling

#### Main data types, attributes, and methods

**Frontend context**

| Type | Kind | Responsibility | Main attributes | Main methods |
| --- | --- | --- | --- | --- |
| `UseBreedIdentification` | service | Orchestrates the UI-side analysis lifecycle | `isAnalyzing`, `result`, `error`, `requestState` | `identifyBreed(file)`, `reset()` |
| `DogApiClient` | service | Sends HTTP requests to backend and parses API errors | `baseUrl` | `postDogFromPhoto(file)`, `parseError(payload)` |
| `SelectedImage` | value object | Encapsulates uploaded image metadata | `name`, `mimeType`, `sizeBytes` | `isValidImage()` |
| `RequestState` | value object | Tracks the current request phase in UI | `phase`, `message` | `isTerminal()` |
| `ApiErrorMessage` | value object | Stores normalized user-facing error text | `message` | `toUserMessage()` |

**Backend orchestration context**

| Type | Kind | Responsibility | Main attributes | Main methods |
| --- | --- | --- | --- | --- |
| `PhotoAnalysis` | aggregate | Orchestrates per-request backend workflow and result assembly | `imagePath`, `status`, `breed`, `rawPredictions`, `advice` | `runDogCheck()`, `runBreedPrediction()`, `runAdviceGeneration()`, `buildResponse()` |
| `BreedLabel` | value object | Stores normalized top breed label | `value` | `normalize(rawLabel)` |
| `RawPrediction` | value object | Represents one classifier prediction item | `label`, `score` | `isTop(minScore)` |
| `AdviceText` | value object | Stores generated recommendation text | `value` | `isEmpty()` |
| `AnalysisResult` | value object | Holds internal merged analysis output | `breed`, `advice`, `rawPredictions` | `toViewModel()` |
| `DogFromPhotoResponse` | DTO | Defines public success response schema | `breed`, `advice`, `raw_predictions` | `toJson()` |
| `ApiErrorResponse` | DTO | Defines public error response schema | `error`, `code` | `toJson()` |

**Dog recognition context**

| Type | Kind | Responsibility | Main attributes | Main methods |
| --- | --- | --- | --- | --- |
| `DogRecognitionModel` | domain service | Performs dog validation and breed classification | `zeroShotModel`, `breedClassifier` | `is_dog(imagePath)`, `predict(imagePath)` |
| `DogValidationResult` | value object | Encodes dog/not-dog decision and reason | `isDog`, `reason` | `passed()` |

**Advice generation context**

| Type | Kind | Responsibility | Main attributes | Main methods |
| --- | --- | --- | --- | --- |
| `DogLLMEngine` | domain service | Calls LLM provider and returns advice text | `client`, `model_name` | `generate_advice(breed)` |
| `AdvicePrompt` | value object | Builds prompt payload for advice generation | `breed`, `template` | `render()` |

**Domain events**

| Type | Kind | Responsibility | Main attributes | Main methods |
| --- | --- | --- | --- | --- |
| `DomainEvent` | abstract event | Provides common event metadata contract | `eventId`, `occurredAt` | `name()` |
| `PhotoReceived` | event | Captures beginning of backend analysis lifecycle | `fileName`, `imagePath` | `name()` |
| `DogValidated` | event | Captures outcome of dog/non-dog verification | `isDog`, `reason` | `name()` |
| `BreedPredictionsComputed` | event | Captures completion of breed prediction stage | `topBreed`, `count` | `name()` |
| `AdviceGenerationCompleted` | event | Captures completion of LLM advice generation | `adviceLength` | `name()` |
| `AnalysisCompleted` | event | Captures final request completion state | `success`, `responseSize` | `name()` |

#### Relationships between data types

- `UseBreedIdentification` sends request and receives `DogFromPhotoResponse` / `ApiErrorResponse`.
- `PhotoAnalysis` composes `BreedLabel`, `RawPrediction`, `AdviceText`, and produces `AnalysisResult`.
- `PhotoAnalysis` depends on `DogRecognitionModel` and `DogLLMEngine`.
- `DogRecognitionModel` creates `DogValidationResult` and `RawPrediction`.
- `DogLLMEngine` consumes `AdvicePrompt` and returns `AdviceText`.
- `PhotoAnalysis` lifecycle is reflected by domain events (`PhotoReceived` -> `AnalysisCompleted`).

![UML Class Diagram](../../pictures/oop-class-diagram.png)

## Interaction

All component communication is **on-demand** — triggered only when the user submits an image. There is no background polling, WebSocket connection, or event bus.

### Communication channels and data exchanged

| From | To | When | Protocol | What |
|------|----|------|----------|------|
| Web Client | Backend API | User submits image | HTTP POST `/api/dog-from-photo` | `multipart/form-data` with image file |
| Backend API | Web Client | Analysis complete | HTTP response (JSON) | `{breed, advice, raw_predictions}` or `{error}` |
| Backend API | `DogRecognitionModel` | On every photo request | In-process method call | image file path |
| `DogRecognitionModel` | Backend API | After dog check | Return value | `bool` (is dog or not) |
| `DogRecognitionModel` | Backend API | After breed prediction | Return value | list of `{label, score}` (top 3) |
| Backend API | `DogLLMEngine` | After breed is known | In-process method call | breed name string |
| `DogLLMEngine` | HuggingFace Inference API | On advice request | HTTPS (chat completion) | prompt with breed name, model config |
| HuggingFace Inference API | `DogLLMEngine` | Response | HTTPS | generated advice text |

### Interaction patterns

**Frontend ↔ Backend: Request/Response over HTTP**

The frontend sends a single `POST /api/dog-from-photo` request with the image as `multipart/form-data`. It then waits synchronously for the backend to complete all processing steps and return one aggregated JSON response. The frontend shows a loading state during this wait.

On error (not a dog, model failure, invalid input), the backend returns `{"error": "..."}` and the frontend displays the message.

**Backend ↔ ML models: Synchronous in-process calls**

`DogRecognitionModel` and `DogLLMEngine` are instantiated lazily on the first request and reused across all requests. The backend calls their methods directly — no serialization, no network. Execution is sequential:

1. `model_instance.is_dog(temp_path)` — runs CLIP zero-shot classification
2. `model_instance.predict(temp_path)` — runs ViT breed classification (only if step 1 passes)
3. `llm_instance.generate_advice(top_breed)` — called after step 2

**Backend → LLM: Remote Procedure Call (HTTPS)**

`DogLLMEngine` calls the HuggingFace Inference API via `InferenceClient.chat_completion()`. This is the only outbound network call in the request flow. The call is synchronous — the backend awaits the response before returning to the frontend.

### Sequence diagram

![UML Interaction Sequence Diagram](../../pictures/interaction-sequence.png)

## Behaviour

### Component behaviour overview

Each component is stateless with respect to individual requests. The only stateful element is the Web Client's local UI state, scoped to a single user session.

| Component | Stateful? | Behaviour |
|---|---|---|
| Web Client (React) | **Stateful** (UI) | Maintains local React state: `isAnalyzing`, `result`, `error`. Reacts to user events (file select, submit) and API responses. Resets state on each new request. |
| Backend API (FastAPI) | **Stateless** | Each HTTP request is independent. Lazy-initialises `DogRecognitionModel` and `DogLLMEngine` on first call; subsequent requests reuse the cached instances. Orchestrates the full pipeline and returns a single aggregated JSON response. |
| DogRecognitionModel | **Stateless** (models cached) | Instantiated once on first use. On each call: opens image with Pillow, runs CLIP zero-shot check (`is_dog`), then ViT breed classification (`predict`). Returns raw results; holds no per-request state. |
| DogLLMEngine | **Stateless** | Instantiated once on first use. On each call sends a single HTTPS `chat_completion` request to the HuggingFace Inference API and returns the text response. |

### State updates

Two components are responsible for state changes:

**Web Client** — updates UI state in response to user actions and API outcomes:
- on submit: sets `isAnalyzing = true`, clears previous `result` and `error`
- on response received: populates `result` or `error`, resets `isAnalyzing = false`

**Backend API** — owns one implicit shared state: the singleton model instances (`dog_model`, `llm`) held in module-level globals. Initialised lazily on the first incoming request and reused for all subsequent ones. Never mutated after initialisation.

### State diagram

The following state diagram complements the activity diagram by focusing only on the **UI/request states** visible in the frontend lifecycle: idle, analyzing, success, and error.

![UML State Diagram](../../pictures/state-diagram.png)

### Activity diagram

The diagram below shows the full request flow from image upload to rendered result, covering lazy initialisation and the dog-check decision branch.

![Activity Diagram – POST /api/dog-from-photo](../../pictures/behaviour-activity.png)

## Data-related aspects (in case persistent storage is needed)

Doggy does not use persistent storage. All data is handled in-memory and discarded after each response. There is no database, no file store, and no session persistence.

### Data-flow diagram

The following data-flow diagram complements the sequence diagram by focusing on the **data artifacts** exchanged across frontend, backend, model services, and the external inference API.

![Data-flow Diagram](../../pictures/data-flow-diagram.png)

### No data needs to be stored

All information exchanged during a request — the uploaded image, breed predictions, and generated advice — exists only for the duration of that request:
- **uploaded image**: saved to a temporary file (`temp_<uuid>.jpg`) during processing, not persisted
- **breed predictions**: computed in-memory, returned in the response, discarded afterwards
- **advice text**: generated per request, returned in the response, not cached

Why no storage: the application is a single-user local tool with no history, accounts, or audit requirements. Statefulness would add complexity with no benefit in the current scope.

### No database queries

No component performs database queries. There is no read or write access to any external or embedded data store. Concurrent read/write concerns do not apply.

### Shared data between components

The only shared data is the **singleton model instances** held in backend module-level globals:
- `dog_model` — shared instance of `DogRecognitionModel`, used by every incoming request
- `llm` — shared instance of `DogLLMEngine`, used by every incoming request

Why shared: loading ML models on every request would be prohibitively slow. Both instances are read-only after initialisation — they hold no mutable per-request state, so sharing them is safe under concurrent load.
