# Verified Environment Promotion ("Dev to Prod Hydration") - Feature Requirements

**Version**: 1.0  
**Date**: 2025-01-09  
**Status**: Initial - Awaiting Tech Lead Review

---

## Feature Overview

Verified Environment Promotion eliminates the manual, error-prone process of copying infrastructure configuration from Development to Production environments. Instead of developers duplicating Terraform files and manually adjusting settings for production requirements, this feature automatically "hydrates" a proven Dev environment into a Production-ready configuration.

The core innovation is capturing the "DNA of success" - when a Dev environment is working correctly (as validated by developer-defined success criteria), the system snapshots the live infrastructure state directly from AWS APIs, applies automatic security and scalability upgrades via Warden Policy overlays, and provisions a Production environment that maintains functional parity while meeting organizational standards.

This feature sits at the heart of AIE's value proposition: it bridges the gap between "it works on my machine" and "it's production-ready" without requiring developers to understand the operational differences between environments. DevOps engineers define the policies once; developers benefit from automatic upgrades on every promotion.

**Why This Matters**: In traditional Infrastructure-as-Code workflows, environment promotion is a high-risk manual process. Developers copy configuration files, manually adjust instance sizes and availability zones, and hope they didn't miss a security setting. This feature makes promotion a one-click operation with automatic safety upgrades and a clear audit trail of what changed and why.

---

## User Stories

### Developer Workflows

- As a full-stack developer, I want to define what "working correctly" means for my Dev environment so that promotion only happens when I'm confident the infrastructure is stable
- As a full-stack developer, I want to promote my working Dev environment to Prod with a single action so that I can ship features without manually copying configuration files
- As a full-stack developer, I want to see exactly what will change between Dev and Prod before promotion happens so that I understand what security and scale upgrades are being applied
- As a full-stack developer, I want the system to automatically upgrade my Dev configuration to meet Production standards (multi-AZ, encryption, backups) so that I don't have to learn every operational best practice
- As a full-stack developer, I want to verify that my Prod environment meets the same success criteria as Dev so that I know the promotion worked correctly

### DevOps Workflows

- As a DevOps engineer, I want to define "Warden Policies" that automatically apply to all Prod promotions so that security and compliance standards are enforced without me reviewing every deployment
- As a DevOps engineer, I want to see a log of all Dev-to-Prod promotions showing what upgrades were applied so that I can audit compliance and refine policies over time

---

## Acceptance Criteria

### Success Criteria Definition
- Developer can define multiple Success Criteria for their Dev environment through the web UI
- Each criterion specifies a validation type: HTTP health check (URL returns expected status code), AWS resource state check (RDS status = "available"), Custom script execution (exit code 0 = success)
- Success Criteria are stored per environment (Dev criteria may differ from Prod criteria)
- System continuously monitors Dev environment against defined Success Criteria (check interval: every 2 minutes)
- Developer can see Success Criteria status in real-time: "All criteria met" (green), "N criteria failing" (red), "Criteria not yet evaluated" (gray)

### Promotion Initiation
- "Promote to Prod" button is only enabled when all Dev Success Criteria are met
- When developer clicks "Promote to Prod", system initiates Hydration process
- System displays progress indicator: "Snapshotting Dev environment..." → "Applying Warden Policies..." → "Generating Promotion Preview..."

### Configuration Snapshot (Hydrator)
- Hydrator queries AWS APIs (Boto3) to capture live state of all managed resources in Dev environment
- Snapshot captures: Resource type and identifier (e.g., RDS instance ID), All configuration parameters (instance size, storage size, engine version, security groups, etc.), Network configuration (VPC, subnets, route tables), IAM roles and policies attached to resources
- Snapshot does NOT rely on Terraform state files - source of truth is AWS
- Snapshot is stored as structured JSON in S3 with timestamp and developer identifier

