---
title: Development
has_children: false
nav_order: 5
---

# Development

## DVCS Conventions

The project uses Git on GitHub as a distributed version control system. Development follows a simple but structured workflow.

The main branch always contains stable code.

Commit messages follow the Conventional Commits style: `type(optional context): short description`. For example: `feat(api): add /api/dog-from-photo endpoint`. This makes it easy to understand which part of the project a change affects and also supports automatic versioning and changelog generation during the release process.

Issues are used to track bugs, enhancements, and planned improvements, creating a clear backlog of work and improving collaboration.

This lightweight workflow was chosen because it balances clarity and speed, making it efficient even for a small development team.

## Implementation Details

Since Doggy is a local, single-user desktop web app (browser-based), many distributed-system concerns are intentionally avoided to keep the project simple and fast:

- Network protocols: none are needed; interactions happen locally with backend API calls from frontend to backend on the same host.
- Data representation: exchanged data is JSON over HTTP.
- Databases: not used. Model input and response data are handled in memory.
- Authentication and authorization: not applicable, as there are no user accounts or online features.

Avoiding these elements keeps the system lightweight, fast, and easy to deploy.

## Technological Details

The project is developed in Python 3.12+ for the backend and TypeScript for frontend. The main dependencies are:

- Backend:
  - FastAPI: web API framework used for routing and HTTP handling.
  - transformers (Hugging Face): model pipelines (`zero-shot-image-classification`, `image-classification`).
  - Pillow: image IO.
- Frontend:
  - React + Vite + TypeScript: UI framework.
  - tailwindcss: styling.

All application logic and static assets are local. There are no external service dependencies except model weights fetched by transformers (can be pre-cached in production if needed).

This stack was selected because it ensures:
- Accessibility: Python/JS are widely supported.
- Performance: Frontend is fast and backend inference is localized.
- Portability: no external DB or auth needed.

The team follows PEP 8 for backend style consistency.
Using an IDE such as VS Code is recommended, as it provides automatic formatting, error checking, and integrated Git support.
For local developer workflow, VS Code terminal tasks (`.vscode/tasks.json`) were configured to simplify bootstrap and run commands for backend/frontend.
