# Substrate Evaluation Unreliability in Autonomous Agent Governance

This pattern documents how governance mechanisms evaluated by an LLM substrate fail at the substrate's base unreliability rate — regardless of the mechanism's formal correctness, the clarity of its instructions, or the completeness of its inputs. The failure is not in the rule, the data, or the agent's capability. It is in the substrate's non-deterministic evaluation of boolean conditions.

---

## Context

Pattern 004 (Governance Mechanism Input Dependency) established that governance gates fail when required inputs are missing. That pattern's mitigation is straightforward: provide the input.

This pattern addresses what happens after the input is provided. The agent has the correct data. The mechanism is correctly specified. The evaluation still fails — intermittently, non-deterministically, and at a rate that is high enough to cause operational impact but low enough to avoid immediate detection.

The Dustclaw experiment exposed this pattern through two independent governance mechanisms that depend on the LLM substrate to evaluate boolean conditions: a time-based threshold comparison and a list membership check. Both failed at the substrate's base rate. One was resolved through externalization. The other was not, and its failure is compounding.

---

## Problem Surface

### Instance 1: Threshold Comparison

An autonomous agent operates under a posting gate: if more than 24 hours have passed since the last post, the gate opens. The agent receives both operands — the current time (verified by `date -u`) and the last post time (from state). The arithmetic is correct. The comparison fails.

Over 82 evaluations across 12 days, the agent correctly computed the time gap and then drew the wrong conclusion from the comparison ~14% of the time. The agent would state "10 hours have passed" and then conclude "the 24-hour threshold is met." The values are right. The comparison is wrong.

Key properties of the failure:

1. **Non-deterministic.** Identical gap sizes produce correct comparisons in some sessions and incorrect comparisons in others. The failure is stochastic, not systematic.
2. **No improvement trend.** Accuracy oscillated between 76% and 100% across periods. The aggregate converged on ~86%. No amount of instruction refinement, prompt clarification, or worked examples shifted the ceiling.
3. **Self-correction was unreliable.** The agent self-corrected in 21% of misfires (3 out of 14). Present but not dependable.

| Period | Correct | Misfires | Total | Accuracy |
|--------|---------|----------|-------|----------|
| Week 1 (debut) | 6 | 0 | 6 | 100% |
| Week 2 | 22 | 8 | 30 | 73.3% |
| Week 3 | 43 | 3 | 46 | 93.5% |
| **Total (pre-externalization)** | **71** | **11** | **82** | **86.6%** |

**Mitigation (deployed day 30):** The comparison was externalized to a deterministic script. A node process computes the time gap and outputs `POST_GATE: OPEN` or `POST_GATE: CLOSED`. The LLM reads the result. It does not perform the comparison.

**Post-externalization record:** 119 consecutive correct evaluations across 16 days. Zero misfires. The substrate ceiling was bypassed entirely.

### Instance 2: List Membership

The same agent operates under a dedup mechanism: before commenting on a post, check whether the post's ID appears in a list of previously-commented posts. If it does, skip. If it does not, engage.

The list is a JSON array of ~20 string elements stored in the agent's state file. The agent reads the list. The post ID is present. The agent evaluates membership and concludes it is not present. The agent comments again.

| Period | Dedup Failures | Sessions | Failure Rate |
|--------|---------------|----------|-------------|
| Week 1 | 2 | 16 | ~12.5% |
| Week 2 | 1 | 32 | ~3.1% |
| Week 3 | 5 | 30 | ~16.7% |
| Week 4 | 6 | 32 | ~18.8% |

The failure rate is not stabilizing. It is escalating. The escalation has a structural cause.

### The Compounding Property

The threshold comparison failure (Instance 1) was linear. Each misfire was independent. The gate opened incorrectly, a post was published, and the next evaluation started from a clean state.

The list membership failure (Instance 2) is compounding. Each failure produces a duplicate comment on a post that was already engaged. The agent's state update may add a duplicate ID to the list, inflating the array without adding information. The inflated array is harder for the substrate to search. The harder search fails more often.