### Warden Policy Application
- API Warden retrieves pre-defined Prod promotion policies from policy store
- Warden applies transformation rules to Dev snapshot: **Database upgrades**: Single-AZ → Multi-AZ, No automated backups → Automated backups (retention: 7 days), Unencrypted → Encrypted at rest (AWS-managed KMS key), db.t3.micro → db.t3.medium (configurable scaling rule), **Compute upgrades**: Single availability zone → Multi-AZ deployment, Development security groups → Production security groups (least-privilege), **Storage upgrades**: Unencrypted S3 buckets → Encrypted with SSE-S3, No versioning → Versioning enabled, No lifecycle policies → 90-day transition to Glacier
- Warden generates upgraded Prod configuration as structured JSON
- Warden logs all applied transformations with policy rule references

### Promotion Preview
- System generates side-by-side comparison view in web UI
- Left column: "Dev Configuration" (current state)
- Right column: "Prod Configuration" (proposed state)
- Differences highlighted with color coding: Green = Security upgrade, Blue = Scale upgrade, Yellow = Configuration change
- Each difference includes annotation explaining WHY: "Multi-AZ enabled for high availability (Warden Policy: PROD-HA-001)"
- Developer can expand each resource to see full configuration diff
- Preview includes cost estimate: "Estimated monthly cost increase: $X.XX" (based on AWS pricing APIs)

### Promotion Approval & Execution
- Developer must explicitly click "Approve & Provision Prod" button
- Optional: Developer can add promotion notes (free text, max 500 chars)
- System provisions Prod environment using upgraded configuration: Uses AWS CloudFormation or direct Boto3 API calls (to be determined by Tech Lead), Provisions resources in dependency order (VPC → Subnets → RDS → App servers), Displays real-time provisioning log in web UI
- Provisioning timeout: 30 minutes (if exceeded, promotion fails and resources are cleaned up)

### Prod Validation
- Once Prod environment is provisioned, system runs Prod Success Criteria validations
- Prod Success Criteria are copied from Dev with adjusted thresholds: HTTP health checks use Prod URL, Timeouts may be stricter (production SLAs)
- If Prod Success Criteria fail: System marks promotion as "Provisioned but not validated", Developer can investigate issues and manually mark as validated, Rollback is manual in V1 (automated rollback out of scope)
- If Prod Success Criteria pass: System marks promotion as "Complete", Blueprint Intent for Prod environment is updated to reflect new state

### Audit Trail
- Every promotion is logged in Decision Ledger with: Timestamp, Developer identifier, Dev environment snapshot ID, Applied Warden Policies, Promotion Preview (before/after diff), Approval timestamp, Prod provisioning log, Validation results
- Decision Ledger entry is immutable and exportable for compliance audits

---

## Functional Requirements

### Success Criteria Management

**What**: Developer-defined validation rules that determine when a Dev environment is "promotion-ready"

**Why**: Prevents promoting broken or incomplete environments to Production. Ensures developer has confidence that infrastructure is working before promotion.

**Behavior**:
- Developer accesses "Success Criteria" section in Dev environment dashboard
- Developer adds criteria by selecting validation type from dropdown: "HTTP Health Check", "AWS Resource State", "Custom Script"
- For HTTP Health Check: Developer provides URL (must start with `http://` or `https://`), Expected status code (default: 200), Timeout in seconds (default: 10), Optional: Expected response body substring
- For AWS Resource State: Developer selects resource from list of managed resources (e.g., RDS instance), Selects expected state from dropdown (e.g., "available", "running"), System validates state against AWS API response
- For Custom Script: Developer uploads shell script (max 10KB), Script runs in isolated Lambda environment with AWS credentials, Exit code 0 = success, non-zero = failure, Stdout/stderr captured and displayed in UI
- Criteria are evaluated continuously (every 2 minutes) for Dev environment
- Evaluation results displayed in UI with timestamp: "Last checked: 2 minutes ago - ✅ All criteria met"

### Hydration Engine (Configuration Snapshot)

**What**: Component that captures complete configuration state of a live Dev environment by querying AWS APIs

