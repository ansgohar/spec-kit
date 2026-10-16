# OpenAPI Specification Guide

A step-by-step guide to authoring, validating, and simulating an OpenAPI specification.

---

## Prerequisites

Verify Node.js is installed:

```bash
node -v
```

> Minimum recommended: Node.js 18 LTS or later.

---

## Step 1 — Author the OpenAPI Specification

Create a file named `users-api.yaml` and author your spec following the [OpenAPI 3.x standard](https://spec.openapis.org/oas/v3.1.0).

### Structural checklist

| Section | Required | Notes |
| --- | --- | --- |
| `openapi` | ✅ | Must be `3.0.x` or `3.1.x` |
| `info.title` | ✅ | Human-readable API name |
| `info.version` | ✅ | Semantic version, e.g. `1.0.0` |
| `info.description` | ✅ | Short summary of the API purpose |
| `servers` | ✅ | At least one server URL |
| `paths` | ✅ | All endpoints with HTTP methods |
| `components.schemas` | Recommended | Reusable data models |
| `components.securitySchemes` | Recommended | Auth definitions (OAuth2, API key, etc.) |
| `tags` | Recommended | Group endpoints logically |

### Minimal working example

```yaml
openapi: 3.1.0

info:
  title: Users API
  version: 1.0.0
  description: Manages user accounts.

servers:
  - url: http://localhost:3000
    description: Local development server

tags:
  - name: Users
    description: User management operations

paths:
  /users:
    get:
      summary: List all users
      operationId: listUsers
      tags:
        - Users
      responses:
        "200":
          description: A list of users
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/User"

  /users/{id}:
    get:
      summary: Get a user by ID
      operationId: getUserById
      tags:
        - Users
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: A single user
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "404":
          description: User not found

components:
  schemas:
    User:
      type: object
      required:
        - id
        - name
        - email
      properties:
        id:
          type: string
          example: "u-001"
        name:
          type: string
          example: "Jane Doe"
        email:
          type: string
          format: email
          example: "jane.doe@example.com"
```

### Authoring standards to follow

- Use **camelCase** for `operationId` values.
- Every path operation **must** have an `operationId`, a `summary`, and at least one `response`.
- Always define a `4xx` response (e.g. `400`, `404`) alongside success responses.
- Use `$ref` to reference shared schemas — avoid inline duplication.
- Add `example` values to every schema property to enable realistic mock responses.
- Avoid `$ref` cycles and deeply nested inline schemas.

---

## Step 2 — Validate the Specification

### Install Spectral

```bash
npm install -g @stoplight/spectral-cli
```

Verify the installation:

```bash
spectral --version
```

### Option A — Lint with the built-in OAS ruleset (no config file needed)

```bash
spectral lint users-api.yaml --ruleset @stoplight/spectral-oas
```

### Option B — Lint with a local config file (recommended for teams)

Create `.spectral.json` in the same directory as your spec:

```json
{
  "extends": ["spectral:oas"]
}
```

Then run Spectral from that directory:

```bash
spectral lint users-api.yaml
```

### Interpreting output

| Severity | Meaning | Action |
| --- | --- | --- |
| `error` | Spec is invalid or breaks OAS rules | **Must fix** before proceeding |
| `warning` | Best-practice violation | Fix to reach a perfect spec |
| `info` | Informational hint | Fix where practical |
| `hint` | Minor style suggestion | Optional |

### Target state

```
0 errors, 0 warnings
```

Iterate on your `users-api.yaml` — fix each reported issue, then re-run `spectral lint` — until you reach zero errors and zero warnings.

#### Common fixes

| Warning / Error | Fix |
| --- | --- |
| `operation-operationId` | Add a unique `operationId` to every operation |
| `operation-summary` | Add a `summary` to every operation |
| `operation-tags` | Add at least one `tag` to every operation |
| `info-contact` | Add an `info.contact` block |
| `oas3-unused-component` | Remove or reference orphaned `components` entries |
| `no-$ref-siblings` | Move sibling properties inside the `$ref` target schema |

---

## Step 3 — Mock and Simulate the API

### Install Prism

```bash
npm install -g @stoplight/prism-cli
```

### Start the mock server

```bash
prism mock users-api.yaml
```

Prism reads your spec and spins up a local HTTP server (default: `http://127.0.0.1:4010`).

### Test the mock endpoints

Use `curl`, Postman, Insomnia, or any HTTP client.

#### List all users

```bash
curl http://127.0.0.1:4010/users
```

#### Get a user by ID

```bash
curl http://127.0.0.1:4010/users/u-001
```

#### Trigger a 404 response

```bash
curl -i http://127.0.0.1:4010/users/does-not-exist
```

> Prism returns example values defined in your spec. If no `example` is set, it generates random values that match the schema type.

### Prism flags worth knowing

| Flag | Purpose |
| --- | --- |
| `--errors` | Return `422` for requests that don't match the request schema |
| `--dynamic` | Generate dynamic (randomised) response data instead of static examples |
| `--port 5000` | Change the default port |

Example with validation enabled:

```bash
prism mock users-api.yaml --errors
```

---

## Summary

```
Author  →  users-api.yaml          (follow OAS 3.x standards)
Validate →  spectral lint           (reach 0 errors, 0 warnings)
Mock     →  prism mock              (simulate real API behaviour)
```

Once all three stages pass cleanly, the spec is considered **production-ready** and can be shared with consumers, published to a developer portal, or used to generate server/client stubs.
