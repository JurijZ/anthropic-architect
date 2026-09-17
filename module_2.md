## Glossary

BAA (Business Associate Agreement)
A contract required under HIPAA between a covered entity (or business associate) and a vendor that handles protected health information on its behalf. 

DPA (Data Processing Agreement)
A contract between a data controller and a data processor defining how personal data may be handled on the controller's behalf, including processing scope, security obligations, sub-processor terms, and breach notification.

Generator-verifier loop
A two-stage pattern in which a model-generated output is checked by a second pass before being used downstream. The verifier may be a deterministic code-based check (schema validation, comparison against an authoritative value) or a second model call scoped to evaluation. Used as a compensating control where the underlying task requires more precision than single-pass generation reliably provides.

Transient error
A transient error is a temporary failure that is expected to resolve on its own without any permanent fix, meaning if you try the same request again after a short wait, it will likely succeed.


## Evals as acceptance criteria

Evaluations let you test your system's behavior before it goes to production or after model updates, so you discover problems before they occur.

An eval suite belongs at the beginning of your build, defined before production code is written, rather than at the end as a QA step. In fact, if you cannot write an eval for behavior, then you have no reliable way to measure whether that behavior is present. This means that every change you make to the system isn't verifiable. Adding an eval suite at the beginning allows you to verify throughout the entire build.

A well-constructed eval workflow runs sequentially through the stages below. Each stage produces an artifact that feeds into the next stage:
1. Define the task - State the behavior you are evaluating in specific, measurable terms.
Output: Task specification with prompt to test and pass criteria.

2. Build the golden dataset	- Assemble the inputs that your system will encounter, including edge cases and counterexamples. T
Output: Labeled dataset with expected outputs

3. Run automated checks	- Pass each prompt through the system and compare the output against your expected result. Use them for behaviors that are unambiguous: format compliance, schema validation, and factual lookups against authoritative data.
Output: Pass/fail record per item.

4. Score with a judge - For behaviors that require interpretation, such as tone, accuracy of reasoning, and appropriateness of edge-case responses, a model-based judge can assess the outputs at scale.	
Output: Score per item with reasoning.

5. Interpret and act - Aggregate scores tell you where the system is and whether a change moved it in the right direction. A change that raises the mean score while quietly degrading performance on edge cases or adversarial inputs doesn't make the system better.	
Output: Overall score, per-category breakdown.


The three types of evals have different speed-versus-flexibility tradeoffs:
* Code-based evals run deterministic checks in milliseconds and cost almost nothing.
* Model-based evals use a judge model to assess outputs that require interpretation and cost roughly as much as the model call itself.
* Human-review evals rely on human judgment for high-stakes or novel behaviors where neither code nor a model judge can be trusted to evaluate reliably. Human-review evals are the slowest and most expensive option.

Everything in the code-based checks is either a structural check or a comparison against a known quantitative value. 
Everything in the model-based checks requires a judgment call that a function cannot supply.

An LLM judge is itself a system that can be wrong. Before you trust its verdicts, make sure to calibrate it. To do this, run it against a set of human-labeled outputs and confirm its similarity with human judgment is high enough to rely on. An uncalibrated judge produces confident scores that may not be high quality at all. This is worse than no automated grade, because it seems trustworthy.

Favor volume over perfection. Many automatically gradable cases beat a handful of manually-graded ones: broad, cheap coverage catches more regressions than a small, painstaking set, and it can run on every change.

Multi-turn evals are a separate category that scores the system over a sequence of exchanges rather than on a single prompt and response. A multi-turn eval checks a few criteria: whether the system keeps prior context straight across turns, whether it answers a follow-up prompt without inventing details that were never said earlier in the conversation, and whether output quality holds as the conversation runs longer.

An eval suite that does not cover the input distribution it will face in production is measuring a different system than the one you are shipping.

Evaluation grading methods: code-based eval, LLM judge, or human review.

## Defining success criteria 
Turning a business requirement into a measurable threshold
A business requirement like "summarize claims accurately" does not really tell you what to measure. 