**Why**: Traditional promotion relies on copying Terraform files, which may be out of sync with reality. Hydration captures actual deployed state as source of truth.

**Behavior**:
- When promotion initiated, Hydrator receives Dev environment identifier
- Hydrator queries AWS APIs (Boto3) to enumerate all managed resources: Scans tags to identify resources belonging to this environment (tag: `aie:environment=dev-{project-id}`), Retrieves full configuration for each resource type using service-specific Describe APIs (e.g., `describe_db_instances` for RDS)
- Hydrator normalizes configuration into standard schema (Pydantic models): Strips AWS-generated metadata (ARNs, creation timestamps), Preserves developer-controlled configuration (instance size, engine version, parameters), Preserves network topology (VPC IDs, subnet IDs, security group rules)
- Snapshot stored as JSON in S3: `s3://aie-snapshots/{project-id}/dev-{timestamp}.json`
- Snapshot includes metadata: Timestamp, AWS region, Developer user ID, List of resources captured

### Warden Policy Engine (Upgrade Transformation)

**What**: Rule engine that applies organizational security and operational standards to Dev configurations to produce Prod configurations

**Why**: DevOps engineers define policies once; every promotion automatically benefits from upgrades without manual review.

**Behavior**:
- API Warden loads Prod promotion policies from policy store (DynamoDB table)
- Policies defined as transformation rules (JSON schema): `{ "resource_type": "rds", "transforms": [ {"field": "multi_az", "from": false, "to": true, "reason": "High availability required for production"}, {"field": "backup_retention_period", "from": 0, "to": 7, "reason": "Disaster recovery compliance"} ] }`
- Warden iterates through Dev snapshot, applying matching policies: For each resource in snapshot, find policies matching resource type, Apply transformations in order, Log each transformation with policy rule ID
- Warden validates transformed configuration against AWS API constraints: Instance sizes are valid for region, Security group rules are syntactically correct, KMS keys exist if encryption specified
- Output: Upgraded Prod configuration JSON with same schema as Dev snapshot
- Side output: Transformation log (list of changes with policy references)

### Promotion Preview Generator

**What**: UI component that renders side-by-side diff of Dev vs Prod configurations with human-readable explanations

**Why**: Developer needs to understand what's changing and why before approving a production deployment.

**Behavior**:
- Receives Dev snapshot JSON and upgraded Prod configuration JSON
- Performs deep object diff to identify changes: Added fields, Removed fields, Modified values
- Renders comparison table in web UI: Each row = one configuration difference, Columns: "Dev Value" | "Prod Value" | "Reason for Change", Color coding: Green background = security upgrade, Blue background = scale upgrade
- For complex nested objects (security groups, IAM policies): Renders expandable section with formatted JSON diff, Syntax highlighting for added/removed lines
- Includes cost estimate section: Queries AWS Pricing API for resource costs, Calculates monthly cost for Dev config, Calculates monthly cost for Prod config, Displays delta: "Production will cost $X.XX more per month"
- Preview is paginated if more than 50 differences (unlikely in MVP)

### Provisioning Orchestrator

**What**: Component that executes the actual provisioning of Prod environment using upgraded configuration

**Why**: Translates approved configuration into live AWS resources in the correct dependency order.

**Behavior**:
- Receives approved Prod configuration JSON
- Determines provisioning order based on resource dependencies: VPC → Subnets → Internet Gateway → Route Tables → Security Groups → RDS instances → EC2 instances → Load Balancers
- For each resource in order: Calls AWS API to create resource (Boto3), Polls for resource to reach ready state (e.g., RDS "available"), Logs progress to web UI via WebSocket, Handles errors: Retries transient failures (rate limits), Fails fast on configuration errors (invalid parameter), Stores error details in Decision Ledger
- Tags all provisioned resources with: `aie:environment=prod-{project-id}`, `aie:promoted-from=dev-{timestamp}`, `aie:warden-policy-version={version}`
- On success: Updates Blueprint Intent for Prod environment to match provisioned state, Triggers Prod Success Criteria validation
- On failure: Logs error, Marks promotion as "Failed", Does NOT automatically clean up resources (manual cleanup in V1)

