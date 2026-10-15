# Task Tracker API — Prompt Library

This library stores reusable prompt templates for the Task Tracker API project. Replace variables in curly braces before running each prompt.

---

## 1. CRISPE Coding Prompt

**CAPACITY:** Act as a Senior Backend Software Engineer specializing in Node.js, Express, PostgreSQL, REST APIs, and secure backend implementation.

**RECIPIENT:** {recipient_team}, consisting of junior developers and QA engineers.

**INSTRUCTION:** Implement these API controllers:
1. `POST /api/v1/tasks` — create a task using `title`, `description`, and `due_date`.
2. `GET /api/v1/tasks/:id` — retrieve one task by ID.

**STYLE:** Follow DRY and SOLID principles. Use clear naming, modular route/controller/data-access responsibilities, input validation, and consistent JSON error responses.

**PARAMETERS:**
- Language/framework: {language_and_framework}
- Database: {database}
- Database driver: {database_driver}
- Schema: `users(id, name, email, password_hash, created_at)` and `tasks(id, user_id, title, description, status, due_date, created_at)`.
- Do not use an ORM.
- Use parameterized raw SQL only.
- Read task ownership from trusted authentication context (`req.user.id`), never from client-provided `user_id`.
- Use status `Pending` by default and permit only `Pending`, `In Progress`, and `Completed`.
- Authentication middleware is assumed to be configured; document the dependency.

**EVALUATION:** Include complete controller and route code, inline comments explaining request parsing and SQL parameter binding, error handling, and example success/error responses. Explain how the code satisfies DRY/SOLID. Do not invent unprovided business requirements.

**PROJECT VARIABLES**
- API base path: `{api_base_path}`
- Required task fields: `{required_task_fields}`
- Allowed task statuses: `{allowed_statuses}`
- Authentication strategy: `{authentication_strategy}`
- Response envelope convention: `{response_convention}`

---

## 2. Least-to-Most Prompt Template

Act as a Business Analyst and Backend Technical Lead. Solve the `{project_name}` task in incremental stages, from basic requirements to implementation. Do not skip stages. State assumptions explicitly and keep each stage consistent with the earlier stages.

**Project context**
- Project: `{project_name}`
- Business objective: `{business_objective}`
- Technology stack: `{technology_stack}`
- Database: `{database}`
- Constraints: `{constraints}`

**Stages**
1. **Clarify the goal:** Restate the required behavior and identify missing information.
2. **Define scope:** List in-scope and out-of-scope features.
3. **Write user stories:** Create user stories for `{user_story_features}` with testable acceptance criteria.
4. **Define data requirements:** Identify entities, fields, constraints, and relationships without writing code yet.
5. **Specify API contract:** Define methods, paths, authentication, request schemas, responses, and HTTP status codes.
6. **Implement incrementally:** Generate database DDL, then data-access functions, then controllers and routes.
7. **Create QA tests:** Produce tests for `{test_scenarios}` and a QA test matrix.
8. **Review:** Check security, consistency, error handling, and requirements coverage.

At each stage, show the output and a short validation checklist. If an earlier decision conflicts with a later requirement, flag the conflict and recommend a resolution instead of silently changing the design.

---

## 3. Reflection Security Audit Prompt

**ROLE:** Act as a Senior Application Security Auditor with expertise in `{language}`, `{framework}`, and `{database}`.

**CODE TO AUDIT:**
```{code_language}
{code_to_audit}
```

**TASK**
1. Identify security weaknesses such as SQL injection, unhandled asynchronous errors, insecure password handling, broken access control, and excessive data exposure.
2. Explain the exploit precondition, likely impact, and severity for each finding.
3. Refactor the code using parameterized SQL queries, safe error handling, input validation, and least-privilege data access.
4. For login endpoints, retrieve the stored password hash by email and verify it using a suitable password-hashing library. Never compare or store plaintext passwords.
5. Return only required non-sensitive response fields.
6. Add inline comments explaining how user input is parsed and how query parameters are bound.
7. Provide verification cases demonstrating that SQL-like input is treated as data, not executable SQL.
8. Separate fixes shown in the sample from additional production controls.

**EVALUATION**
- Do not concatenate user input into SQL strings.
- Use the correct placeholder syntax for `{database_driver}`.
- Handle errors without throwing uncaught exceptions from asynchronous callbacks.
- Do not expose secrets, hashes, stack traces, or database internals.
- Include tests for invalid input, authentication failures, and unauthorized resource access.
- Do not claim tests were executed unless they actually were.

**Additional variables**
- Threat model: `{threat_model}`
- Expected response codes: `{expected_http_statuses}`
- Security controls already present: `{existing_controls}`
- Deployment constraints: `{deployment_constraints}`

---

## Usage guidance

- Use the CRISPE prompt to generate a clearly scoped implementation.
- Use Least-to-Most prompting to develop the project step by step, validating requirements before coding.
- Use the Reflection prompt to review generated code for security flaws and produce a safer revision.
- Store actual prompts and generated outputs used during the assignment separately if an audit trail is required.