The process of turning it into an eval criterion has the following steps:
* Identify the behavior specifically: "Summarize claims accurately" should be updated to "extract the filer's name, claim number, incident date, and claimed amount from each document."
* Set the threshold: Decide what counts as passing. If, for example, your thresholds are 100% accuracy on structured fields, less than 2% hallucination rate, and response within schema 99.5% of the time, those numbers should come from the business requirement. 
* Identify the failure modes: Each failure mode is a category in your eval dataset.
* Include adversarial inputs: It should include documents with missing fields, handwritten sections, unusual formatting, and non-standard layouts. If your golden dataset contains only clean inputs, then your eval scores will not predict production performance.

Every change to a production Claude system, whether it is a model swap, a prompt revision, a context strategy change, or a retrieval configuration update, should run through the eval suite during development before it moves to production. This is the only reliable way to know whether a change has improved the system.


## Cost, latency, reliability and failure modes

Your eval suite tells you whether the system behaves correctly, but nothing about whether it can afford to behave correctly at the volume your partner expects.

* Cost: A POC running 10–50 requests per day produces a negligible bill. Monthly cost projections at production volume are a different calculation entirely.
* Latency: A demo typically runs one request at a time. Latency p95 under concurrent load is a different number than median latency under a single request.
* Reliability: A POC has no retry logic, no fallback, and no circuit breaker. Without retry logic or fallback handling, any transient API failure takes the entire user-facing workflow down rather than degrading gracefully.
* Failure modes: Production raises inputs the developer did not expect. Failure modes are specific to architecture type.

The three inputs you need are: 
* call volume (requests per day or per month), 
* token budget per request (input tokens plus expected output tokens), 
* model tier.

Typical mistakes: 
The cost model was wrong because it was built at the incorrect volume. 
The token distribution assumption was wrong because it was built on the incorrect inputs. 
The reliability failure was invisible in development because failure cases were never tested.

A POC answers the question "can the system do this", but it does not answer "what does it cost to do this at scale" or "what happens when a dependency fails."

## Reliability controls

* When a model returns a transient error, such as a rate limit 429, timeout, or 5xx, the system should retry with progressively longer delays between attempts. This prevents a flood of retries from turning a brief hiccup into a prolonged outage. Set the maximum number of attempts and total wait time based on how much delay your use case can tolerate.
* If the primary model or endpoint is unavailable, the system should automatically route the request to an alternative such as a different model tier or a cached response. It should not raise an error to the user. Fallback behavior should be tested as part of your eval suite.
*  circuit breaker measures the error rate on a downstream dependency and trips when errors exceed an established threshold. Once tripped, requests fail immediately rather than waiting for a timeout. This prevents one degraded dependency from taking down the broader system.

Reliability controls must sit at the right stage to be effective: new attempts belong close to the API call, circuit breakers at the service boundary, and fallback chains in the orchestration layer. Placing them in the wrong layer means protecting the wrong part of the system and leaving the right part exposed.

Model version pinning applies to any architecture. It is an operational discipline, not an architecture choice. 

## Failure modes by architecture type

* Agent	Unbounded tool use and growing context. An agent that can call tools without budget constraints or turn limits will run up cost and latency in ways that are invisible until a single request exceeds the budget ceiling. Eval the agent's stopping behavior, not just its output quality.
* RAG (retrieval-augmented generation)	Retrieval quality drift. The retrieval layer degrades when documents are added or removed from the index without reindexing, when the query and the document representation fall out of alignment, or when the index is refreshed on a schedule that creates staleness for live-state queries. eparate live-state queries from static knowledge queries.
* Document processing pipeline (Evaluator-optimizer)	No exception path for low-confidence extractions. A pipeline that routes all documents through the same flow regardless of extraction confidence will produce wrong outputs on edge cases at the same rate it produces correct outputs on clean documents. Route low-confidence extractions to a human review queue rather than downstream processing. 
* Orchestrator-workers	Failure boundaries between orchestrators and subagents blur, traces fragment, and a dropped subagent can fail silently at synthesis. Define recoverable (subagent: retry or flag) versus unrecoverable (orchestrator) boundaries. Create a shared trace ID across all agents.

## Sizing

Output quality is validated through evals, and system reliability is validated through architecture controls like retries, fallbacks, and circuit breakers. Meeting both bars is what production readiness means.
Sizing tells you whether a specific business problem can meet that bar, and what constraints govern the design. Feasibility fits into one of three states: feasible as scoped, feasible with constraints, and not feasible. Identifying the state correctly is what makes a scoping document useful.

