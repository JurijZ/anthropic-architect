## Glossary

Decision log - A record of each architectural choice that captures not just the decision but the alternatives rejected and the tradeoff each resolved, so a successor does not reverse a load-bearing choice for an understandable wrong reason.

Reversal cost - What it costs to undo a decision after the system has been built around it

## Discovery

Discovery reveals whether you are solving the right problem.

You can break down a request, pick a pattern, size a use case, build evals as acceptance criteria, instrument observability, and stand up an auditable control set for a regulated workload. What none of that resolved is the part of the job that happens in rooms with stakeholders.

project lifecycle: discovery → design → handoff → monitoring → iteration. 

Discovery and tradeoff framing do the discovery-and-design work; 

A discovery conversation functions as a three-step filter: 
* listen - listen to the business goal in plain language and pay attention to the meaning behind the word. Stakeholders usually describe the outcome they want but not the constraint you need to design for.
* translate - translate that into requirements, assumptions, and unresolved constraints. This allows you to see what the design must support, what still needs confirmation, and what could block the solution later.
* write down - write those items down before the conversation moves on. The design needs a clear record to follow.

Stakeholders usually speak in preferences, but design decisions are made against constraints.

Example:
Imagine a stakeholder says, "We want this to feel seamless." If you write down "seamless" as the requirement, you have not learned enough to design anything yet.
The real work starts with the next question: what would make it feel not seamless? That is where the hidden constraints begin to appear. You turn business language into something the system can build and be measured against.

Ask what would break that experience, what the user must never notice, what has to happen behind the scenes, and what must still be true when something goes wrong. A testable, bounded constraint is what the design can be built against.

To turn discovery into requirements ask:
* What the system must do - capabilities.
* What the system must not do - prohibited actinos and constraints.
* What must the system cost - budget constraints, latency, volume.
* What the system must prove - it must be able to produce something of value. 

Each item found in discovery becomes one row in a translation table. The row captures:
* the stakeholder statement as it was said, 
* the constraint it implies, 
* the architectural decision that constraint forces, 
* and any assumption you are making when the constraint has not yet been confirmed.

Example:
1. Stakeholder statement - "It just needs to read the form and route it."
2. Implied constraint - Routing may be a deterministic business rule.
3. Required architectural decision - Keep the routing decision in the rule engine. Claude extracts, and the system routes.
4. Assumption of document - Assumes the routing logic is owned and maintained outside the model. Confirm owner.

Note - The expensive failure is the unstated constraint that survives the test and becomes a production review blocker, when the cost of changing the design is at its highest.

## Tradeoffs

Every meaningful design decision has a tradeoff. One option may reduce cost or complexity but increase latency, risk, or compliance burden. Another may improve the user experience or scale better but require more effort upfront. When the downside appears later, the stakeholder may feel they approved of a recommendation without understanding what came with it.

Architects job is not to resolve the tradeoff before the meeting. Architects job is to describe the options clearly enough that the stakeholder can make an informed choice.

* What do we give up?
* What happens to the business if we make the wrong choice?
* What happens if we choose this now but have to reverse it later?

Description means framing the system's behavior, its limits, and the oversight around it in the stakeholder's goal term.

Example:
Architectural decision - Larger context window per call versus retrieval over chunks
What you gain - Simpler initial design, the full document in view, and fewer moving parts to manage early on.
What you give up - Higher per-call cost and slower response times as production volume grows.
What a wrong, reversed choice costs - Reworking the architecture after cost spikes in production

## Demo

Design proof that the solution fits the buyer's context is part of the demo's job.

1. A capabilities demo answers, "What can this system do?" 
2. A scenario-specific demo answers, "What does this system do with my problem, my workflow, and my constraints?"

The first creates interest; only the second creates confidence.

Demo design choises:
* Scenario selection - Choose a workflow the buyer will immediately recognize from their own operations.
* Limitations placement - Decide in advance which limitations the demo will identify.
* Data preparation - Use information that resembles the buyer's data in structure and volume.

## Scoping

If the demo proves you understand the buyer's problem, the scoping session proves you are ready to shape a credible solution around it. 
In joint scoping, you keep that trust by bringing a structured view of the problem, the options, and the unanswered questions.

## Objections

Objections usually fall into these categories. 
* Capability objections - ask whether the system can do the thing at all. 
* Governance and compliance objections - ask whether the deployment can be trusted, controlled, and evidenced. 
* Design-choice objections - ask why you made this choice instead of another. 

## Approval must be an informed choice

A stakeholder who approved a recommendation without understanding the reversal cost has not made an informed choice. The CTO heard a per-call figure and a simplicity argument and reasonably said yes. The reversal-cost element was the one factor that would have changed the decision. Name all three elements every time and name the reversal cost especially when the design feels obviously simpler.

The CTO approved a per-call number, not a monthly bill, and not the cost of unwinding a decision later.

Example:
Workflow pattern with per-interaction logging built in. 
Gains: full audit trail, quality gate before sending. 
Gives up: a small latency cost from the logging step. 
Reversal: minor, logging can be tuned without redesign.

## Feedback loop

The feedback loop sits on top of observability to answer five questions:
* Signals: What is the system showing us?
* Triage: What needs attention now, and what can wait?
* Decide: Does the issue need a fix, a stakeholder review, or no action?
* Act: What correction, update, or escalation is required?
* Review: Did the response work and does the rule need to change?

The feedback loop is a decision layer that sits above the observability stack.
That judgment layer is what makes the system manageable instead of just measurable.
A live deployment drifts without active monitoring. A drift with no trigger is invisible.
Decide which signals deserve attention, and which are noise.

A feedback loop maps each signal to a trigger, an owner, and a required action.

## SLA

Once the feedback loop decides that something matters, the SLA defines what happens when it crosses the line. 
The threshold should trace back to something tangible.

What are we measuring?
What counts as a breach?
What happens when a breach occurs?

Cost is the expectation that breaks most often after launch.

## Documentation

Documentation outlives a handoff and satisfies a compliance reviewer. 

Architecture documentation serves three readers:
* The inheriting engineer takes over a deployment they had no part in building. 
* The auditor arrives later, looking for evidence that a specific control is live and accounted for. 
* The returning architect, often you, comes back months later with no memory of the design sessions.

A document built for one of these readers and not the others is incomplete even when it is detailed.

The document must carry:
* the decisions that were made, 
* the alternatives that were rejected, 
* and the reason each rejection happened.

The costly failure is a successor who reverses a load-bearing decision because the rationale was never documented.

The diagram tells what the system is, not why it is this way.

The completeness test is whether a competent Architect who was not in the room can make a safe change after reading the document. 

Example:
A performance issue prompted a proposal to switch context strategies, the replacement made the switch, and it reintroduced a data-handling pattern that violated the deployment's data-residency constraint. 

The diagram and the decision log both document intention, but not the evidence.

## Outcome document

An Outcome Document is a concise, executive-facing artifact designed to make the real-world value of a completed deployment clearly visible and understandable to stakeholders who were not involved in the day-to-day build.

Engineering teams focus on outputs (code shipped, tickets closed, systems integrated).
Sponsors and executive leaders care about outcomes. 

Example:
Volume, latency, and error rate are real and worth tracking, but none of them are a business outcome. 
The information that makes the document usable for the CFO conversation is the before-and-after on the business metric the use case targeted.

Without the before number, there is no story. 
Without control, the after number is an assertion.
