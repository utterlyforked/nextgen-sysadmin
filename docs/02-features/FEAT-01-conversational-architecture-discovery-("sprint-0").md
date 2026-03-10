# Conversational Architecture Discovery ("Sprint 0") - Feature Requirements

**Version**: 1.0  
**Date**: 2025-01-09  
**Status**: Initial - Awaiting Tech Lead Review

---

## Feature Overview

Conversational Architecture Discovery replaces traditional Infrastructure-as-Code writing with an AI-powered interview process that extracts application architecture intent from developers through natural conversation. Instead of forcing developers to write Terraform HCL or CloudFormation templates, the system asks targeted questions about their application needs (APIs, databases, workers, queues) and generates a visual topology diagram that serves as the contract between the developer and the infrastructure provisioning system.

This feature is the entry point to the entire AIE system - it's how developers onboard new projects. The goal is to reduce time-to-first-infrastructure from hours (writing and debugging HCL) to minutes (having a conversation and approving a diagram). The output is a "Blueprint Intent" - a structured, machine-readable representation of the application architecture that downstream features (Drift Detection, Environment Promotion) use as the source of truth.

This feature explicitly does NOT provision any cloud resources - it only captures intent and generates a visual preview. Actual provisioning happens in a separate workflow after developer approval.

---

## User Stories

- As a full-stack developer building a new API service, I want to describe what my application needs (database, cache, background jobs) in plain English so that I can get infrastructure without learning Terraform syntax

- As a full-stack developer, I want the system to ask me clarifying questions when my description is ambiguous (like "Do you need a public endpoint or is this internal-only?") so that I don't have to guess what information the system needs

- As a full-stack developer, I want to see a visual diagram of my proposed infrastructure before anything is provisioned so that I can catch mistakes (like accidentally exposing a database publicly) before they become security incidents

- As a full-stack developer, I want to iterate on the architecture diagram conversationally ("Add Redis for session storage", "Make the worker queue private") so that I can refine my infrastructure design without starting over

- As a full-stack developer, I want the approved diagram to become the "source of truth" for my project so that future features (like drift detection) know what the infrastructure should look like

---

## Acceptance Criteria

- Developer can initiate a new project conversation through a web interface by clicking "Create New Project" and providing a project name
- System conducts a structured interview using conversational AI that identifies:
  - **Entry Points**: Public API endpoints, Background Workers, Scheduled Jobs (cron-like tasks)
  - **Backing Services**: Database (type, size), Cache (Redis/Memcached), Message Queue (SQS), Object Storage (S3)
  - **Twelve-Factor App characteristics**: Stateless vs stateful, environment variables needed, horizontal scaling requirements
- System generates a live, interactive topology diagram (Mermaid.js format) that displays:
  - Application components (API, Worker, Scheduler) as nodes
  - Backing services (RDS, ElastiCache, SQS, S3) as nodes
  - Network boundaries (public internet, VPC private subnets) as containers
  - Data flow arrows showing how components connect
- Developer can modify the architecture conversationally (natural language like "Add a Redis cache between API and database") and the diagram updates in real-time
- System prevents nonsensical architectures (like workers with no queue, or public databases) by asking clarifying questions before updating the diagram
- Developer must explicitly approve the diagram ("Looks good, provision this") before the Blueprint Intent is saved
- Once approved, the system stores the Blueprint Intent as a JSON document in S3 with schema version, timestamp, and developer approval signature
- No AWS resources are provisioned during this feature - provisioning is out of scope

---

## Functional Requirements

### Conversation Initialization

**What**: Developer lands on the AIE web application and starts a new project by providing a project name and optional description.

**Why**: Establishes project context and allows the system to personalize questions (e.g., "You mentioned this is an e-commerce API - do you need payment processing integration?")

**Behavior**:
- Web form with two fields: "Project Name" (required, 1-64 chars, alphanumeric + hyphens) and "Description" (optional, max 500 chars)
- System validates project name is unique within the user's account
- On submit, system creates a conversation session (stored in DynamoDB) with session ID, project name, timestamp, and status="in_progress"
- Developer is redirected to the Conversation Interface screen

---

### Structured Interview Flow

**What**: AI agent asks a sequence of questions designed to extract the minimal information needed to generate an infrastructure diagram. Questions adapt based on previous answers.

