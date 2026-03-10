# Active Drift Detection & Reconciliation - Feature Requirements

**Version**: 1.0  
**Date**: 2025-01-XX  
**Status**: Initial - Awaiting Tech Lead Review

---

## Feature Overview

The Self-Healing Sentinel is an active monitoring system that continuously compares live AWS infrastructure against the approved Blueprint Intent, detects unauthorized or unexpected changes within minutes, and provides a conversational interface for developers to reconcile drift through revert or assimilation actions.

This feature solves the "drift paradox" where infrastructure state diverges from intended configuration due to manual console changes, emergency hotfixes, AWS-initiated updates, or external automation. Unlike traditional Infrastructure-as-Code which only detects drift during the next `terraform plan`, the Sentinel actively monitors cloud state every 5 minutes and immediately notifies developers of discrepancies.

The Sentinel doesn't just detect drift—it facilitates intelligent decision-making through a conversational Reconciliation Hub where developers can understand why drift occurred, decide whether to revert unauthorized changes or assimilate emergency fixes back into the blueprint, and maintain a complete audit trail of all infrastructure evolution.

This feature is foundational to AIE's autonomous infrastructure promise: infrastructure that actively maintains alignment with developer intent, self-corrects when possible, and transparently surfaces decisions when human judgment is required.

---

## User Stories

- As a full-stack developer, I want to know within minutes if someone manually changed my infrastructure so that I can catch configuration problems before they cause production outages
- As a full-stack developer, I want to have a conversation with the system about detected drift ("Why did this security group rule change?" "Was this authorized?") so that I can understand the impact before taking action
- As a DevOps engineer, I want to see a chronological log of all infrastructure changes and the reasoning behind reconciliation decisions so that I can audit compliance and troubleshoot incidents
- As a full-stack developer, I want to quickly revert unauthorized console changes with one click so that my infrastructure returns to the approved blueprint state
- As a full-stack developer, I want to incorporate emergency hotfixes I made in the AWS console back into my blueprint so that future deployments don't accidentally undo my fixes
- As a full-stack developer, I want to be notified immediately when drift violates organizational Warden Policies so that I can remediate compliance issues before they're flagged in audits

---

## Acceptance Criteria

- Drift Sentinel agent runs continuously on a 5-minute polling interval for all managed environments (Dev, Staging, Prod)
- Sentinel detects drift within 5 minutes of the change occurring in AWS (MTTD < 5 minutes)
- Drift detection compares three sources: Blueprint Intent (approved architecture diagram), Warden Policy (organizational security/compliance rules), Live Cloud State (queried from AWS APIs)
- System categorizes drift into four types: Resource Added (exists in AWS but not in blueprint), Resource Deleted (in blueprint but missing from AWS), Resource Modified (configuration changed from blueprint values), Policy Violation (resource exists but violates Warden standards)
- Developer receives in-app notification when drift is detected, with notifications optionally sent via email or Slack integration
- Reconciliation Hub displays a timeline view of all drift events for an environment, sorted chronologically
- Each drift event shows: What resource changed (resource type, ID, name), Exact configuration diff (expected vs actual state), Timestamp of detection, Whether change violates Warden Policy, Conversation thread for this drift event
- Developer can interact with each drift event through four actions: Ask Questions (conversational Q&A about the drift), Revert Drift (restore to blueprint state), Assimilate Drift (update blueprint to match live state), Acknowledge Drift (mark as seen, no action needed)
- Revert action uses AWS APIs to restore resource configuration to blueprint state, completes within 2 minutes for standard resources
- Assimilate action requires developer to provide written rationale, updates Blueprint Intent diagram, and creates a new blueprint version
- All drift events, developer actions, and rationales are logged to the Decision Ledger (immutable audit log)
- Decision Ledger is searchable by date range, resource type, action type, and developer
- Decision Ledger exports to JSON or CSV for compliance reporting

---

## Functional Requirements

### Continuous Drift Monitoring

