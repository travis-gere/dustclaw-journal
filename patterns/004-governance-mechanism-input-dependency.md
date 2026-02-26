# Governance Mechanism Input Dependency

This pattern emerged from root cause analysis of recurring governance gate failures in an autonomous agent. It documents how governance mechanisms fail when they depend on environmental state that is never provided to the governed entity, and why this failure class is systematically misdiagnosed as a capability deficit.

---

## Context

Autonomous agents operating under governance constraints — rate limits, scheduling gates, access controls, policy evaluations — must evaluate rules that reference environmental state. A posting cadence gate checks "has 24 hours passed since the last post?" A rate limiter checks "have I exceeded N calls in T minutes?" A compliance evaluator checks "does this transaction meet the current reporting threshold?"

These mechanisms are architecturally sound when the governed entity receives the required state as a verified input. They fail when the entity is expected to determine that state autonomously.

The Dustclaw experiment exposed this pattern through a time-based posting gate that misfired repeatedly across multiple sessions. The root cause was not what it appeared to be.

---

## Problem Surface

### The Observed Failure

The autonomous agent operated under a posting gate: if more than 24 hours had passed since its last post, it was required to create a new post. Otherwise, it would comment on existing posts. The gate enforced a daily posting cadence — preventing both excessive and insufficient output.

Over multiple sessions, the gate misfired. In one session, the agent evaluated a 20-hour gap and concluded it exceeded 24 hours. In another, the agent correctly stated the gap was 6 hours and then concluded the 24-hour threshold was met. Both sessions produced posts that should not have existed, including a near-duplicate of a post published 20 hours earlier.

The initial diagnosis was arithmetic weakness — the model cannot reliably compare timestamps. This diagnosis appeared well-supported: the failures were consistent with an agent that cannot perform temporal arithmetic. Mitigations were proposed accordingly: externalize the time comparison to infrastructure, add a posting cooldown as a simpler evaluation, investigate whether the weakness is model-specific.

### The Actual Root Cause

Investigation of the full input chain revealed the actual failure: the agent was never provided with the current time.

The governance gate required comparing `lastPostTime` (available from the agent's state file) against "now" (not provided anywhere in the agent's operational context). The orchestration framework used `Date.now()` internally for scheduling, but this value was consumed by the framework, never exposed to the agent. The system prompt suggested a tool for checking the time that the agent did not have access to in its heartbeat execution context.

The agent improvised. It estimated "now" from unreliable context clues: a `lastCheck` timestamp from its own state file (written during the previous session, not the current one), timestamps embedded in API responses, and inferred patterns from feed post recency. These estimates were sometimes close enough. Sometimes they were not.

### Why It Was Misdiagnosed

Input absences are invisible in outcome data. When the agent evaluates a 6-hour gap as exceeding 24 hours, the observable output is "the agent failed at arithmetic." The observer sees a wrong answer and attributes it to the agent's capability.

Three properties make this misdiagnosis systematic:

1. **The agent does not report missing inputs.** It does not say "I do not have the current time." It constructs a working estimate and proceeds as though the estimate is authoritative. The improvisation is seamless.

2. **The improvised input looks like a real input.** The agent's internal reasoning includes timestamps, comparisons, and conclusions. The reasoning appears complete. The missing operand is never mentioned because the agent does not know it is missing.

3. **The symptom matches the expected failure mode of a capability deficit.** Wrong temporal comparisons are the expected output of both "the agent cannot do math" and "the agent does not have the right numbers." Without inspecting the input chain, the two are indistinguishable.

This creates a diagnostic trap: the more naturally the agent improvises around missing inputs, the more confidently the failure will be attributed to capability.

---

## The Pattern

**Governance Mechanism Input Dependency:** A governance mechanism that depends on environmental state will fail when the governed entity is expected to determine that state autonomously rather than receiving it as a verified input.

### Defining characteristics

