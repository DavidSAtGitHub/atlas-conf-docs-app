
# Interview Preparation: Cloud/Serverless Architect @ Raiffeisenbank

---

## TABLE OF CONTENTS

1. [Position Analysis & Expected Competencies](#1-position-analysis--expected-competencies)
2. [CV-Based Questions & Project Stories](#2-cv-based-questions--project-stories)
3. [Architecture Questions & Design Scenarios](#3-architecture-questions--design-scenarios)
4. [AWS Serverless Services Deep Dive](#4-aws-serverless-services-deep-dive)
5. [Additional Topics](#5-additional-topics)
6. [Behavioral & Soft Skills](#6-behavioral--soft-skills)
7. [Questions to Ask the Interviewer](#7-questions-to-ask-the-interviewer)

---

## 1. Position Analysis & Expected Competencies

### 1.1 Core Role Expectations

Based on the job description, Raiffeisenbank is looking for someone who can:

**Primary Responsibilities:**

- Design serverless application architectures in AWS
- Create and evolve cloud-native backend services
- Collaborate with developers from design through production deployment
- Define technical standards and best practices
- Implement Infrastructure as Code using AWS CDK

**Key Differentiator:** This is not just a developer role—it's a **platform/standards architect** position. You'll influence how the entire bank approaches cloud development.

### 1.2 Required Technical Knowledge

#### AWS Serverless Stack (Must-Know)

|Service|What You Need to Know|
|---|---|
|**Lambda**|Cold starts, memory/CPU correlation, concurrency limits, VPC considerations, layers, runtime management, error handling patterns|
|**API Gateway**|REST vs HTTP APIs, authentication (Cognito, Lambda authorizers), throttling, caching, request/response transformations|
|**DynamoDB**|Single-table design, GSI/LSI, capacity modes, transactions, streams, TTL, DAX caching|
|**Step Functions**|Standard vs Express, state types, error handling, service integrations, compensation patterns|
|**SQS/SNS**|Fan-out patterns, FIFO vs Standard, DLQ configuration, message filtering, batching|
|**EventBridge**|Event patterns, schema registry, archive/replay, cross-account events|

#### Infrastructure as Code

|Tool|Expected Knowledge|
|---|---|
|**AWS CDK**|Constructs (L1/L2/L3), aspects, context, custom resources, CDK Pipelines|
|**CloudFormation**|Understanding what CDK generates, drift detection, stack policies|
|**Terraform**|(Your strength) - comparison with CDK, when to use each|

#### DevOps & CI/CD

- **GitHub Actions** - workflows, environments, secrets management
- **CI/CD patterns** - multi-environment deployments, approval gates, rollback strategies
- **Monitoring** - CloudWatch (Logs, Metrics, Alarms), X-Ray, structured logging

#### Design Patterns

|Pattern|Application|
|---|---|
|Event-driven architecture|Decoupling, eventual consistency, event sourcing|
|CQRS|Read/write separation, when to apply|
|Saga pattern|Distributed transactions, compensation|
|Circuit breaker|Resilience, failure isolation|
|Strangler fig|Migration from monolith|

### 1.3 Banking/Financial Services Context

**Expect questions around:**

- **Compliance & Audit** - traceability, immutable logs, data retention
- **Security** - encryption at rest/transit, least privilege, network isolation
- **High Availability** - multi-AZ, disaster recovery, RTO/RPO
- **Data Privacy** - GDPR, data residency, PII handling
- **Regulatory Requirements** - ČNB requirements, EBA guidelines

---

## 2. CV-Based Questions & Project Stories

### 2.1 STAR Method Framework

For each project, prepare stories using:

- **Situation** - Context, constraints, business need
- **Task** - Your specific responsibility
- **Action** - What you did, decisions you made
- **Result** - Measurable outcomes, lessons learned

---

### 2.2 Project: Event-Driven DL Components (Oct 2025 – Dec 2025)

**One-liner:** Designed event-driven serverless architecture for automated data asset ingestion and registration in AWS data lake.

#### The Story

**Situation:** "The organization needed to automate the ingestion and registration of data assets into their AWS data lake. Previously, this was a manual process requiring data engineers to validate incoming data and manually register it in the Glue Data Catalog. This created bottlenecks and inconsistencies."

**Task:** "I was responsible for designing and implementing an end-to-end serverless solution that would automatically validate incoming data against defined contracts and register compliant assets in the data catalog."

**Action:** "I designed a manifest-based approach where data producers define their data contracts in manifest files. The architecture used:

- S3 event notifications to trigger Lambda functions on new data arrivals
- Lambda functions to validate manifest consistency and data readability
- Automatic registration in AWS Glue Data Catalog upon successful validation
- Lake Formation for access control and tagging
- Terraform for infrastructure deployment across multiple environments
- CI/CD pipeline for automated testing and deployment"

**Result:** "The solution eliminated manual registration work, reduced time-to-availability for new datasets from days to minutes, and ensured consistent metadata quality across the data lake."

#### Likely Follow-up Questions

**Q: Why manifest-based approach?** "Manifests provide a contract between data producers and the platform. They define expected schema, partitioning, format, and quality expectations. This enables automated validation without custom code per dataset and creates a self-service model for data producers."

**Q: How did you handle validation failures?** "Failed validations trigger SNS notifications to data producers with detailed error information. The manifest and data remain in a 'pending' state until issues are resolved. We implemented a status tracking system so producers can monitor registration progress."

**Q: What formats did you support?** "The solution supported both Apache Iceberg tables and CSV files with conditional validation rules—different requirements based on format. Iceberg tables were registered with schema evolution support."

**Q: How did you ensure idempotency?** "Lambda functions were designed to be idempotent using the manifest file path as a natural idempotency key. Re-processing the same manifest would result in an update rather than duplicate registration."

**Q: Challenges?** "One challenge was handling large manifests with many files—we had to implement batching to stay within Lambda timeout limits. Another was ensuring consistent naming across the catalog, which led to implementing a structured naming convention combining usecase and version attributes."

---

### 2.3 Project: Metadata Consolidation Platform (Mar 2024 – Sep 2025)

**One-liner:** Led architecture of cloud-native AWS solution for metadata consolidation and governance enablement across Commercial department.

#### The Story

**Situation:** "The Commercial department had metadata scattered across multiple systems—lineage in one tool, glossary in another, data quality metrics somewhere else. There was no unified view, making governance and discovery extremely difficult."

**Task:** "I was tasked with architecting a solution that would consolidate metadata from multiple sources into a unified model while supporting automated synchronization and governance processes."

**Action:** "I approached this as a platform problem, not just an integration:

1. Designed a **standardized domain model** that could represent metadata from any source system
2. Architected an **API-driven, event-driven backend** where:
    - APIs provided synchronous access for interactive use cases
    - Events enabled async synchronization from source systems
3. Built **extensibility** into the design so new source systems could be onboarded through configuration rather than code
4. Implemented a **CI/CD pipeline** using Jenkins and Terraform for reliable multi-environment deployments
5. Focused on operational reliability—error handling, retry logic, monitoring"

**Result:** "The platform provided centralized visibility for glossary management, lineage tracking, and data quality processes. The extensible architecture allowed the team to onboard new metadata sources without architectural changes."

#### Likely Follow-up Questions

**Q: How did you design the domain model?** "I started with the core entities that appeared across all systems—datasets, columns, glossary terms, data quality rules. Then I added relationships (lineage, ownership, classification). The model was designed to be additive—new attributes could be added without breaking existing integrations."

**Q: Why both API and event-driven?** "Different use cases require different interaction patterns. Users browsing the glossary need synchronous responses. But when a source system updates lineage for thousands of assets, we don't want to block—events allow async processing with eventual consistency."

**Q: How did you handle conflicts when multiple sources had different views?** "We established source-of-truth rules per attribute type. For example, technical metadata came from the data catalog, business definitions from the glossary tool. The domain model tracked provenance for each attribute."

**Q: What was the most challenging aspect?** "Getting stakeholder alignment on the domain model. Different teams had different mental models of what 'lineage' or 'data quality' meant. We invested significant time in workshops to create a shared vocabulary before building."

---

### 2.4 Project: Reference Data Repository (Mar 2023 – Mar 2024)

**One-liner:** Designed and delivered cloud-native reference data repository as production-ready proof of concept for global adoption.

#### The Story

**Situation:** "The organization lacked a centralized way to manage reference data—code lists, lookup tables, standardized values. Each system maintained its own copies, leading to inconsistencies and no single source of truth."

**Task:** "Design a scalable, secure repository that could serve as the foundation for enterprise-wide reference data standardization."

**Action:** "Key architectural decisions:

1. **API-first design** - all access through versioned APIs, no direct database access
2. **Controlled access** - fine-grained permissions per reference data domain
3. **Discovery capabilities** - search and browse functionality for consumers to find available reference data
4. **Scalability** - designed to handle growth from proof of concept to enterprise scale
5. **Infrastructure as Code** - full Terraform automation for reproducible deployments"

**Result:** "Delivered a production-ready PoC that demonstrated the value of centralized reference data. The architecture was validated for global adoption."

#### Likely Follow-up Questions

**Q: How did you design for scale?** "The architecture separated concerns—API layer, business logic, and storage could scale independently. We used caching for frequently accessed reference data and designed the data model to support partitioning if needed."

**Q: What security model did you implement?** "Role-based access control at the domain level. Consumers could read reference data they were authorized for. Updates went through an approval workflow. All access was logged for audit."

**Q: How did you handle versioning of reference data?** "Reference data sets were versioned with effective dates. Consumers could request current values or historical snapshots. Changes went through a controlled process with impact assessment."

---

### 2.5 Project: Pharmaceutical Data Platform (Nov 2021 – Feb 2023)

**One-liner:** Extended and maintained data processing pipelines within AWS-based platform, focusing on scalability and operational excellence.

#### The Story

**Situation:** "I joined an existing AWS data platform that was experiencing growing pains—pipelines were becoming harder to maintain, data quality issues were increasing, and operational overhead was significant."

**Task:** "Improve scalability, data quality, and reliability while working within existing architectural constraints."

**Action:** "Rather than redesigning, I focused on pragmatic improvements:

1. Refactored critical pipelines for better performance and maintainability
2. Implemented DBT for analytical transformations, bringing testing and documentation
3. Added data quality checks at key pipeline stages
4. Simplified deployment processes to reduce operational burden
5. Improved monitoring and alerting for proactive issue detection"

**Result:** "Measurably improved pipeline reliability, reduced time spent on operational issues, and established patterns the team could apply to other areas."

#### Likely Follow-up Questions

**Q: How do you approach improving an existing system vs. rebuilding?** "I start by understanding the constraints—technical debt, team capacity, business timelines. Usually, incremental improvement is more pragmatic. I identify the highest-pain-point areas and focus there first. Rebuilding is justified when the improvement cost exceeds rebuild cost or when the architecture fundamentally can't meet future needs."

**Q: What did DBT bring to the platform?** "DBT brought three key things: testable transformations (we could define expectations and catch regressions), documentation that lived with the code, and lineage visibility. It also standardized how the team approached analytical modeling."

---

### 2.6 General CV Questions

**Q: Why the transition from data engineering to architecture?** "Natural evolution. As I gained experience, I found myself spending more time on design decisions than implementation. I enjoy the challenge of balancing competing concerns—scalability, cost, security, maintainability. Architecture lets me have broader impact while still staying technical."

**Q: You mention 'hands-on approach to architecture'—what does that mean?** "I believe architects should be able to implement what they design. I stay involved in code reviews, write proof-of-concept code, and maintain working knowledge of the technologies I recommend. This builds credibility with developers and ensures my designs are practical."

**Q: Tell me about a design decision you later regretted.** _(Prepare a genuine example—shows self-awareness)_

**Q: How do you stay current with AWS services?** "I follow AWS announcements, particularly re:Invent. I maintain certifications which forces structured learning. Most importantly, I experiment—I build small projects to understand new services before recommending them."

---

## 3. Architecture Questions & Design Scenarios

### 3.1 Common Design Questions

#### Scenario 1: Design a serverless payment notification system

**Prompt:** "Design a system that sends notifications to customers when their payment is processed. It needs to support email, SMS, and push notifications."

**Key Points to Address:**

- Event source (payment service publishes to EventBridge/SNS)
- Fan-out pattern (SNS to multiple notification channels)
- Channel-specific Lambda functions (email via SES, SMS via SNS, push via Pinpoint)
- DLQ for failed notifications with retry logic
- Idempotency (payment ID as idempotency key)
- Customer preferences (DynamoDB lookup for preferred channels)
- Monitoring and alerting on delivery failures

#### Scenario 2: Design an API for account balance inquiries

**Prompt:** "Design a highly available API that returns current account balance. It needs to handle 10,000 requests per second during peak."

**Key Points to Address:**

- API Gateway with caching for read-heavy workload
- Lambda behind API Gateway
- Consider whether DynamoDB (fast) or RDS Proxy to Aurora (if relational needed)
- Caching strategy (how fresh must data be?)
- Rate limiting per customer
- Authentication (Cognito, API keys for B2B)
- Multi-AZ deployment
- CloudFront for geographic distribution if needed

#### Scenario 3: Design an event-driven loan application workflow

**Prompt:** "Design a system to process loan applications. It involves credit check, document verification, approval workflow, and notification."

**Key Points to Address:**

- Step Functions for orchestration (Standard for long-running with human approval)
- State machine design with error handling
- Service integrations (Lambda for business logic, third-party for credit check)
- Wait states for human approval
- Compensation logic if approval fails after credit check
- Event publishing for downstream systems
- Audit trail (all state transitions logged)

#### Scenario 4: Migrate a monolithic banking application

**Prompt:** "We have a 15-year-old monolithic banking application. How would you approach modernization?"

**Key Points to Address:**

- Strangler fig pattern (not big-bang rewrite)
- Identify bounded contexts (candidates for extraction)
- Start with read-only services (lower risk)
- API Gateway as facade in front of monolith
- Event-driven integration (monolith publishes events)
- Data strategy (shared database initially, then separate)
- Gradual traffic shifting
- Emphasize risk management given banking context

### 3.2 Architecture Trade-off Questions

**Q: When would you NOT use serverless?**

- Consistent low-latency requirements (cold starts problematic)
- Long-running processes (15-min Lambda limit)
- Very high throughput with consistent load (container/EC2 more cost-effective)
- Complex local state requirements
- Specific runtime/dependency needs

**Q: DynamoDB vs Aurora Serverless?**

- DynamoDB: Key-value access patterns, extreme scale, no relational needs
- Aurora: Complex queries, joins, transactions across tables, existing SQL expertise

**Q: SQS vs EventBridge?**

- SQS: Point-to-point, high throughput, simple routing
- EventBridge: Event routing to multiple targets, content-based filtering, schema registry

**Q: Step Functions Standard vs Express?**

- Standard: Long-running (up to 1 year), exactly-once, audit/compliance needs
- Express: Short duration (<5 min), high volume, at-least-once acceptable

### 3.3 Security Architecture Questions

**Q: How do you implement least privilege in serverless?**

- Per-function IAM roles (not shared)
- Resource-level permissions (specific DynamoDB tables, S3 prefixes)
- Condition keys (source IP, VPC)
- Service control policies at org level
- Regular permission audits

**Q: How do you secure API Gateway?**

- Authentication (Cognito, Lambda authorizer)
- Authorization (scopes, custom claims)
- WAF integration (SQL injection, rate limiting)
- mTLS for B2B
- Request validation
- VPC private endpoints for internal APIs

**Q: How do you handle secrets in serverless?**

- Secrets Manager or Parameter Store (SecureString)
- Lambda environment variables for non-sensitive config
- Secrets rotation automation
- VPC endpoints to access secrets without internet
- Never log secrets, use reference IDs

### 3.4 Operational Excellence Questions

**Q: How do you implement observability in serverless?**

- Structured logging (JSON) with correlation IDs
- CloudWatch Logs Insights for querying
- X-Ray for distributed tracing
- Custom metrics for business KPIs
- Dashboards per service
- Alarms with appropriate thresholds
- Log retention policies

**Q: How do you handle Lambda cold starts?**

- Provisioned concurrency for latency-critical functions
- Smaller deployment packages
- Avoid VPC unless necessary (or use Hyperplane)
- SnapStart for Java
- Keep functions warm (scheduled invocation) as last resort

**Q: How do you approach disaster recovery?**

- Multi-AZ by default (most serverless services)
- Multi-region for critical workloads
- DynamoDB global tables
- S3 cross-region replication
- Infrastructure as Code enables quick recovery
- Regular DR testing

---

## 4. AWS Serverless Services Deep Dive

### 4.1 AWS Lambda

#### Core Concepts

- **Execution model:** Request/response, event-based, or streaming
- **Memory:** 128 MB to 10,240 MB (CPU scales proportionally)
- **Timeout:** Max 15 minutes
- **Concurrency:** 1000 default (can increase), per-function reserved concurrency

#### Key Considerations

|Aspect|Details|
|---|---|
|**Cold starts**|100ms-1s+ depending on runtime, memory, VPC, package size|
|**VPC**|Adds latency (Hyperplane improved this), use only when necessary|
|**Layers**|Share code/dependencies, max 5 layers, 250MB unzipped limit|
|**Versions/Aliases**|Immutable versions, aliases for traffic shifting|
|**Destinations**|Async invocation success/failure routing|
|**SnapStart**|Pre-initialized snapshots for Java (reduces cold start to ~200ms)|
|**Provisioned Concurrency**|Pre-warmed instances, costs money|

#### Error Handling

- Sync invocation: Caller handles retry
- Async invocation: 2 retries by default, then DLQ/destination
- Event source mappings: Depends on source (SQS, Kinesis different behaviors)

#### Best Practices

- One function per concern (not monolithic Lambda)
- Externalize configuration (environment variables, Parameter Store)
- Minimize package size (faster cold starts)
- Use structured logging with correlation IDs
- Implement idempotency for retries

---

### 4.2 Amazon API Gateway

#### Types Comparison

|Feature|REST API|HTTP API|
|---|---|---|
|Cost|Higher|70% cheaper|
|Latency|Higher|Lower|
|Features|Full (caching, WAF, validation)|Basic|
|Auth|Cognito, Lambda, IAM|Cognito, Lambda, IAM, JWT|
|Use case|Complex APIs, enterprise|Simple APIs, cost-sensitive|

#### Key Features

- **Stages:** dev, staging, prod with stage variables
- **Throttling:** Account level (10,000 RPS), per-method configurable
- **Caching:** Response caching (0.5GB-237GB), TTL configurable
- **Request validation:** Schema validation before Lambda invocation
- **Transformations:** VTL templates for request/response mapping
- **Custom domains:** Route53 + ACM certificates

#### Security

- **Authentication:** Cognito User Pools, Lambda authorizers, IAM
- **Authorization:** Resource policies, usage plans with API keys
- **WAF:** SQL injection, XSS, rate limiting
- **mTLS:** Mutual TLS for B2B

---

### 4.3 Amazon DynamoDB

#### Core Concepts

- **Primary key:** Partition key (PK) or PK + Sort key (SK)
- **Secondary indexes:** GSI (different PK/SK), LSI (same PK, different SK)
- **Capacity modes:** On-demand (pay per request), Provisioned (RCU/WCU)

#### Data Modeling

**Single-table design principles:**

- Store multiple entity types in one table
- Use generic PK/SK attributes (e.g., PK="USER#123", SK="ORDER#456")
- Design for access patterns first
- Denormalize for read efficiency

#### Key Features

|Feature|Details|
|---|---|
|**Streams**|Change data capture, Lambda triggers|
|**TTL**|Automatic item deletion, cost-free|
|**Transactions**|ACID across up to 100 items|
|**Global tables**|Multi-region, active-active|
|**DAX**|In-memory cache, microsecond reads|
|**PartiQL**|SQL-like query syntax|

#### Limits & Considerations

- Item size: 400 KB max
- GSI: 20 per table
- Query/Scan: 1 MB per call (paginate)
- Hot partitions: Adaptive capacity helps but design for distribution
- Eventually consistent by default (strongly consistent 2x cost)

---

### 4.4 AWS Step Functions

#### Workflow Types

|Feature|Standard|Express|
|---|---|---|
|Duration|Up to 1 year|Up to 5 minutes|
|Execution model|Exactly-once|At-least-once|
|History|Full (90 days)|CloudWatch Logs only|
|Price|Per state transition|Per execution + duration|
|Use case|Orchestration, human approval|High-volume, ETL|

#### State Types

- **Task:** Do work (Lambda, service integration)
- **Choice:** Branching logic
- **Parallel:** Concurrent execution
- **Map:** Iterate over array (inline or distributed)
- **Wait:** Delay execution
- **Pass:** Transform data
- **Succeed/Fail:** Terminal states

#### Service Integrations

- **Optimized:** Direct integration (DynamoDB, SQS, SNS, etc.)
- **SDK:** Any AWS service via API call
- **Request/Response vs Sync/Async:** Control execution patterns

#### Error Handling

- **Retry:** Exponential backoff, max attempts
- **Catch:** Route to error handling states
- **Timeouts:** HeartbeatSeconds, TimeoutSeconds

---

### 4.5 Amazon SQS

#### Queue Types

|Feature|Standard|FIFO|
|---|---|---|
|Throughput|Unlimited|3,000 msg/s (batching), 300 msg/s (individual)|
|Ordering|Best-effort|Guaranteed|
|Delivery|At-least-once|Exactly-once|
|Deduplication|None|5-minute window|

#### Key Concepts

- **Visibility timeout:** Message hidden while processing (default 30s, max 12h)
- **Long polling:** Reduce empty receives, set ReceiveMessageWaitTimeSeconds
- **DLQ:** Dead-letter queue after maxReceiveCount failures
- **Message retention:** 1 minute to 14 days (default 4 days)
- **Message size:** 256 KB (extended with S3)

#### Lambda Integration

- Event source mapping polls SQS
- Batch size configurable (1-10,000)
- Partial batch response (report individual failures)
- Scaling: Up to 1,000 concurrent Lambda invocations

---

### 4.6 Amazon SNS

#### Core Concepts

- **Topics:** Standard (high throughput) or FIFO (ordering)
- **Subscriptions:** Lambda, SQS, HTTP/S, Email, SMS, Kinesis Firehose
- **Message filtering:** Attribute-based subscription filters

#### Fan-out Pattern

```
Producer → SNS Topic → SQS Queue 1 → Consumer A
                    → SQS Queue 2 → Consumer B
                    → Lambda → Consumer C
```

#### Key Features

- **Message attributes:** Key-value metadata
- **Message filtering:** Reduce unnecessary invocations
- **Raw message delivery:** Skip SNS envelope for SQS/HTTP
- **FIFO topics:** Ordering, deduplication (with FIFO SQS only)

---

### 4.7 Amazon EventBridge

#### Core Concepts

- **Event bus:** Default, custom, or partner (SaaS)
- **Rules:** Pattern matching + target routing
- **Targets:** 20+ services (Lambda, SQS, Step Functions, etc.)

#### Event Patterns

json

```json
{
  "source": ["my.application"],
  "detail-type": ["order.created"],
  "detail": {
    "amount": [{"numeric": [">", 100]}]
  }
}
```

#### Key Features

|Feature|Details|
|---|---|
|**Schema Registry**|Auto-discover and version schemas|
|**Archive & Replay**|Store events, replay for recovery/testing|
|**Cross-account**|Send events between accounts|
|**Scheduler**|Cron and rate-based scheduling (new service)|
|**Pipes**|Point-to-point integration with transformation|

#### EventBridge vs SNS

|Aspect|EventBridge|SNS|
|---|---|---|
|Routing|Content-based|Topic-based|
|Schema|Registry support|None|
|Archive|Built-in|None|
|Throughput|Lower|Higher|
|Use case|Event routing|High-volume fan-out|

---

### 4.8 Additional Serverless Services

#### Amazon Cognito

- **User Pools:** User directory, authentication
- **Identity Pools:** AWS credentials for authenticated/guest users
- **Use case:** API Gateway authentication, mobile app auth

#### AWS AppSync

- **GraphQL:** Managed GraphQL API
- **Real-time:** WebSocket subscriptions
- **Resolvers:** Lambda, DynamoDB, HTTP, RDS
- **Use case:** Mobile/web apps needing real-time data

#### Amazon S3 (Event-Driven Context)

- **Event notifications:** Object created/deleted → Lambda, SQS, SNS, EventBridge
- **S3 Object Lambda:** Transform objects on GET
- **Use case:** Data lake triggers, document processing

#### AWS Secrets Manager

- **Rotation:** Automatic secret rotation (Lambda-based)
- **Cross-account:** Share secrets securely
- **Integration:** Lambda retrieves at runtime

#### AWS Systems Manager Parameter Store

- **Tiers:** Standard (free, 4KB), Advanced (paid, 8KB, policies)
- **SecureString:** KMS encryption
- **Use case:** Configuration, non-rotating secrets

---

### 4.9 Service Limits Quick Reference

|Service|Key Limit|
|---|---|
|Lambda|15 min timeout, 10GB memory, 1000 concurrent (default)|
|API Gateway|10,000 RPS (account), 29s timeout|
|DynamoDB|400KB item, 1MB query result, 20 GSI|
|Step Functions|25,000 events history, 1 year duration|
|SQS|256KB message, 14 days retention|
|SNS|256KB message, 12.5M subscriptions/topic|
|EventBridge|5 targets/rule (can increase), 10,000 rules/bus|

---

## 5. Additional Topics

### 5.1 AWS CDK Deep Dive

#### Key Concepts

- **Constructs:** L1 (CloudFormation), L2 (curated), L3 (patterns)
- **Stacks:** Unit of deployment, maps to CloudFormation
- **Apps:** Collection of stacks
- **Context:** Configuration values, environment-specific

#### CDK vs Terraform (Your Transition)

|Aspect|CDK|Terraform|
|---|---|---|
|Language|TypeScript, Python, Java, C#, Go|HCL|
|State|CloudFormation managed|Self-managed (S3 + DynamoDB)|
|Drift|CloudFormation drift detection|terraform plan|
|Learning curve|Lower for developers|Separate language|
|Multi-cloud|AWS only|Multi-cloud|

**When to choose CDK:**

- AWS-only environments
- Development teams more than ops teams
- Complex constructs needing real programming

**When to keep Terraform:**

- Multi-cloud requirements
- Existing Terraform expertise
- Ops-focused teams

#### CDK Best Practices

- Use L2 constructs (not L1 unless necessary)
- One stack per bounded context
- Share code via constructs, not cross-stack references
- Use CDK Pipelines for CI/CD
- Test with assertions (unit tests for infrastructure)

---

### 5.2 Banking/Financial Services Architecture Patterns

#### Compliance & Audit

**Immutable audit logs:**

- CloudTrail for AWS API calls
- Application-level audit logging to S3 (immutable)
- DynamoDB Streams for data change capture
- Kinesis Firehose for log aggregation

**Data retention:**

- S3 lifecycle policies
- DynamoDB TTL with archive
- Legal hold capabilities

#### Security in Banking Context

**Network isolation:**

- Private subnets for sensitive workloads
- VPC endpoints for AWS services
- No public endpoints for internal APIs

**Encryption:**

- KMS customer-managed keys
- Encryption at rest (default for all services)
- TLS 1.2+ in transit

**Access control:**

- AWS Organizations + SCPs
- IAM Identity Center (SSO)
- Role-based access (no long-lived credentials)

#### High Availability

**Multi-AZ (default for most serverless):**

- Lambda: Automatic
- DynamoDB: Automatic
- API Gateway: Automatic

**Multi-region (for critical):**

- DynamoDB Global Tables
- API Gateway with Route53 failover
- S3 Cross-Region Replication

---

### 5.3 Cost Optimization

#### Serverless Cost Model

- **Lambda:** Per request + duration (GB-seconds)
- **API Gateway:** Per request + data transfer
- **DynamoDB:** Capacity mode dependent
- **Step Functions:** Per state transition

#### Optimization Strategies

|Strategy|Application|
|---|---|
|Right-size Lambda|Memory/CPU correlation, test to find optimal|
|DynamoDB capacity|On-demand for variable, provisioned for predictable|
|API Gateway caching|Reduce Lambda invocations|
|Step Functions Express|High-volume workflows|
|S3 storage classes|Intelligent-Tiering, Glacier for archive|

---

### 5.4 Testing Strategies for Serverless

#### Unit Testing

- Test business logic in isolation
- Mock AWS SDK calls
- Use moto (Python) or aws-sdk-mock (JS)

#### Integration Testing

- Test against real AWS services
- Use LocalStack for local development
- Separate test account/environment

#### Contract Testing

- API contracts (OpenAPI)
- Event contracts (EventBridge Schema Registry)

#### End-to-End Testing

- Deploy to test environment
- Test full workflows
- Include failure scenarios

---

## 6. Behavioral & Soft Skills

### 6.1 Expected Behavioral Questions

**Leadership & Influence:**

- "Tell me about a time you influenced a technical decision without direct authority."
- "How do you get buy-in from developers for architectural standards?"

**Problem Solving:**

- "Describe a complex technical problem you solved."
- "Tell me about a time you had to make a decision with incomplete information."

**Collaboration:**

- "How do you handle disagreements with developers about technical approach?"
- "Tell me about working with non-technical stakeholders."

**Adaptability:**

- "Tell me about a time you had to quickly learn a new technology."
- "How do you handle changing requirements mid-project?"

### 6.2 Your Key Strengths to Emphasize

Based on your CV and background:

1. **End-to-end ownership** - You've led from design through production
2. **Practical architecture** - Hands-on, not ivory tower
3. **Data platform expertise** - Deep understanding of data-intensive systems
4. **IaC experience** - Automation-first mindset
5. **Business alignment** - You connect technical decisions to business goals

### 6.3 Potential Concerns to Address

**No explicit CDK experience (yet):** "I've been using Terraform extensively, and I'm actively adopting CDK. The concepts translate well—I understand CloudFormation fundamentals, and CDK's programming model aligns with how I think about infrastructure. I'm confident in my ability to become proficient quickly."

**No direct banking experience:** "While I haven't worked in banking specifically, I've worked with sensitive data and compliance requirements in pharmaceutical and retail contexts. I understand the importance of audit trails, data governance, and security-by-design. I'm excited to learn the specific regulatory requirements of banking."

---

## 7. Questions to Ask the Interviewer

### Technical/Architecture

- "What's your current serverless stack? Are there services you're looking to adopt?"
- "How do you handle cross-team architectural decisions? Is there an architecture review process?"
- "What's the biggest technical challenge you're facing in your serverless adoption?"
- "How do you approach technical debt in your serverless applications?"

### Team & Culture

- "How is the team structured? How many developers would I be working with?"
- "What does success look like for this role in the first 6 months?"
- "How do you balance innovation with stability in a banking context?"
- "What's the relationship between architecture and development teams?"

### Process & Tools

- "What does your CI/CD pipeline look like today?"
- "How do you approach testing for serverless applications?"
- "What monitoring and observability tools do you use?"
- "How do you handle production incidents?"

### Growth & Development

- "What learning and development opportunities are available?"
- "Is there a path toward solution architecture or broader cloud architecture?"
- "How do you stay current with AWS developments as a team?"

---

## Appendix: Quick Reference Cards

### A. STAR Story Checklist

For each project story, ensure you can articulate:

- [ ]  Business context (why was this needed?)
- [ ]  Technical constraints (what were the limitations?)
- [ ]  Your specific decisions (what did YOU decide?)
- [ ]  Trade-offs considered (what alternatives existed?)
- [ ]  Measurable outcomes (what was the result?)
- [ ]  Lessons learned (what would you do differently?)

### B. Architecture Discussion Framework

When presented with a design problem:

1. **Clarify requirements** (functional, non-functional)
2. **Identify constraints** (scale, latency, cost, compliance)
3. **Propose high-level design** (draw components)
4. **Deep dive on key areas** (data model, security, error handling)
5. **Discuss trade-offs** (alternatives considered)
6. **Address operations** (monitoring, deployment, DR)

### C. Key Numbers to Remember

|Metric|Value|
|---|---|
|Lambda max timeout|15 minutes|
|Lambda max memory|10 GB|
|API Gateway timeout|29 seconds|
|DynamoDB item size|400 KB|
|SQS message size|256 KB|
|Step Functions history|25,000 events|
|SNS message size|256 KB|

---

_Good luck with the interview, David! 🚀_