Four inputs drive the model: call volume, token budget per request, model tier, and sensitivity parameters.

* Step 1: Estimate call volume. How many requests are made per day or per month?
* Step 2: Set the token budget per request. The token budget has two components: input tokens (system prompt, retrieved context, and user message) and output tokens (expected response length). Model the distribution rather than just the average. If document lengths vary widely, the cost model should account for the typical cases as well as the extremes. If the system prompt is long and stable, prompt caching can meaningfully reduce input costs.
* Step 3: Project the monthly cost. Multiply call volume by the input token count at the input token rate. Separately, multiply the output token count at the output token rate. Then, add both figures.  If the projection exceeds the ceiling, the architecture needs to change before a line of code is written. 
* Step 4: Run sensitivity analysis. What happens to cost if call volume doubles? What if the token distribution shifts toward the tail? 

Caching requires explicit cache_control markers in the request. Cache writes incur a higher per-token cost than standard input, so the cost model must account for the write cost on first use. The default cache TTL is 5 minutes; workloads with request frequency lower than TTL will not realize consistent caching savings.

## How to scope a use case

* Step 1: Business requirement to capability list. What does the system need to do? Name each capability separately.
* Step 2: Capability list to architecture sketch. For each capability, decide where it belongs. Which capabilities does Claude own? Which belong to existing systems? Which require a human in the loop? 
* Step 3: Architecture sketch to boundary conditions. State the conditions under which the architecture works and the conditions under which it does not.
* Step 4: Boundary conditions to scope in the statement of work. The boundary conditions ensures that the development team and the business owner both understand what the system is designed to handle and what it is explicitly out of scope.

A statement of work - is a formal document that defines all project requirements, deliverables, timelines, and pricing for a service agreement between a client and a contractor.

"technically feasible" is meaningless if the constraints aren't applied to the expected scale.
capability question is answered before the constraint questions are asked.

The volume, latency, and input-size constraints are inputs to the feasibility verdict. The verdict is only as sound as the constraints gathered before it.

## Technical feasibility assessment
Describes an idea that can be done successfully with available means

                   [ Working Memory Axis ]
                   Context Window vs. Retrieval
                                ▲
                                │
[ Knowledge Axis ]              │             [ Reasoning Axis ]
Parametric vs. Non-Parametric ◄─┼─►          Direct Extraction vs.
                                │             Multi-step Drafting
                                ▼
                [ Determinism & Security Axis ]
                Access Bounds & Output Variance

* Next-token prediction
The feasibility question to ask: Does this task require probabilistic generation, or does it require precision on specific values? Classification, summarization, and drafting are probabilistic tasks where the model excels. Extraction of specific authoritative values (account numbers, policy dates, claim amounts) requires verification against the source of truth.
Where design compensates: Generator-verifier loops; code-based evals on extracted values; tool calls to retrieve quantitative data.

* Knowledge
The feasibility question to ask: Does this task depend on information that is rare, contested, recent, or domain-specific in ways that may not be represented in training data? If yes, the design must bring the knowledge into the context window. Do not rely on the model to supply it.
Where design compensates: Retrieval-augmented generation for stable knowledge; tool calls for live-state data; flagging uncertainty on contested claims.

* Working memory
The feasibility question to ask: Do the inputs fit comfortably in the context window, or does the task require processing inputs that, in aggregate, exceed the window? Long documents, multi-document tasks, and extended conversations all hit this constraint.
Where design compensates: Chunking strategies; progressive context loading; summarization across turns; pipeline architecture for inputs exceeding the context limit.

* Steerability
The feasibility question to ask: Are the instructions specific, concrete, and verifiable? Abstract or ambiguous instructions, long reasoning chains, and tasks that require precise numerical or logical computation are all places where the model can drift from intent.
Where design compensates: System prompts with explicit output schemas; structured outputs; code execution for numerical precision; evaluator-optimizer loops.

Once the scoping sequence and technical assessment are complete, the architecture is ready for a feasibility assessment. 
There are three possible outcomes.
* Feasible as scoped
* Feasible with constraints - Document each constraint explicitly. 
* Not feasible - State which constraint is disqualifying and why.

