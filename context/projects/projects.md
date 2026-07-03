# Projects

*Is a detailed list of project items that would look good on a resume. In terms of software engineering roles, it would be: ordered from most to least impressive — weighted by scope, system complexity, team coordination, and breadth of skills demonstrated.*

---

## OpenJudge — Educational Code Evaluation Platform
[github.com/UniversalTze/OpenJudge](https://github.com/UniversalTze/OpenJudge)

**Tech stack:** Python (FastAPI), Celery, PostgreSQL, Docker, Terraform, Task, AWS (ECS, ECR, SQS, RDS, S3, ElastiCache), message queue–based microservices

A team of 6 engineers built an MVP coding-judge platform that reimagines code evaluation around full test-case transparency for educational use, in contrast to opaque competitive-programming platforms. The system is structured as a microservices architecture with asynchronous queue-based communication between services, enabling sandboxed code execution to remain fully isolated from the rest of the platform. I owned the Problem service and contributed to the Submission service, designing RESTful APIs for problem management and submission handling, with submissions dispatched to a Celery-backed worker queue for downstream processing by the language-specific execution services. I also provisioned the AWS infrastructure using Terraform — automating containerised deployment across ECS, RDS, SQS, and supporting services — and produced architecture and deployment diagrams covering service interactions and topology. The project deepened my experience in microservice design, asynchronous workflows, RESTful API design, infrastructure-as-code, and reasoning about quality-attribute trade-offs (security vs. latency, isolation vs. operational complexity) in a distributed system.

---

## CoughOverflow — Pathogen Analysis Service
[github.com/UniversalTze/CoughOverflow](https://github.com/UniversalTze/CoughOverflow)

**Tech stack:** Python (FastAPI), Celery, SQLite/PostgreSQL, Docker, Terraform, Task, AWS (ECS, ECR, SQS), k6, REST

A cloud-hosted REST API backend for a pathology analysis service that ingests pre-processed saliva sample images and detects COVID-19 and H5N1 markers using a provided analysis engine, designed to scale to epidemic-level demand. I built the FastAPI backend with Celery-backed asynchronous workers that offload long-running medical analysis jobs from the web tier to a message broker, keeping API responses fast and responsive under concurrent load. Exposed query and aggregation endpoints — including lab summaries, patient result lookups, and status/outcome counts — with operational filtering by urgency and disease outcome to support both individual queries and batched requests from labs and health authorities. The service persists requests and results to durable storage so analysis jobs survive service restarts or failures, and is fully containerised with Docker and deployed to AWS via Terraform across three iterative stages (containerised web server → cloud deployment → async-optimised pipeline with Celery workers). Stress-tested with k6 under a range of epidemiological surge models — from baseline operation through epidemic peak — ramping to 80 concurrent virtual users across mixed POST/GET/PUT and polling workloads to validate responsiveness for both urgent and non-urgent requests. The project reinforced my skills in async backend design, queue-based scaling patterns, infrastructure-as-code, and load testing distributed systems.

---

## Project Title

**Tech Stack** 

Context on what was built.....

---  <!-- seperator -->

<!-- repeat ## blocks up to projects N -->