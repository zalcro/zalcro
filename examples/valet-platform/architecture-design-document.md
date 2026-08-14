# Valet Platform
## Architecture Design Document

## TL;DR
This is a mobile valet platform launching in New York, Los Angeles, and Chicago, connecting customers who need parking with drivers and garage partners through iOS, Android, and web apps. The system handles real-time driver tracking, bookings, payments, and partner management from a single AWS serverless backend. The two biggest architectural calls were going fully serverless on AWS to minimize operational overhead for a solo maintainer, and using API Gateway WebSockets for live driver location updates instead of a heavier real-time infrastructure.

## Executive Summary
The Valet Platform is a multi-sided marketplace connecting four distinct user types — customers, drivers, garage/lot partners, and internal admins — across three US cities. It handles the complete valet lifecycle: booking, dispatch, real-time tracking, payment, and partner reconciliation.

The architecture is designed around two hard constraints that shape every decision: a solo founder maintaining the system post-launch, and strict cost efficiency. This means serverless-first (no idle compute costs), managed services where available (no self-managed infrastructure), and a deployment footprint that can be operated by one person without heroics.

The stack is prescribed: React Native mobile apps, a web portal, AWS Lambda and API Gateway for the backend, WebSockets for real-time location, and RDS PostgreSQL as the primary data store. The architecture works within these constraints rather than around them.

At launch scale (50–500 concurrent jobs/day), the serverless approach is well-matched to demand — Lambda scales with actual job volume, costs stay proportional to usage, and there's no fixed infrastructure to manage during low-traffic periods. The evening availability requirement is met through Multi-AZ RDS and Lambda's inherent redundancy, without requiring a dedicated ops team.

## System Overview
The platform operates as a real-time logistics and payments system with five distinct surfaces:

1. **Customer Mobile App** (iOS/Android) — Booking flow, live driver tracking on map, payment, receipts, booking history
2. **Driver Mobile App** (iOS/Android) — Job queue, navigation, status updates, earnings, onboarding/background check flow
3. **Partner Web Portal** — Garage/lot management, job visibility, earnings reporting
4. **Admin Web Dashboard** — Ops management, driver approvals, dispute resolution, system monitoring
5. **Serverless Backend** — REST API via API Gateway + Lambda, WebSocket channel for real-time location, RDS PostgreSQL for persistent state

Core flows at launch:
- **Booking**: Customer schedules/modifies/cancels → job created in DB → driver notified → status lifecycle managed through Lambda
- **Real-time tracking**: Driver app broadcasts GPS position via WebSocket at 3–5 second intervals → Lambda broadcasts to subscribed customer sessions
- **Payments**: Stripe handles card capture (customer never touches PCI scope directly), charge on job completion, receipt generation
- **Onboarding**: Driver submits documents → TBD background check provider → admin reviews → approval unlocks job queue access
- **Surge pricing**: Lambda calculates demand multiplier per city zone at booking time based on active job density

## Architectural Constraints Summary

| Constraint | Detail |
|---|---|
| Frontend | React Native (iOS + Android), web portal for partners and admin |
| Backend | AWS Lambda + API Gateway (REST + WebSockets) |
| Database | RDS PostgreSQL (Multi-AZ) |
| AWS Region | US regions only |
| Real-time | API Gateway WebSockets, 3–5s broadcast interval |
| Build approach | Agentic workflow (Claude Code / Cursor) |
| Scale at launch | 50–500 concurrent jobs/day, 3 cities |
| Availability | High availability — no evening downtime acceptable |
| RTO | Under 1 hour for primary database failure |
| Data retention | Job records: 3 years; Driver GPS history: 30 days post-job |
| Compliance | PCI-DSS, CCPA |
| Maintainer | Solo founder post-launch |
| Cost posture | Hard constraint — minimize idle and fixed costs |

## Component Architecture

### Customer Mobile App (React Native)
- Booking creation, modification, and cancellation flows
- Live map view with driver position (consumes WebSocket feed)
- Stripe SDK integration for card management and payment confirmation
- Push notification handling (FCM/APNs)
- Low-connectivity handling: optimistic UI updates, retry logic on failed API calls, graceful degradation when WebSocket drops

### Driver Mobile App (React Native)
- Job queue view with accept/decline
- Turn-by-turn navigation via Google Maps / Apple Maps SDK
- GPS broadcast: posts location to WebSocket endpoint at 3–5s intervals
- Status update controls (en route, arrived, job complete)
- Earnings summary and history
- Onboarding flow: document upload, background check status polling

