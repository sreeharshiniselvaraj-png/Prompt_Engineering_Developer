# Task Tracker API — Unit Tests and QA Test Matrix

## Test setup

Frameworks: Jest and Supertest. The tests assume:
- The Express application is exported from `../app`.
- The API uses PostgreSQL and a test database.
- Authentication middleware accepts `Authorization: Bearer <token>`.
- `TEST_AUTH_TOKEN` is a valid token for a test user.
- The test database is isolated from development/production data.

Install test dependencies:

```bash
npm install --save-dev jest supertest
```

Add this script to `package.json`:

```json
{
  "scripts": {
    "test": "jest --runInBand"
  }
}
```

## `tests/tasks.test.js`

```javascript
const request = require("supertest");
const app = require("../app");

describe("Task Tracker API", () => {
  const token = process.env.TEST_AUTH_TOKEN;

  beforeAll(() => {
    if (!token) {
      throw new Error("TEST_AUTH_TOKEN must be configured for API tests");
    }
  });

  test("TC-01: POST /api/v1/tasks creates a task and returns 201", async () => {
    const taskData = {
      title: "Complete API testing",
      description: "Write automated tests using Jest and Supertest",
      due_date: "2026-11-01"
    };

    const response = await request(app)
      .post("/api/v1/tasks")
      .set("Authorization", `Bearer ${token}`)
      .send(taskData);

    expect(response.status).toBe(201);
    expect(response.body).toHaveProperty("id");
    expect(response.body.title).toBe(taskData.title);
    expect(response.body.description).toBe(taskData.description);
    expect(response.body.status).toBe("Pending");
  });

  test("TC-02: GET /api/v1/tasks/:id returns 404 for a nonexistent task", async () => {
    // Ensure this ID is absent from the isolated test database.
    const nonexistentTaskId = 999999999;

    const response = await request(app)
      .get(`/api/v1/tasks/${nonexistentTaskId}`)
      .set("Authorization", `Bearer ${token}`);

    expect(response.status).toBe(404);
    expect(response.body).toHaveProperty("error", "Task not found.");
  });
});
```

Run tests:

```bash
TEST_AUTH_TOKEN='<test-token>' npm test
```

The output below is illustrative only; the tests have not been run by generating this document.

```text
PASS tests/tasks.test.js
  Task Tracker API
    ✓ TC-01: POST /api/v1/tasks creates a task and returns 201
    ✓ TC-02: GET /api/v1/tasks/:id returns 404 for a nonexistent task
```

## QA test matrix

| Test ID | Scenario | Request / setup | Expected result | Priority |
|---|---|---|---|---|
| TC-01 | Create task successfully | Valid token and valid title/description/due_date | HTTP 201; response includes id, title and default `Pending` status | High |
| TC-02 | Get nonexistent task | Valid token and an ID confirmed absent from test DB | HTTP 404; `error` equals `Task not found.` | High |
| TC-03 | Create task without title | Valid token; omit title | HTTP 400; validation error; no row created | High |
| TC-04 | Create task without authentication | Omit Authorization header | HTTP 401 | High |
| TC-05 | Get task with malformed ID | Valid token; ID `abc` | HTTP 400 | Medium |
| TC-06 | Get another user's task | Valid token for user A; request task owned by user B | HTTP 404 or documented access-denied behavior; no data leak | High |
| TC-07 | SQL injection input | Use SQL-like string as title or ID in a test environment | Input treated as data; no unauthorized access or query alteration | High |
| TC-08 | Database unavailable | Disable test DB connection in isolated environment | Controlled HTTP 500; no stack trace or credentials exposed | High |

## QA notes

- TC-02 uses a valid positive integer that must be confirmed not to exist in the test database. An invalidly formatted ID should normally produce `400`, not `404`.
- Tests need a real test database and configured authentication; the examples are not self-contained mocks.
- Avoid relying on a hard-coded ID if the test database is shared. Use a fixture or create-and-clean-up strategy.
