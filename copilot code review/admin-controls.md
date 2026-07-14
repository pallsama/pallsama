Design: Admin Controls for Cloud-Connected Code Review on GHES
---
# Design: Admin Controls for Cloud-Connected Code Review on GHES

Parent initiative: #18506
Related designs: #18507 (Architecture), #18509 (Identity), #18512 (Entitlement)

## Context

Cloud-connected code review on GHES sends code context to GitHub.com for AI-powered review. GHES admins need granular controls over enablement, data flow, auditing, behavior, and emergency shutdown — consistent with the broader AI controls framework for GHES.

## Controls Framework

Admin controls are organized into 5 categories:
1. **Agent enablement & policy** — instance/org/repo-level toggles, who can invoke
2. **Cloud connection governance** — data flow transparency, context tiers, egress controls
3. **Visibility & auditing** — logs, dashboards, audit integration
4. **Behavior guardrails** — allowed actions, file/path restrictions, fork policies
5. **Kill switch** — instance-wide, per-org emergency stop

## Code Review-Specific Controls

### 1. Agent Enablement & Policy

```yaml
copilot_code_review:
  # Instance-level
  enabled: true                           # Master toggle (site admin)
  
  # Org-level (org admins can restrict further, not expand)
  org_policy:
    default: enabled                      # enabled | disabled | admin_only
    allow_org_override: true              # Can org admins change their setting?
  
  # Repo-level
  repo_policy:
    default: inherit_from_org             # inherit | enabled | disabled
    allow_repo_override: true             # Can repo admins change their setting?
  
  # Who can invoke
  allowed_actors:
    - org_members                         # Default: any org member with repo access
    # Options: org_members, org_owners_only, specific_teams, copilot_seat_holders
    require_copilot_seat: true            # Must have Copilot seat (see #18512)
```

### 2. Cloud Connection Governance (Context Tiers)

```yaml
  # Data flow controls (aligned with context tiers from #18507)
  context_tier: 1                         # 0=diff-only, 1=diff+files, 2=full, 3=agentic
  max_context_tier_allowed: 2             # Instance-level ceiling (orgs cannot exceed)
  
  # Payload controls
  max_payload_size_bytes: 5242880         # 5MB max total
  max_file_size_bytes: 524288             # 512KB per file
  excluded_repos: ["**/secrets-*"]        # Never send to cloud
  excluded_file_patterns: ["*.env", "*.key", "*.pem", "*.p12", "*.pfx"]
  excluded_paths: ["config/credentials/", ".github/workflows/"]
  
  # Redaction
  redact_secrets: true                    # Secret scanning before transmission
  redact_string_literals: false           # Replace literals with placeholders
  custom_redaction_patterns: []           # Enterprise-specific regex patterns
  
  # Egress transparency
  show_data_flow_banner: true             # Show users what tier is active
  require_user_consent_per_review: false  # User must confirm before each review
```

### 3. Auditing & Visibility

```yaml
  # Audit logging
  audit_log:
    log_review_requests: true             # Log every review trigger (who, repo, PR, tier)
    log_payload_metadata: true            # Log file list + sizes sent to cloud
    log_results_received: true            # Log comment count + type returned
    log_entitlement_failures: true        # Log rejected requests
    
  # Admin visibility
  admin_dashboard:
    show_review_volume: true              # Reviews/hour, /day by org/repo
    show_data_egress: true                # Bytes sent to cloud
    show_seat_utilization: true           # Active users vs. assigned seats
    show_error_rates: true                # Failed reviews, timeouts, rejections
```

### 4. Behavior Guardrails

```yaml
  # Review behavior
  behavior:
    review_action: comment_only           # comment_only | changes_requested | may_approve
    auto_review_on_pr_create: false       # Auto-trigger on new PRs
    auto_review_on_push: false            # Auto-trigger on new commits
    require_explicit_request: true        # Only review when assigned or @mentioned
    
  # Fork/external PR policy
  fork_policy:
    allow_fork_prs: false                 # Review PRs from forks?
    fork_max_tier: 0                      # Max context tier for fork PRs
    require_maintainer_trigger: true      # Maintainer must trigger for fork PRs
    
  # Tier 3 (agentic) controls
  agentic:
    enabled: false                        # Disabled by default
    file_allowlist: ["*.go", "*.ts"]      # Only serve these extensions
    max_files_per_review: 20              # Max additional files cloud can request
    require_human_approval: true          # Admin/maintainer approves each file request
    
  # Rate limiting
  rate_limits:
    max_reviews_per_hour: 100             # Instance-wide
    max_reviews_per_org_per_hour: 50      # Per org
    max_reviews_per_repo_per_hour: 20     # Per repo
    max_reviews_per_user_per_hour: 10     # Per user
```

### 5. Kill Switch

```yaml
  # Emergency controls
  kill_switch:
    instance_wide: false                  # Immediately halt all reviews
    per_org: {}                           # { "org-name": true } to halt specific orgs
    # Also available via:
    # - Site admin UI (one-click)
    # - Management console API
    # - ghe-config CLI: ghe-config app.copilot-code-review.enabled false
```

## Admin UI Mockup

```
┌─────────────────────────────────────────────────────────────┐
│  Site Admin > Copilot > Code Review                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ☑ Enable Copilot Code Review          [Kill Switch 🔴]     │
│                                                             │
│  ─── Data & Privacy ───────────────────────────────────     │
│  Context level: [▼ Tier 1: Diff + Files (recommended)]      │
│  Max context tier orgs can use: [▼ Tier 2: Full Context]    │
│  ☑ Redact detected secrets before sending                   │
│  ☐ Require user consent per review                          │
│                                                             │
│  ─── Who Can Use ──────────────────────────────────────     │
│  ☑ Require Copilot seat                                     │
│  Allowed actors: [▼ Any org member with repo access]        │
│                                                             │
│  ─── Review Behavior ──────────────────────────────────     │
│  Review action: [▼ Comment only (non-blocking)]             │
│  ☐ Auto-review on PR create                                 │
│  ☑ Require explicit assignment or @mention                  │
│  ☐ Allow reviews on fork PRs                                │
│                                                             │
│  ─── Rate Limits ──────────────────────────────────────     │
│  Reviews per hour (instance): [100]                         │
│  Reviews per user per hour:   [10]                          │
│                                                             │
│  ─── Exclusions ───────────────────────────────────────     │
│  Excluded repos: [internal/secrets-*          ] [+ Add]     │
│  Excluded files: [*.env, *.key, *.pem         ] [+ Add]     │
│                                                             │
│  [Save changes]                                             │
└─────────────────────────────────────────────────────────────┘
```

## Open Questions

1. Should org admins be able to **increase** the context tier beyond the instance default, or only restrict?
2. Should per-review consent (Tier 3) be a blocking dialog or a PR comment confirmation?
3. How do these controls interact with CODEOWNERS auto-assignment?
4. Should the kill switch also cancel in-flight reviews on the cloud side?
5. What audit log event schema aligns with existing GHES audit log format?

