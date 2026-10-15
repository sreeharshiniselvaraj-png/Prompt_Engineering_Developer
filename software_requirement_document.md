# Software Requirements Document (SRD)
## Task Tracker API

**Version:** 1.0  
**Status:** Draft for implementation  
**API prefix:** `/api/v1`  
**Target stack:** Node.js, Express, PostgreSQL, `pg`  
**Audience:** Business analyst, backend developers, QA engineers

---

## 1. Purpose

The Task Tracker API enables authenticated users to create tasks and retrieve tasks they are authorized to access. This document consolidates the agreed scope, user stories, acceptance criteria, SQL DDL, and reference controllers.

## 2. Scope

### In scope
- User registration data storage.
- Task creation.
- Task retrieval by ID.
- Authentication and task-owner authorization through existing middleware.
- Request validation, parameterized SQL, and standardized errors.

### Out of scope
- Full signup/login controller implementation in this document.
- Task comments, attachments, notifications, reporting, and workflow automation.
- Organization administration and multi-tenant provisioning.
- Task update endpoint implementation, beyond the earlier user story.

## 3. Architecture

Client → Express REST routes → controller/business logic → PostgreSQL via `pg` → JSON response.

Authentication middleware must verify credentials and populate trusted `req.user.id`. The controller must never trust a client-provided owner ID.

## 4. User stories and acceptance criteria

### US-01 — User Signup

As a business employee or authorized business user, I want to register an account using my required details so that I can securely access the Task Tracker API.

- AC-01.1: Valid required details create an account and return its identifier.
- AC-01.2: Missing or invalid required values return a validation error.
- AC-01.3: Duplicate email is rejected by application validation and the database unique constraint.
- AC-01.4: Only a secure password hash is stored; plaintext passwords and hashes are never returned.
- AC-01.5: The endpoint returns a documented status code and response format.

### US-02 — Task Creation

As an authenticated business user, I want to create a task with a title, description, and optional due date so that I can track work.

- AC-02.1: Valid input creates a task and returns `201` with the saved task.
- AC-02.2: Missing, blank, or overlong title returns `400`.
- AC-02.3: If status is omitted, status defaults to `Pending`.
- AC-02.4: Missing or invalid authentication returns `401`.
- AC-02.5: The task is associated with the authenticated user's ID.
- AC-02.6: Invalid data does not create a partial task.

### US-03 — Task Retrieval

As an authenticated and authorized user, I want to retrieve a task by ID so that I can view its current details.

- AC-03.1: An accessible existing task returns `200` and its representation.
- AC-03.2: A malformed ID returns `400`.
- AC-03.3: A nonexistent or inaccessible task returns `404`.
- AC-03.4: Missing or invalid authentication returns `401`.
- AC-03.5: Query values are parameterized and sensitive database fields are not returned.

### US-04 — Task Updates (requirements only)

As an authorized business user, I want to update task details or status so that task information remains accurate as work progresses.

- AC-04.1: Valid changes update the existing task and return the updated representation.
- AC-04.2: Omitted PATCH fields remain unchanged.
- AC-04.3: Only supported status values and permitted transitions are accepted.
- AC-04.4: Missing or inaccessible task returns `404`.
- AC-04.5: Unauthorized changes are denied without modifying data.
- AC-04.6: Invalid values return `400` without applying changes.

Note: A PATCH controller is not included because the current implementation request covers POST create and GET by ID only.

## 5. Business rules

1. User email must be unique.
2. Each task must reference an existing user.
3. A task title is required and limited to 200 characters.
4. Allowed statuses are `Pending`, `In Progress`, and `Completed`.
5. New tasks default to `Pending`.
6. `due_date` and `description` are optional.
7. Only the task owner or a separately authorized principal may access a task. The sample controllers below enforce owner-only access.
8. Passwords must be hashed using a suitable password-hashing algorithm before storage.

## 6. PostgreSQL DDL

```sql
CREATE TABLE users (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tasks (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id BIGINT NOT NULL,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'Pending',
    due_date DATE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    CONSTRAINT fk_tasks_users
        FOREIGN KEY (user_id) REFERENCES users(id),

    CONSTRAINT chk_tasks_status
        CHECK (status IN ('Pending', 'In Progress', 'Completed'))
);

CREATE INDEX idx_tasks_user_id ON tasks(user_id);
```

## 7. API contract

### POST `/api/v1/tasks`
- Authentication: bearer token; middleware must set `req.user.id`.
- Request JSON: `title` (required string, max 200), `description` (optional string/null), `due_date` (optional ISO date/null).
- Success: `201 Created`, task representation.
- Errors: `400` validation error, `401` authentication failure, `500` unexpected server error.

### GET `/api/v1/tasks/:id`
- Authentication: bearer token; middleware must set `req.user.id`.
- Path parameter: positive integer task ID.
- Success: `200 OK`, task representation.
- Errors: `400` malformed ID, `401` authentication failure, `404` not found or not accessible, `500` unexpected server error.

## 8. Reference implementation

Install dependencies:

```bash
npm install express pg
```

### `db.js`

```javascript
const { Pool } = require("pg");

const pool = new Pool({
  host: process.env.DB_HOST,
  port: Number(process.env.DB_PORT || 5432),
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 10,
  connectionTimeoutMillis: 5000
});

module.exports = pool;
```

### `controllers/tasksController.js`

