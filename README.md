# Simple Feature Flag Manager

[![Python](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/downloads/)
[![Framework](https://img.shields.io/badge/FastAPI-0.100.0%2B-009688.svg)](https://fastapi.tiangolo.com/)
[![Frontend](https://img.shields.io/badge/HTMX-1.9.0%2B-336699.svg)](https://htmx.org/)
[![Code Style](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A high-performance, self-hosted feature flag management console and low-latency evaluation service. Simple Feature Flag Manager empowers engineering and product teams to decouple software deployments from feature releases, run deterministic percentage-based rollouts, and instantly toggle features in real time without downtime or redeployments.

---

## Table of Contents

- [Problem Statement & Target Audience](#problem-statement--target-audience)
- [Key Features](#key-features)
- [Architecture & System Design](#architecture--system-design)
- [Technology Stack](#technology-stack)
- [Installation & Setup](#installation--setup)
- [Environment Variables](#environment-variables)
- [Usage Guide](#usage-guide)
  - [Dashboard Interface](#dashboard-interface)
  - [REST API Endpoints](#rest-api-endpoints)
- [Automated Testing](#automated-testing)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [License](#license)

---

## Problem Statement & Target Audience

### The Problem
Deploying code directly to production without runtime controls forces software teams to tie release timing directly to deployment pipelines. This coupling introduces significant operational risks:
* **High-Risk Rollouts:** A single defect in a newly deployed feature requires a full code rollback or emergency patch release.
* **Delayed Releases:** Feature branches remain unmerged for weeks while waiting for business sign-offs or coordinated release windows.
* **Environment Divergence:** Staging and production drift apart when long-lived feature branches are maintained separately.

### Target Audience
* **DevOps & Platform Engineers:** Seeking a light-footprint, self-hosted feature toggle server with zero external SaaS dependencies.
* **Backend & Frontend Developers:** Requiring a low-latency REST API to dynamically resolve feature states based on user attributes and percentage rollouts.
* **Product Managers:** Needing an intuitive UI to perform canary launches, progressive rollouts, and kill-switch toggles safely across `development`, `staging`, and `production`.

---

## Key Features

* **Interactive Web Dashboard:** Modern, reactive UI powered by HTMX and Jinja2 templates. Create, toggle, and tune flags dynamically across multiple environments (`development`, `staging`, `production`) with zero page reloads.
* **Deterministic Evaluation Engine:** Pure Python evaluation engine using MD5 bucket hashing (`hash(flag_key + user_id) % 100`) for stable percentage rollouts, complemented by user attribute targeting rules (e.g., email domain matching, user role, region).
* **Batch Evaluation API:** High-throughput RESTful endpoints allowing client applications to query feature flag evaluations for specific user contexts in a single HTTP request.
* **Immutable Audit Logging:** Built-in tracking system capturing timestamped toggle events, percentage rollout shifts, configuration updates, and actor metadata for compliance and observability.

---

## Architecture & System Design

Simple Feature Flag Manager is designed for low latency, thread safety, and minimalist operational overhead.

```
                  +-----------------------------------+
                  |   HTMX Administrative Console    |
                  |     (Jinja2 + Tailwind/HTML)      |
                  +-----------------+-----------------+
                                    |
                                    v
+-------------------+     +-------------------+     +-------------------+
|  Client App / API | --> |   FastAPI Router   | --> | Evaluation Engine |
|   Evaluations     |     |   (src/api.py)    |     |   (src/core.py)   |
+-------------------+     +-------------------+     +---------+---------+
                                    |                         |
                                    v                         v
                          +-------------------------------------------------+
                          |           SQLAlchemy ORM Data Layer            |
                          |                 (src/models.py)                 |
                          +------------------------+------------------------+
                                                   |
                                                   v
                                          +-----------------+
                                          | SQLite Database |
                                          |   (flags.db)    |
                                          +-----------------+
```

### Module Responsibilities

* `src/models.py`: Defines SQLAlchemy ORM entities (`FeatureFlag`, `Environment`, `TargetingRule`, `AuditLog`) and Pydantic schemas for runtime validation and API serialization.
* `src/core.py`: Implements the deterministic flag evaluation logic (MD5 user hashing for percentages, attribute rule evaluation) and optimized database access helpers.
* `src/api.py`: Exposes FastAPI REST endpoints for flag management, batch evaluation, audit log queries, and HTMX reactive UI routes.
* `src/main.py`: Handles application bootstrapper tasks, SQLite table initialization, Jinja2 template engine context binding, and the Uvicorn server entrypoint.

---

## Technology Stack

* **Language:** Python 3.10+
* **Web Framework:** [FastAPI](https://fastapi.tiangolo.com/)
* **ORM & Database:** [SQLAlchemy 2.0](https://www.sqlalchemy.org/) + SQLite
* **Validation:** [Pydantic v2](https://docs.pydantic.dev/)
* **Frontend:** [Jinja2](https://jinja.palletsprojects.com/) + [HTMX](https://htmx.org/)
* **ASGI Server:** [Uvicorn](https://www.uvicorn.org/)
* **Testing:** [pytest](https://docs.pytest.org/) + `httpx`

---

## Installation & Setup

### Prerequisites
* Python 3.10 or higher installed.
* `pip` and `virtualenv` tools available.

### Setup Instructions

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-org/simple-feature-flag-manager.git
   cd simple-feature-flag-manager
   ```

2. **Create and activate a virtual environment:**
   ```bash
   python3 -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies:**
   ```bash
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

4. **Initialize Environment Variables:**
   ```bash
   cp .env.example .env
   ```

5. **Run the Application:**
   ```bash
   python -m src.main
   ```
   The application will start at `http://127.0.0.1:8000`.

---

## Environment Variables Configuration

Copy `.env.example` to `.env` and adjust the variables as required:

| Variable | Default Value | Description |
| :--- | :--- | :--- |
| `DATABASE_URL` | `sqlite:///./flags.db` | SQLAlchemy connection string for SQLite database. |
| `APP_ENV` | `development` | Runtime environment mode (`development`, `staging`, `production`). |
| `HOST` | `127.0.0.1` | Host address for Uvicorn server. |
| `PORT` | `8000` | Binding port for Uvicorn server. |
| `DEBUG` | `True` | Enable debug mode and auto-reloading. |

---

## Usage Guide

### Dashboard Interface

Access the web dashboard by navigating to `http://127.0.0.1:8000/` in your browser.

* **Flag Overview:** View all flags, target environments, rollout percentages, and active status.
* **Interactive Toggles:** Instantly activate/deactivate flags using HTMX switches without full page refreshes.
* **Targeting Rules:** Configure attribute matching (e.g., `email ENDS_WITH @company.com`).
* **Audit Trail:** Inspect real-time records of every modification event.

---

### REST API Endpoints

#### 1. Evaluate Flags for User Context
Query flag statuses for a user with contextual attributes.

* **Endpoint:** `POST /api/v1/evaluate`
* **Headers:** `Content-Type: application/json`

**Request Body:**
```json
{
  "environment": "production",
  "user_id": "usr_984123",
  "attributes": {
    "email": "developer@company.com",
    "role": "beta_tester",
    "country": "US"
  },
  "flag_keys": ["new_checkout_flow", "dark_mode_v2", "ai_recommendations"]
}
```

**Response (`200 OK`):**
```json
{
  "environment": "production",
  "user_id": "usr_984123",
  "flags": {
    "new_checkout_flow": {
      "enabled": true,
      "reason": "MATCHED_TARGETING_RULE",
      "rule_id": "rule_01H123"
    },
    "dark_mode_v2": {
      "enabled": true,
      "reason": "PERCENTAGE_ROLLOUT_BUCKET_MATCH",
      "bucket": 42
    },
    "ai_recommendations": {
      "enabled": false,
      "reason": "FLAG_GLOBALLY_DISABLED"
    }
  }
}
```

#### 2. Create a Feature Flag
* **Endpoint:** `POST /api/v1/flags`

**Request Body:**
```json
{
  "key": "new_checkout_flow",
  "description": "Enables redesigned 3-step checkout interface",
  "is_enabled": true,
  "rollout_percentage": 50,
  "environment": "production"
}
```

#### 3. Fetch Audit Logs
* **Endpoint:** `GET /api/v1/audit-logs`

**Response (`200 OK`):**
```json
[
  {
    "id": 1,
    "flag_key": "new_checkout_flow",
    "action": "TOGGLE_ENABLED",
    "actor": "admin@company.com",
    "timestamp": "2023-10-24T14:32:10Z",
    "changes": {
      "old_value": false,
      "new_value": true
    }
  }
]
```

---

## Automated Testing Instructions

The project uses `pytest` for unit, integration, and API testing.

### Running Tests

Run the full test suite:
```bash
pytest -v
```

Run core evaluation engine unit tests:
```bash
pytest tests/test_core.py -v
```

Run API endpoint integration tests:
```bash
pytest tests/test_api.py -v
```

Generate test coverage report:
```bash
pytest --cov=src --cov-report=term-missing
```

---

## Project Structure

```
simple-feature-flag-manager/
├── .env.example          # Sample environment variable settings
├── .gitignore            # Git exclusion rules
├── requirements.txt      # Python runtime & testing dependencies
├── src/
│   ├── __init__.py       # Package marker
│   ├── api.py            # FastAPI endpoints & HTMX template rendering
│   ├── core.py           # Evaluation engine algorithms & DB utilities
│   ├── main.py           # Application bootstrapper & ASGI runner
│   └── models.py         # SQLAlchemy ORM models & Pydantic schemas
└── tests/
    ├── __init__.py       # Test package marker
    ├── test_api.py       # REST API integration tests
    └── test_core.py      # Engine evaluation unit tests
```

---

## Roadmap

- [ ] **Redis Evaluation Cache:** Add distributed in-memory caching to reduce DB lookup latency under heavy loads.
- [ ] **SDK Client Libraries:** Publish lightweight SDKs for Python, Node.js, and Go.
- [ ] **Role-Based Access Control (RBAC):** Introduce fine-grained user permissions for production changes.
- [ ] **Webhooks & Slack Alerts:** Trigger notifications on toggle state changes or rollout shifts.

---

## License

This project is open-source software licensed under the [MIT License](LICENSE).