# Backend Logging Documentation

## Overview
Comprehensive logging has been implemented throughout the backend to track requests, responses, and errors.

## Logging Levels

The backend uses the following log levels:

- **[INFO]** - Successful operations and normal flow
- **[WARN]** - Validation errors, client errors (4xx responses)
- **[ERROR]** - Server errors, database errors, system failures (5xx responses)

## Request/Response Logging

### Middleware Logging
The `middleware.Logging` middleware automatically logs:

1. **Incoming Request:**
   ```
   [INFO] --> METHOD /path REMOTE_ADDR
   ```
   Example: `[INFO] --> POST /api/sessions 127.0.0.1:54321`

2. **Response:**
   ```
   [LEVEL] <-- METHOD /path STATUS_CODE DURATION
   ```
   Examples:
   - `[INFO] <-- POST /api/sessions 201 45ms`
   - `[WARN] <-- GET /api/sessions/abc 400 2ms`
   - `[ERROR] <-- POST /api/sessions 500 123ms`

### Handler-Specific Logging

Each handler logs:
- Validation errors with details
- Database operations (success/failure)
- Specific error causes

## Session Handlers

### CreateSession
```
[ERROR] CreateSession: Failed to decode request body: <error>
[WARN] CreateSession: Invalid distance: <value>
[WARN] CreateSession: Invalid duration: <value>
[ERROR] CreateSession: Database error: <error>
[INFO] CreateSession: Created session id=<id>
```

### GetSessions
```
[WARN] GetSessions: Invalid start_date format: <value>, error: <error>
[WARN] GetSessions: Invalid end_date format: <value>, error: <error>
[ERROR] GetSessions: Database error: <error>
[INFO] GetSessions: Retrieved <count> sessions for period <start> to <end>
```

### GetSession
```
[WARN] GetSession: Invalid session ID format: <value>, error: <error>
[ERROR] GetSession: Database error for id=<id>: <error>
[INFO] GetSession: Retrieved session id=<id>
```

### UpdateSession
```
[WARN] UpdateSession: Invalid session ID format: <value>, error: <error>
[ERROR] UpdateSession: Failed to decode request body for id=<id>: <error>
[WARN] UpdateSession: Invalid distance for id=<id>: <value>
[WARN] UpdateSession: Invalid duration for id=<id>: <value>
[ERROR] UpdateSession: Database error for id=<id>: <error>
[INFO] UpdateSession: Updated session id=<id>
```

## Goal Handlers

### CreateGoal
```
[ERROR] CreateGoal: Failed to decode request body: <error>
[WARN] CreateGoal: Invalid target distance: <value>
[WARN] CreateGoal: End date before start date: start=<date>, end=<date>
[ERROR] CreateGoal: Database error: <error>
[INFO] CreateGoal: Created goal id=<id>, target=<distance>km
```

### GetGoals
```
[ERROR] GetGoals: Database error: <error>
[INFO] GetGoals: Retrieved <count> goals
```

### GetGoal
```
[WARN] GetGoal: Invalid goal ID format: <value>, error: <error>
[ERROR] GetGoal: Database error for id=<id>: <error>
[INFO] GetGoal: Retrieved goal id=<id>
```

### DeleteGoal
```
[WARN] DeleteGoal: Invalid goal ID format: <value>, error: <error>
[ERROR] DeleteGoal: Database error for id=<id>: <error>
[INFO] DeleteGoal: Deleted goal id=<id>
```

## Strava Handlers

### Webhook Handlers
- Already have comprehensive logging in webhook_handlers.go
- Logs verification attempts, event reception, and processing

### OAuth Handlers
- Logs connection attempts, token exchanges, and disconnections in strava_handlers.go

### Strava Service
- Extensive logging of activity processing, token refresh, and API calls in strava_service.go

## Example Log Output

```
2024/02/25 15:30:45 [INFO] Server starting on port 8080
2024/02/25 15:31:12 [INFO] --> POST /api/sessions 192.168.1.100:54321
2024/02/25 15:31:12 [INFO] CreateSession: Created session id=42
2024/02/25 15:31:12 [INFO] <-- POST /api/sessions 201 34ms
2024/02/25 15:31:15 [INFO] --> GET /api/sessions 192.168.1.100:54322
2024/02/25 15:31:15 [INFO] GetSessions: Retrieved 15 sessions for period 2024-01-25 to 2024-02-25
2024/02/25 15:31:15 [INFO] <-- GET /api/sessions 200 12ms
2024/02/25 15:31:20 [INFO] --> POST /api/sessions 192.168.1.100:54323
2024/02/25 15:31:20 [WARN] CreateSession: Invalid distance: -5.000000
2024/02/25 15:31:20 [WARN] <-- POST /api/sessions 400 1ms
2024/02/25 15:31:25 [INFO] --> DELETE /api/goals/99 192.168.1.100:54324
2024/02/25 15:31:25 [ERROR] DeleteGoal: Database error for id=99: sql: no rows in result set
2024/02/25 15:31:25 [ERROR] <-- DELETE /api/goals/99 500 8ms
```

## Log Analysis

### Monitoring Recommendations

1. **Track Error Rates**: Monitor `[ERROR]` logs for system health
2. **Watch Validation Issues**: Frequent `[WARN]` logs may indicate client issues
3. **Performance Monitoring**: Check request durations in response logs
4. **Database Issues**: Look for patterns in database error logs

### Log Aggregation

Consider using log aggregation tools like:
- ELK Stack (Elasticsearch, Logstash, Kibana)
- Grafana Loki
- CloudWatch Logs (if on AWS)
- Google Cloud Logging (if on GCP)

### Structured Logging Enhancement (Future)

For production, consider upgrading to structured logging:
- Use `zerolog` or `zap` for JSON-formatted logs
- Add request IDs for tracing
- Include user context when authentication is added
- Add performance metrics

## Configuration

Currently using Go's standard `log` package with format:
```
YYYY/MM/DD HH:MM:SS [LEVEL] Message
```

No additional configuration required. Logs are written to `stderr` by default.