**Why**: Developers shouldn't have to know what information is needed upfront. The system guides them through a logical discovery process.

**Behavior**:
- System uses a question tree with 3 phases:
  1. **Entry Points** (2-4 questions): "Does your application have a public API endpoint?" → "Will it handle HTTP requests or WebSocket connections?" → "Do you have background jobs that process work asynchronously?"
  2. **Backing Services** (3-6 questions): "Do you need a database?" → "What type: relational (PostgreSQL) or document (MongoDB)?" → "Do you need caching?" → "Do you need a message queue for async tasks?"
  3. **Operational Characteristics** (2-3 questions): "Is your application stateless (can run multiple copies)?" → "What environment variables do you need (database URL, API keys, etc.)?" → "Do you expect traffic spikes that require autoscaling?"
- Questions are displayed one at a time in a chat-like interface
- Developer responds in natural language (not multiple choice)
- AI agent parses responses using structured Pydantic models to extract intent (e.g., "Yeah I need Postgres" → `{database: {type: "postgresql", confirmed: true}}`)
- If response is ambiguous, agent asks clarifying follow-up (e.g., "You mentioned 'database' - did you mean a relational database like PostgreSQL or a document store like MongoDB?")
- Developer can go back and change previous answers ("Actually, I don't need Redis")
- Conversation history is visible in left sidebar (like ChatGPT interface)

---

### Topology Diagram Generation

**What**: As the conversation progresses, the system incrementally builds a visual diagram showing the infrastructure components and their relationships.

**Why**: Visual feedback helps developers verify the system understood correctly and catches misunderstandings before provisioning.

**Behavior**:
- Diagram is displayed in the right panel of the screen (split view: conversation on left, diagram on right)
- Diagram updates in real-time as each answer is processed (e.g., after developer confirms "Yes, PostgreSQL database", a new RDS node appears)
- Diagram uses Mermaid.js syntax rendered in the browser
- Diagram structure:
  - **Top layer**: Public internet (if public API exists)
  - **Second layer**: Application components (API Gateway → Lambda, ECS Task, etc.)
  - **Third layer**: Backing services (RDS, ElastiCache, SQS, S3)
  - **Bottom layer**: Private networking (VPC, security groups)
- Arrows show data flow (e.g., API → Database, Worker → Queue)
- Each node shows key properties (e.g., RDS node shows "PostgreSQL 15, db.t3.micro")
- Color coding: Green = public-facing, Blue = private, Orange = data stores

---

### Conversational Architecture Refinement

**What**: After the initial interview, developer can modify the architecture by describing changes in natural language.

**Why**: Developers often realize they forgot something or want to adjust the design after seeing the visual diagram.

**Behavior**:
- Developer types refinement requests in the same conversation interface: "Add a Redis cache for session storage", "Make the background worker private (no public access)", "Change PostgreSQL to MySQL"
- AI agent parses the intent and updates the Blueprint JSON
- Diagram regenerates to reflect changes
- System validates changes for architectural consistency:
  - **Prevent orphan services**: If developer removes the API but Redis cache was for API sessions, system warns "Redis cache was configured for API sessions, but you removed the API. Should I remove Redis too?"
  - **Prevent public data stores**: If developer tries to make database public, system warns "Exposing databases publicly is a security risk. Did you mean to allow public access? (yes/no)"
- Developer can iterate unlimited times - no changes are saved until explicit approval

---

### Blueprint Intent Approval & Storage

**What**: Developer reviews the final diagram and explicitly approves it, which locks in the architecture as the "Blueprint Intent" for the project.

**Why**: Approval creates an audit trail showing the developer confirmed this is the desired infrastructure. Prevents later disputes ("I didn't ask for a public database").

**Behavior**:
- When developer is satisfied with diagram, they click "Approve & Save Blueprint"
- System displays a confirmation modal summarizing:
  - Total estimated monthly cost (rough AWS pricing estimate based on chosen instance sizes)
  - Security summary (e.g., "1 public endpoint, 3 private services, 0 security warnings")
  - Resource count (e.g., "5 AWS resources will be created")