**What**: Background agent that polls AWS APIs every 5 minutes to compare live infrastructure against Blueprint Intent

**Why**: Detects unauthorized changes quickly before they compound into larger problems or cause outages

**Behavior**:
- Sentinel runs as a Lambda function triggered by EventBridge cron (every 5 minutes)
- For each managed environment, Sentinel queries AWS APIs to fetch current state of all resources in the blueprint (EC2 instances, RDS databases, S3 buckets, security groups, IAM roles, etc.)
- Sentinel loads the approved Blueprint Intent from S3 (stored as structured JSON/YAML)
- Sentinel loads applicable Warden Policies from DynamoDB
- Sentinel performs three-way diff: Blueprint vs Warden vs Live State
- Drift is flagged when: Live state has resources not in blueprint, Blueprint specifies resources missing from live state, Resource configuration values differ from blueprint, Resource violates Warden Policy even if it matches blueprint
- Sentinel writes detected drift events to DynamoDB table `DriftEvents` with schema: `{environment_id, resource_id, drift_type, detected_at, expected_state, actual_state, policy_violation_details, status: 'pending'}`
- Sentinel triggers notification Lambda to alert developer

### Drift Classification & Severity

**What**: Categorization system that labels drift by type and assigns severity levels

**Why**: Helps developers prioritize which drift to address first (critical policy violations vs informational AWS updates)

**Behavior**:
- **Drift Types**:
  - `ADDED`: Resource exists in AWS but not in blueprint (e.g., developer manually created an S3 bucket in console)
  - `DELETED`: Resource defined in blueprint but missing from AWS (e.g., someone terminated an EC2 instance)
  - `MODIFIED`: Resource configuration changed from blueprint values (e.g., security group rule added)
  - `POLICY_VIOLATION`: Resource exists and matches blueprint but violates Warden Policy (e.g., RDS lacks encryption)
- **Severity Levels**:
  - `CRITICAL`: Resource deleted, or policy violation of security/compliance rule (e.g., public S3 bucket in Prod)
  - `HIGH`: Modified resource that affects availability or data integrity (e.g., RDS multi-AZ disabled)
  - `MEDIUM`: Added or modified resource that doesn't violate policy but differs from blueprint (e.g., EC2 instance type changed)
  - `LOW`: Informational changes (e.g., AWS updated RDS engine minor version automatically)
- Severity is auto-assigned based on: Drift type (DELETED is always CRITICAL), Resource type (database changes are higher severity than S3 tags), Warden Policy violation flag (always CRITICAL or HIGH)
- Developer can override severity if needed (e.g., downgrade an AWS auto-update from MEDIUM to LOW)

### Reconciliation Hub Interface

**What**: Web UI that displays all drift events in a timeline view with conversation threads for each event

**Why**: Provides a single pane of glass for understanding infrastructure changes and making reconciliation decisions

**Behavior**:
- Accessed at `/environments/{env_id}/drift` route
- Timeline view shows drift events in reverse chronological order (newest first)
- Each drift event card displays:
  - Resource icon and name (e.g., "🗄️ RDS Database: prod-api-db")
  - Drift type badge (ADDED/DELETED/MODIFIED/POLICY_VIOLATION)
  - Severity badge with color coding (CRITICAL=red, HIGH=orange, MEDIUM=yellow, LOW=gray)
  - Timestamp ("Detected 12 minutes ago")
  - Configuration diff (visual side-by-side: Expected ↔ Actual)
  - Policy violation details if applicable
  - Conversation thread icon showing number of messages
- Clicking drift event expands detail panel showing:
  - Full configuration diff (JSON tree diff view)
  - AI-generated explanation: "This security group rule was added manually in the AWS console, allowing SSH access from 0.0.0.0/0. This violates the 'No Public SSH' Warden Policy."
  - Conversation input box for asking questions
  - Action buttons: Revert | Assimilate | Acknowledge
- Filter controls at top: Filter by severity, resource type, status (pending/reverted/assimilated/acknowledged), date range
- Search bar supports queries like "RDS changes last week" or "policy violations"