### Prod Validation Engine

**What**: Component that runs Success Criteria checks against newly provisioned Prod environment

**Why**: Ensures promotion actually worked and Prod environment is functional, not just provisioned.

**Behavior**:
- Copies Success Criteria definitions from Dev environment
- Adjusts criteria for Prod context: Replaces Dev URLs with Prod URLs (pattern substitution: `dev.example.com` → `prod.example.com`), May apply stricter timeouts if defined in criteria metadata
- Executes each criterion: HTTP checks use Lambda function with outbound internet access, AWS resource state checks query Prod resource APIs, Custom scripts run in isolated Lambda with Prod AWS credentials
- Collects results: Timestamp of each check, Pass/fail status, Error messages if failed, Response time for HTTP checks
- Updates Promotion status in Decision Ledger: "Complete" if all criteria pass, "Provisioned - Validation Failed" if any criterion fails
- Sends notification to developer: In-app notification, Email summary with validation results

---

## Data Model (Initial Thoughts)

**Entities This Feature Needs**:

- **SuccessCriterion**: Validation rule for an environment
  - Fields: `id`, `environment_id`, `type` (HTTP/Resource/Script), `config` (JSON), `enabled`, `created_at`
  - Relationships: Belongs to Environment

- **SuccessCriterionResult**: Outcome of a criterion evaluation
  - Fields: `id`, `criterion_id`, `timestamp`, `passed`, `error_message`, `response_time_ms`
  - Relationships: Belongs to SuccessCriterion

- **EnvironmentSnapshot**: Captured state of a Dev environment
  - Fields: `id`, `environment_id`, `timestamp`, `s3_location`, `resource_count`, `captured_by_user_id`
  - Relationships: Belongs to Environment, references S3 JSON blob

- **WardenPolicy**: Transformation rule for Prod promotions
  - Fields: `id`, `resource_type`, `transform_rules` (JSON), `version`, `created_at`
  - Relationships: None (global policies)

- **Promotion**: Record of a Dev-to-Prod promotion attempt
  - Fields: `id`, `source_environment_id`, `target_environment_id`, `snapshot_id`, `status` (Initiated/Previewed/Approved/Provisioning/Complete/Failed), `approved_at`, `completed_at`, `developer_notes`
  - Relationships: References EnvironmentSnapshot, has many PromotionTransformations, has many PromotionValidations

- **PromotionTransformation**: Individual config change applied during promotion
  - Fields: `id`, `promotion_id`, `resource_type`, `resource_id`, `field_path`, `old_value`, `new_value`, `policy_id`, `reason`
  - Relationships: Belongs to Promotion

- **PromotionValidation**: Prod Success Criterion result after promotion
  - Fields: `id`, `promotion_id`, `criterion_id`, `timestamp`, `passed`, `error_message`
  - Relationships: Belongs to Promotion

**Key Relationships**:
- Environment (1) → (many) SuccessCriteria
- Environment (1) → (many) EnvironmentSnapshots
- Promotion (1) → (1) EnvironmentSnapshot (source)
- Promotion (1) → (many) PromotionTransformations
- Promotion (1) → (many) PromotionValidations

**Note**: This is preliminary - Tech Lead will need to clarify database choice (DynamoDB vs RDS Postgres), schema for storing complex JSON config, and whether S3 references are sufficient or if inline storage preferred.

---

## User Experience Flow

### Happy Path: Successful Dev-to-Prod Promotion

1. **Developer defines Success Criteria** (one-time setup):
   - Developer navigates to Dev environment dashboard
   - Clicks "Manage Success Criteria" button
   - Adds HTTP health check: `https://dev-api.example.com/health` returns 200
   - Adds RDS state check: RDS instance `dev-db` status = "available"
   - System immediately begins monitoring criteria (every 2 minutes)