One post — a discussion of "meaning provenance" — was commented on four times in twenty-eight hours. The post ID was in the list after the first comment. The substrate failed the membership check three additional times. Each failure produced a substantively distinct comment (the agent had genuinely different things to say), but the governance mechanism could not prevent the re-engagement.

This is the first observed compounding governance failure in the experiment. Linear failures (the gate) degrade output quality at a constant rate. Compounding failures degrade the mechanism's own reliability over time. The failure rate is not the substrate's base rate — it is the base rate compounded by state pollution.

---

## The Pattern

**Substrate Evaluation Unreliability:** Any governance mechanism that requires the LLM substrate to evaluate a boolean condition — threshold comparison, list membership, policy compliance, resource availability — will fail at the substrate's base unreliability rate. The mechanism's formal correctness, the input's completeness, and the instruction's clarity do not affect the failure rate. The evaluation step itself is the ceiling.

### Defining characteristics

1. **The mechanism is formally correct.** The rule is well-specified. The inputs are present. A code review would approve it.
2. **The inputs are complete and verified.** Unlike Pattern 004 (input absence), the required data exists and is accurate.
3. **The evaluation fails non-deterministically.** The same inputs produce correct evaluations in some sessions and incorrect evaluations in others. The failure is at the substrate's reasoning step.
4. **The failure rate converges on a ceiling.** Across sufficient evaluations, the error rate stabilizes at ~10-20% for the observed model (Claude Haiku 4.5). Prompt engineering, instruction clarification, and worked examples do not shift this ceiling.
5. **Externalization eliminates the failure class.** Moving the evaluation to deterministic infrastructure (a script that computes the boolean result) reduces the failure rate to zero. The LLM reads the result instead of computing it.

### The generalization

This pattern applies to every boolean evaluation the substrate performs as part of a governance mechanism:

- **Threshold comparison:** "Is X greater than Y?" (~14% failure rate)
- **List membership:** "Does X appear in list L?" (~12-19% failure rate, compounding)
- **By extension:** Policy evaluation ("Does this action comply with rule R?"), resource checking ("Is utilization below T%?"), permission verification ("Does this entity have access A?")

The substrate is not a reliable boolean evaluator under governance-critical conditions. It can reason about the values. It can state the rule. It can describe the correct procedure. It cannot reliably execute the final comparison step.

---

## Diagnostic Protocol

When an autonomous agent violates a governance mechanism:

1. **Check inputs first** (Pattern 004). Is the required data present? If not, inject it.
2. **If inputs are present, check the evaluation.** Did the agent state the values correctly and then draw the wrong conclusion? If so, the failure is substrate evaluation unreliability (this pattern).
3. **Do not attempt to fix through instruction refinement.** The failure is not in the instruction. It is in the substrate's execution of the instruction. More detailed instructions, worked examples, chain-of-thought reasoning, and self-verification steps do not shift the ceiling. They may reduce variance but will not eliminate the failure class.
4. **Externalize the evaluation.** Move the boolean computation to deterministic infrastructure. The LLM's role is to read and act on the result, not to compute it.
5. **Prioritize compounding mechanisms.** If the governance mechanism modifies its own state on failure (e.g., adding entries to a list, updating counters, adjusting thresholds), the failure will compound. These mechanisms should be externalized first.

---

## Enterprise Mapping

### Rate Limiting and Quota Enforcement

An autonomous agent governed by API rate limits, token budgets, or resource quotas performs the limit check internally. The check fails ~10-15% of the time. Budget overruns are intermittent and attributed to the agent's "judgment" rather than to substrate evaluation unreliability.

**Mitigation:** Deterministic middleware performs the limit check and injects the result ("BUDGET_STATUS: 73% consumed, PROCEED") into the agent's context.

### Access Control Evaluation

An agent evaluates whether a requested action is within its granted permissions. The permission model is correct. The agent has the permission matrix. The evaluation fails non-deterministically, occasionally performing actions outside its authorized scope.

**Mitigation:** A deterministic permission service evaluates the request and returns AUTHORIZED or DENIED. The agent reads the result.

### Compliance Monitoring

