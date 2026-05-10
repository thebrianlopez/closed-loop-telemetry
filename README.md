# Closed-Loop Telemetry for AI-Augmented Engineering
## A JSONL Event Bus for Non-Deterministic Systems

*Status: DRAFT (sanitization pass required before publish)*

---

## The Problem

Imagine you hired a tutor to help you study. The tutor is smart and fast, but you notice something strange: some days they explain things clearly and efficiently, and other days they spend twenty minutes re-reading the textbook out loud when you just needed a one-sentence answer. You can see that time is being wasted, but you cannot prove it. You have no record of what happened. You only know the session felt expensive and slow.

This is the situation engineers face when they add an AI coding agent to their workflow.

Before AI tools, software behaved *deterministically*, meaning the same input always produced the same output. If something broke, you could replay the inputs and reproduce the failure. Classic monitoring tools (called APM, for Application Performance Monitoring) were built for this world. They measure things like: how long did a request take? Did it succeed or fail? How much memory did it use?

But AI coding agents are *non-deterministic*. The same question can produce different tool selections, different edits, different reasoning paths through a codebase. Latency tells you the model was slow, but not whether it made good decisions. Token counts tell you how much the session cost, but not whether the cost was justified. The monitoring tools designed for deterministic systems give you the wrong measurements.

What you actually want to know is behavioral: Is the agent following your instructions? Is it getting better over time, or are the same bad habits recurring across sessions? If you add a rule to the agent's configuration file, did the rule actually change anything?

These are the questions that *MLOps* (Machine Learning Operations) is trying to answer. MLOps is the practice of keeping ML-powered systems working well in production, not just at launch. One of its core techniques is *observability*: designing systems so that you can understand what is happening inside them by watching what comes out.

The standard approach to observability in ML is an *eval harness*, a test suite that scores the model on a fixed set of examples. That works for measuring model quality in isolation. It breaks down for agent-level behavioral telemetry, because the behavior you care about is spread across hundreds of real work sessions, not a fixed test set. Manually labeling 1,929 sessions is not a realistic option.

What you need instead is a system that watches the agent in real time, records what it does in a structured format, and automatically converts those recordings into actionable guidance. That system is what this repo documents.

---

## What We Built

The foundation of the system is an *event bus*. Think of an event bus like a flight data recorder for your AI agent. Every time something significant happens, a record of it gets written to a file. The records stack up over time and become the raw material for analysis.

Our event bus writes to one file per calendar day, stored locally on the machine. No database, no cloud infrastructure, no server to manage. Every observable event the agent produces is written as a single line of JSON to `~/.automation-metrics/events/YYYY-MM-DD.jsonl`.

The format is called *JSONL* (JSON Lines): each line is a self-contained, valid JSON object. This makes the files easy to parse, easy to append to, and easy to read with standard tools. You can open a daily file in any text editor and immediately see what happened.

**Schema v2.1** defines 44 event types organized into 11 *source layers*. A source layer is a specific point in the agent's lifecycle where we have instrumented an observation. Think of each layer as a different camera angle on the same game:

| Source Layer | What it watches |
|---|---|
| `session_start` | Session metadata: which model, how much context, which workspace |
| `user_prompt_submit` | When a prompt arrives and what context was injected |
| `pre_tool` | What the agent is about to do, before it does it |
| `post_tool` | What the agent actually did, and how long it took |
| `bash_validator` | Whether shell commands were blocked or allowed |
| `session_stop` | End-of-session summary: total tokens, cost estimate, tool count |
| `recommendation` | Output from the automated analysis pipeline |
| `directive_proposal` | A suggested change to the agent's configuration |
| `directive_decision` | The human's response: approve, snooze, or reject |
| `scoring` | The quality score for a completed session |
| `anti_pattern` | A named bad behavior that was observed, with severity |

Every event shares a common *envelope*, a consistent wrapper that makes events from different source layers comparable:

```json
{
  "schema_version": "2.1",
  "session_id": "8076c539",
  "timestamp": "2026-04-24T19:16:13Z",
  "source_layer": "post_tool",
  "event_type": "tool_call_completed",
  "payload": { ... }
}
```

The `session_id` field connects all the events from a single session, so you can reconstruct what happened in sequence. The `schema_version` field tells you which version of the format a given event uses. That field sounds trivial, but it becomes critical the moment you change the schema: without it, you cannot tell which events use the old structure and which use the new one.

Before tuning anything, we backfilled 1,929 historical sessions to establish *baselines*. A baseline is the starting measurement before any changes. In ML, baselines are essential: without one, you cannot tell if an improvement is real or just noise. The backfill took a weekend. We observed zero schema drift over a 6-day soak period afterward, which confirmed the schema was stable enough to run analysis against.

The analysis produces a quality score per session on a 0 to 100 scale. The scoring *rubric* (the set of rules and weights used to compute the score) has a validated confidence of 91.25/100, meaning its scores correlate highly with manual labels on a held-out sample. The rubric measures four things: tool selection efficiency, output token budget discipline, loop behavior (did the agent get stuck repeating itself?), and model routing appropriateness (did the agent use the right model for the complexity of the task?).

