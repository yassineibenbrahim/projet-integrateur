# Milestone 1 — Software engineering (Design and Technical Stack)

Owners: Titouan Mathieu (infrastructure) and Erwan Derrien (application / frontend).

## 1. Functional scope and priorities

**MVP**

1. An authenticated researcher uploads a **CSV file** on the website.
2. Rows are **preprocessed** (cleaning, normalization, language detection, dates) and stored as *preprocessed data*.
3. The pipeline **analyzes** each item with the **DIMA** and **DISARM** frameworks (rule-based lexicon) and computes a **severity score**. The result is stored as *analyzed data*.
4. The frontend shows the **list of items with their score**, sortable **by severity** and **by publication date**.

**Priorities after the MVP**

| Priority | Feature |
|---|---|
| 1 | **Online data collection (scraping)**, legal and compliant (see section 4) |
| 2 | Machine learning model to improve DIMA / DISARM classification |

## 2. Diagrams

| Diagram | File | Use |
|---|---|---|
| Architecture v1 (history) | [UML/ApplicationArchitecture_v1.puml](UML/ApplicationArchitecture_v1.puml) | First version (C4, Streamlit frontend) |
| Class diagram | [UML/ClassDiagram.puml](UML/ClassDiagram.puml) | The 3 data types stored in PostgreSQL |
| Deployment diagram (full) | [UML/DeploymentDiagram.puml](UML/DeploymentDiagram.puml) | Full reference: every service, port and flow |
| Deployment overview (slides) | [UML/DeploymentOverview_presentation_milestone_1.puml](UML/DeploymentOverview_presentation_milestone_1.puml) | Oral presentation: backend collapsed into one block |
| Backend detail (slides) | [UML/DeploymentBackend_presentation_milestone_1.puml](UML/DeploymentBackend_presentation_milestone_1.puml) | Oral presentation: inside the backend |
| CI/CD pipeline | [UML/CICDPipeline.puml](UML/CICDPipeline.puml) | From Pull Request to deployment |

### Data stored in the database (class diagram)

| Group | Classes | Role |
|---|---|---|
| Researcher accounts | `User`, `Role` | Access restricted to researchers. An admin creates the accounts. |
| Preprocessed data | `CollectionQuery`, `Import`, `PreprocessedData` | An `Import` is one uploaded CSV or one run of an online collection query, with its processing status. |
| Analyzed data | `AnalyzedData`, `Detection`, `Technique` | Severity score, detected DIMA / DISARM techniques and their confidence. |

## 3. OpenStack layout (ens0insa-tp6-04)

Resources: 6 instances, 24 vCPU, 24 GB RAM, 2 floating IPs. Each VM runs Ubuntu with Docker and starts its containers with `docker compose`.

| VM | Services | vCPU | RAM | Floating IP | Rationale |
|---|---|---|---|---|---|
| VM1 `mars-web` | Nginx (reverse proxy, TLS) + React build | 2 | 2 GB | **#1** (public) | Single entry point of the website |
| VM2 `mars-api` | FastAPI (auth, upload, results) | 3 | 3 GB | — | Moderate load, isolated from the Internet |
| VM3 `mars-scraper` | Scraping service + scheduler | 4 | 4 GB | — | Isolates outbound traffic; a headless browser needs RAM |
| VM4 `mars-analysis` | Preprocessing + DIMA / DISARM analysis (+ ML later) | 8 | 8 GB | — | CPU-heavy batch processing |
| VM5 `mars-db` | PostgreSQL + Cinder volume | 4 | 4 GB | — | Isolated data, backups on a volume |
| VM6 `mars-ops` | SSH bastion, GitHub runner, Ansible, monitoring | 3 | 3 GB | **#2** (admin) | Administration and deployment access |
| **Total** | | **24** | **24 GB** | **2** | |

**Why a dedicated scraping VM?**

- **Network isolation:** the scraper is the only backend service that needs outbound Internet access. Security groups allow outbound HTTPS on VM3 only. It goes through the OpenStack router, so no floating IP is needed.
- **Resource isolation:** a headless browser (Playwright) uses a lot of RAM. A scraping spike will not slow down the API or the analysis.
- **Priority:** scraping comes before ML, so ML shares the analysis VM until it needs more resources.

### Security: access restricted to researchers

- **No public sign-up:** an admin creates the accounts (`Role.ADMIN`).
- Passwords hashed with **Argon2** (or bcrypt). Authentication with a short-lived **JWT** in an `HttpOnly` + `Secure` cookie.
- Every API route (except `/auth/login` and `/health`) requires a valid token.
- **HTTPS only** (Nginx, TLS certificate). Port 80 redirects to 443.
- **OpenStack security groups:** only VM1 is exposed (443). VM6 accepts SSH only from the team's IPs. PostgreSQL accepts only VM2, VM3 and VM4.
- Secrets (DB password, JWT key) live in **GitHub Secrets** and in `.env` files on the VMs, never in the repository.
- Size limit and schema validation for uploaded CSV files in the API.

## 4. Legal and compliant online data collection

| Rule | How we apply it |
|---|---|
| Prefer official APIs | YouTube Data API, Reddit API, Mastodon API, Bluesky (AT Protocol). Browser-based scraping only for public pages without an API. |
| Respect terms of service and `robots.txt` | The scraper reads `robots.txt` before visiting a site. Platforms whose terms forbid scraping are excluded. |
| Public data only | No login-protected content, no private groups or messages. |
| Rate limiting | Request limits per domain and per API quota, with an explicit User-Agent identifying the research project. |
| GDPR: data minimization | Only the fields needed for analysis are stored. Authors are pseudonymized (`authorPseudonym` = hash). |
| Retention | Raw collected data deleted after a defined period (to be set with the supervisors). |
| Traceability | Every collection is linked to a `CollectionQuery` and an `Import` (who, when, what). |