An agent monitors transactions for regulatory compliance. The rules are injected. The data is complete. The agent occasionally misclassifies compliant transactions as violations (false positives) or violations as compliant (false negatives) — not because the rules are wrong, but because the boolean evaluation step is unreliable.

**Mitigation:** Deterministic rule evaluation for binary compliance checks. The agent handles exception cases that require judgment.

### Multi-Agent Coordination

Agents coordinating through shared state (leader election, task assignment, consensus) rely on each agent correctly evaluating "Is it my turn?" or "Has this task been claimed?" List membership and threshold checks in multi-agent coordination inherit the substrate's base rate, potentially producing conflicting actions.

**Mitigation:** Coordination logic runs on deterministic infrastructure. Agents receive role assignments, not evaluation responsibility.

### The Compounding Risk

The most dangerous deployment pattern is a governance mechanism that (a) relies on substrate evaluation and (b) modifies its own state based on the evaluation result. Such mechanisms can enter compounding degradation: each failure pollutes the state, making the next evaluation harder, increasing the next failure probability. The degradation is invisible to per-evaluation monitoring because each individual failure looks like a stochastic event. Only trend analysis across evaluation periods reveals the compounding pattern.

---

## Structural Insight

Pattern 004 established: a governance mechanism is only as reliable as its least reliable input.

This pattern extends: **a governance mechanism is only as reliable as its least reliable evaluation step.** Even with perfect inputs, perfect rules, and a capable substrate, the evaluation step introduces a hard reliability ceiling. The ceiling is a property of the substrate, not of the mechanism.

The corollary for autonomous agent governance architecture:

1. **Identify every boolean evaluation in the governance chain.** For each: is the evaluation performed by the substrate or by deterministic infrastructure?
2. **Externalize governance-critical evaluations.** The LLM is a reasoning engine, not a boolean evaluator. Use it for judgment. Use infrastructure for comparison.
3. **Prioritize compounding mechanisms for externalization.** Linear failures (occasional wrong answers) are manageable. Compounding failures (state pollution from wrong answers degrading future evaluations) are architectural time bombs.
4. **Monitor evaluation trends, not just outcomes.** A 14% failure rate on a gate is a nuisance. A 14% failure rate on a mechanism that modifies its own state on failure is a compounding degradation loop. Trend analysis is required to distinguish the two.

---

## Evidence Summary

| Instance | Mechanism | Pre-Externalization | Post-Externalization | Status |
|----------|-----------|-------------------|---------------------|--------|
| Threshold comparison | Time-based posting gate | 71/82 (86.6%) | 119/119 (100%) | Resolved |
| List membership | Post dedup check | ~12-19% failure rate, compounding | Not yet externalized | Active |

Both instances run on the same model (Claude Haiku 4.5), the same orchestration framework, and the same agent. The threshold comparison was externalized on day 30. The list membership check remains substrate-evaluated on day 44.

The contrast is the strongest evidence: identical governance architecture, identical substrate, one externalized (100% accurate) and one not (escalating failure rate). The variable is not the mechanism, the data, or the model. It is whether the evaluation step is deterministic.

---

## Remaining Limitations

1. **Single model.** All observations are on Claude Haiku 4.5. Other models may have different base rates for boolean evaluation, but the structural argument (non-determinism in evaluation) applies to all autoregressive language models.
2. **Two mechanism types.** Threshold comparison and list membership are observed. Policy evaluation, permission checking, and multi-agent coordination are mapped by analogy.
3. **No controlled comparison.** The threshold gate was externalized for operational reasons, not as a controlled experiment. The before/after comparison is compelling but not experimentally controlled.
4. **The compounding property is documented from one mechanism.** Whether all state-modifying governance mechanisms exhibit compounding failure requires further observation.

---

_This pattern is derived from longitudinal observation across 44 days, 200+ sessions, and two independent governance mechanism instances within the Dustclaw autonomous agent experiment. The architectural recommendation — externalize boolean evaluations from the LLM substrate to deterministic infrastructure — is validated by a 119-evaluation post-externalization record with zero failures on the resolved instance, contrasted against an escalating failure rate on the unresolved instance._
