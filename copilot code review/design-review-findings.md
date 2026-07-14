Design Review Findings: Cloud Code Review for GHES
---
# Design Review Findings: Cloud-Connected Code Review for GHES

This issue tracks unresolved design flaws, security gaps, and missing edge cases identified during review of the GHES cloud code review design.

---

## 🚨 Blocking Issues (Must Resolve Before Implementation)

### 1. GitHub Connect identity vs. local reviewer identity are conflated
The design uses the existing GitHub Connect App (JWT/installation token) for GHES→cloud auth, and a separate local GHES App (`copilot-code-review[bot]`) for posting reviews. The trust boundary and token flow between these two principals is never defined.

**To resolve:** Explicitly separate Connect identity (authenticates GHES to cloud), local reviewer app identity (posts reviews on GHES), and define the correlation/authorization model between them.

---

### 2. Reviewer UX assumptions are unvalidated on GHES
The design assumes a GitHub App bot can:
- Appear in suggested reviewers
- Be assigned as a reviewer via UI
- Be referenced in CODEOWNERS
- Satisfy branch protection required reviews

These capabilities may not be available for App bots on GHES without platform changes.

**To resolve:** Validate each UX primitive against current GHES App/bot behavior. Document required platform changes if any.

---

### 3. Fork/untrusted PR behavior is completely missing
A malicious fork contributor could craft a small diff that causes Tier 1+ to send sensitive base-file content to the cloud. No policies exist for external PRs.

**To resolve:** Add explicit fork policies:
- Tier 0 only for untrusted fork PRs (default)
- Maintainer-triggered review for external PRs
- Never serve Tier 3 files for untrusted actors
- Document risk of base-file context exposure

---

### 4. Per-actor authorization is absent
No check that the actor requesting review is allowed to send that repo's code to the cloud. External contributors, limited-access users, or bots triggering via @mention are uncontrolled.

**To resolve:** GHES proxy must authorize locally: actor has repo access, actor is allowed by org policy to invoke cloud review, blocked users/bots cannot trigger via comments.

---

### 5. Result delivery model is internally contradictory
Architecture diagram shows `CallbackSvc → Proxy` (cloud pushes), text recommends SSE (GHES pulls), and "Connect reverse channel" is mentioned without definition.

**To resolve:** Pick one canonical transport. Recommend: GHES-initiated long-poll/SSE with defined reconnect, ACK, retry, dedup, and offline-GHES behavior.

---

### 6. Stale head SHA — reviews can land on wrong lines
Reviews are generated asynchronously but no verification ensures the PR head SHA still matches before applying comments. Force-pushes during review can cause comments on wrong lines.

**To resolve:** Include base/head SHA in all request/result payloads. GHES proxy must verify SHA match before applying. Discard stale results or mark them appropriately.

---

### 7. "Only diffs leave GHES" claim contradicts Tier 1 default
The security section, summary, and architecture overview repeatedly claim "only diff hunks transit," but the recommended default (Tier 1) sends full file contents of all changed files.

**To resolve:** Update all claims to accurately reflect Tier 1 behavior: "Full contents of changed files leave GHES by default." Make the privacy implications clear to admins.

---

### 8. Data retention across all hops is undefined
The design defines 24h TTL for the result store, but says nothing about retention of: request payloads in transit, CAPI prompts/responses, model telemetry, Hydro messages, dead-letter queues, worker logs, or traces.

**To resolve:** Define retention for every system that touches source code. Document: region residency, subprocessors, training/non-training guarantees, deletion semantics.

---

## ⚠️ Important Issues (Should Resolve Before GA)

### 9. CCRA changes are underestimated
The design claims "minimal" CCRA changes. In reality, CCRA assumes GitHub.com repo IDs, PR API models, feature flag lookups, callback mechanisms, comment path mapping, Snippy integration, abuse/rate-limit attribution, and telemetry partitioning.

**To resolve:** Conduct a compatibility audit of all CCRA assumptions that break for GHES-originated reviews. Size the actual work.

---

### 10. Bot approval can bypass human review
The design allows the agent to "Approve" PRs and be a required reviewer in branch protection. This could accidentally satisfy merge requirements without human review.