### Partner Web Portal (React — web)
- Garage/lot profile management
- Live and historical job view for their location
- Earnings dashboard and payout history
- Built as a standard React SPA, served via CloudFront + S3

### Admin Web Dashboard (React — web)
- Driver approval workflow (post background check)
- Dispute management and resolution
- System-wide job and booking visibility
- User account management
- Served via CloudFront + S3

### API Gateway (REST)
- Single entry point for all mobile and web client REST requests
- JWT-based auth enforced at Gateway level via Lambda authorizer
- Routes to appropriate Lambda functions by resource/method
- Throttling configured per route to protect downstream services

### API Gateway (WebSockets)
- Manages persistent WebSocket connections for real-time driver location
- Handles connect/disconnect/message routes via Lambda
- Connection IDs stored in RDS to map driver sessions to active jobs
- Broadcasts driver location updates to subscribed customer connections

### Lambda Functions (organized by domain)
- **Auth Lambda**: Token issuance, refresh, role validation
- **Booking Lambda**: CRUD for bookings, schedule logic, cancellation rules
- **Dispatch Lambda**: Job assignment logic, driver availability, surge pricing calculation
- **Location Lambda**: Receives driver GPS, stores to location history, broadcasts to customer WebSocket sessions
- **Payment Lambda**: Stripe charge initiation, webhook handling, receipt generation
- **Notification Lambda**: Triggers Twilio SMS and FCM/APNs push on job status changes
- **Partner Lambda**: Partner-facing job and earnings data
- **Admin Lambda**: Admin operations, driver approvals, dispute handling
- **Onboarding Lambda**: Document handling, background check provider integration, status polling

### RDS PostgreSQL (Multi-AZ)
- Primary data store for all transactional data
- Multi-AZ for automatic failover (supports RTO < 1 hour)
- Daily automated snapshots
- Connection pooling via RDS Proxy to handle Lambda cold-start connection bursts

### Supporting AWS Services
- **S3**: Document storage (driver onboarding docs), static web portal hosting, receipt PDFs
- **CloudFront**: CDN for web portals and S3-hosted assets
- **CloudWatch**: Logs, metrics, alarms for failed jobs and payment errors
- **AWS X-Ray**: Distributed tracing across Lambda functions
- **AWS Cost Explorer + Budgets**: Cost monitoring and alerting for the solo maintainer
- **Secrets Manager**: Stores API keys (Stripe, Twilio, background check provider)
- **CodePipeline / CodeDeploy**: CI/CD with Lambda alias-based rollback capability

## Data Architecture

### Core Schema (PostgreSQL)

**Users**
- `user_id`, `role` (customer/driver/partner/admin), `email`, `phone`, `created_at`
- CCPA-relevant fields flagged for deletion/export workflows

**Drivers**
- `driver_id` (FK to users), `status` (pending/approved/suspended), `background_check_status`, `background_check_provider_ref`, `onboarding_completed_at`

**Partners**
- `partner_id` (FK to users), `location_name`, `address`, `city`, `capacity`, `payout_account_ref`

**Jobs**
- `job_id`, `customer_id`, `driver_id`, `partner_id`, `city`, `status` (enum: scheduled/dispatched/en_route/arrived/in_progress/complete/cancelled), `scheduled_at`, `completed_at`, `surge_multiplier`, `base_price`, `final_price`

**Bookings**
- `booking_id`, `job_id`, `created_at`, `modified_at`, `cancellation_reason`

**Payments**
- `payment_id`, `job_id`, `stripe_payment_intent_id`, `amount`, `currency`, `status`, `receipt_url`, `created_at`

**Driver Location History**
- `location_id`, `driver_id`, `job_id`, `latitude`, `longitude`, `recorded_at`
- Partition or TTL strategy: records purged 30 days post-job completion
- Not used for real-time routing — raw history only, for ops/dispute purposes

**WebSocket Connections**
- `connection_id`, `user_id`, `role`, `job_id`, `connected_at`
- Short-lived; cleaned up on disconnect

### Retention Policy

| Data type | Retention |
|---|---|
| Job records | 3 years |
| Payment/receipt records | 3 years (PCI requirement) |
| Driver GPS history | 30 days post-job |
| WebSocket connection records | Purged on disconnect |
| User PII (CCPA) | Retained until deletion request honored |