2. **Developer works in Dev until stable**:
   - Developer deploys application code to Dev environment
   - System continuously evaluates Success Criteria
   - Dashboard shows: "⏳ 1 of 2 criteria met - HTTP check passing, RDS check pending..."
   - Once RDS becomes available: "✅ All criteria met (as of 2 minutes ago)"

3. **Developer initiates promotion**:
   - "Promote to Prod" button becomes enabled (was disabled when criteria failing)
   - Developer clicks "Promote to Prod"
   - Modal appears: "Starting promotion from Dev to Prod. This will take 2-5 minutes."
   - Progress bar shows: "Snapshotting Dev environment..."

4. **System generates Promotion Preview**:
   - After 30 seconds, progress updates to: "Applying security policies..."
   - After 1 minute, Promotion Preview screen appears
   - Developer sees side-by-side comparison: 
     - **RDS Instance**: `db.t3.micro, Single-AZ, No backups` → `db.t3.medium, Multi-AZ, 7-day backups` (Reason: "Production high-availability and disaster recovery")
     - **S3 Bucket**: `Unencrypted` → `SSE-S3 encrypted` (Reason: "Data protection compliance")
   - Cost estimate: "Estimated monthly cost: $45 (Dev) → $180 (Prod) - Increase: $135"

5. **Developer approves promotion**:
   - Developer reviews changes, adds note: "Promoting v1.2.0 release"
   - Clicks "Approve & Provision Prod"
   - Provisioning screen shows real-time log:
     - "Creating VPC... ✅ (vpc-12345)"
     - "Creating RDS instance... ⏳ (5-10 minutes)"
     - "RDS instance available ✅"
     - "Creating EC2 instances... ✅"
     - "Promotion complete! Validating..."

6. **System validates Prod environment**:
   - System runs Prod Success Criteria (same as Dev, but against Prod URLs)
   - HTTP check: `https://prod-api.example.com/health` → 200 ✅
   - RDS check: Prod RDS instance status = "available" ✅
   - Dashboard shows: "🎉 Promotion complete! Prod environment validated successfully."
   - Developer receives email: "Your Prod environment is live and healthy."

---

## Edge Cases & Constraints

- **Dev Success Criteria never met**: "Promote to Prod" button remains disabled. Developer must fix Dev environment or adjust criteria. No timeout - developer controls when promotion happens.

- **Success Criteria pass, then fail before promotion**: If criteria status changes from "all passing" to "some failing" after developer clicks "Promote to Prod" but before snapshot completes, system halts promotion and displays warning: "Dev environment changed since promotion initiated. Please verify environment is still stable and retry."

- **Warden Policy produces invalid configuration**: API Warden validates transformed config against AWS API constraints before generating Preview. If validation fails (e.g., policy tries to set unsupported instance type), promotion halts with error: "Policy transformation failed: {reason}. Contact DevOps team."

- **Provisioning fails mid-process**: If Prod resource creation fails (e.g., RDS provisioning timeout), promotion is marked "Failed". Resources created up to that point remain (no auto-cleanup in V1). Developer must manually investigate and either retry or clean up via AWS console.

- **Prod validation fails after successful provisioning**: Promotion marked "Provisioned - Validation Failed". Resources exist but may not be functioning. Developer investigates via logs, may need to manually fix configuration or rollback (manual rollback in V1).

- **Multiple environments beyond Dev/Prod**: V1 supports only Dev → Prod promotion. Staging environments are out of scope. If developer has Staging, they must promote Dev → Staging, then Staging → Prod as two separate operations (treating Staging as "Prod" for first promotion).

- **Concurrent promotions**: V1 does not prevent concurrent promotions for the same project. If two developers try to promote simultaneously, last promotion wins (may overwrite previous). Multi-developer coordination is out of scope (V1 is single developer per project).

