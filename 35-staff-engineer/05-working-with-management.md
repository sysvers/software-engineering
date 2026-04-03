# Working with Management

Staff Engineers and Engineering Managers are partners, not competitors. You own the "how" — technical direction, architecture, and engineering quality. They own the "who" and "when" — people, process, and delivery timelines. When this partnership works, the team moves faster than either of you could achieve alone. When it breaks down, teams get pulled in conflicting directions, morale suffers, and delivery stalls.

This partnership does not happen automatically. It requires deliberate effort, clear communication, and mutual respect for each other's domain.

The Staff Engineer who ignores management dynamics will find their technical recommendations deprioritized. The one who invests in these relationships will find that their initiatives get resourced, their proposals get fast-tracked, and their influence extends further than their technical skills alone could carry it.

---

## The Staff-Manager Partnership

### Complementary Roles

| Staff Engineer | Engineering Manager |
|---------------|-------------------|
| Technical direction | People management |
| Architecture decisions | Hiring and career growth |
| Code quality and standards | Process and delivery |
| Technical risk assessment | Stakeholder management |
| Mentoring on technical skills | Mentoring on career growth |
| System design ownership | Resource allocation |
| Cross-team technical alignment | Cross-team process alignment |

The overlap is in strategy and prioritization. You both care about what the team works on — you from a technical perspective, they from a business and people perspective. The best outcomes come from genuine collaboration in this overlap zone.

A real-world example: at a fintech company, the Staff Engineer identified that the payment processing service needed a major refactor to support upcoming regulatory requirements. The Engineering Manager knew that two senior engineers were about to go on parental leave and that the product team had committed to a major feature launch in the same quarter.

Together, they designed a phased approach: the critical regulatory changes in phase one (before the team capacity dropped), the broader refactor in phase two (after the feature launch and when the team was back to full strength). Neither could have created this plan alone — it required both technical judgment and people context.

### How to Work Together

- **Align on priorities weekly.** A 30-minute sync where you discuss what is technically important and what is business-critical. When these conflict — and they will — resolve it together. Come with a recommendation, not just a list of problems. "I think we should spend 30% of the sprint on tech debt this quarter because the deploy pipeline is adding 20 minutes to every deploy" gives the EM something concrete to react to.

- **Present a unified front.** Disagree in private, align in public. If the team sees you and the EM pulling in different directions, they lose confidence in both of you. This does not mean you suppress disagreement — it means you resolve it before it reaches the team.

- **Respect their domain.** Do not undermine their people decisions. If you disagree with a hiring call or a performance assessment, raise it privately. An engineer who hears the Staff Engineer criticize the manager's decision to promote someone will trust neither of you.

- **Share context bidirectionally.** You hear things in code reviews and design discussions that the EM does not — technical risks, quality concerns, quiet frustrations about tooling. They hear things in skip-levels and stakeholder meetings that you do not — organizational changes, budget pressures, career concerns from team members. Share freely and regularly.

### The Weekly Sync in Practice

A productive Staff-EM weekly sync covers:

```text
1. What shipped last week — any technical concerns?
2. What is coming this week — any dependencies or risks?
3. Technical debt status — anything getting worse?
4. People signals — anyone struggling, anyone excelling?
5. Cross-team coordination — anything needed from other teams?
6. One strategic topic — a 10-minute deep dive on one issue
```

Keep it to 30 minutes. If a topic needs more time, schedule a separate session. The weekly sync is for alignment, not for problem-solving.

### Anti-Patterns in the Partnership

Several common dynamics poison the Staff-Manager relationship:

- **The Shadow Manager.** The Staff Engineer starts making people-management decisions — assigning work, giving career advice that contradicts the manager's guidance, or holding private performance conversations. This creates confusion about who is in charge and undermines the manager.

- **The Ignored Technical Voice.** The manager makes technical decisions unilaterally, treating the Staff Engineer as a senior individual contributor rather than a strategic partner. If you find yourself learning about technical commitments made in stakeholder meetings after the fact, the partnership is broken.

- **The Blame Dynamic.** When something goes wrong, the Staff Engineer blames the manager's process and the manager blames the Staff Engineer's architecture. Healthy partnerships own failures jointly and diagnose root causes without assigning blame.

- **The Absent Partner.** The Staff Engineer is always working on cross-team initiatives and is never available for their home team's needs. The manager feels they have lost their technical partner.

When you notice these patterns, address them directly in a 1:1 conversation. "I noticed I have been giving career advice to team members without aligning with you first. I want to fix that — can we talk about how to coordinate?"

The earlier you catch these patterns, the easier they are to fix. Left unaddressed, they calcify into permanent dysfunction.

## Working with Directors & VPs

As a Staff Engineer, you will interact with leadership above your direct manager. This is normal and expected — your scope extends beyond a single team and so must your relationships.

