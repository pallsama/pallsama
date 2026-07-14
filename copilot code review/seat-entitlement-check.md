Design: Copilot Seat Entitlement Check for GHES Code Review
---
# Design: Copilot Seat Entitlement Check for GHES Code Review

## Problem

When a user triggers a code review on GHES, we need to verify they have a Copilot seat before extracting context and sending it to the cloud. Today, the GHES license file does **not** include Copilot seat information.

## Decision: Check Both GHES and Cloud

| Check Location | Purpose | What It Prevents |
|---|---|---|
| **GHES (fast, local)** | Gate before context extraction | Wasting appliance CPU/IO (git diff, file reads, symbol parsing, redaction) for unlicensed users |
| **Cloud (authoritative)** | Definitive entitlement validation | Stale GHES cache, revoked seats, billing lapses, enterprise plan changes |

### Why both?

- **GHES-only** is insufficient because seat data is a cached copy that can drift from billing reality
- **Cloud-only** is wasteful because unlicensed requests still trigger expensive local context extraction and network transmission before rejection

---

## Flow

```
┌─────────────────────────────────────────────────┐
│  GHES (fast local gate)                         │
│                                                 │
│  1. User triggers review                        │
│  2. Proxy checks: does user have local Copilot  │
│     seat assignment?                            │
│     - YES → proceed with context extraction     │
│     - NO  → reject immediately (no network)    │
│                                                 │
│  Seat data source: synced from cloud via        │
│  Connect (periodic pull, e.g., every 1h)        │
└─────────────────────────────────────────────────┘
                    │
                    ▼ (only if local check passes)
┌─────────────────────────────────────────────────┐
│  Cloud (authoritative check)                    │
│                                                 │
│  3. Gateway receives request with user identity │
│  4. Validates:                                  │
│     - Enterprise has active Copilot plan        │
│     - User's seat is currently valid            │
│     - Billing is current                        │
│     - Usage quota not exceeded                  │
│     - NO → return 403 (proxy shows error on PR)│
│     - YES → route to CCRA pipeline             │
└─────────────────────────────────────────────────┘
```

---

## Seat Sync Mechanism

### Option A: Extend Connect License Sync (Recommended for GA)

GitHub Connect already syncs license data (`license_usage_sync` feature). Extend this to include Copilot seat assignments.

- Cloud pushes seat roster to GHES on existing sync schedule
- GHES stores locally: `{user_id, seat_type, assigned_at, last_verified_at}`
- Refresh interval: hourly (configurable)
- Aligns with existing Connect patterns; transparent to admins

### Option B: On-Demand Check with Cache (Fallback / Bootstrap)

- First request from an unrecognized user → proxy calls cloud to verify seat
- Cache result locally for N hours (e.g., 4h TTL)
- Handles: new users, just-assigned seats, sync hasn't run yet
- Con: First request per user has extra latency

### Recommendation

**Option A** for steady state, **Option B** as fallback when sync hasn't run or for newly-assigned users.

---

## GHES Local Seat Store

```sql
-- New table on GHES (conceptual)
CREATE TABLE copilot_seats (
  ghes_user_id     INTEGER PRIMARY KEY,
  dotcom_user_id   INTEGER,          -- linked GitHub.com account (if available)
  seat_type        TEXT NOT NULL,     -- 'enterprise', 'business', 'individual'
  features         TEXT[],            -- ['completions', 'chat', 'code_review']
  assigned_at      TIMESTAMP,
  last_verified_at TIMESTAMP,
  expires_at       TIMESTAMP          -- TTL for cache invalidation
);
```

### Seat check logic (pseudocode):

```ruby
def user_entitled_to_code_review?(user)
  seat = CopilotSeat.find_by(ghes_user_id: user.id)
  
  return false if seat.nil?
  return false if seat.expires_at < Time.now  # stale cache
  return false unless seat.features.include?('code_review')
  
  true
end
```

---

## User Identity Mapping (GHES → Cloud)

The cloud needs to identify which GHES user is requesting review for seat validation.

| Approach | Pro | Con |
|----------|-----|-----|
| **Linked GitHub.com user ID** | Direct seat lookup; already exists for Connect users | Requires account linking (unified contributions/search) |
| **EMU (Enterprise Managed User) ID** | Already mapped 1:1 | Only works for EMU enterprises |
| **Hashed GHES email** | Privacy-preserving | Cloud needs email→seat mapping; collision risk |
| **GHES server_id + local user_id** | No PII leaks | Cloud needs per-instance seat table |

**Recommendation**: Use **linked GitHub.com user ID** (already required for several Connect features). For enterprises without account linking, fall back to **hashed email** with a mapping table maintained during seat sync.

---

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Seat just revoked, GHES cache stale | GHES allows, cloud rejects with 403 → proxy posts error on PR |
| Seat just assigned, not synced yet | Option B fallback: on-demand cloud check, cache result |
| GHES offline / cloud unreachable | Allow if local cache is fresh (< TTL); reject if expired |
| User has Copilot but not code_review feature | Check `features` array; reject if code_review not included |
| Auto-review on PR create | Check seat of PR author (not a bot/system trigger) |
| @mention trigger by user without seat | Reject; post comment explaining user needs Copilot seat |
| Fork PR from external contributor | Check contributor's seat; if none, reject or require maintainer trigger |
| Bulk sync fails | Keep serving from last successful sync; alert admin after N missed syncs |

---

## Failure UX

When a user without a seat triggers a review:

```
❌ copilot-code-review[bot] commented:

  Unable to perform code review: @username does not have a Copilot code review
  seat assigned. Contact your enterprise admin to request access.
  
  [Learn more about Copilot for GHES](https://docs.github.com/...)
```

When cloud rejects (stale cache):

```
❌ copilot-code-review[bot] commented:

  Unable to perform code review: entitlement validation failed.
  This may be a temporary issue — please try again in a few minutes.
  If the problem persists, contact your enterprise admin.
```

---

## Open Questions

1. **Feature granularity**: Should seats be per-feature (`code_review`, `completions`, `chat`) or bundle-level (`copilot_enterprise` = all features)?
2. **Seat count enforcement**: Should GHES enforce max seat count locally, or only cloud?
3. **Grace period**: When a seat is revoked, allow a grace period (e.g., 24h) before local rejection?
4. **Offline mode**: If GHES can't reach cloud for extended time, should it continue serving from cache indefinitely or hard-fail after N days?
5. **Org-level vs enterprise-level**: Can individual orgs on GHES have different Copilot plans?

