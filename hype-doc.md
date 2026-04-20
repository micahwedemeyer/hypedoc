# Hype Doc — Micah Wedemeyer

> Running record of accomplishments, impact, and wins. Updated weekly.

---

## 2026

### Q1 (Jan–Mar)

- **Identified and drove an urgent alerting consolidation that no one else was prioritizing.** After the Acquiring reorg merged three sub-teams (APM, PayAPI, Payment Foundations), alerting infrastructure was silently fragmenting — 4 legacy Slack channels, 3 independent alert systems, ~45 PagerDuty services, and 3 orphaned escalation policies all still pointing to old team boundaries. While the rest of the team treated this as business-as-usual, I recognized the risk: the post-reorg on-call rotation was days away from going live with no unified alerting, meaning engineers would be paged for services whose alerts they couldn't even see. Took ownership of the entire effort unprompted and delivered under a tight timeline.

- **Navigated competing stakeholder priorities to design a phased approach that satisfied both leadership and front-line engineers.** Management directive was to unify all alerting into shared channels immediately. Front-line engineers pushed back hard — they were comfortable with their existing per-sub-team channels and didn't want to drown in cross-team noise overnight. Bridged the gap by proposing a two-phase strategy: Phase 1 delivered a non-unified but standardized dual-channel setup per domain (`#acquiring-sq-monitoring` / `#acquiring-sq-pager` for Square-native, `#acquiring-ap-monitoring` / `#acquiring-ap-pager` for Afterpay), giving engineers familiar separation while establishing the infrastructure and naming conventions for Phase 2 full unification (`#acquiring-monitoring` / `#acquiring-pager`). This earned buy-in from both sides — leadership got a concrete path to unification with committed timelines, and engineers got a gradual transition that didn't disrupt their workflows.

- **Conducted a comprehensive audit of the Acquiring alerting infrastructure** spanning 20+ Terraform repos, 45 PagerDuty services, and 3 escalation policies inherited from pre-reorg sub-teams. Identified and documented the full routing topology — including a critical gap where PagerDuty had been silently removed from `#payapi-pager` with no replacement, meaning pages were being dropped with no one noticing. Catalogued ~90 services and their PD mappings, classified alert volume (~31 messages/day across channels), and identified flappy monitors contributing to alert fatigue.

- **Designed and executed a multi-track migration strategy** covering Square-native (Terraform/SKI) and Afterpay (Kompute/CAKE) services with different infrastructure stacks. Coordinated Terraform changes across all tf-* repos, managed the singleton Datadog↔Slack integration lifecycle (applying `create = true` repo first), and updated Sentry routing in `tf-pay-observ`. Standardized inconsistent notification routing patterns across all monitors — normalizing double-notifications and filtering non-alerting noise (LaunchDarkly, PayswingBot) out of pager channels.

- **Launched "Self Appointed AI Visionary Thought Leader," a LinkedIn post series** sharing perspectives on AI adoption, tooling, and engineering culture. Building a public presence as a thought leader in the AI-for-engineering space. Posts will eventually be available on micahwedemeyer.com.
  - *TODO: Once posts are live on micahwedemeyer.com, pull in summaries of each post here.*

