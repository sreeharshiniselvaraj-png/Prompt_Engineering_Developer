# Task Tracker API — OpenAPI Documentation

This document contains the OpenAPI 3.0.3 specification for the Task Tracker API.

```yaml
openapi: 3.0.3
info:
  title: Task Tracker API
  version: 1.0.0
  description: REST API for creating and retrieving tasks.
servers:
  - url: http://localhost:3000
    description: Local development server
tags:
  - name: Tasks
    description: Task management operations
paths:
  /api/v1/tasks:
    post:
      tags: [Tasks]
      summary: Create a new task
      operationId: createTask
      security:
        - bearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskRequest'
            example:
              title: Complete API documentation
              description: Write the OpenAPI specification
              due_date: '2026-11-01'
      responses:
        '201':
          description: Task created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
              example:
                id: 101
                user_id: 1
                title: Complete API documentation
                description: Write the OpenAPI specification
                status: Pending
                due_date: '2026-11-01'
                created_at: '2026-10-15T09:00:00Z'
        '400':
          description: Invalid request data
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                error: Title is required.
        '401':
          description: Authentication required
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
  /api/v1/tasks/{id}:
    get:
      tags: [Tasks]
      summary: Get a task by ID
      operationId: getTaskById
      security:
        - bearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          description: Positive integer task identifier
          schema:
            type: integer
            format: int64
            minimum: 1
          example: 101
      responses:
        '200':
          description: Task retrieved successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
        '400':
          description: Invalid task ID format
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                error: Task ID must be a positive integer.
        '401':
          description: Authentication required
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
        '404':
          description: Task not found or not accessible
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                error: Task not found.
        '500':
          description: Internal server error
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    CreateTaskRequest:
      type: object
      required: [title]
      properties:
        title:
          type: string
          minLength: 1
          maxLength: 200
        description:
          type: string
          nullable: true
        due_date:
          type: string
          format: date
          nullable: true
    Task:
      type: object
      required: [id, user_id, title, status, created_at]
      properties:
        id:
          type: integer
          format: int64
        user_id:
          type: integer
          format: int64
        title:
          type: string
        description:
          type: string
          nullable: true
        status:
          type: string
          enum: [Pending, In Progress, Completed]
        due_date:
          type: string
          format: date
          nullable: true
        created_at:
          type: string
          format: date-time
    Error:
      type: object
      required: [error]
      properties:
        error:
          type: string
```

## Endpoint notes

- `POST /api/v1/tasks` creates a task and returns `201 Created`.
- `GET /api/v1/tasks/{id}` returns `200 OK` for an accessible task.
- Invalid request bodies or malformed IDs return `400 Bad Request`.
- Missing/invalid authentication returns `401 Unauthorized`.
- A task that does not exist or is not accessible to the caller returns `404 Not Found`.
- The OpenAPI path syntax uses `{id}`; Express route syntax uses `:id`.
- This contract assumes bearer-token authentication middleware sets `req.user.id`.
