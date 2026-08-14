> Phase 1 of 9 — this file contains the Phase 1 tickets (Infrastructure Foundation) only. The full ticket set across all nine phases is generated inside Zalcro.

# Valet Platform — Tickets 

## Ticket 1: Initialize Infrastructure as Code (IaC) Project

**Objective:** This ticket establishes the foundational AWS SAM or CDK project that will define all cloud resources for the application.

**Context:** As per the Deployment Architecture, all infrastructure must be defined as code to ensure repeatable and reliable deployments. This initial project setup is the first step in the "Infrastructure Foundation" epic, creating the container into which all subsequent resource definitions (database, storage, networking) will be placed.

**Technical Directives:**
- Initialize a new Infrastructure as Code project using either AWS SAM or AWS CDK.
- Configure the project for the `us-east-1` AWS region as specified in the Deployment Architecture.
- The project structure should accommodate definitions for Lambda functions, API Gateway, RDS, S3, and other AWS resources.
- Create an initial, empty stack definition that can be deployed to AWS to verify the project setup.

**Scope Boundaries:**
- Infrastructure as Code (IaC) project configuration
- Local development environment setup for SAM/CDK

**Acceptance Criteria:**
- An AWS SAM `template.yaml` or AWS CDK application file is created in a new repository.
- The project is configured to deploy to the `us-east-1` region.
- A deployment of the initial empty stack to a development AWS account succeeds without errors.

---

## Ticket 2: Define VPC and Networking

**Objective:** This ticket defines the Virtual Private Cloud (VPC) and associated networking resources that will house the application's backend components.

**Context:** The Security Architecture mandates that the RDS database be deployed in a private subnet, inaccessible from the public internet. This ticket creates the necessary network isolation within the IaC project established in the previous ticket, providing a secure foundation for the database and Lambda functions.

**Technical Directives:**
- Within the IaC project (SAM/CDK), define a new AWS VPC.
- The VPC must contain at least two private subnets and two public subnets, spread across different availability zones to support Multi-AZ deployments.
- Define an Internet Gateway and attach it to the VPC.
- Define route tables to direct traffic from public subnets through the Internet Gateway.
- Define a NAT Gateway to allow resources in the private subnets (like Lambda functions) to access the internet for external API calls, while preventing inbound traffic.

**Scope Boundaries:**
- AWS Networking Layer (VPC, Subnets, Route Tables, Gateways)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- The IaC stack successfully deploys a VPC into the `us-east-1` region.
- The deployed VPC contains public and private subnets across at least two availability zones.
- Resources launched in a private subnet can successfully make outbound requests to the internet.
- Resources launched in a private subnet are not directly accessible from the public internet.

---

## Ticket 3: Provision RDS PostgreSQL Database and Proxy

**Objective:** This ticket provisions the primary RDS PostgreSQL database, including its connection proxy, automated backups, and high-availability configuration.

**Context:** The architecture specifies RDS PostgreSQL as the primary data store, requiring Multi-AZ for high availability and an RDS Proxy to manage connections from Lambda. This ticket implements these critical data architecture components within the VPC created in the previous ticket.

**Technical Directives:**
- Define an AWS RDS instance using the PostgreSQL engine within the private subnets of the VPC.
- Enable the Multi-AZ deployment option as required for an RTO of less than one hour.
- Configure daily automated snapshots with a retention period of at least 7 days.
- Enable Point-in-Time Recovery (PITR).
- Define an RDS Proxy and attach it to the PostgreSQL instance. The proxy must also be deployed into the private subnets.
- Configure security groups to only allow traffic to the RDS instance from the RDS Proxy.

**Scope Boundaries:**
- AWS Database Layer (RDS, RDS Proxy)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- The IaC stack successfully deploys an RDS PostgreSQL instance into the private subnets.
- The RDS instance has Multi-AZ enabled and automated backups configured with a 7-day retention.
- An RDS Proxy is deployed and successfully connected to the RDS instance.
- The RDS Proxy endpoint is available for use by resources within the VPC.
- The RDS instance's security group blocks direct connections from any source other than its associated RDS Proxy.

---

## Ticket 4: Configure Secrets Manager and IAM Roles

**Objective:** This ticket creates placeholders for all required application secrets in AWS Secrets Manager and defines the necessary IAM roles for Lambda functions to access them.

**Context:** The Security Architecture mandates that all credentials and API keys be stored in AWS Secrets Manager, with Lambda functions granted least-privilege access. This ticket prepares the secure storage for credentials needed by Stripe, Twilio, and other services before any application code is written.

**Technical Directives:**
- In AWS Secrets Manager, create placeholder secrets for the values listed in the Environment Setup section (e.g., `DB_SECRET_ARN`, `JWT_SECRET`, `STRIPE_SECRET_KEY`, etc.).
- The database secret for RDS (`DB_SECRET_ARN`) should be configured to use Secrets Manager's automated credential rotation for PostgreSQL.
- Within the IaC project, define IAM roles for each Lambda domain specified in the Component Architecture (Auth, Booking, Payment, etc.).
- Each IAM role must have a policy granting it read-only access to *only* the specific secrets it requires, following the principle of least privilege.

**Scope Boundaries:**
- AWS Secrets Management Layer (AWS Secrets Manager)
- AWS Identity and Access Management (IAM) Layer
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- All secrets listed in the Environment Setup section exist as placeholders in AWS Secrets Manager in the `us-east-1` region.
- The RDS database secret is configured for automatic rotation.
- IAM roles for each Lambda domain (Auth, Booking, etc.) are defined in the IaC project.
- The IAM policy for the "Payment Lambda" role grants access to `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET` but not `TWILIO_AUTH_TOKEN`.
- The IAM policy for the "Notification Lambda" role grants access to Twilio, FCM, and APNs secrets but not Stripe secrets.

