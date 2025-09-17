### Task Management API

Modern REST API for managing tasks and subtasks with JWT authentication.

## Base URL

- Local: `https://task-management-nodejs-ba46.onrender.com`
- All endpoints are prefixed with `/api`

## Authentication

- Scheme: Bearer JWT
- Header: `Authorization: Bearer <token>`
- Obtain a token via `POST /api/auth/login` or after successful `POST /api/auth/register`.

## Environment Variables

- `PORT`: server port (default `3000`)
- `DATABASE_URL`: PostgreSQL connection string
- `JWT_SECRET`: secret for signing JWTs (default fallback exists, set your own in production)

## Data Models (Prisma)

- User: `{ id, email, password, name, createdAt, updatedAt }`
- Task: `{ id, title, description, status, priority, dueDate?, assignedTo?, userId, createdAt, updatedAt, subtasks[] }`
  - status: one of `pending | in_progress | completed | cancelled`
  - priority: one of `low | medium | high | urgent`
- Subtask: `{ id, title, description, completed, taskId, createdAt, updatedAt }`

## Conventions

- Request/Response Content-Type: `application/json`
- Dates: ISO 8601 strings (e.g., `2025-09-17T10:00:00.000Z`)
- Protected routes require the `Authorization` header

## Auth Endpoints

### POST /api/auth/register

Register a new user and return a JWT.

Request body:

```json
{
  "email": "user@example.com",
  "password": "strongPassword123",
  "name": "User Name"
}
```

Response 201:

```json
{
  "success": true,
  "data": {
    "user": { "id": "cuid", "email": "user@example.com", "name": "User Name" },
    "token": "<jwt>"
  }
}
```

Notes:

- Email must be unique.
- Password is stored hashed.

Example:

```bash
curl -X POST http://localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret","name":"User"}'
```

### POST /api/auth/login

Authenticate an existing user.

Request body:

```json
{
  "email": "user@example.com",
  "password": "strongPassword123"
}
```

Response 200:

```json
{
  "success": true,
  "data": {
    "user": { "id": "cuid", "email": "user@example.com", "name": "User Name" },
    "token": "<jwt>"
  }
}
```

Example:

```bash
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret"}'
```

### GET /api/auth/me

Return the current authenticated user.

Headers:

```
Authorization: Bearer <token>
```

Response 200:

```json
{
  "success": true,
  "data": { "id": "cuid", "email": "user@example.com", "name": "User Name" }
}
```

## Task Endpoints

All task and subtask endpoints are protected. Add `Authorization: Bearer <token>` to every request.

### GET /api/tasks

List all tasks for the current user (most recent first).

Response 200:

```json
{
  "success": true,
  "count": 2,
  "data": [
    {
      "id": "cuid",
      "title": "Plan sprint",
      "description": "Outline sprint backlog",
      "status": "pending",
      "priority": "high",
      "dueDate": "2025-09-20T00:00:00.000Z",
      "assignedTo": "john",
      "userId": "cuidUser",
      "createdAt": "2025-09-15T10:00:00.000Z",
      "updatedAt": "2025-09-15T10:00:00.000Z",
      "subtasks": [ { "id": "cuid", "title": "Draft", "description": "", "completed": false, "taskId": "cuid", "createdAt": "...", "updatedAt": "..." } ]
    }
  ]
}
```

Example:

```bash
curl http://localhost:3000/api/tasks \
  -H "Authorization: Bearer <token>"
```

### GET /api/tasks/:id

Get a single task (owned by user).

Responses:

- 200: `{ success: true, data: { ...task } }`
- 404: `{ success: false, error: "Task not found" }`

Example:

```bash
curl http://localhost:3000/api/tasks/<taskId> \
  -H "Authorization: Bearer <token>"
```

### POST /api/tasks

Create a new task (optionally with subtasks).

Request body:

```json
{
  "title": "Plan sprint",
  "description": "Outline sprint backlog",
  "status": "pending", // or "in-progress" (client) → saved as "in_progress"
  "priority": "high",
  "dueDate": "2025-09-20",
  "assignedTo": "john",
  "subtasks": [
    { "title": "Draft", "description": "" },
    { "title": "Review", "description": "" }
  ]
}
```

Responses:

- 201: `{ success: true, data: { ...taskWithSubtasks } }`
- 400: `{ success: false, error: "..." }`

Example:

```bash
curl -X POST http://localhost:3000/api/tasks \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"title":"Plan sprint","description":"Outline","status":"pending","priority":"high"}'
```

### PUT /api/tasks/:id

Update fields on an existing task.

Request body (any subset):

```json
{
  "title": "Updated title",
  "description": "Updated description",
  "status": "in-progress", // will be stored as "in_progress"
  "priority": "urgent",
  "dueDate": "2025-09-22",
  "assignedTo": "mary"
}
```

Responses:

- 200: `{ success: true, data: { ...updatedTask } }`
- 404: `{ success: false, error: "Task not found" }`
- 400: `{ success: false, error: "..." }`

Example:

```bash
curl -X PUT http://localhost:3000/api/tasks/<taskId> \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"status":"completed"}'
```

### DELETE /api/tasks/:id

Delete a task and its subtasks.

Responses:

- 200: `{ success: true, data: { ...deletedTask } }`
- 404: `{ success: false, error: "Task not found" }`

Example:

```bash
curl -X DELETE http://localhost:3000/api/tasks/<taskId> \
  -H "Authorization: Bearer <token>"
```

## Subtask Endpoints

### GET /api/tasks/:taskId/subtasks

List subtasks for a specific task (must belong to user).

Responses:

- 200: `{ success: true, count, data: [ ...subtasks ] }`
- 404: `{ success: false, error: "Task not found or access denied" }`

Example:

```bash
curl http://localhost:3000/api/tasks/<taskId>/subtasks \
  -H "Authorization: Bearer <token>"
```

### GET /api/subtasks/:id

Get a single subtask (must belong to a task owned by user).

Responses:

- 200: `{ success: true, data: { ...subtask } }`
- 404: `{ success: false, error: "Subtask not found or access denied" }`

### POST /api/tasks/:taskId/subtasks

Create a subtask under a task.

Request body:

```json
{
  "title": "Write tests",
  "description": "Cover CRUD"
}
```

Responses:

- 201: `{ success: true, data: { ...newSubtask } }`
- 404: `{ success: false, error: "Task not found or access denied" }`
- 400: `{ success: false, error: "..." }`

### PUT /api/subtasks/:id

Update a subtask.

Request body (any subset):

```json
{
  "title": "Write tests",
  "description": "Cover all branches",
  "completed": true
}
```

Responses:

- 200: `{ success: true, data: { ...updatedSubtask } }`
- 404: `{ success: false, error: "Subtask not found or access denied" }`
- 400: `{ success: false, error: "..." }`

### DELETE /api/subtasks/:id

Delete a subtask.

Responses:

- 200: `{ success: true, data: { ...deletedSubtask } }`
- 404: `{ success: false, error: "Subtask not found or access denied" }`

## Protected Route Example (from server)

### GET /api/protected

Simple example to verify JWTs; returns the decoded user stored on the request.

## Error Handling

- 401 Unauthorized: missing/invalid/expired token
- 404 Not Found: resource not found or not owned by user
- 400 Bad Request: validation errors (creation/update)
- 500 Internal Server Error: unhandled errors

## Headers Cheat Sheet

- `Content-Type: application/json`
- `Authorization: Bearer <token>`

## Notes for Clients

- When sending `status` from clients, you may use `in-progress`; the API persists it as `in_progress`.
- `dueDate` accepts a date string; the API stores it as a Date.
- List endpoints return `count` to aid pagination planning (server currently returns all results for the user).