### Conversational Drift Investigation

**What**: LLM-powered Q&A interface that explains drift context and impact

**Why**: Developers may not understand AWS configuration nuances or why a change matters—conversational explanation reduces cognitive load

**Behavior**:
- Developer types question in drift event conversation thread (e.g., "Why does this matter?" or "What breaks if I revert this?")
- Question is sent to LLM (GPT-4) with context: Drift event details (resource type, config diff), Blueprint Intent for this resource, Warden Policy rules, AWS best practices knowledge
- LLM generates natural language response explaining: What changed in plain English, Why it might have changed (common scenarios), Impact of keeping vs reverting the change, Security/compliance implications
- Example conversation:
  - **Developer**: "Why was this security group rule added?"
  - **AIE**: "Someone manually added ingress rule allowing TCP port 22 (SSH) from 0.0.0.0/0 (public internet) to this security group at 2:34 PM today. This is likely a developer troubleshooting access issues. However, this violates the 'No Public SSH' Warden Policy because it exposes SSH to brute-force attacks. Recommended: Revert this and use AWS Systems Manager Session Manager for secure instance access instead."
- Conversation history is preserved in DriftEvents table for audit trail
- Developer can ask follow-up questions in same thread

### Drift Reversion

**What**: One-click action that restores AWS resource configuration to blueprint state using AWS APIs

**Why**: Quickly undo unauthorized changes without manual AWS console work

**Behavior**:
- Developer clicks "Revert" button on drift event
- System shows confirmation modal:
  - "Revert [Resource Name] to blueprint state?"
  - Shows config diff being reverted
  - Warning if resource has dependent resources (e.g., "Reverting this security group will affect 3 EC2 instances")
  - Checkbox: "I understand this will immediately change live infrastructure"
- On confirmation, Reversion Lambda executes:
  - Loads blueprint expected configuration from S3
  - Calls appropriate AWS API to restore configuration (e.g., `modify_db_instance`, `authorize_security_group_ingress`, `put_bucket_encryption`)
  - Waits for AWS to confirm change applied (with 2-minute timeout)
  - Updates DriftEvent status to `reverted` with timestamp and developer ID
- Developer sees success notification: "✅ Reverted [Resource Name] to blueprint state at [timestamp]"
- Reversion is logged to Decision Ledger with reasoning: "Unauthorized change detected and reverted by [developer_email]"
- If reversion fails (AWS API error), developer sees error message with details and drift event status set to `revert_failed`

### Drift Assimilation

**What**: Action that updates Blueprint Intent to match live AWS state, incorporating manual changes into the source of truth

**Why**: Emergency hotfixes made in AWS console need to be preserved in blueprint, otherwise next deployment will overwrite them

**Behavior**:
- Developer clicks "Assimilate" button on drift event
- System shows assimilation form:
  - "Update blueprint to match live state?"
  - Shows config diff being assimilated
  - Required text input: "Why are you making this change?" (min 20 characters)
  - Dropdown: "Change category" (Emergency Fix | Intentional Improvement | AWS Auto-Update | Other)
  - Warning: "This will update your blueprint. Future environments will include this change."
- On submission, Assimilation Lambda executes:
  - Loads current Blueprint Intent from S3
  - Updates blueprint JSON to reflect live AWS state for this resource
  - Increments blueprint version (e.g., v1.2 → v1.3)
  - Re-generates topology diagram with updated resource configuration
  - Stores updated blueprint to S3 with version tag
  - Updates DriftEvent status to `assimilated` with rationale and timestamp
- Developer sees success notification: "✅ Blueprint updated to v1.3. [Resource Name] changes are now part of your approved architecture."
- Assimilation is logged to Decision Ledger with full rationale
- If assimilated change violates Warden Policy, system shows warning: "⚠️ This change violates [Policy Name]. You can assimilate it for Dev/Staging, but it will be upgraded when promoted to Prod."

### Drift Acknowledgement

