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

## Phase 4: Strava Webhook Integration

### 4.1 Overview
Implement Strava webhook integration to automatically sync running activities from Strava to Runna. When a user creates, updates, or deletes an activity in Strava, our system will be notified via webhook and can process the activity accordingly.

### 4.2 Backend Implementation

#### Database Schema Updates
- **New Table**: `strava_connections`
  - `id` (INTEGER PRIMARY KEY AUTOINCREMENT)
  - `user_id` (INTEGER) - Future-proofing for multi-user support
  - `strava_athlete_id` (INTEGER NOT NULL UNIQUE)
  - `access_token` (TEXT NOT NULL) - Encrypted token for API calls
  - `refresh_token` (TEXT NOT NULL) - Encrypted token for token refresh
  - `token_expires_at` (DATETIME NOT NULL)
  - `connected_at` (DATETIME DEFAULT CURRENT_TIMESTAMP)
  - `last_sync` (DATETIME)
  
- **Update `sessions` Table**:
  - Add `strava_activity_id` (INTEGER, nullable, unique)
  - Add `source` (TEXT DEFAULT 'manual') - Values: 'manual', 'strava'
  - This allows tracking which sessions came from Strava vs manual entry

#### Webhook Subscription Management
- **Models** (`internal/models/webhook.go`):
  - `WebhookEvent` struct matching Strava's event payload:
    - `aspect_type` (string): "create", "update", or "delete"
    - `event_time` (int64)
    - `object_id` (int64): Activity ID
    - `object_type` (string): "activity" or "athlete"
    - `owner_id` (int64): Athlete ID
    - `subscription_id` (int)
    - `updates` (map[string]interface{}): Changed fields

- **Webhook Endpoints** (`internal/handlers/webhook_handlers.go`):
  - **GET /api/webhooks/strava** - Subscription verification endpoint
    - Parse query params: `hub.mode`, `hub.verify_token`, `hub.challenge`
    - Verify token matches configured `STRAVA_VERIFY_TOKEN`
    - Return JSON: `{"hub.challenge": "<challenge_value>"}`
    - Must respond within 2 seconds
  
  - **POST /api/webhooks/strava** - Event receiver endpoint
    - Parse webhook event from request body
    - Validate event structure
    - Queue event for async processing (to respond within 2 seconds)
    - Return 200 OK immediately
    
- **Event Processing Logic** (`internal/services/strava_service.go`):
  - **ProcessActivityCreated**:
    1. Look up athlete's tokens from `strava_connections` using `owner_id`
    2. Check if token needs refresh (if expired, use refresh token to get new access token)
    3. Fetch detailed activity data from Strava API: `GET /api/v3/activities/{id}`
    4. Filter for running activities only (type = "Run")
    5. Extract relevant fields:
       - `start_date` → session date
       - `distance` (meters) → convert to km
       - `moving_time` (seconds) → duration
       - `name` → notes (optional)
    6. Create session in database with `strava_activity_id` and `source='strava'`
    7. Handle duplicates (check if `strava_activity_id` already exists)
  
  - **ProcessActivityUpdated**:
    1. Look up athlete's tokens
    2. Refresh token if needed
    3. Fetch updated activity data from Strava API
    4. Find existing session by `strava_activity_id`
    5. Update session fields based on `updates` map:
       - Title changed → update notes
       - Type changed → if no longer "Run", consider deleting session
       - Privacy changed to "true" (Only You) → handle per privacy requirements
    6. Update session in database
  
  - **ProcessActivityDeleted**:
    1. Find session by `strava_activity_id`
    2. Delete session from database
    3. Handle case where session doesn't exist (no-op)
  
  - **ProcessAthleteDeauthorized**:
    1. Look up athlete's connection by `owner_id`
    2. Delete from `strava_connections` table
    3. Optionally: Delete or mark all associated sessions as orphaned

- **Token Refresh Logic** (`internal/services/strava_auth.go`):
  - Check if `token_expires_at` is in the past
  - POST to `https://www.strava.com/oauth/token`:
    - `client_id`, `client_secret`, `grant_type=refresh_token`, `refresh_token`
  - Update `strava_connections` with new tokens and expiry
  - Return fresh access token

- **Strava API Client** (`internal/services/strava_client.go`):
  - `GetActivity(accessToken, activityID)` - Fetch activity details
  - Handle rate limiting (Strava: 100 requests per 15 min, 1000 per day)
  - Error handling for 401 (token issues), 404 (not found), etc.

#### Configuration & Environment Variables
Add to `.env` and `.env.example`:
- `STRAVA_CLIENT_ID` - From Strava API application
- `STRAVA_CLIENT_SECRET` - From Strava API application
- `STRAVA_VERIFY_TOKEN` - Random string for webhook validation (e.g., "RUNNA_STRAVA_WEBHOOK")
- `STRAVA_WEBHOOK_CALLBACK_URL` - Public URL for webhook endpoint (e.g., `https://api.runna.com/api/webhooks/strava`)

#### Async Processing
- Implement a simple job queue or background worker:
  - In-memory channel-based queue (for MVP)
  - Worker goroutines to process events from queue
  - Graceful shutdown handling
- Alternatively: Use a lightweight job queue library (e.g., `asynq` with Redis, but adds infrastructure complexity)