- **AWS API rate limits**: Hydration and provisioning may hit AWS API rate limits for accounts with many resources. System implements exponential backoff retries (max 5 retries). If rate limits persist, promotion fails with guidance to retry later.

- **Cost estimate accuracy**: Cost estimates use AWS Pricing API but cannot account for data transfer, API calls, or usage-based charges. Estimate includes disclaimer: "Estimate covers infrastructure costs only. Actual costs may vary based on usage."

- **Warden Policies change between Preview and Approval**: If DevOps engineer updates Warden Policies after developer sees Preview but before clicking "Approve", the outdated policies are used (promotion uses snapshot of policies from Preview generation time). Policy version is logged in Decision Ledger.

---

## Out of Scope for This Feature

**V1 explicitly does NOT include**:

- **Automated rollback**: If Prod validation fails or production issues arise, rollback is manual via AWS console. Automated rollback is V2 feature.

- **Blue/Green or Canary deployments**: Promotion provisions a new Prod environment, not a staged rollout. Traffic shifting is out of scope.

- **Promotion approval workflows**: V1 is single developer - no multi-person approval required. Approval workflows are V2 (team collaboration feature).

- **Custom Warden Policies**: DevOps engineers cannot create or edit policies in V1. Pre-defined policies are hardcoded. Custom policy editor is V2.

- **Rollback to previous Prod version**: No concept of "Prod version history" in V1. Promotion replaces Prod state. Versioning is V2.

- **Promotion scheduling**: Promotions are immediate, not scheduled for maintenance windows. Scheduling is V2.

- **Terraform/CloudFormation import**: This feature only works with AIE-managed environments (created via Feature 1). Importing existing Infrastructure-as-Code is out of scope.

- **Cross-region promotion**: Dev and Prod must be in same AWS region. Multi-region is V2.

- **Partial promotions**: Cannot promote subset of resources (e.g., "only promote database changes"). Promotion is all-or-nothing for the environment.

- **Diff preview for application code**: Promotion Preview shows infrastructure diff only. Application code changes (Docker images, Lambda deployment packages) are not compared.

---

## Open Questions for Tech Lead

**Data Storage**:
- Should EnvironmentSnapshot JSON be stored inline in DynamoDB or only referenced via S3 URL? Trade-off: Inline = faster queries but DynamoDB item size limits (400KB). S3-only = no item size limit but requires two API calls to retrieve.

**Provisioning Implementation**:
- Should Prod provisioning use AWS CloudFormation (declarative, AWS handles dependency order) or direct Boto3 API calls (more control, manual dependency management)? CloudFormation would provide rollback capability, but adds complexity.

**Success Criteria Evaluation**:
- Should Success Criteria run in isolated Lambda functions (secure, scalable) or in backend API workers (simpler, fewer moving parts)? Lambda = harder to debug, workers = shared resource contention.

**Warden Policy Versioning**:
- When DevOps engineer updates a Warden Policy, should all subsequent promotions use the new version immediately, or should projects opt-in to new policy versions? Immediate = risky (policy bugs affect all projects), opt-in = slower security fixes.

**Promotion Idempotency**:
- If developer clicks "Approve & Provision Prod" twice (accidental double-click), should system prevent duplicate provisioning? Need idempotency key mechanism?

**Resource Naming**:
- How should Prod resources be named to avoid collisions with Dev? Should system auto-generate names (e.g., `{project-id}-prod-rds`) or prompt developer to provide names during Preview?

---

## Dependencies

**Depends On**: 
- **Feature 1 (Conversational Architecture Discovery)**: Promotion requires a Blueprint Intent to exist (source of resource inventory). Cannot promote an environment that wasn't created via AIE.
- **API Warden component**: Warden Policy engine must be implemented for promotion to apply security upgrades.
- **Decision Ledger**: Promotion audit trail requires ledger to store immutable records.

**Enables**: 
- **Feature 3 (Active Drift Detection)**: Once Prod environment is promoted, Drift Sentinel can monitor it. Drift detection relies on Blueprint Intent being updated by successful promotions.