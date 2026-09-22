## Safety stack

Safety is built around context.

Safety is a full set of controls, each covering a different part of the request path, each with a blind spot the next one has to catch. 

Claude arrives with broad safety behavior in place, but it does not know the partner's data-handling rules, authorization model, or domain policy. 

Before you start adding controls you need to understand:
* how much safe behavior is already handled by the model, 
* and how much is still yours to build 
You need to understand this boundary clearly to avoid creating duplicate protections or assuming the model is enforcing a rule it has never seen.

Anthropic trains Claude against a constitution: a written document that describes the values and behavior the model should exhibit. Anthropic revises this document over time, and the most recent published version is from January 2026. It sets a priority order for the model to follow when goals conflict: 
1. be broadly safe, 
2. be ethical, 
3. comply with guidelines,
4. and be genuinely helpful to operators and users. 
That ordering matters because a helpful answer is sometimes unsafe. Higher-priority goals generally take precedence when they conflict, but important to understand that the model weighs them together rather than applying them in a rigid sequence. 

Treat safety as four layers stacked from Claude outward. Each one covers something the layer below cannot, and each one fails in a way the next must catch:
* Model trained behaviour (owner: Anthropic) - covers broad classes of harmful or unsafe output, applied to every request without configuration. It does not cover your domain policy, your data rules, your authorization model. 
* System-prompt instructions (owner: Architect) - covers role, tone, and stated constraints that steer Claude inside one request. Anything an adversarial or unusual input can talk Claude out of, since instructions are not enforcement.
* Runtime screening (owner: Architect) - covers input and output screening that detects disallowed content. Does not cover actions with side effects and novel attacks.
* Authorization (owner: Architect) - covers whether a specific action with a side effect is permitted for this caller in this context. It does not coveers content quality and fairness.

Each added layer costs latency and engineering. Four layers means four places to design, version, and test. The system prompt and the screening logic drift independently if not governed.

## Trained policy vs Domain policy

Trained refusals can be mistaken for a domain policy. It may seem that if Claude already refuses broadly harmful requests in testing, then it covers your partner's data handling policy. 
Any rule that is specific to your partner must be enforced in a layer you build.

## LLM-system risk

* Direct prompt injection: A user crafts input that overrides the system's instructions.
* Indirect prompt injection: Malicious instructions arrive through retrieved content or tool outputs.
* Token-budget exhaustion: Oversized or adversely padded inputs.
* Tool and action abuse: The model is induced to call a side-effecting tool.
* Data exposure: Sensitive fields enter the context window.

To identify week points walk the request and data paths together. At each entry point, user input, retrieved content, tool outputs, the model's own output, and the logs. The risk assessment should be a written artifact. For each identified risk, record the category, the affected component, a likelihood-and-impact judgment, and the mitigation control with an owner and an evidence artifact.

Examples:
Treat retrieved text as untrusted.
Screen tool/content inputs, not just user input.
Action-authorization check that runs before the tool executes.
Limit input so a large document cannot truncate the work silently.
Server-side redaction of sensitive fields before anything is logged.

## Guardrails

A guarded request path has a few decision points:
* Input screening runs before the model call and decides whether the request should reach the model at all.
* Output screening runs before the response reaches the user and decides whether what the model produced is safe to return.
* Tool-call authorization runs before any action with side effectst.

Because they sit at different points and check different things, a control at one place does nothing for the others, which is why a single filter cannot cover the whole path.

Tool-call authorization is almost always deterministic check, like an allowlist of permitted actions, identity checks, and scope validation.

There is no control that catches everything. Model-based and deterministic checks fail differently so these controls are deployed in series. Identifying what each one misses ensures each gap is deliberately covered by a different control rather than left open.

A second injection vector: instructions arriving through retrieved content and tool outputs. 
In a RAG system, a malicious instruction in a retrieved document reaches the model after input screening has already passed the request. In an agentic system, a tool response can carry instructions the model treats as authoritative. 

On the API, responses return a refusal when streaming classifiers intervene. The Messages API reports this as stop_reason: "refusal" accompanied by a stop_details object (available since Claude Opus 4.7). That object carries a policy category along with a readable explanation; both fields are null when the refusal does not map to a named category. 

As a rule, once a refusal is received, reset the conversation context before continuing: remove or rephrase the turn that triggered the refusal, or clear the history. Sending the next request on the same refused context returns further refusals.

How the guardrail layer behaves under failure? Anthropic's built-in model safety controls fail closed, that is they block the traffic on failure. 
On the other hand the Architect has to decide what happens when the Guardrails fail, should it fail open (allow traffic) or fail closed (block the traffic). Each gate can pass, block, or fail, and each fail resolves to the direction you chose.

## Skill supply-chain security

An untrusted skill can carry a code-execution exploit: logic that runs commands, reaches out to the network, or touches files the moment it's invoked.

The defense must move earlier in the chain. Before you can trust and call a skill, you need to audit it: open the bundle and read it for two things.
A skill that passes review clean can still reach out at runtime to fetch code that was never in the package you read. The log review can reveal this.

Watch for out-of-scope operations - behavior that doesn't match the job it claims to perform. 

## Fairness

The retrieval corpus can over-represent or under-represent groups, so the context the model sees is already skewed.
The framing of the prompt can encode an assumption that pushes outcomes in one direction.
The examples used in few-shot prompting can carry the same skew the corpus does.

Which of the system points could skew the outcome?

A corpus that over-represents some cases produces unequal outcomes.
Without decision logging, the team cannot explain the harm or find its source.
Examples:
* A routing step that sends some cases down a different path with no log entry.
* A retrieval step whose returned context is not captured.
* A model output stored without the inputs that produced it.

Fairness and explainability are architecture requirements. Instrument them at the points where skew can enter.
Decision must be reconstructable.

## Explain the system behaviour

Differnet audience needs differnet format of explanaition of the system behaviour.

A developer wants a prompt, retrieved context, model output, and every routing step.
A user wants the inputs that drove the decision and the reason the outcome was reached, in a digestible form.
A regulator wants a durable, queryable record of inputs, outputs, and decision path.

## Routing decisions to people by stakes, not by volume

A low-confidence, irreversible, high-cost decision almost always needs human review.
A confident, easily reversed, low-cost decision can usually run without a human. 

Deciding which decisions count as high stakes takes judgment. 
A routing rule has three controls: 
* a confidence threshold, 
* the cost of a wrong answer, 
* and a reversibility setting.

Where the human sits is a tradeoff between safety and speed.
Place a gate before any irreversible or high-stakes action an agent would otherwise take autonomously.

What you put in front of the reviewer determines whether the review is accurate. Ensure your reviewers have three things: 
* the inputs that drove the decision, 
* the model's output, 
* and the reason it was flagged. 
Without this information they cannot tell if the behaviour is correct. 

Consent fatigue - is when a system asks for approval dozens of times in a row, and reviewers start clicking through and approving items without reading or providing the quality of review needed. That pattern is what led to plan-level review in Claude Code, where a person approves the plan rather than each step.

Routing by volume rather than stakes either overwhelms reviewers and risks review quality degradation, or allows a high-stakes, irreversible action with no gate at all.

## Regulations

A regulation states an outcome, but you supply the control and the proof it is operating.
They say what must be true: that protected data must be handled a certain way, that access must be controlled, and that processing must happen in an authorized environment, but they leave the technical control to you.

three things you own: 
* a specific technical control that achieves the outcome, 
* an owner accountable for it,
* an evidence artifact that shows it is live.

A control no one can demonstrate is indistinguishable from one that is not running.