---

## The Closed Loop

The event bus tells you what happened in any individual session. The *closed loop* is what makes the system useful for improving behavior over time.

In control systems engineering, a closed loop is a feedback circuit where the output of a process becomes an input that adjusts the process itself. Your home thermostat is a closed loop: it measures temperature, compares the measurement to the target, and turns heating or cooling on or off. The measurement feeds back into the control.

Our closed loop works the same way. The stages are:

```
Hooks emit events
      |
      v
JSONL accumulates (one file per day)
      |
      v
Batch analysis produces recommendations
      |
      v
adirective proposes a configuration change
      |
      v
Human reviews the proposal (approve / snooze / reject)
      |
      v
Configuration updated
      |
      v
Subsequent sessions re-measured
      |
      v
(back to the top)
```

**Stage 1: Instrumentation hooks.** The agent runtime supports *hook scripts*, small programs that fire automatically at specific lifecycle events. We wired three:

- `UserPromptSubmit` fires when a prompt arrives. It injects context (current date, working directory, branch state) so the agent does not need to run shell commands to look up values it can already be given. It also emits a `user_prompt_submit` event to the bus.
- `PreToolUse` fires before each tool call. It validates the command against a blocklist: for example, if the agent tries to use the shell command `grep` when a dedicated `Grep` tool is available, the hook blocks it, explains why, and emits a `pre_tool` event recording the violation.
- `PostToolUse` fires after each tool call completes. It records how long the call took, how large the output was, and whether it succeeded. It emits a `post_tool` event.

These hooks are Fish shell scripts. They add roughly 2ms of overhead per tool call, which is not noticeable in practice.

**Stage 2: Accumulation.** Events accumulate in daily files. A typical busy session produces 80 to 200 events. Daily files grow at about 200 to 400 KB per active day. A full year of history fits comfortably under 200 MB, which any laptop can handle without trouble.

**Stage 3: Batch analysis.** Once per day, a batch job reads the event files, computes per-session scores, counts anti-pattern occurrences by type, and writes a file called `recommendations.jsonl`. Each recommendation names a specific problem, how often it happened, how confident the analysis is, and which configuration file it would target:

```json
{
  "signal_id": "rec-003",
  "anti_pattern": "bash_ls_instead_of_glob",
  "occurrences_14d": 123,
  "severity": "medium",
  "confidence": 0.91,
  "evidence_window_days": 30,
  "proposed_rule_target": "~/.claude/rules/tool-selection.md"
}
```

The *evidence window* is the number of days of history the analysis used. A longer window catches patterns that appear infrequently. A 30-day window is standard; a 14-day window is too short for rare events.

**Stage 4: Directive proposals.** A component called `adirective` reads `recommendations.jsonl` and generates candidate edits to the agent's configuration. Before proposing anything, it checks the *confidence threshold* for the tier of change being proposed:

| Tier | Class | Threshold |
|---|---|---|
| T1 | Wording reinforcement | 0.75 confidence over 14 days |
| T2 | New rule addition | 0.85 confidence over 30 days |
| T3 | Model routing changes | Human drafts only |
| T4 | Observability infrastructure | Human drafts only |

T3 and T4 proposals are never auto-generated. The system flags that a change is needed, but a human writes the actual directive. Model routing and observability changes have high *blast radius*, meaning a bad change can degrade many sessions at once. The cost of a wrong automatic change is much higher than the cost of a slow manual review, so the human gate stays.

Deduplication prevents the same proposal from surfacing repeatedly. Each proposal is fingerprinted on its (signal ID, target file, proposed change). If an identical fingerprint was already approved or snoozed within the evidence window, the new proposal is suppressed.

**Stage 5: Human review.** Proposals surface in a review queue. For each one, the human chooses: approve (add to the configuration now), snooze (re-evaluate after N days), or reject (close permanently). Approved directives land in the rule file immediately. The next agent session picks them up without any deployment step or restart.

The loop closes when you check the same metric that triggered the proposal and observe it moving in the right direction after the directive landed. If the rate went down, the loop worked. If it did not, the directive was not specific enough and needs iteration.

---

## A Concrete Outcome: The Model Routing Fix

Here is one complete pass through the loop.

**The signal.** The scoring model flagged a metric called `opus_pct`: the fraction of sessions where Opus (the most capable and expensive model tier) was used as the primary model. When `opus_pct` exceeded 40% of session volume, the rubric treated it as a medium-severity anti-pattern.

Opus is the right choice for genuinely hard problems: multi-step architectural decisions with three or more competing constraints. It is the wrong choice for reading a file, searching for a string, or answering a question a skilled engineer could resolve in under two minutes. When Opus handles routine work, cost increases and the quality signal gets noisier. A session scored "high cost" could mean the problem was hard, or it could mean the wrong model was selected. Without separate tracking, you cannot distinguish the two.