- **Pioneered a triple-AI review process to safely ship ~70 auto-generated Terraform PRs without drowning human reviewers.** The Slack consolidation effort required updating monitoring configs across 52 Terraform repos — far too many PRs for traditional human code review. Designed and validated a novel cross-agent review pipeline using three independent AI reviewers: Amp (Claude) for full safety-checklist analysis against both the code diff and CI terraform plan output, Oracle (GPT-5.4) as a blind independent second opinion on the same checklist, and GitHub Copilot for inline code-level review. Each reviewer catches different failure modes — Copilot flags undocumented changes in the diff, Amp validates resource types and blast radius against the plan, and Oracle provides a true independent cross-check that caught issues Amp missed (and vice versa). Built in mandatory safeguards: reviews cannot start until ALL CI check runs reach `completed` status (learned the hard way on tf-everlink PR #91 when a still-queued global plan hid 8 Aurora DB parameter group drift items), and every PR body is updated with a clear ✅ SAFE or ❌ UNSAFE verdict visible at a glance. This reduced human reviewer burden from "carefully review 70 Terraform plans" to "spot-check the AI's work on Tier 0 services and rubber-stamp the rest" — turning a multi-week review bottleneck into a same-day operation.

- **Built an automated post-migration Datadog verification system to catch silent routing failures across 52 repos.** Recognized that merging Terraform PRs is only half the battle — a failed pipeline, a missed `.observ-cli.yaml` update, or a partial apply could leave monitors silently pointing at the old channels with no visible error. Wrote a verification script that queries every monitor for a given service via the Datadog API and checks four dimensions: Slack notification handle, team tag, monitor name prefix, and PagerDuty escalation handle. Accounts for sub-monitors (QPS breakdowns, latency sub-metrics) that intentionally have no `@slack-` reference, so they don't false-positive as failures. Built both a per-repo mode (run after each merge) and a batch sweep that finds *any* monitor across the entire Acquiring org still referencing old channels — catching stragglers from repos we missed, non-Terraform monitors, or failed applies. Dry-run tested against the tf-afterpay baseline (93 monitors, 59 with Slack handles, 34 sub-monitors) to validate the script's output before any migration began. This turns a "merge and pray" migration into a verifiable, auditable process — every monitor is accounted for, and any regression is caught within minutes of apply.

- **Drove AI adoption on the team from skepticism to enthusiasm in under a month** by integrating Builderbot into the new consolidated monitoring channels to auto-analyze every incoming Datadog alert. Within weeks, on-call engineers reported that the analysis was consistently high-quality — surfacing root causes, relevant runbook steps, and correlated signals that would have taken minutes to find manually. Designed a lightweight feedback loop where engineers could correct Builderbot's analysis in-thread and say "put that in the runbook," creating a self-improving system where each on-call shift made the bot smarter. Turned a team of AI skeptics into advocates actively proposing new automation opportunities — a cultural shift that has compounding returns as the team now defaults to "can we automate this?" instead of accepting manual toil.

### Q2 (Apr–Jun)

- **Built an agentic AI pipeline that uses agents to write runbook documentation, then uses other agents to test it — creating a self-validating loop that systematically closed documentation gaps across 13 high-burden payment gateways.** The newly consolidated Acquiring team owned ~99 services across 5 former sub-teams, but the `acquiring-oncall` AI skill only had alert-specific references for 7 categories — leaving engineers without agent-assisted guidance for the most common pages on critical gateways like eero (Chase Paymentech), cafis (Japan), and cuscal (Australia). Rather than manually auditing and rewriting each service's runbook, I designed a multi-agent orchestration system with a novel write-then-test feedback loop:

  **Agent orchestration layer:** A coordinator agent fans out parallel sub-agents to gather data from multiple sources simultaneously — one reads the service runbook, another extracts Datadog monitor definitions from Terraform, another pulls 90 days of PagerDuty incident history. The coordinator synthesizes findings into a gap analysis identifying the highest-value missing documentation per service. This turned what would be a full day of manual cross-referencing per service into a ~15-minute automated sweep. Ran 5 services in parallel in a single session.

  **Iterative write-then-test loop:** The core innovation is treating documentation as testable code. After a reference doc is drafted, a *completely clean* agent — with zero context from the writing process — is spawned to triage a real historical PagerDuty alert using only the skill and the new reference. This "contamination-controlled" testing ensures the docs work for any engineer (or agent) encountering the alert cold, not just the author who already knows the answer. Results are evaluated against a 10-point rubric covering service identification, severity calibration, partner contacts, historical patterns, and remediation steps. If the test agent misses something, the doc is revised and re-tested — true red-green-refactor for documentation.

  **Measured improvement on eero (first service through the pipeline):** Baseline agent improvised generic advice, treated a routine transient alert as a major incident, missed the Chase Tech Ops phone number, and had no historical context. Post-change agent immediately identified the alert as matching a known transient pattern (56% of eero pages, all auto-resolving in 3-5 minutes), provided the Chase number, correctly advised "do NOT escalate unless >10 minutes," and cited 3 specific historical incidents. 7 of 10 evaluation criteria improved, 0 regressed.

  *TODO: Update with final ship count and measured outcomes as services complete the pipeline.*

### Q3 (Jul–Sep) — Projected Outcomes

- **Reduced mean time to triage (MTTT) for oncall pages by an estimated 60-70% across the team's 13 highest-burden gateways.** Before the runbook rewrite, oncall engineers receiving an unfamiliar page had to: read a 20-60KB runbook searching for relevant sections, cross-reference PagerDuty and Datadog manually, search Slack history for past incidents, and improvise a triage plan — a process that took 10-20 minutes per page. After shipping agent-optimized reference docs for all 13 high-burden services, the oncall agent now provides a complete triage plan with specific Presidio queries, partner contact info, escalation thresholds, and historical context within seconds of receiving the page. For the most common alert type (gateway approval rate drops), triage time dropped from "investigate for 15 minutes then escalate" to "check if recovering, wait 5 minutes, close" — because the reference doc teaches the agent that these are almost always transient.

- **Eliminated unnecessary expert escalations for transient gateway alerts, reducing expert interrupt load by an estimated 40+ interruptions per quarter.** Historical analysis showed that approval rate alerts across eero, cafis, cuscal, and other gateways were generating 3-4 simultaneous PagerDuty incidents per event (due to grouped composite monitors), and oncall engineers without historical context were escalating every occurrence to domain experts. The reference docs include historical incident data showing auto-resolution patterns, explicit "do NOT escalate unless >10 minutes" guidance, and explanation of the composite monitor grouping behavior — dramatically reducing false escalations.

- **Enabled confident cross-domain oncall coverage for the consolidated 20-person rotation.** The team's 3-rotation structure means engineers are regularly paged for services they didn't build. Before the rewrite, a PayCore engineer paged for a cafis 4xx ratio alert had no skill reference and a runbook that only covered 5xx/connectivity issues — they were flying blind. After the rewrite, every high-burden service has structured triage guidance that any engineer (or their oncall agent) can follow, regardless of prior domain expertise. This directly supports the team's goal of eventually unifying from 3 rotations to 1.

- **Established a repeatable, test-driven methodology for maintaining oncall documentation that scales with team growth.** The 6-step pipeline (identify → baseline test → draft → post-change test → evaluate → ship) with contamination-controlled agent testing is now documented and reusable. New services can be onboarded by any engineer following the playbook. The baseline/post-change comparison creates an auditable record of documentation quality improvements, and the evaluation rubric ensures consistency. This transforms runbook maintenance from "someone should update this eventually" into a systematic, measurable process.

### Q4 (Oct–Dec)

---

## 2025

### Q4 (Oct–Dec)

### Q3 (Jul–Sep)

### Q2 (Apr–Jun)

### Q1 (Jan–Mar)

---

## Performance Evaluations

### December 2025 — Adam Edwards (Manager)

**Rating: Consistently Exceeds Expectations**

> Micah is operating as an L6 technical expert with clear team-level and cross-team influence and is consistently exceeding expectations.
>
> This year, Micah led the Mercado Pago integration when the Mexico launch needed a viable Plan B, scoping and delivering a new gateway in weeks while making deliberate trade-offs around auth flows and live testing. He has taken principled ownership of PayAttr, turning a critical but under-owned system into something the organization can rely on: 10x faster backfills, a clearer understanding of queue behavior, and a roadmap to simplify over-engineered paths. This has already paid off, as multiple teams have needed fast backfills and new features in Payattr. He has shaped technical direction across our services through monitor-based deploys, Kotlin adoption, and pragmatic use of cloud tools.
>
> In H1, Micah stepped up as the interim EM, keeping the team focused and on track. He supports peer growth through hands-on mentoring, frequent Team Time sessions, and models humility and bias for action. He is a visible champion for AI, sharing concrete practices that raise the bar for AI use across the team. He is an effective cross-team ambassador, building trust with partner teams and representing APM interests constructively. For example, when someone presented the team with a design for a new way to backfill payment attributes, Micah suggested a more robust alternative and helped co-write the redesign.
>
> Micah has consistently demonstrated exceptional technical leadership and strategic impact throughout 2025. His ability to deliver critical solutions under pressure, drive architectural improvements, and enable team growth exemplifies L6+ performance. From leading the team during interim EM responsibilities to pivoting strategically on Mexico expansion, Micah combines deep technical expertise with strong cross-team influence and mentorship.
>
> A growth area is deepening his mental model of the broader Payments/BFP ecosystem. Giving Micah insight on when to seek wider alignment for projects with cross-team impact. Overall, Micah is exceeding L6 expectations in technical leadership, ownership, and team impact.

### July 2025 — Lauren Fragnito (Manager)

**Rating: Consistently Exceeds Expectations**

> Micah has demonstrated exceptional leadership during H1 when he led the team during my maternity leave. While balancing his role as interim EM, he successfully drove multiple high-impact international expansion initiatives and ensured engineers were unblocked and had opportunities for growth as we progressed toward the $2B GPV goal for APM.
>
> When dLocal integration timelines were at risk, he quickly pivoted and successfully implemented MercadoPago integration for Mexico in record time. This alternative integration met Alpha timelines and provided valuable insights that derisk our upcoming GA with dLocal. He resolved persistent DGFT onboarding issues, reducing problematic onboardings and DLQ issues to zero, which improved our POb/DLQ monitoring effectiveness.
>
> Micah drove architectural improvements by championing Kotlin adoption for new development and led the APM re-tiering initiative, enhancing code quality and system maintainability.

### October 2024 — Self + Lauren Fragnito (Manager)

**Rating: Meets Expectations (Impact, Behavior) · Exceeds Expectations (Betterment)**

**Self-evaluation highlights:**

- Led Scalable QR (DGFT) project day-to-day: troubleshooting onboarding issues, building the Omniview admin tool for comparing DGFT/gateway/Caper views of sellers, adapting to KYC changes
- Drove Kotlin adoption, starting with shared ReDB library work, with plan to spread to the team
- Led Aurora and Terraform updates across team services with careful rollout methodology
- Top code contributor and active reviewer; mentored junior engineers with a balance of challenging their work while staying open-minded
- Strongest area: Betterment — constantly cleaning up feature flags, filing tickets for TODOs, improving processes, presenting useful Team Times with quizzes

**Manager (Lauren Fragnito) highlights:**

> Micah is a technical leader and drives the technical direction of the team. He is constantly evaluating how we do things and considering ways to improve. He is passionate about team and pager health and constantly reviews sentry methods, pages, and logs for potential improvements.
>
> Micah is a top code contributor on the team and an active reviewer. After leading the dgft payments integration, Micah dove deep into the complexity of dgft onboarding to ensure he could support Brandon and guide the direction of our onboarding solution.
>
> Micah led the Aurora and Terraform updates for the team's services. He is methodical in assessing possible issues and rolls new changes out carefully. He is currently leading the redb migration for APM services. He created a shared library that could be used in all of our gateways and plans to abstract it for other java services.