### Backup Strategy
- RDS automated daily snapshots, retained 7 days minimum
- S3 versioning enabled for document and receipt storage
- Point-in-time recovery (PITR) enabled on RDS for granular recovery within the backup window

## Integration Architecture

### Stripe (Payments)
- Customer card details captured via Stripe SDK on device — card data never touches the platform's backend (reduces PCI scope to SAQ A-EP or better)
- Payment Intents API used for charge lifecycle
- Stripe webhooks received by Payment Lambda for async status updates (charge succeeded, refund processed, dispute opened)
- Webhook signature verification enforced
- Stripe-hosted receipts linked from platform receipt records; PDF copies stored in S3

### Google Maps / Apple Maps (Navigation + Tracking)
- Driver app uses the native maps SDK for turn-by-turn navigation
- Customer app renders driver location on map using the same SDKs
- No server-side maps API calls required for tracking — location data flows through the platform's own WebSocket pipeline
- Maps SDK API keys managed per platform (iOS/Android), stored in Secrets Manager for any server-side usage

### Twilio (SMS)
- Notification Lambda triggers Twilio REST API for SMS on key job status events: booking confirmation, driver dispatched, driver arrived, job complete
- Twilio credentials in Secrets Manager
- SMS delivery status webhooks optionally consumed for logging

### FCM / APNs (Push Notifications)
- Notification Lambda sends push payloads via FCM (Android) and APNs (iOS)
- Device tokens registered and stored against user records at login
- Push used for real-time job status changes; SMS used as fallback for critical events

### Background Check Provider (TBD)
- Onboarding Lambda submits driver PII to provider API on document completion
- Provider returns a reference ID; Lambda polls or receives webhook for result
- Result stored against driver record; admin notified for manual approval step
- Provider API credentials in Secrets Manager
- Architecture is provider-agnostic: the integration is behind an internal interface so the provider can be swapped without changing the approval workflow

## Security Architecture

### Authentication and Authorization
- JWT-based auth issued by Auth Lambda on login
- Role-based access control (RBAC) with four roles: customer, driver, partner, admin
- Lambda authorizer on API Gateway validates JWT and extracts role before routing
- Admin endpoints additionally restricted by IP allowlist where feasible
- Refresh token rotation with short-lived access tokens (15-minute expiry)

### PCI-DSS Compliance
- Card data never transits or is stored on platform infrastructure — Stripe SDK handles all card capture
- Stripe.js / Stripe SDK tokenizes card data client-side
- Backend only handles Stripe Payment Intent IDs and status
- Scope reduction targets SAQ A-EP; formal assessment recommended before launch
- Stripe webhook payloads verified using Stripe signature header on every inbound request

### CCPA Compliance
- PII fields in the user schema documented and tagged
- Data deletion workflow: soft-delete with PII nullification on request, retained job records anonymized
- Data export workflow: Lambda-driven export of user-linked records on request
- Driver GPS history auto-purged at 30-day mark via scheduled Lambda
- No third-party marketing data sharing at launch

### Secrets Management
- All API keys, database credentials, and provider secrets stored in AWS Secrets Manager
- Lambda functions granted least-privilege IAM roles — each function accesses only the secrets it needs
- No secrets in environment variables, source code, or CloudFormation outputs

### Network Security
- API Gateway as the sole public entry point — no Lambda functions exposed directly
- RDS deployed in private VPC subnet — not publicly accessible
- Lambda functions in VPC where database access is required
- S3 buckets for driver documents set to private; access via pre-signed URLs only
- CloudFront distribution with HTTPS enforced for web portals

### Operational Security
- CloudWatch alarms on: Lambda error rates, payment webhook failures, failed login spikes
- X-Ray traces for full request visibility across Lambda hops
- All admin actions logged to CloudWatch with user ID for audit trail

## Deployment Architecture

### AWS Region Strategy
- Primary region: **us-east-1** (New York as primary launch city)
- No multi-region active-active at launch — cost and operational complexity not justified at this scale
- RDS Multi-AZ within us-east-1 provides automatic failover for database-level failures (RTO < 1 hour)
- Lambda and API Gateway are inherently highly available within a single region
- CloudFront serves web assets globally from edge locations automatically

### Infrastructure as Code
- AWS SAM or CDK used to define all Lambda functions, API Gateway routes, RDS config, and supporting resources
- Enables repeatable deployments and fast rollback via CloudFormation stack updates