1. **The mechanism is formally correct.** The rule, threshold, comparison, and action are well-specified. A design review would approve it.
2. **The input is assumed, not provided.** The mechanism references a value (e.g., "current time") that is expected to exist in the evaluation context but is never explicitly injected.
3. **The governed entity improvises.** Rather than failing with "input not found," the agent constructs an estimate from ambient signals. The estimate is unreliable but plausible.
4. **The failure is attributed to capability.** Observers see wrong evaluations and conclude the agent lacks the ability to perform the operation, rather than that the agent lacks the data to perform it correctly.
5. **The correct mitigation is the opposite of the assumed mitigation.** Capability deficits call for removing responsibility from the agent (externalization). Input absences call for adding data to the agent's context (injection).

### Structural formula

```
Governance Mechanism  =  Rule  +  Threshold  +  Evaluation  +  Input
                                                    |              |
                                              (agent performs)   (must be provided)

When Input is missing:
  Agent improvises Input' ~ Input (unreliable)
  Evaluation(Rule, Threshold, Input') -> intermittent failure
  Observer sees failure -> attributes to Evaluation capability
  Actual cause: Input absence
```

---

## The Two Failure Classes

This pattern introduces a diagnostic distinction between two failure classes that produce identical symptoms but require opposite mitigations:

### Failure Class A — Capability Deficit

The agent cannot perform the required operation even with correct inputs. The model lacks the reasoning capability to evaluate the governance rule.

**Mitigation:** Remove the operation from the agent's responsibilities. Externalize to infrastructure. Upgrade the model. Simplify the evaluation.

### Failure Class B — Input Absence

The governance mechanism lacks a required input. The agent may or may not be capable of the operation — it was never given the chance to demonstrate capability with correct data.

**Mitigation:** Provide the input. Inject verified environmental state into the agent's context before evaluation.

**The diagnostic sequence matters.** Verify the input chain before attributing governance failures to agent capability. The natural tendency is to skip step one because capability deficits are visible (wrong answers) while input absences are invisible (the agent improvises silently).

---

## Intervention

The fix for the observed failure was minimal: six lines, 253 characters.

Before the agent reads its state or evaluates any gate, it executes `date -u` to obtain the current UTC timestamp. The gate evaluation instructions explicitly reference this output as the authoritative current time. The agent is told not to estimate or guess.

This converts the evaluation from a two-operand comparison with one missing operand into a two-operand comparison with both operands present.

---

## Observed Impact

### Pre-fix

Two consecutive gate evaluations, both incorrect:

- **Session 1:** 20-hour gap evaluated as exceeding 24-hour threshold. Agent believed the date was one day later than reality. Gate misfired. Near-duplicate post published.
- **Session 2:** 6-hour gap correctly identified as "more than 6 hours" but gate still triggered. Agent stated the number and immediately contradicted it in the conclusion. Post published.

### Post-fix

Three manual heartbeats triggered for validation. Two independent gate evaluations, both correct:

- **Evaluation 1:** Agent ran `date -u`, obtained verified timestamp, calculated gap as 13 hours 40 minutes, correctly determined "24h threshold not met." Gate did not fire.
- **Evaluation 2:** Agent ran `date -u`, obtained verified timestamp, calculated gap as 13 hours 43 minutes, correctly determined "NOT triggered yet (need 24h)." Gate did not fire.

Clean reversal. Both post-fix evaluations referenced the verified timestamp, calculated the gap correctly, and compared it against the threshold with the correct conclusion. No improvised timestamps. No ambient signal estimation.

The agent's arithmetic capability was not the variable. The input was.

---

## Enterprise Mapping

### Time-Based Rate Limiters

An autonomous agent tasked with API calls is governed by a rate limit: "no more than N calls per T minutes." The agent tracks its call history but derives "current time" from context clues rather than a verified clock. It occasionally exceeds the rate limit — not because it cannot count calls, but because its temporal reference is imprecise.

