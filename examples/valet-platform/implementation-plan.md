# Valet Platform
## Implementation Plan

### Environment Setup

#### What you'll need

Before any code gets written, you'll need a handful of accounts set up and configured. Getting these sorted upfront means your coding agent can work without hitting walls mid-build — credentials, keys, and services all need to exist before the code that uses them does.

#### Accounts to create

- **AWS** — [aws.amazon.com](https://aws.amazon.com) — Everything backend lives here: your API, database, file storage, notifications pipeline, CI/CD, and monitoring. This is the core of the platform.
- **Stripe** — [dashboard.stripe.com/register](https://dashboard.stripe.com/register) — Handles all payment processing. You'll use both a test account (for staging) and a live account (for production).
- **Twilio** — [twilio.com/try-twilio](https://www.twilio.com/try-twilio) — Sends SMS messages to customers and drivers at key moments in a job (booking confirmed, driver on the way, job complete).
- **Google Cloud** — [console.cloud.google.com](https://console.cloud.google.com) — Provides the Google Maps SDK for Android navigation and driver tracking, and FCM for push notifications to Android devices.
- **Apple Developer Program** — [developer.apple.com/programs](https://developer.apple.com/programs) — Required to use Apple Maps on iOS and to send push notifications to iPhones via APNs. There's an annual fee.
- **Background Check Provider** — TBD per your selection. Whichever provider you choose, you'll need an account and API credentials before the driver onboarding flow can be wired up.

#### Before you start

**AWS setup:**
- Create an IAM user (or use IAM Identity Center) for your coding agent with programmatic access. Do not use your root account credentials for anything.
- Enable Multi-AZ on RDS when you create the PostgreSQL instance — this isn't something you can easily toggle on later without downtime.
- Enable RDS automated backups with a retention window of at least 7 days, and turn on point-in-time recovery.
- Create an RDS Proxy and attach it to your PostgreSQL instance — this manages database connections from Lambda without connection exhaustion.
- Store every API key and database credential in AWS Secrets Manager before any Lambda is deployed. Nothing sensitive goes in environment variables or code.
- Set up an AWS Budget with an alert at a threshold you're comfortable with (a monthly dollar amount). You'll want an email or SMS alarm before a cost surprise hits.
- Create an SNS topic for operational alerts. This is what CloudWatch alarms will use to reach you by email or SMS when something goes wrong.
- Enable AWS X-Ray tracing at the account level so it's ready when Lambda functions are deployed.
- Create an S3 bucket for driver onboarding documents (private, no public access) and a separate bucket for receipt PDFs. Enable versioning on both.
- Create S3 buckets for the Partner Portal and Admin Dashboard static files, and set up a CloudFront distribution pointing to each.

**Stripe setup:**
- In your Stripe dashboard, enable webhooks and point them at the URL your Payment Lambda will expose once deployed. You'll need to configure this for both test mode (staging) and live mode (production).
- Note your webhook signing secret — you'll need it so your backend can verify that incoming webhook payloads are genuinely from Stripe.

**Twilio setup:**
- Purchase a phone number in your Twilio account to send SMS from.
- Note your Account SID and Auth Token.

**Google Cloud setup:**
- Create a project in Google Cloud Console.
- Enable the Maps SDK for Android and the FCM API.
- Generate separate API keys for Android maps usage and for FCM server-side sending.

**Apple Developer setup:**
- Register your app's bundle identifier.
- Generate an APNs authentication key (a .p8 file) under Certificates, Identifiers & Profiles — this is what your Notification Lambda will use to send iOS push notifications.

**Background check provider setup:**
- Complete account registration and any required compliance agreements with your chosen provider.
- Obtain your API key and note the webhook URL format they use for sending back results.

#### Environment variables

These are the values your coding agent will need wired up across Lambda functions, the CI/CD pipeline, and local development. All sensitive values should live in AWS Secrets Manager — the variable names below are what the code will reference when fetching them.

**Database**
- `DB_HOST` — RDS Proxy endpoint — AWS Console → RDS → Proxies → your proxy → Endpoint
- `DB_PORT` — Almost always `5432` for PostgreSQL
- `DB_NAME` — The name you gave your PostgreSQL database at creation
- `DB_SECRET_ARN` — AWS Secrets Manager → the secret holding your RDS username and password → Secret ARN

**Auth**
- `JWT_SECRET` — A secret string you generate and store in Secrets Manager; used to sign and verify JWTs

**Stripe**
- `STRIPE_SECRET_KEY` — Stripe Dashboard → Developers → API Keys → Secret key (use test key for staging, live key for production)
- `STRIPE_WEBHOOK_SECRET` — Stripe Dashboard → Developers → Webhooks → your endpoint → Signing secret

**Twilio**
- `TWILIO_ACCOUNT_SID` — Twilio Console → Account Info → Account SID
- `TWILIO_AUTH_TOKEN` — Twilio Console → Account Info → Auth Token
- `TWILIO_PHONE_NUMBER` — Twilio Console → Phone Numbers → your purchased number

**Google / FCM**
- `GOOGLE_MAPS_API_KEY` — Google Cloud Console → APIs & Services → Credentials → your Maps API key
- `FCM_SERVER_KEY` — Google Cloud Console → your project → Cloud Messaging → Server key

**Apple / APNs**
- `APNS_KEY_ID` — Apple Developer → Certificates, Identifiers & Profiles → Keys → your APNs key → Key ID
- `APNS_TEAM_ID` — Apple Developer → Account → Membership → Team ID
- `APNS_PRIVATE_KEY_SECRET_ARN` — AWS Secrets Manager → the secret holding your .p8 key contents → Secret ARN

**Background Check Provider**
- `BGC_API_KEY_SECRET_ARN` — AWS Secrets Manager → the secret holding your provider API key → Secret ARN
- `BGC_WEBHOOK_SECRET` — Your provider's dashboard → webhook signing secret (if supported)

**AWS Infrastructure**
- `AWS_REGION` — `us-east-1`
- `S3_DOCUMENTS_BUCKET` — AWS Console → S3 → your driver documents bucket name
- `S3_RECEIPTS_BUCKET` — AWS Console → S3 → your receipts bucket name
- `CLOUDFRONT_DOMAIN` — AWS Console → CloudFront → your distribution → Domain name

---

## Phases

### Phase 1: Infrastructure Foundation

This phase stands up the entire AWS infrastructure skeleton before a single line of application logic is written. Every other phase depends on the database, networking, secrets management, storage, and CI/CD pipeline being in place and working. Getting this right first means the coding agent has a stable target to deploy against from Phase 2 onward.

Deliverables:
- RDS PostgreSQL instance running in a private VPC subnet with Multi-AZ enabled, automated backups configured, and RDS Proxy attached
- AWS Secrets Manager populated with placeholder secrets for all external services, with IAM policies scoped per Lambda domain
- S3 buckets created for driver documents, receipts, and static web portals, with versioning enabled and appropriate access policies
- CloudFront distributions provisioned for the Partner Portal and Admin Dashboard
- SNS topic created and wired to an email/SMS alert for the founder
- AWS SAM or CDK project initialized with the full infrastructure defined as code and deployable to a dev stage
- CodePipeline configured with build, test, and deploy stages targeting Lambda aliases, with alias-based rollback capability confirmed working

---

### Phase 2: Authentication and Core Data Schema

This phase delivers working auth across all four user roles and creates the full PostgreSQL schema. Nothing else in the system can be built safely without a working identity layer and the data structures that every other Lambda will read from and write to.

Deliverables:
- Full PostgreSQL schema deployed: users, drivers, partners, jobs, bookings, payments, driver location history, and WebSocket connections tables
- Auth Lambda issuing short-lived JWTs (15-minute expiry) with refresh token rotation, covering all four roles: customer, driver, partner, admin
- Lambda authorizer on API Gateway validating JWTs and extracting roles before routing
- Login, token refresh, and logout endpoints tested and returning correct role-scoped responses
- CCPA-relevant PII fields documented and tagged in the schema
- Data retention rules for driver GPS history (30-day purge) and job records (3-year retention) defined in schema comments and ready for enforcement via scheduled Lambda

---

### Phase 3: Booking and Dispatch

This phase delivers the core transactional loop of the platform — the flow that takes a customer from "I need a valet" to a driver being assigned and en route. Surge pricing is included here because it's calculated at booking time and is part of the job creation record.

Deliverables:
- Booking Lambda handling job creation, modification, and cancellation with correct status enum transitions (scheduled → dispatched → en_route → arrived → in_progress → complete → cancelled)
- Dispatch Lambda assigning available drivers to jobs, including driver availability checks
- Surge pricing calculation implemented in Dispatch Lambda: demand multiplier calculated per city zone based on active job density, stored as `surge_multiplier` on the job record
- Customer-facing booking endpoints (create, modify, cancel, view history) tested across all valid and invalid state transitions
- Driver-facing job queue endpoint returning available jobs for the driver's city, with accept/decline responses updating job state

---

### Phase 4: Real-Time Driver Location

This phase wires up the WebSocket infrastructure for live driver tracking — the feature that makes the platform feel like a real-time logistics product rather than a static booking tool. It's sequenced after booking and dispatch because the WebSocket connection is scoped to an active job.

Deliverables:
- API Gateway WebSocket API deployed with connect, disconnect, and message routes handled by Location Lambda
- Driver app WebSocket connection successfully broadcasts GPS coordinates at 3–5 second intervals, with connection IDs stored in the WebSocket connections table linked to the active job
- Location Lambda receiving driver GPS payloads, writing to driver location history table, and broadcasting to the subscribed customer WebSocket session
- Customer app receiving live driver position updates over WebSocket with graceful reconnection logic when the connection drops
- WebSocket connection records cleaned up correctly on disconnect
- CloudWatch alarm configured on WebSocket connection count

---

### Phase 5: Payments and Receipts

This phase integrates Stripe end-to-end: card capture on the client, charge on job completion, webhook handling for async status updates, and receipt generation. Payment is the highest-stakes integration in the platform from a compliance and reliability perspective, so it's built and hardened as its own focused phase.

Deliverables:
- Stripe SDK integrated in the customer mobile app for client-side card tokenization — card data never reaches the platform backend
- Payment Lambda creating and confirming Stripe Payment Intents on job completion
- Stripe webhook endpoint live and verifying Stripe's signature header on every inbound request; handles charge succeeded, refund processed, and dispute opened events idempotently
- Receipt PDF generated and stored in the S3 receipts bucket on payment completion; receipt URL written to the payments table
- Payment status correctly reflected on the customer's booking record
- CloudWatch alarm configured on Payment Lambda error rate

---

### Phase 6: Notifications

This phase delivers SMS and push notifications for all key job status events. Notifications are sequenced after payments because the complete job status lifecycle — including payment confirmation — needs to exist before the full notification trigger map can be implemented correctly.

Deliverables:
- Notification Lambda triggering Twilio SMS on: booking confirmation, driver dispatched, driver arrived, and job complete
- Push notifications delivered via FCM (Android) and APNs (iOS) for the same job status events
- Device tokens registered and stored against user records at login; Notification Lambda reading the correct token per user
- SMS used as fallback for critical events when push delivery cannot be confirmed
- Twilio and push delivery failures logged to CloudWatch without crashing the job status update flow

---

### Phase 7: Driver Onboarding and Background Checks

This phase builds the driver onboarding flow: document upload, background check submission, status polling or webhook handling, and the admin approval step that unlocks job queue access. It's sequenced late because it depends on auth, the driver schema, S3 document storage, and the admin dashboard that approves the result.

Deliverables:
- Onboarding Lambda accepting driver document uploads and storing them in the private S3 documents bucket via pre-signed URLs
- Background check submission implemented behind an abstraction interface: Onboarding Lambda sends driver PII to the provider API, stores the provider reference ID on the driver record, and handles the result via webhook or polling
- Driver `status` field transitions correctly: pending → approved or suspended based on check result plus admin action
- Admin-facing approval endpoint allowing manual approval or rejection of drivers post background check
- Admin notification (push or email) triggered when a background check result is ready for review
- Manual admin override path functional for cases where the background check provider is unavailable

---

### Phase 8: Partner Portal and Admin Dashboard

This phase delivers the two web surfaces — the React SPA for garage/lot partners and the React SPA for internal admins. Both are sequenced last because they are read/write layers on top of data that the earlier phases produce and manage.

Deliverables:
- Partner Portal deployed to S3 and served via CloudFront: garage profile management, live and historical job view scoped to the partner's location, earnings dashboard and payout history
- Admin Dashboard deployed to S3 and served via CloudFront: system-wide job and booking visibility, driver approval workflow, dispute management and resolution, user account management
- Admin endpoints restricted by IP allowlist in API Gateway
- All admin actions logged to CloudWatch with the acting user ID for audit trail
- Partner Lambda and Admin Lambda endpoints returning correctly role-scoped data, verified against the Lambda authorizer

---

### Phase 9: Observability, Compliance, and Launch Hardening

This phase is not cleanup — it's the work that makes the platform safe and operable at launch. It covers the monitoring, alerting, retention enforcement, compliance workflows, and cost controls that a solo maintainer needs to run this system without being on-call heroics.

Deliverables:
- CloudWatch dashboards live: Lambda invocation counts, error rates, duration, RDS connections, and WebSocket connection count all visible in one place
- CloudWatch alarms active and routing to SNS for: Lambda error rate spikes, payment webhook failures, failed login spikes, RDS failover events, and budget threshold breaches
- X-Ray service map tracing confirmed across all Lambda hops; pre-built Log Insights queries documented for: find failed jobs, trace a payment, locate driver activity
- Scheduled Lambda purge job running on the driver GPS history table, removing records older than 30 days post-job, with a CloudWatch alarm if the purge job fails
- CCPA data deletion workflow implemented: soft-delete with PII nullification, job records anonymized; data export workflow generating a user-linked record export on request
- AWS Budgets alert configured at the founder's monthly cost threshold
- Lambda concurrency limits set per function, with reserved concurrency on Booking and Payment Lambdas
- Warm-up pings configured on critical-path Lambda functions (Booking, Dispatch, Payment) to reduce cold start impact
- Staging environment confirmed running against Stripe test mode, separate RDS instance, and Twilio test credentials; full end-to-end smoke test passing before production promotion