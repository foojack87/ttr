# TTR AI Scheduling Architecture

## 1. Architecture Goal

Build a reliable scheduling system with AI-assisted customer interaction across web, Instagram, and phone.

The booking system itself should remain deterministic and backend-controlled. The AI agent should help users interact naturally, but it should not be the source of truth.

The system must also support:

- Variable appointment durations based on service and dog
- Shared update capability for both staff and owners
- Channel-specific customer verification before sensitive reads or writes
- A future in-house LLM based on Llama without coupling core business logic to any one model
- Reusable multi-tenant platform

## 2. High-Level Flow

```text
Customer Channels
  -> AI Agent Orchestrator
  -> FastAPI Booking API
  -> PostgreSQL
  -> WebSocket Events / Notifications
```

Staff and owners use a separate dashboard connected to the same FastAPI backend.

All booking mutations must pass through authenticated backend APIs that enforce scheduling, authorization, and audit rules.

## 3. Main Components

### Customer Channels

Supported channels:

- Customer Web UI
- Instagram DM
- Phone call handled by AI voice agent

The web and Instagram channels provide text input directly to the AI agent. The phone channel uses speech-to-text before sending text to the AI agent.

### AI Agent Orchestrator

The AI agent is responsible for:

- Understanding customer intent
- Asking clarification questions
- Calling backend tools
- Producing customer-friendly replies
- Logging interactions for future fine-tuning

The AI agent should not directly access PostgreSQL.

The AI agent also should not make trust decisions on its own. Identity verification, authorization, and final scheduling validation belong to FastAPI.

The intended operating model is bounded autonomy:

- The agent is autonomous in conversation flow and clarification
- The agent can choose from approved backend tools
- The agent is not allowed to invent new business actions outside approved tool boundaries
- Sensitive actions such as booking changes, payment changes, or protected status access must still pass backend verification and confirmation rules

The AI layer should also be reusable across clients:

- Core orchestration, tool-calling, and safety rules should be shared
- Client-specific prompts, policies, branding, and knowledge should be injected through configuration
- Client onboarding should not require forking the core agent runtime unless there is a true product-domain difference

### LLM Runtime

Initial runtime:

- Ollama for fast local/private deployment of a Llama-family model

Future runtime:

- vLLM for higher-throughput production inference
- Future in-house LLM derived from Llama when available

Important distinction:

- Llama is the model family/base model direction
- Ollama or vLLM are serving runtimes
- The architecture should stay compatible with either runtime so long as the model adapter contract stays stable

### FastAPI Backend API

The backend owns all business rules:

- Create booking
- Reschedule booking
- Cancel booking
- Check availability
- Assign groomer
- Update dog status
- Mark payment as paid/unpaid
- Trigger reminders and notifications
- Verify customer identity for channel-specific actions
- Enforce role permissions and write audit records

### Scheduling Service

The scheduling service must be an explicit backend subsystem rather than a generic availability check.

It is responsible for:

- Calculating appointment duration from service type, dog attributes, and configured business rules
- Determining groomer capacity and overlap eligibility
- Reserving or moving appointments atomically
- Preventing race conditions and double-booking
- Applying buffer time, lateness, and reassignment policies

Recommended scheduling rules:

- Treat the published times such as `09:00`, `10:00`, `13:00`, and `14:00` as business-facing preferred start windows, not as proof that all appointments are fixed-length
- Use PostgreSQL transactions and constraints to protect booking invariants
- Make booking and reschedule requests idempotent so retries from AI, telephony, or webhook systems do not create duplicate bookings

### PostgreSQL

PostgreSQL stores core business data:

- Customers
- Dogs
- Groomers
- Bookings
- Appointment notes
- Payment status
- Grooming status history
- Audit history for sensitive changes

Important data concerns:

- Keep booking state, payment state, and operational audit data in the source-of-truth database
- Model training datasets should be derived from reviewed logs, not embedded as raw transactional tables

### Redis

Redis is used for:

- WebSocket Pub/Sub
- Background job queues
- Reminder queues
- Session/cache support
- Short-lived conversation state where needed

### WebSocket Gateway

WebSockets are used for real-time updates:

- Dog grooming status changed
- Booking created
- Booking cancelled
- Booking rescheduled
- Payment marked paid
- Appointment running late

This supports the user story where customers can check their dog’s grooming status and staff can see updates in real time.

WebSocket subscriptions should be authorized per actor and scoped to only the data each user is allowed to view.

### Background Worker

A Celery or RQ worker processes scheduled jobs:

- 48-hour appointment reminders
- 4-week rebooking reminders
- Notification retries
- Data cleanup/export jobs

### Notification Service

The notification service sends:

- SMS reminders
- Email reminders
- Instagram replies
- Booking confirmations
- Rebooking prompts

The notification layer should also handle:

- Delivery status tracking
- Retry with deduplication
- Opt-out / consent rules where applicable
- "Still valid?" checks before sending delayed reminders

### Phone AI Flow

Phone booking requires both speech-to-text and text-to-speech:

```text
Customer speaks
  -> Telephony Provider
  -> Speech-to-Text
  -> AI Agent
  -> Identity / verification check when needed
  -> FastAPI Booking API
  -> AI Agent response
  -> Text-to-Speech
  -> Customer hears response
```

Phone-specific notes:

- Caller ID alone should not authorize booking changes or detailed status access
- Sensitive actions should require an extra confirmation step, with SMS OTP fallback if needed
- The voice flow should support timeout, retry, and human-fallback behavior even if initial MVP keeps this simple