**To resolve:** Default to comment-only. If approval is supported, make it opt-in, separate from human-review requirements, and clearly auditable.

---

### 11. Deduplication and idempotency are missing
PR opened, synchronized, review-requested, CODEOWNERS auto-request, @mention, and auto-review can all trigger overlapping reviews for the same PR/SHA.

**To resolve:** Add deduplication keyed by `(repo_id, pr_number, head_sha, tier, trigger_type)` with cancellation/supersession semantics.

---

### 12. Failure UX is not designed
No design for: cloud timeout, entitlement failure, payload rejected, redaction failure, SSE disconnect, CAPI error, or comment-application failure.

**To resolve:** Post a check run or PR timeline event with failure state and retry instructions. Define timeout thresholds and user-facing error messages.

---

### 13. Secret redaction is presented as a guarantee
The design implies `redact_secrets: true` prevents credential exposure. Secret scanning is pattern-based and probabilistic — it cannot catch custom tokens, proprietary secrets, or non-standard formats.

**To resolve:** Reframe as best-effort. Document residual risk. Add: custom patterns, entropy scanning, deny-by-default path rules, payload preview for admins.

---

### 14. Snippy is both included and skipped
Tier 2 example payload includes `"snippy_enabled": true`, while the CCRA modifications table says Snippy "may skip for GHES."

**To resolve:** Decide whether Snippy (code clone/license detection) is supported for GHES. If not, remove from Tier 2 parity claims.

---

### 15. Skipped/excluded files are invisible to CCRA
If the proxy strips files due to size/pattern/redaction rules, CCRA doesn't know and may produce invalid or incomplete comments.

**To resolve:** Include skipped-file metadata in the payload. Surface skipped files in the review summary so users understand limitations.

---

### 16. CODEOWNERS support is both claimed and listed as open question
Doc 2 says `@copilot-code-review` works in CODEOWNERS, then later lists it as an open question.

**To resolve:** Remove the claim until validated on GHES. Track as future work.

---

## 🧩 Missing Edge Cases (Track for Implementation)

- [ ] Force-push during in-flight review
- [ ] Closed/reopened PRs triggering stale reviews
- [ ] Draft PRs and review-request semantics
- [ ] Renamed/deleted files, binary files, LFS, submodules, generated/vendor files
- [ ] Very large diffs exceeding payload limits (partial review behavior)
- [ ] Merge conflicts or missing merge base
- [ ] GHES offline for extended period while results accumulate
- [ ] Cloud outage after request submitted (timeout behavior)
- [ ] GHES version skew / feature availability mismatch
- [ ] Multiple GHES HA nodes sharing one server ID
- [ ] Org/repo granular opt-in vs "installed on all repos"
- [ ] Review dismissal permissions and audit history
- [ ] Interaction with required status checks vs PR reviews
- [ ] Tier 3 path traversal, symlinks, submodule access attempts
- [ ] Compression and caching for Tier 1 full-file payloads at scale

---

## 📊 Scalability Concerns

- **Tier 1 payload size**: Full HEAD+BASE contents for large PRs may exceed 5MB limit frequently in monorepos
- **GHES appliance resources**: Git diff/show + secret scanning + symbol parsing competes with core GHES workloads — need worker pools, CPU/memory limits, backpressure
- **Fixed rate limits**: `max_reviews_per_hour: 100` is arbitrary — should be adaptive based on seat count, appliance capacity, queue depth
- **SSE connection management**: Enterprise proxies/load-balancers may have idle timeouts that break persistent connections — need heartbeats, reconnect, jitter
- **Auto-install on all repos**: Could trigger massive review volume if auto-review is also enabled

---

## ✅ What Context Tiers Successfully Addressed

- Privacy/quality tradeoff is now explicit and admin-configurable
- Acknowledges dotcom needs full files + symbols + metadata (not just diffs)
- Admin controls for file patterns, size limits, secret redaction, audit logging
- Tier 3 has consent requirement, allowlist, per-request audit logging
- Tier 0 provides a genuine "minimum data exposure" option for sensitive enterprises

