Decision: Connect-Based vs Runner-Based Code Review for GHES
---
# Decision: Connect-Based vs Runner-Based Code Review for GHES

Parent initiative: #18506
Related: #18507 (Connect design), #18529 (Runner design)

## Two Approaches

| | **Connect-Based** (#18507) | **Runner-Based** (#18529) |
|---|---|---|
| **How it works** | GHES proxy sends context to cloud CCRA via GitHub Connect | Actions workflow runs code review CLI on customer's self-hosted runner, calls CAPI directly |
| **Who controls execution** | GitHub (cloud CCRA worker pool) | Customer (their runner infrastructure) |
| **Where code goes** | Context leaves GHES → cloud (tier-controlled) | Code stays on runner; only LLM prompts reach CAPI |

---

## Connect-Based: Pros & Cons

### Pros
- **Uniform experience** — GitHub controls CCRA infrastructure; every customer gets same latency, quality, reliability
- **Platform-native UX** — `copilot-code-review[bot]` identity, suggested reviewers, re-request button, branch protection integration
- **Centralized admin controls** — Management Console UI, not workflow YAML scattered across repos
- **No customer infra required** — no runners, no Actions, just enable Connect
- **Enterprise seat enforcement built-in** — no PAT management, no secret rotation
- **Telemetry & billing integrated** — usage metering, compliance, audit logs are first-class
- **Quality improvements ship automatically** — model upgrades, prompt tuning, detection improvements require zero customer action
- **Only GitHub can build this** — deep platform integration that customers cannot replicate

### Cons
- **Code leaves customer network** — even with tiers, full file content (Tier 1 default) transits to cloud
- **Complex to build** — new proxy on GHES, Connect feature, cloud gateway, result store (~5-6 months)
- **Cross-team dependencies** — Connect team, GHES platform team, CAPI team, security review
- **Requires GitHub Connect** — some GHES customers refuse to enable Connect
- **New infrastructure on both sides** — ongoing maintenance burden
- **Privacy/compliance concerns** — some enterprises will never send code to cloud regardless of controls

---

## Runner-Based: Pros & Cons

### Pros
- **Privacy by architecture** — source code never leaves customer network (same boundary as Copilot completions in editors)
- **Fast to build** — ~8-10 weeks for 2 engineers (CLI + Action wrapper)
- **No new GHES infrastructure** — uses existing Actions + runners
- **No new cloud infrastructure** — uses existing CAPI
- **Agentic/full-context is trivial** — runner has full repo access, no reverse channel needed
- **Customer-buildable today** — validates demand; customers can prototype with CAPI tokens now
- **Works without GitHub Connect** — only needs runner outbound to CAPI endpoint
- **Customers control their data** — no trust delegation to GitHub cloud for code storage

### Cons
- **Non-uniform experience** — review latency, reliability, and availability depend on customer runner capacity, specs, and queue depth. No SLA GitHub can provide.
- **Customer manages infrastructure** — runner setup, capacity planning, network rules, maintenance
- **Auth complexity falls on customer** — PAT with Copilot seat stored as secret, rotation, seat management
- **Less native UX** — requires hybrid approach (App identity) for reviewer assignment; without it, reviews come from `github-actions[bot]`
- **Admin controls are distributed** — workflow YAML + org secrets, not centralized Management Console
- **Customers can already build this** — thin moat; value is mainly standardization and maintenance
- **Quality depends on prompt/model access** — without the proprietary autofind detector, review quality is "generic LLM" not "CCRA quality"
- **No enterprise-wide enforcement** — individual repos must adopt the workflow; can't enforce at instance level
- **Fork PRs are difficult** — `GITHUB_TOKEN` is read-only for forks; PAT secrets unavailable to fork workflows
- **Requires Actions enabled** — some GHES customers don't use Actions

---

## Head-to-Head Comparison

| Dimension | Connect-Based | Runner-Based | Winner |
|-----------|--------------|--------------|--------|
| **Experience uniformity** | Same for all customers | Varies by runner infra | Connect |
| **Privacy** | Tier-controlled, code transits cloud | Code stays on-prem | Runner |
| **Time to ship** | 5-6 months (beta) | 8-10 weeks | Runner |
| **UX nativeness** | Fully native (bot identity, admin UI) | Bolted-on (workflow + secrets) | Connect |
| **Customer effort** | Enable Connect toggle | Configure runners + secrets + workflow | Connect |
| **Admin control** | Centralized Management Console | Distributed YAML + secrets | Connect |
| **Maintenance burden (GitHub)** | High (new services both sides) | Low (Action + CLI binary) | Runner |
| **Maintenance burden (customer)** | None | Runner capacity + secret rotation | Connect |
| **Moat / defensibility** | Deep (platform changes) | Thin (standardization only) | Connect |
| **Works without Connect** | No | Yes | Runner |
| **Agentic/deep context** | Complex (reverse channel) | Trivial (local access) | Runner |
| **Enterprise enforcement** | Instance-wide policy | Per-repo adoption | Connect |
| **Billing integration** | Built-in | Customer-managed PAT/seat | Connect |

**Score: Connect wins on 8 dimensions, Runner wins on 4.**

---

## Strategic Assessment

### Why Connect-Based is the primary investment
1. **Uniform, reliable experience** — GitHub can guarantee quality and latency
2. **Only GitHub can build it** — deep platform integration customers cannot replicate
3. **Enterprise-grade controls** — centralized policy, enforcement, compliance
4. **Aligns with GHES product direction** — Connect is the bridge for cloud features on GHES

### Where Runner-Based fits
1. **Customers who refuse Connect** — privacy-first enterprises that won't send code to cloud
2. **Customer-buildable today** — validates demand without GitHub investment
3. **If GitHub builds it**: value is standardization (one official Action vs. N customer hacks) but the experience will never be uniform
4. **Possible "community tier"** — supported Action for customers who want DIY with guardrails

### Recommendation

**Invest in Connect-based as the primary product.** The runner approach is a valid option for privacy-constrained customers but doesn't require significant GitHub engineering investment — customers can build it themselves with CAPI tokens, or GitHub can ship a lightweight official Action later as a secondary offering.

---

---

## Why Not Let Customers Build the Runner Approach Themselves?

While the runner-based approach is technically customer-buildable today (Actions workflow + CAPI token + self-hosted runner), there are reasons GitHub might still standardize it:

**Argument for GitHub building it (standardization):**
- Without an official Action, every customer reinvents the wheel — different prompts, error handling, security posture
- GitHub can ship the proprietary autofind detector and tuned prompts that individual customers cannot access
- One maintained Action removes N customer maintenance burdens

**Arguments against GitHub investing heavily here:**
- **Non-uniform experience** — Review latency, reliability, and availability vary by customer runner capacity and specs. GitHub cannot provide an SLA. One customer gets reviews in 30s, another waits 5 minutes because their runners are saturated.
- **Thin moat** — The core value (standardization, maintenance) is real but not defensible. Any third party could build a similar Action.
- **Customers who care enough will build it regardless** — Privacy-first enterprises willing to manage runners are also willing to write glue code.
- **Dilutes the Connect story** — Offering a runner-based option may reduce urgency for customers to enable Connect, fragmenting the product.

**Bottom line:** The runner approach is more of a community/ecosystem play than a product investment. GitHub's engineering effort is better spent on the Connect-based experience that only GitHub can deliver and that provides a uniform, platform-native experience across all GHES customers.

---

## Future Consideration: CCA (Copilot Coding Agent) on GHES

CCA requires fundamentally different capabilities than CCR:
- **Full repo access** (read + write across entire codebase)
- **Build/test execution** (customer toolchain, private dependencies)
- **Iterative loops** (run tests → read output → fix → re-run)
- **Git operations** (create branches, commit, push)
- **Access to customer network** (private package registries, internal APIs)

### Which CCR foundation makes CCA easier?

| CCA Need | Connect-Based CCR | Runner-Based CCR |
|----------|-------------------|------------------|
| Full repo read/write | ❌ Massive transfer or complex reverse channel | ✅ Already checked out |
| Run builds & tests | ❌ Cloud cannot run customer toolchains | ✅ Runner has build environment |
| Commit & push | ❌ Needs reverse write channel to GHES | ✅ Local git operations |
| Iterative agent loop | ❌ Each iteration round-trips through Connect | ✅ Fast local loop |
| Private dependencies | ❌ Cloud can't reach customer package registries | ✅ Runner is on customer network |

**CCA on GHES essentially requires the runner-based pattern.** The Connect approach cannot support the read-write, iterative, unbounded nature of a coding agent without rebuilding the entire execution model.

### Strategic implication

If GitHub builds CCR on runners first, it establishes infrastructure that CCA directly reuses:
- Auth pattern (CAPI token on runner)
- Actions trigger model (PR/issue events → workflow dispatch)
- Identity model (App posting results back to GHES)
- Customer familiarity and trust with the pattern
- Runner capacity and network policies already configured

If GitHub only builds Connect-based CCR, the runner infrastructure must be built from scratch for CCA anyway — making the Connect CCR investment partially redundant for customers who later adopt CCA.

### Revised recommendation

This shifts the calculus. Rather than viewing runner-based as "thin moat / community tier," it may be the **foundational pattern** for all agentic Copilot features on GHES. The question becomes:

- **Connect-based CCR**: Better standalone product, but does not pave the road for CCA
- **Runner-based CCR**: Less polished standalone, but establishes the platform CCA needs

A possible strategy: invest in runner-based CCR as the **platform foundation**, then layer Connect-based admin controls and identity (bot provisioning, Management Console UI, seat enforcement) on top — getting the best of both worlds.

---

## Critical Review: Runner-Based Privacy Claim is Overstated

### The prompts ARE the code

The runner design claims "code never leaves customer network — only prompts go to CAPI." This is misleading. The CLI constructs prompts containing diff hunks, full file contents, and symbol context — the same source code that the Connect approach transmits. CAPI's LLM sees identical source code either way.

| | Connect-Based | Runner-Based |
|---|---|---|
| Code goes to... | Cloud CCRA → CAPI | Runner CLI → CAPI |
| Who sees the code? | CCRA service + LLM | LLM (same LLM) |
| Data retention risk | CCRA logs + CAPI logs | CAPI logs (same) |
| Network transit | GHES → GitHub cloud | Runner → GitHub cloud |

The runner approach removes one intermediate hop (the CCRA service), but source code still leaves the customer network and hits GitHub's cloud infrastructure (CAPI). "Privacy by architecture" is more accurately "fewer hops," not "code stays on-prem."

### Additional critical issues with the runner approach

**1. Autofind detector distribution is non-trivial**
The design assumes `codeml-detector` (proprietary Go library) can be packaged as a standalone binary for arbitrary runner environments. Questions: binary size, model file dependencies, supported OS/arch matrix, update mechanism, licensing for on-prem distribution.

**2. Comment line-mapping is the hardest part**
CCRA has significant logic for mapping LLM output to exact PR diff positions. This is not trivial CLI work — it involves understanding unified diff format, translating model output coordinates to GitHub review comment positions, handling renames/deletes, and gracefully degrading when mappings fail.

**3. No telemetry or observability**
CCRA has Hydro events, StatsD metrics, and OpenTelemetry tracing. A CLI running on a customer's runner has no telemetry path back to GitHub. How do you: debug quality regressions? Monitor success/failure rates? Track usage for billing? Detect abuse? A/B test model changes?

**4. Model and prompt upgrades are slow**
When CAPI routes to a new model or prompts are tuned, CCRA gets the improvement instantly (server-side). The runner CLI requires customers to update their Action version — which many won't do promptly. This creates version skew and inconsistent quality across the fleet.

**5. "Same as Copilot completions" framing is inaccurate**
Copilot completions send ~100-200 lines of surrounding context per request. Code review at Tier 1-2 sends entire changed files (potentially thousands of lines across multiple files). The data exposure surface is significantly larger than completions.

**6. Billing and metering gaps**
Without server-side telemetry, GitHub cannot accurately meter usage for billing. CAPI token-based tracking may work at a coarse level, but lacks the per-review, per-repo, per-user granularity that the Connect approach provides natively.

## Open Questions

1. **Should GitHub ship an official runner-based Action at all?** Or let customers/community build it?
2. **If both exist, how do we avoid confusing customers?** Clear docs: "Connect = recommended, Runner = for customers who can't use Connect"
3. **Can we reuse the same CLI binary for both?** (Runner uses it directly; Connect proxy uses it internally for context extraction)
4. **Is there a middle ground?** E.g., runner-based execution but with Connect-based admin controls and identity?