### Identity and Access Control

Authentication and verification should vary by channel:

- Web: customer account session, magic link, or OTP login
- Phone: caller number plus a confirming detail such as dog name or appointment date, with OTP fallback
- Instagram: linked identity or verification handoff before allowing status access, rescheduling, or cancellation
- Staff/Admin dashboard: authenticated staff sessions with role-based access

Recommended roles:

- `customer`: limited to their own records and allowed customer-facing actions
- `staff`: daily schedule view, notes access, dog status updates, payment updates
- `owner`: all staff capabilities plus higher-privilege reporting/configuration

All sensitive mutations should record who performed the action, over which channel, and what changed.

## 4. Privacy and Data Handling

Because the system is intended for deployment in Australia, privacy controls should be designed around the Australian Privacy Principles model even if the initial business footprint is small.

Architecture-level privacy expectations:

- Minimize collection of personal information to what is needed for booking, reminders, verification, and support
- Separate transactional booking data, business audit logs, and AI interaction/training data
- Encrypt personal data in transit and at rest
- Apply role-based access control at the application layer and avoid broad direct database access
- Define retention and deletion rules for bookings, transcripts, call recordings, and logs
- Support customer access/correction workflows for personal information held by the system
- Maintain a breach-response process suitable for Australian notifiable data breach obligations if the business falls under the Privacy Act

## 5. Deployment and Runtime Model

The system should use:

- Dockerized application services
- Self-managed AWS EC2 infrastructure
- GitHub Actions for CI/CD
- Ansible for provisioning and deployment automation
- Nginx as the reverse proxy/web entry layer where needed
- Alembic for PostgreSQL schema migrations
- AWS KMS-backed secret handling for service-to-service credentials and other sensitive runtime secrets
- Prometheus + Grafana style monitoring for services, jobs, and infrastructure
- A monorepo with a modular monolith backend
- Separate runtime deployment for API, worker, and AI-serving components where needed

This architecture is designed to scale horizontally at the API layer behind a load balancer without forcing early microservices or Kubernetes.

Detailed deployment, topology, CI/CD, scaling, and operational guidance is documented in [infra.md](infra.md).

Tenant-aware deployment expectations:

- Application services should remain reusable across clients/tenants
- Tenant identity must flow through APIs, logs, jobs, notifications, and AI interactions
- Client-specific configuration should be injected without requiring separate codebases for each client

## 6. Data Collection for Future In-House LLM

The system should log structured interaction data from day one.

Important data to capture:

- Original user input
- Channel: web, Instagram, phone, staff
- Detected intent
- Tool/function call selected
- Tool/function arguments
- API result
- Final response
- Human correction, if any
- Failure reason, if any

This creates a dataset for future fine-tuning or evaluation.

However, raw interaction logging and model-training data are not the same thing.

Operational logging should be split into:

- Business audit logs: booking changes, payment changes, dog status changes, actor, timestamp, previous value, new value
- AI interaction logs: prompts, tool calls, tool outputs, confidence/failure markers, human corrections

Training pipeline guidance:

- Curate training data from interaction logs after review and redaction
- Remove or mask unnecessary PII before adding examples to training/evaluation datasets
- Version datasets separately from production logs
- Keep the Llama fine-tuning path behind a model adapter so inference infrastructure can evolve without changing business APIs

## 7. Recommended Build Phases

### Phase 1: Core Booking System

- FastAPI backend
- PostgreSQL schema
- Customer and dog records
- Booking creation/reschedule/cancel
- Scheduling engine with duration calculation and conflict protection
- Initial Docker, EC2, CI/CD, and deployment automation setup
- Staff daily schedule view
- Manual payment status
- Manual dog grooming status updates
- Audit logging for sensitive changes
- WebSocket real-time updates

### Phase 2: AI Text Agent

- Web agent
- Tool calling layer
- Structured logging
- Channel-aware identity verification handoff
- Basic natural language booking
- Clarification flow

### Phase 3: Reminders and Notifications

- 48-hour reminders
- 4-week rebooking reminders
- SMS/email notifications
- Notification retry worker
- Operational monitoring for scheduled jobs

### Phase 4: Phone AI Agent

- Telephony integration
- Speech-to-text
- Text-to-speech
- Verification flow for sensitive actions
- Phone booking and rescheduling

### Phase 5: Multi-Client Expansion

- Client configuration system
- Per-client rules
- Per-client branding
- Per-client knowledge base
- Tenant-aware logging and reporting

Even if only one client is active at first, tenant identifiers and tenant-safe data boundaries should be designed into the schema and logs from the beginning.

### Phase 6: Future In-House LLM

- Clean logged data
- Build training/evaluation dataset
- Fine-tune base model
- Evaluate against production logs
- Swap model through LLM provider adapter

## 8. Design Principles

1. Keep AI replaceable.
2. Keep business rules in FastAPI, not prompts.
3. Use PostgreSQL as the source of truth.
4. Use WebSockets for real-time events.
5. Use Redis for Pub/Sub, queues, and sessions.
6. Make scheduling rules explicit, transactional, and idempotent.
7. Separate business audit logs from AI training data.
8. Use bounded AI autonomy: natural conversation and approved tools, but no open-ended business actions.
9. Prefer Dockerized services on a monorepo/modular-monolith foundation until scale or ownership pressures justify service splits.
10. Log every agent decision for future model improvement.
11. Start with one client, but design for multi-tenant expansion.