A feasible-with-constraints verdict that is not documented becomes an infeasible system when the constraints are violated in production. The constraints are part of the design and carry the same weight as the architecture they qualify.

## Business value and ROI
A feasibility verdict tells the business owner that the system can be built within the budget and the constraints. Hovewer it does not tell them whether building it is worth doing.

The business case rests on five main pillars:
* efficiency (the same work done faster or cheaper), 
* transformation (work that was not feasible before becoming possible), 
* productivity (more output from the same people), 
* solution cost (the run cost of the system itself),
* performance SLAs (the service levels the deployment must hold). 

The ROI estimate is a comparison between two states: The baseline state is how the work is done today, measured in the unit the business cares about. The projected state is how the work is done once Claude is in the workflow, measured in the same unit. The value is the difference between the two states, minus the cost of running the system.
* Step 1: Name the baseline in a business unit. 
* Step 2: Predict the post-deployment state in the same unit.
* Step 3: Subtract the run cost from the sizing model.
* Step 4: State the payback period and the sensitivity. 

Typical ROI estimation mistakes:
* The baseline is estimated rather than measured.
* The projection assumes full automation when the design requires human review.
* The run cost is taken from an average rather than the sizing distribution.

## Enterprise integration 

Sizing tells you what the system needs to do and whether it can do it within the constraints. Integration patterns tell you how it connects to the enterprise stack. 

Regulatory and policy constraints, laws and data-residency requirements eliminate entry point options before any other decisions are made. 

Before any integration design begins, work through the constraints in order: identify the governing regulation or policy, determine which entry point and route are still available, choose the integration pattern that fits, and document the identity, data handling, and observability requirements that follow. Skipping any step risks building something that works technically but fails a legal or security review.

The architectural decision:
* Compliance - Which delivery routes and entry points survive the governing constraint? 
* Authentication - Where does the user identity boundary sit relative to the Claude integration point? 
* Authorization - Which capabilities does this user or role have? What data can they access?
* PII - What data goes into the context window?
* Audit - What questions will you need to answer after an incident?

Every tool you connect to a Claude system is an attack surface and a cost. Establish the trust hierarchy by scoping each subagent's tool access to its task, so a subagent cannot reach tools its job does not require.

Identity verification belongs on the server, before the Claude call. The user's identity and role should be injected into the system prompt by your server rather than provided by the user in their message.

For each field that enters the context window, ask whether it is necessary for Claude to produce the intended output. Reference identifiers like account numbers or claim numbers are often needed for routing but not for the language task itself. 

## Logging

A production Claude system should log four things:

* The request: model version, input token count, prompt identifier
* The response: output token count, latency, stop reason
* The context: user role, session ID, whether caching was applied
* The outcome: whether the downstream system accepted the output and any rejection signals

Security organizations increasingly treat observability as the precondition for enabling agents at all, since without a trustworthy audit trail, an autonomous system is not approved to act.

Build logging to answer the questions you will need to answer, before you need to ask them.

A multi-tenant system running on a shared API key has no way to attribute a rate limit breach to the tenant that caused it. Separate API keys per tenant are required for attribution and isolation in any production multi-tenant deployment.

## PII

If the field is not required for the language task Claude is performing, it should not be in the context window.

Adding a PII redaction layer, building a server-side identity injection, and instrumenting the observability stack all add time. They also add no visible capability, as the system works without them. The cost of skipping them does not appear until the first audit.

The fix was a data architecture change: a server-side redaction step that strips non-essential PII fields before the Claude call, and a retrieval function that supplies only the fields the language task needs.

## A/B testing

Observability answers the monitoring question. Structured A/B testing answers the improvement question. Without both, you are either flying blind or making changes you cannot measure.

An A/B test for a Claude system follows the same structure as any experiment: 
a hypothesis (must be specific and testable),
a treatment group, 
a control group, 
a metric,
a sample size large enough to make the result statistically meaningful. 

The difference from traditional software A/B testing is that LLM outputs are probabilistic, which makes the results noisier and the interaction effects harder to control.
For LLM systems, the variance in outputs is higher than for deterministic systems, which means the required sample size is larger.
A primary metric is a single metric defined before the experiment runs. 