**What**: Dismissal action for informational drift that requires no reconciliation

**Why**: Not all drift is problematic (AWS auto-updates, known external changes)—acknowledgement removes noise from pending queue

**Behavior**:
- Developer clicks "Acknowledge" button on drift event
- System shows confirmation: "Mark this drift as acknowledged? It will remain in the timeline but won't show in pending drift count."
- Optional text input: "Note" (reason for acknowledgement)
- On confirmation, DriftEvent status updated to `acknowledged` with timestamp
- Acknowledged drift no longer appears in default timeline view (only shows if "Show Acknowledged" filter enabled)
- Acknowledgement logged to Decision Ledger

### Decision Ledger Audit Log

**What**: Immutable, searchable log of all drift events and reconciliation decisions

**Why**: Compliance audits, incident post-mortems, and debugging require full history of infrastructure changes

**Behavior**:
- DynamoDB table `DecisionLedger` stores entries with schema:
  ```
  {
    ledger_id: UUID,
    environment_id: string,
    timestamp: ISO8601,
    event_type: 'drift_detected' | 'drift_reverted' | 'drift_assimilated' | 'drift_acknowledged',
    resource_id: string,
    resource_type: string,
    actor: 'sentinel' | developer_email,
    details: {
      drift_type: string,
      severity: string,
      expected_state: JSON,
      actual_state: JSON,
      policy_violations: [string],
      rationale: string (for assimilations),
      conversation_history: [messages]
    }
  }
  ```
- Ledger is append-only (no updates or deletes)
- Accessed via `/environments/{env_id}/audit-log` route
- UI provides:
  - Searchable table view with columns: Timestamp | Resource | Event Type | Actor | Details
  - Export button: Download as JSON or CSV
  - Filter by: Date range, resource type, event type, actor
  - Detail modal: Click row to see full JSON including conversation history
- Ledger retention: Infinite for Prod environments, 90 days for Dev/Staging (configurable)

---

## Data Model (Initial Thoughts)

**Entities This Feature Needs**:

- **DriftEvent**: Represents a single detected drift instance
  - Attributes: `drift_id`, `environment_id`, `resource_id`, `resource_type`, `drift_type` (enum: ADDED/DELETED/MODIFIED/POLICY_VIOLATION), `severity` (enum: CRITICAL/HIGH/MEDIUM/LOW), `detected_at` (timestamp), `expected_state` (JSON), `actual_state` (JSON), `policy_violations` (array of policy IDs), `status` (enum: pending/reverted/assimilated/acknowledged), `conversation_thread` (array of messages), `resolved_at` (timestamp), `resolved_by` (developer_id)

- **DecisionLedger**: Immutable audit log entry
  - Attributes: `ledger_id`, `environment_id`, `timestamp`, `event_type`, `resource_id`, `resource_type`, `actor`, `details` (JSON blob with drift context and rationale)

- **WardenPolicy**: Organizational rules that infrastructure must comply with
  - Attributes: `policy_id`, `policy_name`, `description`, `resource_types` (applies to which AWS resources), `rule_schema` (JSON schema defining compliance check), `severity` (CRITICAL/HIGH)

- **BlueprintIntent**: (existing entity) Approved architecture diagram
  - New field needed: `version_history` (array of blueprint versions with timestamps)

**Key Relationships**:
- DriftEvent `belongs_to` Environment (one environment has many drift events)
- DriftEvent `references` BlueprintIntent (expected state comes from blueprint)
- DriftEvent `references` WardenPolicy (if policy violation detected)
- DecisionLedger `contains` DriftEvent details (audit log of all drift lifecycle)

**Data Flow**:
1. Sentinel Lambda queries AWS APIs → generates DriftEvent record
2. DriftEvent triggers notification to developer
3. Developer interacts with DriftEvent (ask questions, revert, assimilate, acknowledge)
4. Each interaction appends to `conversation_thread` in DriftEvent
5. Reconciliation action updates DriftEvent `status` and `resolved_at`
6. All state transitions append to DecisionLedger (immutable log)