### CI/CD Pipeline
- CodePipeline triggers on main branch push
- Build stage: runs tests, packages Lambda deployment bundles
- Deploy stage: updates Lambda function code via alias promotion (new version deployed to staging alias, promoted to production alias after smoke test)
- Rollback: Lambda alias pointer reverted to previous version in under 5 minutes — no redeployment required
- Agentic build workflow (Claude Code / Cursor) generates code; human review gate before pipeline trigger is strongly recommended

### Environment Strategy
- **Development**: Local + Lambda deployed to dev stage in API Gateway
- **Staging**: Full environment mirror in AWS, separate RDS instance, Stripe test mode
- **Production**: Live environment with Multi-AZ RDS, production Stripe, real SMS/push

### Monitoring and Alerting (Solo Maintainer Friendly)
- CloudWatch dashboards: Lambda invocation counts, error rates, duration, RDS connections, WebSocket connection count
- Alarms → SNS → email/SMS to founder for: payment errors, Lambda error rate spikes, RDS failover events, budget threshold breaches
- X-Ray service map for tracing request paths across functions
- AWS Cost Explorer with monthly budget alerts — Lambda, RDS, API Gateway, and data transfer are the primary cost drivers to watch
- Log Insights queries pre-built for common ops tasks: find failed jobs, trace a payment, locate driver activity

### Scaling Approach
- Lambda concurrency limits set per function to prevent runaway cost from traffic spikes
- RDS Proxy manages connection pooling — critical for Lambda-to-RDS at higher concurrency
- At 500 concurrent jobs/day, the serverless stack is well within comfortable operating range with no manual scaling required
- Reserved concurrency can be added for critical-path Lambdas (Booking, Payment) as traffic grows

## Technology Stack

| Layer | Technology | Notes |
|---|---|---|
| iOS App | React Native | Shared codebase with Android |
| Android App | React Native | Shared codebase with iOS |
| Partner Portal | React (web SPA) | Hosted on S3 + CloudFront |
| Admin Dashboard | React (web SPA) | Hosted on S3 + CloudFront |
| API Layer | AWS API Gateway (REST + WebSockets) | Single public entry point |
| Compute | AWS Lambda (Node.js or Python) | Serverless, per-invocation cost |
| Database | AWS RDS PostgreSQL (Multi-AZ) | Transactional data store |
| Connection Pooling | AWS RDS Proxy | Lambda-safe connection management |
| Real-time | API Gateway WebSockets | Driver location broadcast |
| Object Storage | AWS S3 | Docs, receipts, static assets |
| CDN | AWS CloudFront | Web portals + S3 assets |
| Payments | Stripe (SDK + API + Webhooks) | PCI scope reduction via client-side tokenization |
| Maps / Navigation | Google Maps SDK, Apple Maps SDK | Native per platform |
| SMS | Twilio | Job status notifications |
| Push Notifications | FCM (Android), APNs (iOS) | Real-time status alerts |
| Background Checks | TBD provider | Behind abstraction layer |
| Auth | JWT + Lambda Authorizer | RBAC, four roles |
| Secrets | AWS Secrets Manager | All credentials |
| Monitoring | CloudWatch + X-Ray | Logs, metrics, traces |
| CI/CD | AWS CodePipeline + CodeDeploy | Lambda alias-based rollback |
| IaC | AWS SAM or CDK | Full infrastructure definition |
| Build tooling | Claude Code / Cursor | Agentic workflow |

## Key Architectural Decisions

### 1. Serverless-First (Lambda + API Gateway)
**Decision**: All backend compute runs on Lambda with no EC2 or container-based services.
**Rationale**: At 50–500 jobs/day, fixed compute costs are hard to justify. Lambda's per-invocation pricing means near-zero cost during off-peak hours, with automatic scaling during evenings when demand peaks. Critically, there's no infrastructure to patch, restart, or monitor for a solo maintainer.
**Trade-off**: Lambda cold starts can add latency on first invocation after idle periods. Mitigated by keeping critical-path functions warm via scheduled pings, and by the fact that booking/dispatch flows aren't latency-sensitive at sub-second levels.

### 2. API Gateway WebSockets for Real-time Location
**Decision**: Driver GPS broadcast uses API Gateway WebSockets rather than a dedicated real-time service (e.g., Pusher, Ably, or a self-managed Socket.io server).
**Rationale**: Keeps the stack fully within AWS, avoids a third-party real-time dependency, and the 3–5 second broadcast interval is well within WebSocket capabilities. At launch scale, the connection count is manageable.
**Trade-off**: API Gateway WebSocket connection limits and per-message pricing require monitoring as connection counts grow. If the platform scales to thousands of concurrent connections, a dedicated real-time service may become cost-competitive.