**The hypothesis.** The agent's configuration file had no explicit model routing guidance. The agent had to infer when to escalate to a more expensive model, and it was over-escalating. Adding an explicit routing policy with concrete examples for each model tier should reduce Opus usage on tasks that do not warrant it.

**The intervention.** A T3 directive (human-drafted, because model routing is a Tier 3 change) added a new rule file with a three-row routing table:

- Haiku: file search, exploration, any sub-task a skilled human resolves in under 30 seconds
- Sonnet: standard engineering work, including coding, debugging, and code review (the default)
- Opus: genuine multi-constraint architectural planning requiring reasoning across more than three system-level trade-offs

The rule explicitly prohibited Opus for any task resolvable in under two minutes. It also added `opus_pct > 40%` to the scoring rubric as a named anti-pattern.

**The re-measurement.** Within three sessions of the directive landing, `opus_pct` dropped below the threshold. The improvement showed up in the daily scoring output automatically. No code changed. No model changed. A behavioral instruction, enforced by the agent at runtime, was sufficient.

The more important result is not the cost reduction. It is the ability to *verify* that the directive worked. Without the event bus, adding a rule to a configuration file is an act of faith. With it, you have a timestamped before-and-after signal tied to the specific change that caused it. That is what distinguishes a feedback loop from a guess.

---

## What Did Not Work

**Schema drift in the first two weeks.** The original schema had inconsistent field names across source layers: `session_id` in some events, `sid` in others; `timestamp` versus `ts` versus `event_time`. Analysis code that worked for one event type broke silently on another. The fix was straightforward: add `schema_version` to every event from day one, and run a strict validator that rejects malformed events at write time rather than at analysis time. The 6-day soak period after the backfill existed specifically to verify that the validator caught all remaining drift before the recommendation pipeline ran against it.

**Runaway loop detection in the schema.** Early versions tried to detect runaway loops (the agent calling the same failing tool repeatedly) at the event bus level. The plan was to define a rule, fire an event when it triggered, and surface the alert. This approach did not hold up. The definition of "runaway" is context-dependent: three calls to the same tool might be normal behavior in one situation and a bug in another. A schema-level detector baked in too many assumptions. The better solution was a directive: "if the same tool fails three times, stop and surface the blocker to the user." The agent enforces the rule; the event bus observes whether it was triggered and scores the session accordingly. Enforcement and observation are separate responsibilities, and mixing them creates fragile detectors.

**Cost attribution.** Assigning a token cost to a specific decision is harder than it sounds. A session that ends up expensive might be expensive because the problem was genuinely hard, because the context window was bloated with unnecessary files, or because the wrong model was selected. The scoring model tries to separate these dimensions, but the attribution is probabilistic. A "high cost" score is a signal to investigate, not a diagnosis.

**Session correlation beyond tool calls.** The event bus captures what the agent did: which tools it called, what it edited, whether it succeeded. It does not capture why. Connecting a specific tool call to the reasoning that preceded it requires reading the full session transcript, which is large and unstructured. The system measures behavior, not intent. That is a real limitation when debugging subtle anti-patterns where the action was correct but the reasoning path was wasteful.

---

## What Generalizes

The specific stack here (Fish shell hooks, JSONL files, a scoring rubric, a directive taxonomy) is less important than the underlying patterns. Here is what transfers to other contexts:

**Instrument at all lifecycle boundaries, not just around the model.** The most useful events in this system come from the edges: prompt submission, tool call validation, session completion. The model's internal reasoning is not directly observable, but its inputs and outputs are. Design your schema around the boundaries you can see.

**Include schema versioning in every event from day one.** You will change the schema. When you do, you need a way to tell which events use the old format and which use the new one. Adding this field retroactively to 1,929 sessions is a painful weekend. Adding it at the start costs one extra field.

**Organize events by source layer, not just event type.** Grouping events by where in the lifecycle they originate (`pre_tool`, `post_tool`, `session_start`) rather than a flat list of event type names makes analysis code more maintainable and forces you to think about lifecycle position before defining fields.

**Gate automated proposals behind confidence thresholds and evidence windows.** Not every signal is strong enough to act on. A recommendation that fires after two sessions is probably noise. One that fires consistently over 30 days with 0.91 confidence is worth taking seriously. Design your pipeline with explicit thresholds rather than applying every recommendation automatically.

**Keep enforcement and observation separate.** The event bus records what happened. The configuration file says what should happen. The scoring model measures the gap between them. When you try to collapse these roles into the same component, you end up with a system that is hard to reason about and easy to misconfigure. Separation of concerns is not just a software design principle; it is a system reliability principle.

**A feedback loop is only useful if it closes fast enough to be actionable.** A 30-day batch cycle means you are always operating on a month of stale signal. A per-session batch run that finishes in seconds means you can observe the effect of a directive within a few days of landing it, not a few months.

---

*Questions, feedback, or corrections welcome. Open an issue or reach out on [LinkedIn](https://linkedin.com/in/thebrianlopez).*