---

## User Experience Flow

### Drift Detection Happy Path

1. Developer has approved Blueprint Intent for their Prod environment (3-tier web app: ALB + EC2 + RDS)
2. DevOps engineer manually changes RDS instance from `db.t3.medium` to `db.t3.large` in AWS console (emergency performance fix during incident)
3. Within 5 minutes, Sentinel Lambda runs its polling cycle
4. Sentinel queries RDS API, detects instance type mismatch: Blueprint expects `db.t3.medium`, AWS shows `db.t3.large`
5. Sentinel creates DriftEvent record with severity=HIGH (resource modified, database config change)
6. Notification Lambda sends in-app alert to developer: "🔴 Drift detected: RDS instance prod-api-db configuration changed"
7. Developer opens Reconciliation Hub, sees new drift event at top of timeline
8. Developer clicks drift event card to expand details
9. Developer sees diff:
   - **Expected**: `InstanceType: db.t3.medium`
   - **Actual**: `InstanceType: db.t3.large`
10. Developer types question: "Why would someone change this?"
11. LLM responds: "This RDS instance type was upgraded from t3.medium to t3.large, which doubles CPU and memory. This is commonly done during performance incidents to handle increased load. Cost will increase by approximately $50/month. The change is valid and doesn't violate any Warden Policies."
12. Developer decides this was an intentional emergency fix that should be kept
13. Developer clicks "Assimilate" button
14. System prompts for rationale: Developer types "Emergency performance fix during Black Friday traffic spike. Keep this instance size."
15. System updates Blueprint Intent to v1.4 with `db.t3.large` as new expected state
16. Developer sees confirmation: "✅ Blueprint updated. Future deployments will use db.t3.large."
17. DriftEvent status updated to `assimilated`
18. DecisionLedger records: "Developer assimilated RDS instance type drift due to emergency performance fix"

### Drift Reversion Path

1. Junior developer manually adds security group rule in AWS console allowing 0.0.0.0/0 SSH access (debugging network issue)
2. Sentinel detects security group modification within 5 minutes
3. Drift flagged as CRITICAL severity (violates "No Public SSH" Warden Policy)
4. Developer receives notification: "🔴 CRITICAL: Security policy violation in prod-api-sg"
5. Developer opens drift event, sees policy violation warning
6. Developer asks: "Is this a security risk?"
7. LLM explains: "Yes. This security group rule allows SSH access from any IP address on the internet. This exposes the instance to brute-force attacks and is a common entry point for breaches. This violates organizational policy. Recommended action: Revert immediately and use AWS Systems Manager Session Manager for secure instance access."
8. Developer clicks "Revert" button
9. System confirms: "Revert security group to blueprint state? This will remove the 0.0.0.0/0 SSH rule."
10. Developer confirms
11. System calls AWS API to revoke security group ingress rule
12. Within 30 seconds, rule is removed from AWS
13. Developer sees: "✅ Reverted prod-api-sg to blueprint state. Unauthorized SSH rule removed."
14. DriftEvent status updated to `reverted`
15. DecisionLedger logs reversion with rationale: "Security policy violation reverted"

---

## Edge Cases & Constraints

- **Drift detected during active deployment**: If Sentinel detects drift while a deployment is in progress (blueprint is being applied), system marks drift as `deployment_in_progress` and re-checks after 10 minutes. Avoids false positives during legitimate changes.

- **Cascading drift (dependent resources)**: If developer reverts a security group change, and that security group is attached to 5 EC2 instances, system shows warning: "This will affect 5 resources: [list instances]". Reversion proceeds if confirmed, but creates separate DriftEvent records for each affected resource to track full impact.

- **AWS API throttling**: If Sentinel makes too many AWS API calls and hits rate limits, it implements exponential backoff and extends polling interval temporarily. Developer sees notification: "Drift detection delayed due to AWS rate limiting. Next check in 15 minutes."

