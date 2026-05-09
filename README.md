# TTR AI Scheduling System

This repository contains the initial technical architecture for an AI-assisted scheduling product for TTR.

The platform should be designed so core scheduling, communications, and AI orchestration capabilities can be reused for additional clients over time.

The system supports customer booking and appointment management through multiple channels:

- Web agent
- Instagram DM
- AI phone answering
- Staff/admin dashboard

The core principle is:

> The AI agent is the interface. The FastAPI backend is the source of truth.

This means:

- The backend owns scheduling rules, identity checks, permissions, and audit history
- The AI agent can assist with conversation and tool selection, but it cannot bypass backend validation
- Production audit logs and future LLM training data are related but separate concerns

## Platform Direction

- multi-tenant/client on a reusable platform
- Core backend logic should be tenant-aware rather than client-specific
- The AI layer should be reusable across clients through shared tooling and orchestration
- Client-specific behavior should come from configuration, policies, branding, and knowledge, not hard-coded forks wherever possible

## Proposed Tech Stack

| Layer | Technology | Purpose |
| --- | --- | --- |
| Customer UI | React web app | Web booking agent, appointment status, reschedule/cancel flows |
| Staff/Admin UI | React web app | Daily appointment view, dog status updates, payment markers |
| Backend API | Python FastAPI | Booking, customers, dogs, groomers, scheduling rules |
| Database | PostgreSQL | Core business data and booking records |
| Realtime | FastAPI WebSockets | Live booking/status/payment updates |
| Event Bus / Cache / Jobs | Redis | Pub/Sub, session cache, reminder queues |
| Background Jobs | Celery or RQ | 48-hour reminders, 4-week rebooking reminders, notification retries |
| AI Serving Runtime | Ollama initially or vLLM later | Serve open-source models in local/private infrastructure |
| Base Model Direction | Llama-family model | Base model direction for future in-house fine-tuning |
| Future Model | In-house LLM derived from Llama | Fine-tuned/custom model trained from collected data |
| Phone | Twilio / Vonage / SIP provider | Receive calls and connect to AI voice flow |
| Speech-to-Text | Whisper / provider STT | Convert customer voice to text |
| Text-to-Speech | ElevenLabs / provider TTS / open-source TTS | Convert AI response back to voice |
| Notifications | SMS / Email / Instagram | Appointment reminders, confirmations, rebooking prompts |

## Suggested MVP Scope

### Customer Features

- Book appointment
- Reschedule appointment
- Cancel appointment
- Receive 48-hour reminder
- Receive 4-week rebooking reminder if no future booking exists
- Check dog grooming status in real time

Customer access should be verified per channel:

- Web: account login, magic link, or OTP
- Phone: caller ID plus confirmation step, with SMS OTP fallback for sensitive actions
- Instagram: linked customer identity or verification link / OTP before booking changes or status access

### Business Owner Features

- Auto-validate appointment availability
- Auto-assign bookings to groomers based on rules
- Mark bookings as paid/unpaid
- View customer and dog history

Scheduling is not based on fixed-length appointments only. Appointment duration varies by dog and service, so the backend must calculate duration, enforce overlap rules, and reserve capacity atomically.

### Staff Features

- View daily schedule
- View appointment notes
- Update dog grooming status
- See payment status clearly

Staff and owners can both update dog status and payment status, but permissions and audit history must be enforced by the backend.

## Key Architecture Rule

The LLM should not directly modify the database.

Instead:

1. The AI agent interprets the user request.
2. The agent calls controlled backend tools/APIs.
3. FastAPI validates identity, permissions, scheduling rules, and business constraints.
4. FastAPI writes to PostgreSQL.
5. FastAPI writes an audit trail and emits domain events.
6. WebSocket events notify connected clients.

## Key Design Additions

### Scheduling Engine

- Appointment duration is computed from service type, dog attributes, and business rules
- Daily scheduling may still use preferred start windows such as `09:00`, `10:00`, `13:00`, and `14:00`, but booking acceptance must be based on computed duration and groomer capacity
- Booking, reschedule, and cancellation flows must be atomic and idempotent to prevent double-booking across web, phone, and Instagram
- Buffer time, overlap policy, and reassignment rules belong in backend configuration, not prompts

### Roles and Permissions

- `customer`: can view and manage only their own bookings and dog status data allowed by policy
- `staff`: can view daily schedules, appointment notes, and update dog and payment status
- `owner`: can perform all staff actions and manage higher-risk configuration/reporting actions

### AI Operating Model

- The AI agent is autonomous in conversation
- The AI agent can choose from approved backend tools
- The AI agent is not allowed to invent unapproved business actions
- Sensitive actions still require backend validation, and may require confirmation or verification

### Logging Strategy

- Business audit logs record who changed bookings, payment state, and dog status
- AI interaction logs record prompts, tool calls, tool results, failures, and corrections
- Training datasets for the future in-house LLM are curated from interaction logs after review/redaction, not copied directly from raw production traffic

## Privacy and Data Handling

Because the system is intended for deployment in Australia, it should be designed to align with the Australian Privacy Principles model from day one.

- Collect only the personal information needed for booking, reminders, verification, support, and audit
- Keep booking data, audit data, and AI/training data as separate concerns
- Restrict access by role and channel, not by broad shared database visibility
- Support retention, deletion, access, and correction workflows for customer data
- Review overseas vendors and hosted services for cross-border personal data handling
- Prepare for notifiable data breach response even if the initial deployment is small

## Deployment and Delivery

- Use Docker for application services
- Deploy onto self-managed AWS EC2 infrastructure
- Use GitHub Actions for CI/CD and Ansible for provisioning/deployment automation
- Use Nginx as the reverse proxy/web server layer on EC2 hosts where needed
- Use Alembic for PostgreSQL schema migrations
- Use AWS KMS-backed secret management for machine-to-machine credentials and sensitive application secrets
- Use Prometheus + Grafana style monitoring for service, job, and infrastructure visibility
- Separate `dev`, `staging`, and `production` environments
- Use a monorepo and start with a modular monolith, not early microservices
- Scale API instances horizontally behind a load balancer; scale workers and AI-serving separately as needed

Detailed infrastructure guidance lives in [Infrastructure](docs/infra.md).

## Documents

- [Architecture](docs/architecture.md)
- [Infrastructure](docs/infra.md)
- [Mermaid Diagram](docs/architecture.mmd)