### What They Need from You

- **Technical translation.** Directors and VPs make resource allocation decisions that affect multiple teams. Help them understand the technical implications in business terms. "If we defer this migration, we will lose approximately 2 hours of engineering time per team per week to workarounds. With 12 teams affected, that is 24 engineer-hours per week, or roughly one full-time engineer's output lost to inefficiency."

- **Risk assessment.** They need to know what can go wrong and how likely it is. Be specific and calibrated. "There is a 30% chance this integration misses the deadline because we are waiting on the vendor's API changes. If that happens, here is our fallback plan." Vague warnings ("this is risky") are useless. Quantified risks with mitigation plans are actionable.

- **Options, not just problems.** "We have a scaling problem" is unhelpful. "We have a scaling problem. Here are three options: (A) vertical scaling buys us 6 months for $50K, (B) horizontal sharding gives us 2 years for $200K and 3 months of engineering, (C) rewrite on a new platform gives us 5 years for $500K and 6 months of engineering. I recommend option B because..." is the format that gets decisions made.

- **Early warning signals.** Leadership hates surprises. If you see a technical risk emerging — a dependency that is flakier than expected, a migration that is taking longer than planned, a system approaching its scaling limit — raise it early. "I want to flag that the search service is at 70% capacity. At current growth, we will hit limits in four months. Here is what I recommend we do now" is far better than "The search service went down because we hit our scaling limit."

### What You Need from Them

- **Air cover.** When you drive a large initiative that requires multiple teams to invest effort, you need leadership to back you when teams push back on priorities. Before you launch a major initiative, explicitly ask: "I will need your support when teams tell me they are too busy for this. Are you willing to make this a priority?"

- **Context.** Business strategy, upcoming organizational changes, budget constraints, and competitive pressures all affect your technical decisions. Ask for this context proactively. Most directors and VPs are willing to share if asked directly. "What should I know about the business context that might affect our platform strategy this year?" often unlocks information that changes your technical recommendations.

- **Feedback.** Your direct manager may not have the technical depth to evaluate the quality of your architectural decisions. Directors and VPs who have seen your proposals and your results can provide feedback that your manager cannot. Ask them directly: "How do you think the database migration went? What would you have done differently?"

- **Sponsorship.** There is a difference between a manager who supports your work and a leader who sponsors it. A sponsor actively advocates for your initiatives in rooms you are not in, allocates resources proactively, and connects you with the right people. Cultivate sponsorship by consistently delivering results and being transparent about your goals.

### Communication Format for Leadership

Leadership has limited time and broad scope. Optimize your communication:

```text
Format for written updates:
- Status: Green / Yellow / Red
- One-line summary of current state
- Key decision needed (if any)
- Timeline impact (if any)
- What you need from them (specific ask)

Example:
- Status: Yellow
- Search migration is 60% complete, but the data team is blocked on schema changes
- Need: Can you confirm with the data team's director that this is their P1 for next sprint?
- Timeline: 2-week delay if the blocker is not resolved this week
```

In meetings, lead with the conclusion and provide supporting detail only if asked. "We should proceed with option B. It costs more upfront but reduces operational risk by 60%." If they want the analysis, they will ask for it.

## Navigating Disagreements

You will sometimes disagree with management on technical decisions. This is healthy and expected. How you handle it determines your long-term effectiveness.

### When You Are Right & They Are Wrong

This happens. Management sometimes makes technically unsound decisions because they lack the technical depth or context. Your job:

1. **Raise the concern with data, not emotion.** "I am worried this will not scale" is weak. "This architecture handles 1,000 requests per second. Our projections show we will need 5,000 within six months. Here is the load test data" is strong.

2. **Explain the consequences clearly:** cost, risk, timeline impact. Frame it in terms they care about. Engineering time wasted, customer impact, revenue at risk.

3. **Propose an alternative.** Never just say "no." Always provide a path forward. "Instead of building this from scratch, we could integrate with the existing service and add the custom logic as a plugin. This saves three months and reduces operational burden."

4. **If they still decide to proceed, commit and execute.** Document your concern in an ADR so there is a record, then give the decision your full effort. Half-hearted execution of a decision you disagreed with is worse than either full commitment or successful persuasion.

### When They Are Right & You Are Wrong

This also happens, more often than most engineers admit. Management sometimes has business context that changes the technical calculus entirely.

A Staff Engineer at a logistics company argued against a "quick and dirty" integration with a new carrier, insisting on building a proper abstraction layer first. The VP overruled them, explaining that the carrier contract had a 90-day exclusivity window — if the integration was not live in 30 days, the deal died.

The quick integration went live in three weeks, generated $2M in revenue in the first quarter, and the abstraction layer was built afterward with the revenue justifying the investment.

