## Safety stack

Safety is a full set of controls, each covering a different part of the request path, each with a blind spot the next one has to catch. Safety is built around context.

Claude arrives with broad safety behavior in place, but it does not know the partner's data-handling rules, authorization model, or domain policy. Before you start adding controls, how much safe behavior is already handled by the model, and how much is still yours to build? You need to understand this boundary clearly to avoid creating duplicate protections or assuming the model is enforcing a rule it has never seen.

Anthropic trains Claude against a constitution: a written document that describes the values and behavior the model should exhibit. Anthropic revises this document over time, and the most recent published version is from January 2026. It sets a priority order for the model to follow when goals conflict: be broadly safe, be ethical, comply with guidelines, and be genuinely helpful to operators and users. That ordering matters because a helpful answer is sometimes unsafe. Higher-priority goals generally take precedence when they conflict, but important to understand that the model weighs them together rather than applying them in a rigid sequence. 

Treat safety as four layers stacked from Claude outward. Each one covers something the layer below cannot, and each one fails in a way the next must catch:
* Model trained behaviour (owner: Anthropic) - covers broad classes of harmful or unsafe output, applied to every request without configuration. It does not cover your domain policy, your data rules, your authorization model. 
* System-prompt instructions (owner: Architect) - covers role, tone, and stated constraints that steer Claude inside one request. Anything an adversarial or unusual input can talk Claude out of, since instructions are not enforcement.
* Runtime screening (owner: Architect) - covers input and output screening that detects disallowed content. Does not cover actions with side effects and novel attacks.
* Authorization (owner: Architect) - covers whether a specific action with a side effect is permitted for this caller in this context. It does not coveers content quality and fairness.

Each added layer costs latency and engineering. Four layers means four places to design, version, and test. The system prompt and the screening logic drift independently if not governed.

## Trained policy vs Domain policy

Trained refusals can be mistaken for a domain policy. It may seem that if Claude already refuses broadly harmful requests in testing, then it covers your partner's data handling policy. 

A domain policy was conflated with trained alignment. Any rule that is specific to your partner must be enforced in a layer you build.

## LLM-system risk

* Direct prompt injection: A user crafts input that overrides the system's instructions.
* Indirect prompt injection: Malicious instructions arrive through retrieved content or tool outputs.
* Token-budget exhaustion: Oversized or adversely padded inputs.
* Tool and action abuse: The model is induced to call a side-effecting tool.
* Data exposure: Sensitive fields enter the context window.

To identify week points walk the request and data paths together. At each entry point, user input, retrieved content, tool outputs, the model's own output, and the logs. The risk assessment should be a written artifact. For each identified risk, record the category, the affected component, a likelihood-and-impact judgment, and the mitigation control with an owner and an evidence artifact.

Examples:
Treat retrieved text as untrusted and screen tool/content inputs, not just user input.
Action-authorization check that runs before the tool executes.
Limit input so a large document cannot truncate the work silently.
Server-side redaction of sensitive fields before anything is logged.