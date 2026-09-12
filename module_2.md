
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

## Technical feasibility assessment
Describes a idea that can be done successfully with available means

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