### 3. RDS PostgreSQL over DynamoDB
**Decision**: PostgreSQL on RDS rather than a serverless NoSQL option like DynamoDB.
**Rationale**: The job/booking/payment data model is inherently relational. Surge pricing calculations, earnings reporting, and dispute resolution all benefit from SQL query flexibility. The operational overhead of RDS is managed via Multi-AZ, automated snapshots, and RDS Proxy — no manual tuning required at this scale.
**Trade-off**: RDS has a fixed minimum cost regardless of usage (unlike DynamoDB on-demand). At very low traffic, this is the largest fixed cost in the stack. Accepted given the query flexibility and maintainability advantages.

### 4. Single US Region (us-east-1) at Launch
**Decision**: No multi-region deployment at launch.
**Rationale**: Multi-region active-active adds significant operational complexity and cost that isn't justified at 50–500 jobs/day. High availability is achieved within us-east-1 via Multi-AZ RDS and Lambda's built-in redundancy. Chicago and LA are served from the same region with acceptable latency for a booking/dispatch use case (not a real-time gaming scenario).
**Trade-off**: A full us-east-1 regional outage (rare but possible) would take the platform down. Accepted risk at launch scale; revisit at significant volume growth.

### 5. Stripe Client-Side Tokenization for PCI Scope Reduction
**Decision**: All card data handled by Stripe SDK on the client — the backend never sees raw card numbers.
**Rationale**: This dramatically reduces PCI-DSS compliance scope. The platform handles only Payment Intent IDs and status, which targets SAQ A-EP compliance rather than full SAQ D. This is achievable by a solo maintainer without a dedicated compliance team.
**Trade-off**: Slightly less control over the payment UI experience. Accepted — Stripe's SDK provides a good user experience and the compliance reduction is the priority.

### 6. Background Check Provider Behind an Abstraction Layer
**Decision**: The background check integration is implemented behind an internal interface, not tightly coupled to a specific provider.
**Rationale**: The provider is TBD at architecture time. An abstraction layer means the provider can be selected, changed, or swapped without modifying the approval workflow, driver onboarding flow, or admin dashboard.
**Trade-off**: Slightly more upfront engineering for the abstraction. Low cost, high flexibility.

### 7. Lambda Alias-Based Rollback for Deployments
**Decision**: Deployments use Lambda versioning and alias promotion rather than direct code replacement.
**Rationale**: This gives the solo maintainer a fast, reliable rollback path — reverting a bad deployment is a pointer swap, not a redeployment. Critical for a system where evening availability is a hard requirement.
**Trade-off**: Requires disciplined CI/CD configuration. Managed via CodePipeline.

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| RDS failure during peak evening hours | Low | High | Multi-AZ automatic failover; RTO < 1 hour; daily snapshots + PITR |
| WebSocket connection limit reached at scale | Low (at launch) | Medium | CloudWatch alarm on connection count; evaluate dedicated real-time service if approaching limits |
| Lambda cold starts degrading booking experience | Medium | Low–Medium | Scheduled warm-up pings on critical-path functions; acceptable at current scale |
| Stripe webhook delivery failure causing missed payment status | Low | High | Webhook signature verification; idempotent payment processing; CloudWatch alarm on payment Lambda errors; Stripe dashboard for manual reconciliation |
| Background check provider outage blocking driver onboarding | Medium | Medium | Admin manual override path; abstraction layer allows fast provider swap |
| Solo maintainer unavailability during incident | Medium | High | Runbook documentation for common failure scenarios; CloudWatch alarms with clear escalation paths; IaC enables environment rebuild from scratch |
| RDS connection exhaustion from Lambda concurrency | Medium | High | RDS Proxy manages connection pooling; Lambda concurrency limits set per function |
| AWS cost overrun from misconfigured Lambda or data transfer | Low–Medium | Medium | AWS Budgets alert at threshold; Lambda concurrency limits; CloudWatch cost anomaly detection |
| CCPA deletion request not fully honored | Low | High | PII fields documented in schema; deletion workflow implemented at launch; legal review recommended |
| Driver GPS data retained beyond 30-day policy | Low | Medium | Scheduled Lambda purge job with CloudWatch alarm if purge fails |
| Agentic-generated code introducing security vulnerabilities | Medium | High | Human code review gate before CI/CD trigger; dependency scanning in pipeline; least-privilege IAM on all Lambda roles |