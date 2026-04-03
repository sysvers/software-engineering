# Working with Management

Staff Engineers and Engineering Managers are partners, not competitors. You own the *how* — technical direction, architecture, and engineering quality. They own the *who* and *when* — people, process, and delivery timelines. When this partnership works, the team moves faster than either of you could achieve alone.

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

The overlap is in strategy and prioritization. You both care about what the team works on — you from a technical perspective, they from a business and people perspective. The best outcomes come from genuine collaboration in this overlap zone.

### How to Work Together

- **Align on priorities weekly.** A 30-minute sync where you discuss what is technically important and what is business-critical. When these conflict, resolve it together.
- **Present a unified front.** Disagree in private, align in public. If the team sees you and the EM pulling in different directions, they lose confidence in both of you.
- **Respect their domain.** Do not undermine their people decisions. If you disagree with a hiring call or a performance assessment, raise it privately.
- **Share context.** You hear things in code reviews and design discussions that the EM does not. They hear things in skip-levels and stakeholder meetings that you do not. Share freely.

## Working with Directors and VPs

As a Staff Engineer, you will interact with leadership above your direct manager. This is normal and expected — your scope extends beyond a single team.

### What They Need from You

- **Technical translation.** They make resource allocation decisions. Help them understand the technical implications: "If we defer this migration, we will lose approximately 2 hours of engineering time per team per week to workarounds."
- **Risk assessment.** They need to know what can go wrong and how likely it is. Be specific: "There is a 30% chance this integration misses the deadline because we are waiting on the vendor's API changes."
- **Options, not just problems.** "We have a scaling problem" is unhelpful. "We have a scaling problem. Here are three options with different cost-time trade-offs" is useful.

### What You Need from Them

- **Air cover.** When you drive a large initiative, you need leadership to back you when teams push back on priorities.
- **Context.** Business strategy, upcoming organizational changes, and budget constraints all affect your technical decisions. Ask for this context proactively.
- **Feedback.** Your manager may not have the technical depth to evaluate your work. Ask Directors and VPs for feedback on your proposals and your impact.

## Navigating Disagreements

You will sometimes disagree with management on technical decisions. This is healthy. How you handle it determines your effectiveness:

### When You Are Right and They Are Wrong

This happens. Management sometimes makes technically unsound decisions because they lack context. Your job:

1. Raise the concern with data, not emotion.
2. Explain the consequences clearly: cost, risk, timeline impact.
3. Propose an alternative.
4. If they still decide to proceed, commit and execute. Document your concern in an ADR so there is a record.

### When They Are Right and You Are Wrong

This also happens. Management sometimes has business context that changes the technical calculus. If they explain why and it makes sense, update your position openly. "I was wrong about this — given the business constraint, their approach is better" builds enormous credibility.

### When It Is Genuinely Unclear

Most disagreements live here. The right call depends on assumptions about the future. In these cases, propose a way to reduce uncertainty: a prototype, a time-boxed experiment, or a phased rollout. Avoid extended debate when data can settle the question.