- Developer confirms by typing project name (prevents accidental approval)
- System generates Blueprint Intent JSON with schema:
  ```json
  {
    "version": "1.0",
    "project_name": "my-api",
    "approved_at": "2025-01-09T10:30:00Z",
    "approved_by": "developer@example.com",
    "components": [
      {
        "type": "api_gateway",
        "name": "public-api",
        "properties": {"protocol": "https", "auth": "jwt"}
      },
      {
        "type": "rds_postgresql",
        "name": "main-db",
        "properties": {"version": "15", "instance": "db.t3.micro", "storage_gb": 20}
      }
    ],
    "network": {
      "vpc": true,
      "public_subnets": ["public-api"],
      "private_subnets": ["main-db"]
    }
  }
  ```
- Blueprint JSON is stored in S3 at `s3://aie-blueprints/{project_id}/intent-v1.json`
- Conversation session status is updated to "approved"
- Developer is redirected to Project Dashboard (out of scope for this feature - just show success message)

---

## Data Model (Initial Thoughts)

**Entities This Feature Needs**:

- **ConversationSession**: Tracks the interview process
  - `session_id` (UUID, primary key)
  - `project_name` (string)
  - `developer_email` (string)
  - `status` (enum: in_progress, approved, abandoned)
  - `created_at`, `updated_at` (timestamps)
  - `conversation_history` (JSON array of {role: "agent"|"developer", message: string, timestamp})

- **BlueprintIntent**: The approved architecture
  - `blueprint_id` (UUID, primary key)
  - `project_name` (string)
  - `version` (string, e.g., "1.0")
  - `approved_at` (timestamp)
  - `approved_by` (developer email)
  - `schema_json` (JSON blob containing components, network, properties)
  - `s3_uri` (S3 location of full blueprint document)

- **ComponentTemplate**: Predefined infrastructure patterns (pre-populated, not user-created)
  - `template_id` (UUID)
  - `component_type` (string: "api_gateway", "rds_postgresql", "elasticache_redis")
  - `default_properties` (JSON: default instance sizes, versions)
  - `validation_rules` (JSON: e.g., "rds cannot be in public subnet")

**Key Relationships**:
- ConversationSession → BlueprintIntent (one-to-one: each approved session creates one blueprint)
- BlueprintIntent references multiple ComponentTemplates (many-to-many: blueprint uses many component types)

**Storage Decisions**:
- ConversationSession: DynamoDB (fast lookups by session_id, supports JSON conversation history)
- BlueprintIntent: DynamoDB for metadata + S3 for full JSON document (DynamoDB for queries, S3 for large documents)
- ComponentTemplate: DynamoDB (read-heavy, pre-populated during deployment)

---

## User Experience Flow

**Happy Path**:

1. Developer opens AIE web app and clicks "Create New Project"
2. Developer enters project name "my-ecommerce-api" and description "REST API for online store with product catalog"
3. System creates conversation session and asks first question: "Does your application have a public API endpoint that users will access?"
4. Developer types "Yes, it's a REST API"
5. System updates diagram (adds API Gateway node) and asks: "Do you need a database to store data?"
6. Developer responds "Yes, PostgreSQL"
7. System updates diagram (adds RDS PostgreSQL node, adds arrow from API to RDS) and asks: "Do you need caching to improve performance?"
8. Developer responds "Yes, Redis for session storage"
9. System updates diagram (adds ElastiCache Redis node, adds arrow from API to Redis)
10. System asks: "Do you have background jobs that process work asynchronously?"
11. Developer responds "No background jobs"
12. System asks: "Is your API stateless? Can we run multiple copies for scaling?"
13. Developer responds "Yes, stateless"
14. System summarizes: "I've designed an architecture with: Public API (AWS Lambda), PostgreSQL database (RDS), Redis cache (ElastiCache). Does this look correct?"
15. Developer reviews diagram and types "Looks good"
16. Developer clicks "Approve & Save Blueprint"
17. System shows confirmation modal with cost estimate ($47/month) and security summary (1 public endpoint, 2 private services)
18. Developer types project name to confirm
19. System saves Blueprint Intent to S3 and DynamoDB
20. Developer sees success message: "Blueprint approved! Next step: provision Dev environment."

**Alternate Path (Iterative Refinement)**:

- At step 15, developer notices they forgot something: "Wait, I also need an S3 bucket for file uploads"
- System asks: "Should the S3 bucket allow public uploads or only from the API?"
- Developer responds "Only from the API"
- System updates diagram (adds S3 node in private subnet, adds arrow from API to S3)
- Developer reviews updated diagram and clicks "Approve & Save Blueprint"

---

## Edge Cases & Constraints