**Diagnostic trap:** "The agent can't enforce its own rate limit" leads to externalizing rate limiting to the gateway. **Actual fix:** Provide a verified timestamp in the agent's context.

### Policy-Version-Dependent Access Controls

An agent evaluates access requests against a governance policy. The policy is versioned. The agent is expected to use "the current version." The current version is not injected — the agent must query a policy store. If the query fails silently or returns a cached version, the agent evaluates against stale policy and the failure is attributed to judgment.

**Diagnostic trap:** "The agent made wrong access decisions" leads to removing access control from the agent. **Actual fix:** Inject the current policy version into the agent's context with a verified timestamp.

### Resource-Aware Scheduling

An agent is instructed to "schedule tasks only when resource utilization is below 80%." The agent does not receive current resource metrics. It estimates from historical patterns. It occasionally schedules during high utilization.

**Diagnostic trap:** "The agent can't assess resource availability" leads to removing scheduling authority. **Actual fix:** Provide current resource metrics as a verified input before scheduling decisions.

### Compliance Evaluation

An agent reviews transactions for regulatory compliance. The compliance rules reference "current reporting thresholds" that change quarterly. The thresholds are not injected. The agent uses thresholds from training data or from the most recent context it has seen. It flags or passes transactions against stale thresholds.

**Diagnostic trap:** "The agent can't keep up with regulatory changes" leads to replacing the agent with a rule engine. **Actual fix:** Inject current thresholds as verified inputs at evaluation time.

---

## Diagnostic Protocol

When an autonomous agent fails a governance check, follow this sequence:

1. **Verify the input chain.** Does the agent have all required inputs for the evaluation? For time-based gates: does the agent know the current time? For policy-based gates: does the agent have the current policy version? For resource-based gates: does the agent have current resource state?

2. **If inputs are missing:** Provide the input. Test again. If the failure resolves, the root cause was input absence (Failure Class B).

3. **If inputs are present and correct:** Test the agent's evaluation capability with known inputs and expected outputs. If the evaluation fails, the root cause is capability deficit (Failure Class A).

4. **Do not skip step 1.** The natural tendency is to attribute governance failures to the agent's capability because capability deficits are visible and input absences are invisible. The input was never provided, so there is no failed attempt to observe — only the agent's improvised workaround, which looks like a capability failure.

---

## Structural Insight

The core finding generalizes: **a governance mechanism is only as reliable as its least reliable input.**

A mechanism's formal specification may be correct — the threshold, the comparison, the action all well-defined. Its operational reliability is determined by the input chain. If the input depends on the agent's ability to improvise from ambient signals, the mechanism inherits that improvisation's reliability characteristics.

This applies recursively. Any governance architecture that validates mechanisms by reviewing their rules without auditing their inputs will produce false confidence. The rules are correct. The inputs may not be present.

The corollary is that governance mechanism audits must include input chain verification: for each rule, what inputs does it reference, where do those inputs come from, and are they verified or improvised?

---

## Remaining Limitations

1. **Single mechanism type.** The validated instance is a time-based posting gate. Other mechanism types (policy evaluation, resource scheduling) are mapped by analogy, not by direct observation.
2. **Single model.** All observations are on Claude Haiku 4.5. Different models may improvise differently around missing inputs, potentially making the diagnostic trap easier or harder to detect.
3. **Short validation window.** Two correct post-fix evaluations confirm the fix works. Longer-term validation across varied conditions (edge cases near the 24-hour threshold, timezone transitions, sessions of varying length) would strengthen the evidence.
4. **No adversarial testing.** The agent was not deliberately given conflicting time signals. Adversarial scenarios — where ambient signals suggest a different time than the injected timestamp — have not been tested.

---

_This pattern is derived from root cause analysis and controlled validation within the Dustclaw autonomous agent experiment. The diagnostic reframe — from capability deficit to input absence — was triggered by a question from the system architect and confirmed through pre/post comparison of governance gate evaluations. The architecture described here is open and reproducible._
