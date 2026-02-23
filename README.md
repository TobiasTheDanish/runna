# Runna

Runna is a running session tracking application consisting of a Go backend API and a SvelteKit frontend, both containerized with Docker.

## Project Structure

```
/
├── backend/            (Go API)
└── frontend/           (SvelteKit app)
```

## Getting Started

### Prerequisites

- Docker and Docker Compose
- Turso database connection string

### Running the Application

1.  **Backend Setup**:
    -   Run `git clone https://github.com/TobiasTheDanish/runna-backend.git backend` to clone the backend project
    -   Navigate to `backend/`
    -   Copy `.env.example` to `.env` and fill in your Turso database credentials.
    -   Run `docker-compose up` to start the backend service.

2.  **Frontend Setup**:
    -   Run `git clone https://github.com/TobiasTheDanish/runna-frontend.git frontend` to clone the frontend project
    -   Navigate to `frontend/`
    -   Copy `.env-example` to `.env` and configure the backend API URL.
    -   Run `docker-compose up` (or `npm run dev` for local development) to start the frontend.

## Features

-   **Track Sessions**: Record your running sessions with distance, duration, and notes.
-   **Analyze Performance**: View your sessions, filter by date range, and see calculated pace.
-   **Edit Sessions**: Update details of your past runs.

## Technology Stack

-   **Backend**: Go (Golang), Turso (libSQL)
-   **Frontend**: SvelteKit, Tailwind CSS, shadcn-svelte
-   **Containerization**: Docker
