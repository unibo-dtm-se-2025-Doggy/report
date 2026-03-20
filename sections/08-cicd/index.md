---
title: CI/CD
has_children: false
nav_order: 9
---

# CI/CD

The project uses **GitHub Actions** for CI/CD with three workflows.

## Continuous Integration

### Backend CI (`.github/workflows/backend-ci.yml`)

Triggers on push and pull requests to `main` and `dev` branches when files inside `backend/` change.

**`lint-and-type-check` job:**

| Tool | What it checks |
|------|----------------|
| `ruff check` | Code style and common errors (unused imports, undefined variables, etc.) |
| `ruff format --check` | Consistent code formatting (indentation, spacing, line length) |
| `ruff check --select I` | Import sorting order |
| `mypy` | Static type checking — ensures type annotations are consistent and correct |

**`test` job:**

Runs `pytest` with coverage across a matrix of **3 operating systems × 3 Python versions** (9 combinations total):

- OS: Ubuntu, macOS, Windows
- Python: 3.10, 3.11, 3.12

`HF_TOKEN=dummy` is set as an environment variable so tests can run without a real HuggingFace token.

---

### Web CI (`.github/workflows/web-ci.yml`)

Triggers on pull requests to `dev`.

**`lint-and-type-check` job — runs ESLint:**

ESLint is a static analysis tool for JavaScript/TypeScript. It reads the source files and checks them against a set of rules without executing the code. In this project (`eslint.config.js`) it checks:

| Rule set | What it catches |
|----------|-----------------|
| `@eslint/js` recommended | Common JS errors: unused variables, unreachable code, etc. |
| `typescript-eslint` recommended | TypeScript-specific issues: incorrect types, unsafe `any` usage, unused variables |
| `eslint-plugin-react-hooks` | Violations of React Hooks rules (e.g. hooks inside conditions or loops) |
| `eslint-plugin-react-refresh` | Ensures components are exported correctly for hot reload to work |

If any rule is violated, the job fails and the PR cannot be merged.

**`test` job:**

Runs Vitest in CI mode (`npm run test:ci`). Vitest runs tests in a `jsdom` environment, which simulates a browser DOM without opening a real browser. This allows testing React components and UI logic in CI.

---

## Continuous Deployment

**Backend Deploy** (`.github/workflows/backend-deploy.yml`) triggers automatically when a git tag matching `backend/v*` is pushed, or manually via workflow dispatch. It deploys the backend to Fly.io using `flyctl deploy --remote-only`.

## Secrets and Environment Variables

| Name | Where | Purpose |
|------|-------|---------|
| `FLY_API_TOKEN` | GitHub Secret | Authenticates `flyctl` during deployment |
| `HF_TOKEN` | Fly.io Secret | HuggingFace API token for advice generation (set via `flyctl secrets set`) |
| `HF_TOKEN=dummy` | CI env var | Allows backend tests to run without a real token |