- **Conflicting assimilations**: If two developers attempt to assimilate different drift on the same resource simultaneously, system uses optimistic locking. Second developer sees error: "Blueprint was updated by [other_developer] 30 seconds ago. Please refresh and try again."

- **Drift on deleted environments**: If an environment is deleted but Sentinel still has DriftEvents in pending status, system auto-archives those events to `archived` status and stops polling.

- **Very large diffs**: If a resource has >100 configuration attributes changed (e.g., large IAM policy), diff view truncates display and provides "Download full diff" button to avoid UI performance issues.

- **Non-reversible drift**: Some AWS changes cannot be reverted via API (e.g., RDS engine version upgrade). System detects this and disables "Revert" button, shows message: "This change cannot be automatically reverted. You must manually restore from backup or assimilate this drift."

- **Warden Policy violations in assimilated drift**: If developer assimilates drift that violates policy (e.g., keeps an unencrypted RDS in Dev), system allows it but tags blueprint with `policy_exception` flag. When promoting Dev to Prod, API Warden will still enforce policy and upgrade the resource (developer sees: "Note: RDS encryption was disabled in Dev but will be enforced in Prod per Warden Policy").

---

## Out of Scope for This Feature

This feature explicitly does NOT include:

- **Automatic drift reversion without developer approval**: No auto-remediation in V1. All reconciliation actions require explicit developer decision.
- **Predictive drift prevention**: No ML model that predicts drift before it happens or blocks manual console changes.
- **Drift simulation/dry-run**: No "what-if" analysis showing impact of reverting drift before execution.
- **Cross-environment drift comparison**: No feature to compare drift across Dev vs Prod environments simultaneously.
- **Custom drift detection rules**: Developer cannot define custom drift detection logic (e.g., "ignore tag changes"). All drift is detected; developer can acknowledge to dismiss.
- **Integration with external monitoring tools**: No webhooks to Datadog, PagerDuty, or other observability platforms (just in-app, email, Slack).
- **Drift remediation playbooks**: No automated runbooks that execute multi-step fixes for complex drift scenarios.
- **Historical drift replay**: No ability to "time travel" and see what infrastructure looked like at a specific point in past.
- **Drift on non-AWS resources**: Only AWS resources monitored; no support for external SaaS APIs or on-premise infrastructure.

---

## Open Questions for Tech Lead

- **Polling interval trade-off**: 5-minute polling provides MTTD < 5 min but may hit AWS API rate limits for large environments (100+ resources). Should we implement adaptive polling (faster for small envs, slower for large) or make interval configurable per environment?

- **Conversation context retention**: Should conversation threads in drift events persist indefinitely or expire after X days? If developer asks a question about a 6-month-old drift event, should the LLM still have full context?

- **Reversion safety checks**: What pre-flight validations should run before allowing revert? (e.g., check if resource has active connections, verify no dependent resources will break, confirm resource isn't in "deleting" state in AWS)

- **Assimilation approval workflow**: Should assimilations to Prod environments require an additional approval step (e.g., senior developer or DevOps engineer must confirm), or is developer rationale + audit log sufficient?

- **Sentinel failure handling**: If Sentinel Lambda crashes or fails to run for >15 minutes, should system alert developers with "Drift detection may be delayed" warning, or silently retry?

---

## Dependencies

**Depends On**: 
- Feature 1 (Conversational Architecture Discovery) must be complete because Blueprint Intent is the source of truth for expected state
- Warden Policy definition must exist (may be hardcoded standard policies for V1 if Feature 2 hasn't implemented policy engine yet)
- AWS IAM credentials with read permissions for all monitored resource types (EC2, RDS, S3, SecurityGroups, IAM, etc.)

**Enables**: 
- Feature 2 (Verified Environment Promotion) benefits from drift detection because it can validate that Dev environment has no unacknowledged drift before allowing promotion to Prod
- Future V2 feature: Automated compliance reporting (Decision Ledger provides audit trail for SOC2, ISO27001 compliance)