If management explains their reasoning and it makes sense, update your position openly. "I was wrong about this — given the business constraint, their approach is better" builds enormous credibility. Engineers and managers alike respect intellectual honesty.

### When It Is Genuinely Unclear

Most disagreements live here. The right call depends on assumptions about the future that neither side can verify. In these cases:

- **Propose a way to reduce uncertainty.** A prototype, a time-boxed experiment, or a phased rollout. "Let us build the core module in two weeks and validate performance before committing to the full migration" is better than extended debate.

- **Define the reversibility.** If the decision is easily reversible, bias toward action. Ship it, learn from it, adjust. If the decision is hard to reverse (a new database technology, a public API contract), invest more time in analysis.

- **Agree on evaluation criteria.** "We will proceed with approach A. In three months, we will evaluate based on these metrics. If latency exceeds 200ms or the error rate exceeds 0.1%, we switch to approach B." This turns an opinion-based disagreement into a data-driven experiment.

## Managing Across Multiple Managers

Staff Engineers often work with multiple managers — their direct manager, the managers of teams they influence, and directors who sponsor their initiatives. Each relationship requires slightly different management:

- **Your direct manager** needs to understand your priorities, your impact, and your growth areas. Give them enough context to represent your work in calibration meetings. A monthly written summary of your top three accomplishments helps them advocate for you.

- **Managers of other teams** need to understand what you need from their teams and what you offer in return. Be explicit: "I would like to review all design docs that change the payment API. In return, I will provide architecture guidance and help with scaling issues."

- **Directors and VPs** need the executive summary, not the details. Lead with impact and decisions needed. Save the technical depth for when they ask for it.

### Navigating Manager Changes

Your EM will eventually change — they get promoted, transfer, or leave. When a new manager arrives:

1. Schedule a longer-than-usual first 1:1. Share your current priorities, your working style, and what you need from them.
2. Explain the Staff-Manager partnership model you expect. Not every manager has worked with a Staff Engineer before.
3. Give them time to establish themselves before pushing for changes. A new manager who feels pushed by the Staff Engineer will push back.
4. Be explicitly helpful during their ramp-up. Share the team's technical context, history of decisions, and current risks.

## Common Pitfalls

- **Treating management as adversarial.** The "us vs. them" mentality between engineering and management poisons everything. Your manager is your partner, not your opponent. If the relationship feels adversarial, diagnose why and fix it — or change teams.

- **Over-rotating on technical purity.** Insisting on the technically ideal solution when the business context requires a pragmatic compromise. The Staff Engineer who always says "we need to do it right" without acknowledging business constraints loses influence quickly.

- **Under-communicating with leadership.** Assuming that leadership knows what you are working on and why it matters. They do not. Proactive, concise communication is essential. If your director does not know about your biggest initiative, it effectively does not exist for resource allocation purposes.

- **Failing to adapt your communication style.** Using the same level of technical detail with your EM, your director, and the VP. Each audience needs a different depth and framing. Directors do not need to know about the specific query optimization — they need to know it will reduce page load time by 40%.

- **Avoiding conflict.** Disagreeing with management is part of the job. Staying silent when you see a bad technical decision to preserve the relationship is a failure of your role. The key is how you disagree — with data, respect, and a proposed alternative.

- **Making commitments on behalf of the manager.** Telling a team "we should prioritize X" when the manager has not agreed. Always coordinate before making prioritization statements that affect other people's work.

- **Not investing in the relationship.** Treating the Staff-EM sync as a status update rather than a strategic partnership session. If your weekly sync feels like a standup, you are underutilizing the most important working relationship in your role.

## Key Takeaways

- The Staff-Manager partnership is built on complementary ownership: you own the technical direction, they own the people and process. The overlap in strategy and prioritization is where you must collaborate closely.
- Weekly alignment syncs, a unified public front, mutual respect for each other's domain, and bidirectional context sharing are the foundations of an effective partnership.
- Watch for anti-patterns: the Shadow Manager, the Ignored Technical Voice, the Blame Dynamic, and the Absent Partner. Address them directly when they appear.
- When working with Directors and VPs, provide technical translation in business terms, quantified risk assessments, options with trade-offs, and early warning signals.
- Handle disagreements with data rather than emotion, always propose alternatives rather than just objections, and commit fully once decisions are made — even when you disagreed.
- Recognize that management sometimes has business context that changes the technical calculus; updating your position openly when proven wrong builds credibility.
- Adapt your communication format to your audience: detailed with your EM, summarized with directors, executive-level with VPs.
- Invest in the Staff-EM relationship as a strategic partnership, not just a reporting line.
- When your manager changes, proactively share your priorities, working style, and the Staff-Manager partnership model during the first 1:1.
- Give your direct manager enough context to represent your work in calibration — a monthly written summary of your top three accomplishments is a simple, effective tool.
