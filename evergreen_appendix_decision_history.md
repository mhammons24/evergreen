# Evergreen Appendix — Decision Options Considered

Use this as input to Microsoft Copilot in PowerPoint.

## Slide 1 — Why move routine maintenance toward centralized automation?

**Subtitle:** Anticipated question: Why not continue letting application teams decide when to consume each base image?

### Option 1 — Centralized Mandatory Patch Pipeline
Platform owns the upgrade cycle and timeline, detects new releases, drives rebase/build/staging, and hands production execution to the existing production process.

**Strength:** Fastest fleet coverage and consistency.  
**Tradeoff:** More coordination and risk of broad impact if a defect is discovered late.

### Option 2 — Controlled Autonomy
Platform provides tooling and a migration deadline, while application teams decide when to rebase, test, and move through staging.

**Strength:** More flexibility for application development cycles.  
**Tradeoff:** Preserves the adoption lag that centralization is intended to solve.

### Current target direction
**Centralize routine base-image maintenance while preserving application ownership of correctness and production risk.**

---

## Slide 2 — How should the fleet be scheduled at scale?

**Subtitle:** Anticipated question: Why use recurring cohorts and RTO sequencing instead of a simpler queue?

### Model 1 — Wave / Batch
Static groups based on LOB, dependency depth, or deployment frequency execute in predefined waves.

**Best at:** Predictable throughput  
**Weakness:** Scheduling rigidity

### Model 2 — Dependency & LOB Based
Work is distributed by organizational capacity, with dependency-aware ordering across teams.

**Best at:** Distributing workload  
**Weakness:** Coordination and decentralized execution

### Model 3 — Continuous Drift / Adaptive
Continuously monitor currency, place eligible work into a managed queue, and execute as capacity becomes available.

**Best at:** Resilience and scale  
**Weakness:** Requires mature scheduling controls

### Prioritization approaches explored
- **Critical First:** RTO 0 first. Fast critical coverage, but the newest image reaches critical workloads before lower-risk workloads establish confidence.
- **Time-Based Rolling Window:** Prioritize age since last patch. Smooth and predictable, but critical workloads can wait behind the calendar.
- **Adaptive Risk Queue:** Blend RTO and patch age into dynamic priority. Flexible, but more complex and harder to explain.

### Current target direction
**Use recurring 3-week capacity cohorts, then sequence each weekly cohort from lower-criticality to higher-criticality applications.**

---

## Slide 3 — How should deferrals and platform intervention work?

**Subtitle:** Anticipated questions: Can teams defer forever? Where should deferral intent live? What happens during an urgent event?

### Deferral models considered
1. **Federated Intent — Git Signal:** Developer-controlled flag or commit signal. Low bureaucracy and strong team ownership, but higher audit complexity and risk of stale intent.
2. **Centralized Intent — UI / Ticket:** Centrally visible and auditable. Cleaner control, but adds process overhead and potential approval bottlenecks.
3. **Platform Enforcement — Debt Manager:** Deferred work remains tracked, accumulates debt, and can eventually be escalated into mandatory execution by policy.

### Emergency / override models considered
1. **Global Toggle — “The Hammer”:** Immediate firm-wide stop. Simple and fast, but extremely coarse and operationally disruptive.
2. **Gatekeeper Approval:** Governance approval required for intervention. Strong accountability, but slows emergency response.
3. **Dynamic Constraint / Safety Valve:** Scheduler rules raise emergency work above normal deferrals and debt. Flexible, but more complex and less explicit operationally.

### Current target direction
**Keep deferral as governed maintenance debt and retain explicit platform governance controls.**

---

## Copilot prompt

Create three executive-level appendix slides from this content. These slides should show the decision-making process behind the Evergreen target state and answer anticipated executive questions. Do not make them architecture slides.

For each slide:
- Show the options considered side-by-side
- Summarize the main strength and tradeoff of each option
- Clearly identify the current target direction
- Keep language concise and executive-level
- Match the existing presentation's fonts, colors, and visual style
- Do not invent new policies, thresholds, approval processes, or technical details
- Make the current target direction visually prominent