```javascript
const db = require("../db");

function parseTaskId(rawId) {
  // req.params values are strings; reject malformed or non-positive IDs.
  if (!/^[1-9]\d*$/.test(rawId)) return null;

  const id = Number(rawId);
  if (!Number.isSafeInteger(id)) return null;

  return id;
}

async function createTask(req, res, next) {
  try {
    // req.user.id must be set by trusted authentication middleware.
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ error: "Authentication required." });
    }

    // Parse the JSON body; never accept user_id from the client.
    const { title, description = null, due_date = null } = req.body ?? {};

    if (
      typeof title !== "string" ||
      title.trim().length === 0 ||
      title.trim().length > 200
    ) {
      return res.status(400).json({
        error: "Title is required and must be at most 200 characters."
      });
    }

    if (description !== null && typeof description !== "string") {
      return res.status(400).json({
        error: "Description must be a string or null."
      });
    }

    if (
      due_date !== null &&
      (typeof due_date !== "string" ||
        !/^\d{4}-\d{2}-\d{2}$/.test(due_date) ||
        Number.isNaN(Date.parse(`${due_date}T00:00:00Z`)) ||
        new Date(`${due_date}T00:00:00Z`).toISOString().slice(0, 10) !== due_date)
    ) {
      return res.status(400).json({
        error: "due_date must be a valid date in YYYY-MM-DD format."
      });
    }

    // $1-$4 are PostgreSQL placeholders. Values are bound separately,
    // so user input is treated as data, not executable SQL.
    const query = `
      INSERT INTO tasks (user_id, title, description, due_date)
      VALUES ($1, $2, $3, $4)
      RETURNING id, user_id, title, description, status, due_date, created_at
    `;

    const values = [userId, title.trim(), description, due_date];
    const result = await db.query(query, values);

    return res.status(201).json(result.rows[0]);
  } catch (err) {
    // Forward to centralized Express error middleware; do not leak DB details.
    return next(err);
  }
}

async function getTaskById(req, res, next) {
  try {
    const userId = req.user?.id;
    if (!userId) {
      return res.status(401).json({ error: "Authentication required." });
    }

    const taskId = parseTaskId(req.params.id);
    if (taskId === null) {
      return res.status(400).json({
        error: "Task ID must be a positive integer."
      });
    }

    // Both task ID and owner ID are bound parameters. Owner filtering prevents
    // a caller from retrieving another user's task by guessing its ID.
    const query = `
      SELECT id, user_id, title, description, status, due_date, created_at
      FROM tasks
      WHERE id = $1 AND user_id = $2
      LIMIT 1
    `;

    const result = await db.query(query, [taskId, userId]);

    if (result.rows.length === 0) {
      return res.status(404).json({ error: "Task not found." });
    }

    return res.status(200).json(result.rows[0]);
  } catch (err) {
    return next(err);
  }
}

module.exports = { createTask, getTaskById };
```

### `routes/tasks.js`

```javascript
const express = require("express");
const { createTask, getTaskById } = require("../controllers/tasksController");
const requireAuth = require("../middleware/requireAuth");

const router = express.Router();

router.post("/", requireAuth, createTask);
router.get("/:id", requireAuth, getTaskById);

module.exports = router;
```

### `app.js`

```javascript
const express = require("express");
const tasksRouter = require("./routes/tasks");

const app = express();
app.use(express.json());
app.use("/api/v1/tasks", tasksRouter);

// Centralized error handler. Log details server-side only.
app.use((err, req, res, next) => {
  console.error("API error:", err.message);
  if (res.headersSent) return next(err);

  return res.status(500).json({
    error: "An internal server error occurred."
  });
});

module.exports = app;
```

The `requireAuth` middleware is a project dependency and must verify a session/token and populate `req.user.id`. Do not implement a placeholder that trusts an ID from the request body.

## 9. Error response examples

```json
{ "error": "Title is required and must be at most 200 characters." }
```

```json
{ "error": "Task ID must be a positive integer." }
```

```json
{ "error": "Task not found." }
```

## 10. Security requirements

- Use parameterized SQL; never concatenate user input into SQL.
- Store password hashes only; never store plaintext passwords.
- Authenticate all task endpoints.
- Enforce task ownership in SQL or a trusted authorization layer.
- Do not return database errors, stack traces, password hashes, or secrets.
- Use environment variables or a secrets manager for database credentials.
- Use HTTPS and rate limiting in production.
- Run tests against an isolated test database.
- Consider adding audit logging and organization-level tenant isolation if the product will serve multiple businesses.

## 11. QA test summary

| Test | Expected behavior |
|---|---|
| Create task with valid data | `201 Created` |
| Create task without title | `400 Bad Request` |
| Create task without authentication | `401 Unauthorized` |
| Get existing owned task | `200 OK` |
| Get malformed task ID | `400 Bad Request` |
| Get missing or non-owned task | `404 Not Found` |
| SQL-like input in a value | Treated as data; query structure unchanged |
| Database failure | Controlled `500`; no internal details exposed |

## 12. Assumptions and implementation notes

- Database is PostgreSQL and the driver is `pg`.
- The implementation is a reference scaffold and has not been executed against a real project database.
- `created_at` is generated by PostgreSQL.
- The `users` and `tasks` tables use the DDL above.
- The OpenAPI and tests must be kept synchronized with actual middleware and response behavior.