## 5. Choice / alternative / rationale

| Area | Choice | Alternative(s) | Rationale |
|---|---|---|---|
| Frontend | **React** (Vite, TypeScript) | Streamlit, Vue.js | Full control over the interface (sortable table, filters, login). Streamlit is fast for prototypes but limited for a real multi-user app with authentication. |
| Backend / API | **FastAPI** (Python) | Django REST, Flask, Node.js (Express) | Same language (Python) as scraping, the pipeline and ML. Automatic validation with Pydantic, generated OpenAPI docs, async support. Lighter than Django. |
| Database | **PostgreSQL** | MongoDB, SQLite | Relational data (user → import → items → analyses). Efficient SQL sorting and filtering (severity, date). `JSONB` for variable CSV metadata. SQLite handles several concurrent services poorly. |
| ORM / migrations | **SQLAlchemy + Alembic** | Django ORM, raw SQL | Standard with FastAPI. Versioned migrations applied automatically on deployment. |
| Architecture | **Microservices** (web, api, scraper, analysis, db) | Monolith | Each service evolves and deploys on its own. Scraping and analysis, the heaviest parts, get their own VMs. Fits the 6 instances. |
| Service communication | **HTTP / REST** + import status table in the database | Message queue (RabbitMQ, Redis + Celery) | Enough for MVP volume and easier to debug. `Import.status` tracks progress. A queue can be added if volume grows. |
| Scraping | **Official API clients + Playwright** for public pages | Scrapy, BeautifulSoup only | APIs give clean, legal data. Playwright handles JavaScript-rendered pages that BeautifulSoup cannot read. |
| AI analysis | **Rules (lexicon) first, then ML** (scikit-learn) on **DIMA / DISARM** | LLM only, or rules only | Rules give explainable results from the MVP onward. ML improves classification later using annotations. Far lower cost and footprint than an LLM (eco-design). |
| Containers | **Docker + docker compose** on each VM | Kubernetes (k3s), Docker Swarm | Easy for the team to learn, suited to 6 VMs. Kubernetes would be too heavy for the project size. |
| Cloud | **OpenStack** (ens0insa-tp6-04) | Public cloud (AWS, Azure) | Resources provided by the school, at no cost. |
| Reverse proxy | **Nginx** | Traefik, Caddy | Serves the React build and routes `/api` to FastAPI from a single public IP. Handles TLS. Well documented. |
| Authentication | **JWT + admin-created accounts** | Keycloak, OAuth (Google) | Meets the need (researchers only) without hosting an extra service. |
| CI/CD | **GitHub Actions** + self-hosted runner | GitLab CI, Jenkins | Built into the GitHub repository, free. The runner on `mars-ops` reaches the private VMs without opening any inbound port. |
| Image registry | **GHCR** (GitHub Container Registry) | Docker Hub, private registry on OpenStack | GitHub's built-in Docker image storage. Works directly with GitHub Actions, no rate limits for our use, nothing extra to host. |
| Deployment | **Ansible** | SSH scripts, Terraform alone | Repeatable, idempotent deployment on several VMs. Terraform / OpenTofu could create the VMs later. |
| Tests | **pytest** (Python), **Vitest** (React), integration tests with docker compose | unittest, Jest | Standard, fast tools. Integration tests check the full CSV → analysis → results path. |

## 6. Target CI/CD pipeline

Diagram: [UML/CICDPipeline.puml](UML/CICDPipeline.puml). Tool: **GitHub Actions**.

### Branches

- `main`: always deployable, protected (no direct push).
- `feature/*`: one branch per Jira ticket, merged through a Pull Request after review (1 approval).

### Stage 1 — Continuous integration (every Pull Request)

| # | Job | Tools | Blocking |
|---|---|---|---|
| 1 | Python lint and typing | ruff, mypy | Yes |
| 2 | Python unit tests (api, scraper, analysis) | pytest + coverage | Yes |
| 3 | React lint | ESLint, Prettier | Yes |
| 4 | React unit tests | Vitest | Yes |
| 5 | Dependency audit | pip-audit, npm audit | Yes (critical vulnerabilities) |
| 6 | Build Docker images | docker build (one Dockerfile per service) | Yes |
| 7 | Integration tests | docker compose + test PostgreSQL: sample CSV → analysis → results | Yes |

Jobs 1 to 5 run in parallel. Only changed services are tested (path filters).

### Stage 2 — Continuous delivery (merge on `main`)

1. Rebuild the images, tagged with the **commit SHA** (and `latest`).
2. Push them to **GHCR**.
3. **Manual approval** through the GitHub `production` environment (one team member approves).

### Stage 3 — Automated deployment on OpenStack

Run by the **self-hosted GitHub runner on `mars-ops`**:

1. Ansible connects to each VM over the private network.
2. `docker compose pull`, then `docker compose up -d` with the new tag.
3. Database migrations with **Alembic**.
4. **Smoke tests**: `/health` on every service, login with a test account, upload of a small CSV.
5. On failure: automatic **rollback** to the previous tag.

### Target repository structure

```text
MARS/
├── frontend/           # React (Vite, TypeScript) + Dockerfile
├── api/                # FastAPI + Dockerfile
├── scraper/            # Online collection (APIs + Playwright) + Dockerfile
├── analysis/           # Preprocessing + DIMA / DISARM analysis + Dockerfile
├── ml/                 # Training and inference (later phase)
├── infra/              # Ansible, docker-compose per VM, Nginx config
├── .github/workflows/  # ci.yml, cd.yml
└── docs/
```
