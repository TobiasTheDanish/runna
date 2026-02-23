# Implementation Plan for Runna

## Overview
Runna is a running session tracking application consisting of a Go backend API and a SvelteKit frontend, both containerized with Docker.

## Project Structure
```
/Users/thc/dev/runna/
├── requirements.md      (project requirements)
├── plan.md             (this file)
├── backend/            (Go API - separate git repo)
└── frontend/           (SvelteKit app - separate git repo)
```

## Phase 1: Backend API (Go)

### 1.1 Project Setup
- Initialize new Git repository in `backend/` directory
- Set up Go module with necessary dependencies:
  - HTTP router (e.g., chi, gin, or standard library)
  - Turso/libSQL driver
  - Environment variable management
- Create project structure:
  ```
  backend/
  ├── cmd/
  │   └── api/
  │       └── main.go
  ├── internal/
  │   ├── handlers/
  │   ├── models/
  │   ├── database/
  │   └── middleware/
  ├── Dockerfile
  ├── docker-compose.yml
  ├── go.mod
  └── .env.example
  ```

### 1.2 Database Layer
- Set up Turso/libSQL connection using environment variable
- Define Session model/schema:
  - ID (primary key)
  - Date/timestamp
  - Distance
  - Duration
  - Pace (calculated or stored)
  - Notes (optional)
  - Created_at/Updated_at
- Create database initialization and migration logic
- Implement database access layer (repository pattern)

### 1.3 API Endpoints
- **POST /api/sessions** - Create new session
  - Request body: session data
  - Response: created session with ID
  - Validation for required fields
- **GET /api/sessions** - Get sessions within time period
  - Query parameters: `start_date`, `end_date`
  - Response: array of sessions
  - Default to current month if no dates provided

### 1.4 Docker Setup
- Create Dockerfile:
  - Multi-stage build (builder + runtime)
  - Expose API port (e.g., 8080)
- Create docker-compose.yml:
  - API service configuration
  - Environment variables for Turso DB URL
  - Port mapping
  - Health checks

### 1.5 Additional Considerations
- CORS configuration for frontend access
- Error handling and logging
- Input validation
- Environment variable validation on startup

## Phase 2: Frontend (SvelteKit)

### 2.1 Project Setup
- Initialize SvelteKit project in `frontend/` directory
- Initialize new Git repository
- Install necessary dependencies:
  - Date handling library (e.g., date-fns)
  - HTTP client (fetch API)
- Create project structure:
  ```
  frontend/
  ├── src/
  │   ├── routes/
  │   │   ├── +page.svelte       (overview)
  │   │   └── create/
  │   │       └── +page.svelte   (create session)
  │   ├── lib/
  │   │   ├── components/
  │   │   ├── api/
  │   │   └── utils/
  │   └── app.html
  ├── static/
  ├── Dockerfile
  ├── docker-compose.yml
  ├── svelte.config.js
  └── package.json
  ```

### 2.2 API Integration Layer
- Create API client module for backend communication
- Configure API base URL via environment variable
- Implement functions:
  - `createSession(data)` - POST to /api/sessions
  - `getSessions(startDate, endDate)` - GET from /api/sessions

### 2.3 UI Components

- Use shadcn-svelte for all ui components

#### Create Session Page
- Form with fields:
  - Date picker
  - Distance input (with unit selector: km/miles)
  - Duration input (hours:minutes:seconds)
  - Notes textarea (optional)
- Form validation
- Submit handler to call API
- Success/error feedback
- Redirect or clear form after successful creation

#### Sessions Overview Page
- Date range selector:
  - Start date and end date pickers
  - Quick filters (this week, this month, last 30 days)
- Sessions list/table display:
  - Date
  - Distance
  - Duration
  - Pace (calculated)
  - Notes preview
- Loading states
- Empty state when no sessions found
- Summary statistics (total distance, total time, average pace)

### 2.4 Docker Setup
- Create Dockerfile:
  - Build step for SvelteKit
  - Node.js adapter configuration
  - Expose port (e.g., 3000)
- Create docker-compose.yml:
  - Frontend service configuration
  - Environment variable for backend API URL
  - Port mapping
  - Depends on backend service

### 2.5 Additional Considerations
- Responsive design for mobile use
- Form validation and error handling
- Loading indicators
- Date/time formatting
- Unit conversion helpers

## Phase 3: Integration & Testing

### 3.1 Combined Docker Compose
- Create root-level docker-compose.yml that orchestrates both services
- Network configuration for inter-service communication
- Volume mounts for development
- Environment variable management

### 3.2 Testing
- Backend:
  - Test database connection
  - Test API endpoints with curl/Postman
  - Validate CORS settings
- Frontend:
  - Test form submission
  - Test data retrieval and display
  - Test error scenarios
- Integration:
  - End-to-end flow: create session and verify it appears in overview

### 3.3 Documentation
- README for each repository with:
  - Setup instructions
  - Environment variable requirements
  - Docker commands to run
  - API documentation (for backend)
- Development workflow documentation

## Phase 4: Deployment Preparation

### 4.1 Environment Configuration
- Document all required environment variables
- Create .env.example files
- Set up Turso database and obtain connection URL

### 4.2 Production Optimizations
- Backend:
  - Production logging configuration
  - Rate limiting
  - Security headers
- Frontend:
  - SvelteKit adapter configuration for deployment target
  - Build optimization
  - Static asset optimization

## Success Criteria
- ✅ Backend API running in Docker with two functional endpoints
- ✅ Frontend running in Docker with two pages (create, overview)
- ✅ Successful creation of sessions via UI
- ✅ Sessions display correctly filtered by date range
- ✅ Both services can be started with docker-compose
- ✅ Data persists in Turso database
- ✅ Proper error handling and validation
