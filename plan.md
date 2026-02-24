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

## Phase 2.5: Session Management Enhancements (Edit & Update)

### 2.5.1 Backend Expansion
- **GET /api/sessions/{id}** - Retrieve single session
  - Logic to fetch session by ID from database
  - Handle 404 if not found
- **PUT /api/sessions/{id}** - Update existing session
  - Request body: updated session data
  - Update database record
  - Update `updated_at` timestamp
  - Return updated session
- **Database Updates**
  - Implement `GetSession` and `UpdateSession` query methods

### 2.5.2 Frontend Edit Capability
- **API Client Updates**
  - Add `getSession(id)`
  - Add `updateSession(id, data)`
- **Edit Session Page** (`src/routes/edit/[id]/+page.svelte`)
  - Reuse form layout from Create page, by extracting form layout from Create page into separate .svelte file
  - Fetch session data on mount to pre-fill inputs
  - Handle submission as update
- **Overview Integration**
  - Add "Edit" action/button to session list table (routes to edit page)

## Phase 3: Goal Tracking

### 3.1 Backend Implementation

#### Database Schema
- **New Table**: `goals`
  - `id` (INTEGER PRIMARY KEY AUTOINCREMENT)
  - `target_distance` (REAL NOT NULL)
  - `start_date` (DATETIME NOT NULL)
  - `end_date` (DATETIME NOT NULL)
  - `created_at` (DATETIME DEFAULT CURRENT_TIMESTAMP)
  - `updated_at` (DATETIME DEFAULT CURRENT_TIMESTAMP)

#### Models (`internal/models/goal.go`)
- Create `Goal` struct mirroring the database table.
- Create `CreateGoalRequest` struct for API input.
- Create `GoalProgress` struct for response, including:
  - `Goal` details
  - `CurrentDistance` (sum of relevant session distances)
  - `TargetDistance`
  - `ProgressPercentage`
  - `Status` (e.g., "On Track", "Behind", "Ahead")
  - `Sessions` ([]Session) - List of sessions contributing to this goal

#### Database Logic (`internal/database/database.go`)
- Update `Init()` to create the `goals` table.
- Implement `CreateGoal(req models.CreateGoalRequest) (*models.Goal, error)`
- Implement `GetGoals() ([]models.Goal, error)`
- Implement `DeleteGoal(id int) error`
- Implement `GetGoal(id int) (*models.GoalProgress, error)`
  - This method will fetch the goal details.
  - Query sessions where `date >= goal.start_date` and `date <= goal.end_date`.
  - Calculate progress metrics.
  - Return the goal info, metrics, and the list of sessions.

#### Handlers (`internal/handlers/handlers.go` & `goal_handlers.go`)
- `CreateGoal`: POST `/api/goals`
- `GetGoals`: GET `/api/goals` (Can return list of goals with their progress)
- `GetGoal`: GET `/api/goals/{id}` (Returns single goal details + contributing sessions)
- `DeleteGoal`: DELETE `/api/goals/{id}`

#### Routing (`cmd/api/main.go`)
- Register the new routes.

### 3.2 Frontend Implementation

#### API Client (`src/lib/api/client.ts`)
- Add methods: `createGoal`, `getGoals`, `getGoal(id)`, `deleteGoal`.

#### UI Components
- **Goal Card Component**: Displays a single goal's progress.
  - Visual indicator (e.g., progress bar).
  - "On Track" status badge.
- **Create Goal Form**:
  - Inputs for Target Distance, Start Date, End Date.

#### Pages
- **Goals Dashboard (`src/routes/goals/+page.svelte`)**:
  - Lists existing goals.
  - Shows a "Create New Goal" button.
- **Create Goal Page (`src/routes/goals/create/+page.svelte`)**:
  - Hosts the Create Goal Form.
- **Goal Details Page (`src/routes/goals/[id]/+page.svelte`)**:
  - Displays the specific goal's progress card.
  - **Sessions List**: A table/list view of all sessions that fall within this goal's time period (contributing to the total distance).
  - Delete goal button.

### 3.3 "On Track" Logic
- `Expected Distance` = `Target Distance` * (`Days Elapsed` / `Total Duration in Days`)
- `Status`:
  - If `Current Distance` >= `Expected Distance`: "On Track"
  - If `Current Distance` < `Expected Distance`: "Behind"

## Phase 4: Integration & Testing

### 4.1 Combined Docker Compose
- Volume mounts for development
- Environment variable management

### 4.2 Testing
- Backend:
  - Test database connection
- Frontend:
  - Test form submission
  - Test data retrieval and display
  - Test error scenarios
- Integration:
  - End-to-end flow: create session and verify it appears in overview

### 4.3 Documentation
- README for each repository with:
  - Setup instructions
  - Environment variable requirements
  - Docker commands to run
  - API documentation (for backend)
- Development workflow documentation

## Phase 5: Deployment Preparation

### 5.1 Environment Configuration
- Document all required environment variables
- Create .env.example files

### 5.2 Production Optimizations
- Backend:
  - Production logging configuration
  - Rate limiting
  - Security headers
- Frontend:
  - SvelteKit adapter configuration for deployment target
  - Build optimization
  - Static asset optimization

## Success Criteria
- ✅ Backend API running in Docker with functional endpoints
- ✅ Frontend running in Docker with create, overview, and goal pages
- ✅ Successful creation of sessions via UI
- ✅ Sessions display correctly filtered by date range
- ✅ Both services can be started with docker-compose
- ✅ Data persists in Turso database
- ✅ Proper error handling and validation
- ✅ Users can create distance-based goals
- ✅ Users can view goal progress, "on track" status, and related sessions