### 4.3 Frontend Implementation

#### OAuth Flow for Strava Connection
- **New Page**: `src/routes/strava/connect/+page.svelte`
  - Display "Connect with Strava" button
  - On click, redirect to Strava authorization URL:
    ```
    https://www.strava.com/oauth/authorize?
      client_id=<CLIENT_ID>&
      redirect_uri=<REDIRECT_URI>&
      response_type=code&
      scope=activity:read
    ```
  - Note: Request `activity:read` scope (or `activity:read_all` for private activities)

- **New Page**: `src/routes/strava/callback/+page.svelte`
  - Receives OAuth callback with `code` parameter
  - Sends code to backend endpoint: `POST /api/strava/connect`
  - Backend exchanges code for tokens and stores in database
  - Redirect to dashboard with success message

#### Backend OAuth Endpoints
- **POST /api/strava/connect** (`internal/handlers/strava_handlers.go`):
  - Receive authorization `code` from frontend
  - Exchange code for tokens via Strava API:
    - POST to `https://www.strava.com/oauth/token`
    - Body: `client_id`, `client_secret`, `code`, `grant_type=authorization_code`
  - Response includes: `access_token`, `refresh_token`, `expires_at`, `athlete` object
  - Store in `strava_connections` table (encrypt tokens)
  - Return success response

- **GET /api/strava/status** (`internal/handlers/strava_handlers.go`):
  - Check if user has active Strava connection
  - Return connection status and athlete info
  - Used by frontend to show connection status

- **DELETE /api/strava/disconnect** (`internal/handlers/strava_handlers.go`):
  - Remove Strava connection from database
  - Optionally: handle sessions created from Strava (mark as orphaned or delete)

#### UI Updates
- **Settings/Integrations Page** (`src/routes/settings/integrations/+page.svelte`):
  - Show Strava connection status
  - "Connect with Strava" button (if not connected)
  - "Disconnect" button (if connected)
  - Display last sync time
  - Show athlete name/profile from Strava

- **Sessions Overview Updates**:
  - Add visual indicator (icon/badge) for Strava-synced sessions
  - Filter option to show only manual or only Strava sessions
  - Prevent editing Strava sessions (or allow with warning)

### 4.4 Security Considerations
- **Token Storage**:
  - Encrypt `access_token` and `refresh_token` before storing in database
  - Use environment variable for encryption key: `TOKEN_ENCRYPTION_KEY`
  - Consider using `crypto/aes` with GCM mode for encryption

- **Webhook Validation**:
  - Verify `verify_token` matches expected value
  - Validate event structure to prevent malicious payloads
  - Rate limit webhook endpoint to prevent abuse

- **API Scopes**:
  - Request minimal required scopes (`activity:read` for public activities)
  - Respect activity privacy (handle private activities appropriately)
  - Follow Strava API Agreement regarding data usage

### 4.5 Error Handling & Edge Cases
- **Token Expiry**: Auto-refresh tokens before API calls
- **Rate Limiting**: Implement exponential backoff for Strava API calls
- **Duplicate Events**: Strava may send duplicate events; use idempotent processing
- **Partial Failures**: Log errors but don't block other events from processing
- **Activity Type Filtering**: Only process "Run" activities; ignore others
- **Multiple Updates**: Strava may send multiple events for one user action; handle updates gracefully
- **Webhook Retry**: Strava retries up to 3 times if not receiving 200; ensure idempotency
- **Connection Loss**: Handle case where athlete deauthorizes without webhook (regular token validation)

### 4.6 Deployment Considerations
- **Public Endpoint**: Backend must have public URL for webhook callbacks
- **SSL/TLS**: Strava requires HTTPS for callback URLs
- **Subscription Management**: Create subscription during deployment or via admin endpoint
- **Monitoring**: Log webhook events for debugging and monitoring
- **Graceful Degradation**: App should work without Strava integration

### 4.7 Testing Strategy
- **Unit Tests**:
  - Token refresh logic
  - Event parsing and validation
  - Activity data transformation

- **Integration Tests**:
  - Webhook verification flow
  - Event processing with mocked Strava API
  - OAuth token exchange

- **Manual Testing**:
  - Use ngrok to test webhooks locally
  - Create test activities in Strava
  - Verify sessions appear in Runna
  - Test update and delete flows
  - Test deauthorization

### 4.8 Future Enhancements
- Bulk import of historical activities
- Webhook health monitoring and alerting
- Support for multiple connected services (Garmin, Apple Health, etc.)

## Phase 5: Integration & Testing

### 5.1 Combined Docker Compose
- Volume mounts for development
- Environment variable management

### 5.2 Testing
- Backend:
  - Test database connection
- Frontend:
  - Test form submission
  - Test data retrieval and display
  - Test error scenarios
- Integration:
  - End-to-end flow: create session and verify it appears in overview

### 5.3 Documentation
- README for each repository with:
  - Setup instructions
  - Environment variable requirements
  - Docker commands to run
  - API documentation (for backend)
- Development workflow documentation

## Phase 6: Deployment Preparation

### 6.1 Environment Configuration
- Document all required environment variables
- Create .env.example files

### 6.2 Production Optimizations
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