- **Incomplete Conversation**: If developer abandons conversation (closes browser), session remains in "in_progress" state. Session expires after 24 hours of inactivity. Developer can resume from conversation history if they return within 24 hours.

- **Ambiguous Responses**: If AI agent cannot confidently parse developer's response (confidence score < 0.7), agent asks explicit clarifying question rather than guessing. Example: Developer says "I need storage" → Agent asks "Do you mean object storage (S3) for files, or a database for structured data?"

- **Unsupported Components**: If developer requests infrastructure not supported in V1 (e.g., "I need Kubernetes"), system responds: "Kubernetes (EKS) is not available in this version. For V1, we support Lambda functions and ECS containers. Which would you prefer?"

- **Contradictory Requests**: If developer makes contradictory statements (e.g., "I need a database" then later "I don't need any data storage"), system highlights the conflict: "Earlier you mentioned needing a database, but now you said no data storage. Should I remove the database? (yes/no)"

- **Diagram Complexity Limit**: If diagram exceeds 15 components, system warns: "Your architecture is complex (15+ components). Consider breaking this into multiple projects for better manageability." Still allows approval, just warns.

- **Cost Threshold**: If estimated monthly cost exceeds $500, system shows additional confirmation: "This architecture will cost ~$X/month. Confirm you understand the cost. (yes/no)"

- **Security Red Flags**: If diagram contains obvious security issues (public database, unencrypted storage), system blocks approval with error message explaining the issue and how to fix it.

---

## Out of Scope for This Feature

**Explicitly NOT included**:

- **Actual AWS resource provisioning**: This feature only captures intent and generates diagrams. Provisioning happens in a separate workflow (future feature).

- **Cost optimization**: Cost estimate is rough order-of-magnitude only, not exact AWS pricing. Detailed cost analysis is a V2 feature.

- **Multi-region architectures**: V1 supports single AWS region only (us-east-1 default). Developer cannot specify region during interview.

- **Custom VPC configurations**: System uses default VPC setup (public/private subnets). Advanced networking (VPN, VPC peering, custom CIDR blocks) is out of scope.

- **IAM role/policy design**: System applies default least-privilege IAM policies. Custom IAM permissions are out of scope.

- **Editing approved blueprints**: Once approved, blueprint is immutable. To change architecture, developer must start a new conversation (V2 will support blueprint versioning).

- **Team collaboration**: Single developer per project. No approval workflows, commenting, or sharing (V2 feature).

- **Import existing infrastructure**: This feature is for greenfield projects only. Importing existing AWS resources is out of scope.

---

## Open Questions for Tech Lead

1. **AI Model Selection**: Should we use OpenAI GPT-4, Anthropic Claude, or a self-hosted open-source model (like Llama)? Trade-offs: GPT-4 is expensive but highly capable, Claude has better security for sensitive data, open-source avoids vendor lock-in but requires more prompt engineering.

2. **Conversation State Management**: How should we handle long conversations that exceed LLM context windows (e.g., developer asks 50 questions)? Options: (a) Summarize old conversation history, (b) Use RAG to retrieve relevant past exchanges, (c) Limit conversation to 20 exchanges and force restart.

3. **Diagram Rendering**: Mermaid.js renders in the browser (client-side). Should we also generate static PNG/SVG images server-side for audit trail purposes? PNG would make it easier to include diagrams in email notifications or reports.

4. **Schema Validation**: Who validates that the Blueprint JSON matches expected schema - the AI agent (pre-save) or a separate validation service (post-save)? Trade-off: Agent validation prevents bad data but increases prompt complexity; post-save validation is cleaner separation but requires rollback logic.

5. **Developer Authentication**: How should we identify the "approved_by" developer? Options: (a) Email/password auth, (b) OAuth (Google, GitHub), (c) AWS IAM identity federation. Preference for security and ease of use?

---

## Dependencies

**Depends On**: 
- User authentication system (out of scope for this feature spec, but required before this feature can function)
- AWS credentials management (how does AIE get permission to eventually provision resources? Assume developer provides IAM role ARN during account setup)

**Enables**: 
- Feature 2 (Verified Environment Promotion) - requires Blueprint Intent as input
- Feature 3 (Active Drift Detection) - requires Blueprint Intent as "expected state"
- Future provisioning workflow - requires approved Blueprint Intent to know what to create