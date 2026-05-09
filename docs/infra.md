# TTR Infrastructure and Deployment

## 1. Recommended Approach

For the current scope, the recommended infrastructure model is:

- Monorepo
- Modular monolith application backend
- Dockerized services
- Self-managed AWS EC2 deployment
- GitHub Actions for CI/CD
- Ansible for server provisioning and deployment automation
- Nginx as the reverse proxy/web server layer where needed

This is preferred over early microservices or Kubernetes because the product is still one primary bounded domain and the team will move faster with simpler deployment and operational overhead.

Platform direction:
- Infrastructure and deployment patterns should support reuse across future clients
- The AI runtime and backend services should stay shared unless clear isolation or scaling needs justify separation

## 2. Repository and Service Model

### Repository Model

Use a monorepo for:

- Customer web app
- Staff/admin web app
- Backend API
- Background worker
- Shared API schemas and contracts
- Shared tool definitions and validation logic
- Infrastructure automation and deployment configs
- Tenant/client configuration where appropriate

Recommended high-level structure:

```text
repo/
  apps/
    customer-web/
    staff-web/
    api/
    worker/
  packages/
    shared-schemas/
    shared-tooling/
  infra/
    ansible/
    docker/
    github-actions/
```

### Service Model

Use a modular monolith for the main product backend:

- Booking
- Scheduling
- Auth / verification
- Notifications
- Audit logging
- Customer / dog / groomer records

Tenant-awareness should be built into the shared platform from the start:

- Tenant/client identifiers in core records, logs, jobs, and notifications
- Client-specific configuration for branding, policies, and knowledge
- Shared runtime and codebase unless isolation requirements later justify splits

Deploy supporting runtimes separately where operational characteristics differ:

- `api`: FastAPI HTTP/WebSocket service
- `worker`: background jobs and delayed notifications
- `ai-serving`: Llama-family model runtime, served by Ollama initially or vLLM later

This keeps the codebase cohesive while still allowing runtime separation where it matters.

## 3. Deployment Topology

Recommended AWS shape:

- `ALB` (AWS Application Load Balancer) in front of application instances
- `Nginx` on application hosts as the local reverse proxy/web server layer
- `2+ EC2 instances` for the API across availability zones where practical
- `1+ worker instance` or worker process group
- `1+ dedicated AI-serving instance` if inference is self-hosted
- `PostgreSQL` preferably managed rather than self-hosted
- `Redis` preferably managed rather than self-hosted

Recommended traffic shape:

```text
Internet
  -> ALB
  -> EC2 instance(s)
  -> Nginx
  -> web and API containers

Background jobs
  -> Worker EC2 instance(s)
  -> PostgreSQL / Redis / notification providers

AI runtime
  -> Dedicated AI-serving EC2 instance(s)
  -> API tool-calling path
```

If cost or operational simplicity requires a smaller initial footprint, the first deployment can be reduced, but the application should not assume a single-host architecture.

Initial hosting assumption:

- Customer web, staff web, and API may be co-hosted on the same EC2 application instances at the start
- Nginx can proxy web traffic, API traffic, and WebSocket traffic to the correct container/service
- Worker and AI-serving runtimes should remain separately deployable even if early environments are smaller

## 4. Docker Strategy

Docker should be used for all application services.

Recommended container split:

- `customer-web`
- `staff-web`
- `api`
- `worker`
- `ai-serving` where applicable

Benefits:

- Consistent local and server runtime
- Repeatable builds in CI/CD
- Cleaner deploy automation with Ansible
- Easier rollback and version pinning

Guidance:

- One Docker image for `api`
- One Docker image for `worker`, built from the same codebase with a different runtime command
- Separate image/runtime for `ai-serving`
- `customer-web` and `staff-web` can be built as static frontend artifacts and served behind Nginx
- Nginx may run on the host or in its own container depending on the deployment layout
- Environment-specific configuration injected at deploy time, not baked into images

## 5. CI/CD

### Recommended Tooling

- `GitHub Actions` for CI/CD workflows
- `Ansible` for server provisioning and deployment automation

### CI Pipeline

On pull requests:

- Run linting
- Run tests
- Build Docker images
- Validate migrations and application startup where practical

On merge to main:

- Build versioned Docker images
- Push images to a container registry such as AWS ECR
- Deploy automatically to `staging`

Production release:

- Manual approval gate in GitHub Actions
- Ansible rollout to production EC2 hosts
- Controlled Alembic migration step
- Post-deploy health verification

## 6. Ansible Role

Use Ansible for:

- EC2 host bootstrap
- Docker and runtime dependency installation
- Environment file and secret wiring
- Nginx reverse proxy / service config
- Deployment rollouts
- Restart/reload procedures

