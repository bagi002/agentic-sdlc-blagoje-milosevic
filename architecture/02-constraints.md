# Chapter 2: Architecture Constraints

## 2.1 Technical Constraints

| ID | Constraint | Rationale |
|----|------------|-----------|
| TC-01 | Backend is **Python 3.12** | Course-mandated language version |
| TC-02 | REST API framework is **Flask** | Chosen for its simplicity and low boilerplate for small REST APIs |
| TC-03 | Frontend is **React 18** SPA | Course-mandated UI technology |
| TC-04 | Database is **SQLite** | Chosen to avoid running a separate database server; sufficient for the scale of this kata |
| TC-05 | Testing uses **pytest** and **pytest-bdd** | Course-mandated test stack |
| TC-06 | All services run in **Docker** containers | Reproducible local dev and CI environments |
| TC-07 | **No AWS** | AWS is not available for this project |

## 2.2 Organisational Constraints

| ID | Constraint | Rationale |
|----|------------|-----------|
| OC-01 | Architecture is documented using the **arc42** template | Course requirement |
| OC-02 | Architecture diagrams follow the **C4 model** and are written in **PlantUML** | Course requirement |
| OC-03 | Domain design follows **Domain-Driven Design** principles | Course requirement |
| OC-04 | Behavioural requirements are expressed as **BDD scenarios** (Gherkin) | Enables pytest-bdd test generation |

## 2.3 Conventions

| ID | Convention | Detail |
|----|------------|--------|
| CV-01 | API responses use **JSON** | Content-Type: application/json for all endpoints |
| CV-02 | HTTP status codes follow **RFC 9110** semantics | 200 OK, 201 Created, 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 409 Conflict, 422 Unprocessable Entity |
| CV-03 | Domain events are named in **past tense** | e.g. `CardPlayed`, `TurnEnded`, `PlayerDamaged`, `GameWon` |
| CV-04 | All game logic lives in the **domain layer**, isolated from Flask and SQLite | Keeps the core testable without I/O |
| CV-05 | SQLite schema uses **explicit migrations** | Reproducible DB state across environments |