Random assignment of requests to treatment (new version) or control (current version). Assignment must be consistent for a given user or session to avoid contamination. Non-random assignment means the groups are not comparable. If the treatment group happens to receive more complex queries, an apparent win may be an artifact of input distribution.

The two questions to ask before declaring a winner are: is the effect large enough to justify the operational overhead of maintaining the new version? And did any secondary metric degrade?
A prompt change that improves performance on typical inputs may degrade performance on edge-case inputs that appear rarely in the test period but frequently in a future seasonal spike.

There is a way to test against real traffic without exposing - you run the new version in parallel with the current one, send it a copy of live requests, and serve every user the current version's response. The new version's outputs are logged rather than returned, and you score them offline after the fact.

Deploying an unproven or new version of a system carries the risk that it will perform worse than the current version. You should only run a live A/B test if your business and system can tolerate this potential negative impact. Because only a small percentage of your traffic (e.g., 5% or 10%) is routed to the new version (Version B), the exposure to a "worse version" is strictly bounded and controlled.

If your application only gets a few dozen visitors a day, it could take months to gather enough data, making the test impractical. A live A/B test is ideal for high-traffic environments where you can reach the required sample size quickly (e.g., within a few days or weeks).

Offline evaluations (like testing on historical datasets or using automated benchmarks) are highly controlled but cannot perfectly predict how humans will react in real life. For a regulated-industry deployment, where exposing users to an unvalidated model change may not be permissible at all, shadow testing is often the only acceptable way to validate the change.

There is a danger in selecting the metric after seeing the results. It turns a test into a search for whatever metric happened to move. An underpowered experiment with metric selection after the fact produces confirmation rather than evidence.

## Observability

Production observability for a Claude system needs to answer four questions: 
* what is the system doing?
* how well is it performing? 
* when did it change?
* why did it change? 

Request-level tracing - Every request should produce a trace that includes the model, model version, input token count, output token count, latency, stop reason, and any tool calls made.

Metric aggregation - Aggregate the request-level data into the metrics the dashboard displays: cost per request, latency p50 and p95, task success rate by error type.

Anomaly detection - Set threshold alerts on the metrics that matter for the deployment. A cost spike that exceeds 150% of the 7-day average deserves an alert. A latency p95 that crosses the SLA threshold deserves an alert.

Change attribution - When a metric moves, the instrumentation should be able to distinguish Model drift (the model's behavior on stable inputs changed), data drift (the input distribution changed), and model update effects (the model version changed and the new version behaves differently on existing inputs). 

Classifying what you are looking at before making a change:
* Prompt failure - The fix is in the prompt, not the model.
* Hallucination - The fix is grounding through retrieval, tool use, or verification. Stronger instruction will not resolve it.
* Model mismatch - The chosen tier is wrong for the task.
* Orchestrator-workers failure - a recoverable subagent failure (retry or flag) looks different from an unrecoverable orchestrator failure.

Discernment - means moving from passive consumption of AI outputs to critical evaluation. You can't just observe metrics you need to ask are they actually good.

Connecting observability data to business value - The observability stack needs a translation layer that connects the technical metrics to the business metrics they drive.

## Efect impact and effect confidence
Changes has two axes: the expected effect size (how large a difference you expect to see) and the confidence requirement (how certain you need to be before acting on the result). Confidence requirement is driven by the consequence of a wrong call and how reversible it is.

A: Small effect · Low confidence. The effect of a wording change on a clarification message is unlikely to be large. The cost of being wrong is low. A small, fast comparison is appropriate.

B: Small or unknown effect · High confidence. In a high-consequence deployment, a small sample that happens to look positive is not sufficient. The confidence requirement is driven by the consequence of a wrong call, not the expected effect size.

C: Large effect · High confidence. A 30% routing shift has a large effect that affects a substantial fraction of requests. High confidence is required before deploying a change of this magnitude.

D: Large effect on cost · Moderate confidence. The cost effect of a model tier change is expected to be large and is easy to measure. Quality degradation is the risk to monitor, but the cost signal is strong enough to reduce the confidence requirement for the cost component.

E: Small effect · Moderate confidence. Retrieval prompt changes tend to have subtle, distributed effects on output quality. At 200 requests per day, reaching significance on a small effect takes longer, which raises the effective confidence requirement.