Ansible should not replace CI/CD. It should be the execution layer for repeatable host configuration and deployment steps, triggered by GitHub Actions or by controlled release workflows.

## 7. Database Migrations

Recommended migration tooling:

- `Alembic` for PostgreSQL schema migrations

Guidance:

- Keep migration files in the application repo
- Run migrations through CI/CD release workflows, not manually from developer laptops
- Treat migrations as versioned application artifacts
- Validate forward migration in `staging` before production rollout
- Prefer additive/compatible schema changes where possible to reduce deploy risk

## 8. Secrets, KMS, and Machine-to-Machine Credentials

Recommended approach:

- Use AWS KMS-backed secret storage/protection for sensitive runtime configuration
- Avoid storing raw machine-to-machine secrets in repo files, AMIs, or ad hoc server config
- Inject secrets into services at boot/deploy time through config/bootstrap tooling

Recommended usage:

- Database credentials
- Redis credentials
- API signing keys
- Internal service credentials
- Third-party provider secrets for telephony, messaging, STT, and TTS

For machine-to-machine authentication:

- Prefer short-lived credentials or signed tokens where practical
- Use KMS-backed key material or secret-store managed credentials rather than long-lived shared plaintext tokens
- Rotate credentials through controlled operational workflows
- Audit which service can access which secret

Recommended implementation pattern for this project:

- Secrets/keys are stored in a KMS-backed system
- A bootstrap/config mechanism retrieves and injects environment-specific values into services when they start
- Service-to-service credentials should still be scoped, rotated, and audited even if injected through environment configuration

KMS is not itself the full token-management layer, but it should anchor encryption and protection of the secrets/keys used by service-to-service authentication.

## 9. Environments

Maintain separate:

- `dev`
- `staging`
- `production`

Expectations:

- `dev`: fast iteration, local Docker-based workflows, test credentials
- `staging`: production-like topology for integration testing
- `production`: isolated secrets, hardened networking, monitored services

Do not share databases or secrets across environments.

## 10. Monitoring and Observability

Recommended monitoring stack:

- `Prometheus` for metrics collection
- `Grafana` for dashboards and visualization
- Centralized log aggregation for application and infrastructure logs
- Alerting for API failures, job failures, host health, and dependency outages

Recommended monitoring coverage:

- API latency, error rate, throughput, and health check status
- Worker queue depth, retry counts, job failures, and execution latency
- PostgreSQL connection health and resource pressure
- Redis connectivity and memory pressure
- EC2 host CPU, memory, disk, and network health
- AI-serving latency, availability, and inference load
- Third-party provider failures for telephony, messaging, STT, and TTS

Recommended dashboard groups:

- Application health
- Background job health
- Infrastructure health
- AI runtime health
- Third-party integration status

Monitoring should be treated as production infrastructure, not as a later nice-to-have.

## 11. Scaling and Clustering

The system should be designed for horizontal scaling, but full clustering should not be treated as a blanket day-one requirement.

Recommended initial stance:

- `API`: yes, support horizontal scaling behind an ALB from the beginning
- `Worker`: scale worker count based on queue pressure
- `AI serving`: keep separately deployable and scale only when inference demand justifies it
- `Redis`: start simple, add failover/replication only if it becomes operationally critical
- `PostgreSQL`: prioritize backups, restore, and managed failover before complex self-managed clustering
- Docker containers
- EC2 instances
- Ansible-managed rollout
- Load balancer for API scaling

Introduce heavier clustering/orchestration only when traffic, uptime, or team structure clearly requires it.

## 12. Security and Secrets

Infrastructure expectations:

- Secrets stored outside source control
- Separate secrets per environment
- KMS-backed protection for sensitive keys and secrets
- TLS at public entrypoints
- Nginx and public entrypoints configured with least-exposed routing and header handling
- Restricted network access to PostgreSQL and Redis
- SSH access limited and auditable
- Principle of least privilege for CI/CD and runtime credentials

## 13. Observability and Recovery

Minimum production operations baseline:

- Centralized logs
- Prometheus metric collection
- Grafana dashboards
- Error monitoring
- Job failure alerts
- API health checks
- Worker health checks
- Backup and restore procedures for PostgreSQL
- Basic rollback procedure for failed deploys

Recommended operational rule:

- A release is not complete unless it is observable and recoverable

## 14. Future Evolution

Revisit infrastructure design later if one or more of these become true:

- AI inference traffic materially exceeds booking API traffic
- Separate teams own different runtime areas
- Multi-tenant scale introduces stronger isolation requirements
- Availability requirements justify more advanced failover and orchestration

At that point, consider:

- Stronger Redis/PostgreSQL failover strategy
- More formal service decomposition
- More advanced container orchestration
- Separate deployment pipelines per runtime domain