---

## Ticket 5: Provision S3 Buckets for Storage

**Objective:** This ticket creates the S3 buckets required for storing driver documents, payment receipts, and static web assets.

**Context:** The architecture relies on S3 for multiple storage needs: secure, private storage for sensitive driver documents and receipts, and public-read access for hosting the Partner and Admin web portals. This ticket sets up these buckets with the correct access policies and features as defined in the architecture.

**Technical Directives:**
- Within the IaC project, define three separate S3 buckets.
- Bucket 1 (`S3_DOCUMENTS_BUCKET`): For driver onboarding documents. Must be private with no public access. Enable versioning.
- Bucket 2 (`S3_RECEIPTS_BUCKET`): For receipt PDFs. Must be private with no public access. Enable versioning.
- Bucket 3 (for Partner Portal & Admin Dashboard): A single bucket or two separate buckets to host the static files for the React web applications. These must be configured for static website hosting.

**Scope Boundaries:**
- AWS Object Storage Layer (S3)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- Three S3 buckets are successfully created via the IaC deployment.
- The buckets for documents and receipts are configured to block all public access.
- Versioning is enabled on the document and receipt buckets.
- The bucket(s) for web assets are configured with a policy that allows them to be served as a static website.

---

## Ticket 6: Configure CloudFront Distributions for Web Portals

**Objective:** This ticket sets up AWS CloudFront distributions to serve the Partner Portal and Admin Dashboard static websites from S3.

**Context:** To provide fast, secure, and global access to the web portals, the architecture specifies using CloudFront as a CDN. This ticket fronts the S3 buckets created in the previous step with CloudFront distributions, enforcing HTTPS and improving performance.

**Technical Directives:**
- Within the IaC project, define two AWS CloudFront distributions.
- One distribution should be configured to serve content from the S3 bucket designated for the Partner Portal.
- The second distribution should be configured to serve content from the S3 bucket for the Admin Dashboard.
- Both distributions must be configured to enforce HTTPS.
- Configure the distributions to correctly serve a React single-page application (SPA) by routing all 404 errors to `index.html`.

**Scope Boundaries:**
- AWS CDN Layer (CloudFront)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- Two CloudFront distributions are successfully created and associated with their respective S3 buckets.
- Attempting to access the portals via HTTP automatically redirects to HTTPS.
- When a placeholder `index.html` file is placed in the S3 buckets, it is successfully served via the CloudFront domain name.
- Requesting a non-existent path (e.g., `CLOUDFRONT_DOMAIN/some-react-route`) returns the content of `index.html` with a 200 OK status code.

---

## Ticket 7: Set Up Alerting and Budgeting

**Objective:** This ticket establishes the foundational monitoring and alerting infrastructure to notify the solo maintainer of operational issues and cost anomalies.

**Context:** A key operational constraint is the system being maintained by a solo founder. This requires robust, automated alerting. This ticket creates the SNS topic for alarms and a baseline AWS Budget to prevent cost overruns, as specified in the architecture.

**Technical Directives:**
- Within the IaC project, define an SNS topic for operational alerts.
- Create an AWS Budget for the account with an alert threshold.
- Configure the budget alert to publish a notification to the newly created SNS topic when the threshold is breached.
- Manually subscribe an email address to the SNS topic to receive notifications.
- Define a baseline CloudWatch alarm (e.g., for RDS CPU > 80%) that also publishes to the same SNS topic.

**Scope Boundaries:**
- AWS Monitoring and Alerting Layer (CloudWatch, SNS, AWS Budgets)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- An SNS topic for alerts is created via the IaC deployment.
- An email subscription to the topic is confirmed and active.
- A test message published to the SNS topic is successfully delivered to the subscribed email address.
- An AWS Budget with an alert that targets the SNS topic is visible in the AWS Billing console.
- A CloudWatch alarm targeting the SNS topic is visible in the CloudWatch console.

---

## Ticket 8: Implement CI/CD Pipeline with Rollback Capability

**Objective:** This ticket creates a CodePipeline to automate the deployment of the IaC stack, including a safe, alias-based rollback strategy for future Lambda deployments.

**Context:** To enable rapid and safe deployments for a solo maintainer, the Deployment Architecture requires a CI/CD pipeline with a fast rollback capability. This ticket builds that pipeline, ensuring that any changes to the infrastructure or future application code can be deployed and reverted reliably.

**Technical Directives:**
- Define an AWS CodePipeline pipeline using the IaC project (SAM/CDK).
- The pipeline source must be a Git repository (e.g., AWS CodeCommit, GitHub).
- The pipeline must include a `build` stage that runs tests and packages the IaC application.
- The pipeline must include a `deploy` stage that executes the IaC deployment (e.g., `sam deploy` or `cdk deploy`).
- For future Lambda deployments, the deployment action must be configured to use Lambda aliases for promotion (e.g., deploying to a 'staging' alias first, then promoting to 'production'). This setup only needs to be defined; no Lambda function is deployed in this ticket.

**Scope Boundaries:**
- AWS CI/CD Layer (CodePipeline, CodeBuild, CodeDeploy)
- Infrastructure as Code (IaC) project

**Acceptance Criteria:**
- A CodePipeline is created in the AWS account.
- Pushing a change to the main branch of the linked Git repository automatically triggers a pipeline run.
- The pipeline successfully builds and deploys the existing IaC stack (VPC, RDS, S3, etc.) to a staging environment.
- The pipeline definition includes stages and actions configured for a Lambda alias-based deployment, ready for use in future epics.
- A manual approval step exists in the pipeline before deploying to a production stage.