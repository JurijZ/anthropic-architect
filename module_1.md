## Glossary
Authoritative source - means the source you have agreed to treat as correct
Adaptive thinking - Extended thinking where the model itself, rather than you, decides whether to think and how much, based on the complexity of each request.
CSP - Cloud Service Provider 
Live state - Data that changes during the lifetime of a conversation or process
Monolithic - everything the model might need is loaded into context up front, in one block. 
Progressive - an approach where context, instructions, or capabilities are loaded in stages as the work requires them, rather than all at once at the start. 


## Architecturing basics

When you're architecting a solution, you're making these decisions: 
1. identify what the ask is, 
2. which systems you have available to address it, 
3. where human judgment needs to be involved, 
4. determine where Claude can help.

The architectural decision depends on five factors: 
* predictability (how predictable the task is), 
* error cost (how expensive a wrong answer would be), 
* observability (how visible the work is while it runs), 
* latency (how long you can wait),
* cost (how much you can spend per run). 

The tightest constraint is the factor that drives the architecture.
For example, if the error cost is the binding constraint, error cost picks the pattern.

Core design decisions:
* the platform entry point, 
* workflow pattern,
* how the work is split across Claude and existing systems, 
* the model and context strategy,
* the human-in-the-loop posture.

Task assignment: 
* assign to Claude, 
* assign to an existing system, 
* assign to a human.
Getting this wrong by over-assigning to Claude is the most common and most expensive early mistake.

Pattern selection:
* is work is an augmented call, 
* is work is a workflow, or 
* is work is an agent. 
Each one takes a different position on two axes: predictability (how predictable the path through the work is) and model autonomy (how much autonomy you're willing to hand to the model).
Each choice costs you something, naming that costs is the objective.

## Entry points

Anthropic designs different products to act as the primary "way in" for different audiences to access Claude:
* Claude.ai (web, mobile, and desktop apps) - This is the entry point for applied AI users, not builders. The consumer tiers fit individuals and small teams and Claude for Work fits organizations that need governance and identity controls on the same product.
* Claude Code (terminal, IDE plugin, desktop, web) - For engineers doing real development work.
* Claude Cowork - A desktop agent for non-developers that works with local files and applications, automating file and task management on the user's machine under configurable permissions.
* Claude in Chrome - A browsing agent that operates inside the Chrome browser, navigating pages and taking actions on behalf of the user.
* Claude for Excel - A spreadsheet agent that operates inside Excel, working directly with cells, formulas, and structured data.

Custom applications implement product related entry point.

Multi entry points - a deployment that spans more than one entry point exposes a class of problems a single-entry-point system does not.
Which entry-point owns which task, and why?

An entry-point chosen for one task gradually taking on another because the routing logic was never documented.

Multi-platform routing multiplies integration points, each with its own auth, logging, and failure profile. The entry-point-responsibility map is the only thing that keeps them understandable over time.

## Built-time interfaces

* Direct API - For teams building Claude directly into their own product. Use this when an SDK has not exposed a feature you need.
* SDKs - SDKs offer the same capability as the API, wrapped in language-native types and helpers that cut down on boilerplate. 
* MCP - For teams that need the same tools reachable from multiple Claude clients. If only one client will ever use it, MCP adds overhead without much payback.
* Agent SDK - Running a managed agent loop, the same loop that powers Claude Code, from the partner's own application code. The Agent SDK handles iteration and termination.

### Augmented LLM
A single model invocation: you send the request, the model completes the task, and your code handles the wiring around it. You can add tool use, retrieval, or extended thinking to that call, but the model is still doing one bounded job in one pass. The control flow never branches based on what the model decides. Use this when the task is well-defined, the output is something you can verify, and there's no reason to split the work across multiple steps.

### Workflow
You decompose the task into named steps and orchestrate them in your own code. Each step may or may not call Claude. Because the control flow lives in your code rather than inside the model, you can log it, test it, and reason about its behavior the same way you would any other piece of software. Use this when error cost is real, observability matters, and the steps can be determined in advance.

### Agent
You give Claude a goal and a set of tools and the model determines its own sequence of steps to reach that goal. The control flow lives inside the model, not in your code. Use this only when the path through the work cannot be enumerated in advance, and only when the cost of an unexpected or inconsistent output is acceptable and recoverable. 
In production, agents are typically bound by constrained tool entry points, per-turn budgets, explicit permissions, and stopping criteria. These constraints are not options; they keep an agent from becoming a liability. An agent that can act autonomously is the right choice only when the stakes and reversibility of its actions justify the autonomy it is given.

## Reference architectures

Reference architectures are where a known, good blueprint either fits the problem shape or is misapplied.
Popular blueprints:
* Agent, 
* RAG, 
* Document processing pipeline (Evaluator-optimizer), 
* Routing,
* Coding agent.
Combine them when different parts of your system break differently and pick one when you are still uncertain what the system will need to handle.
 
The most common mistake is to use retrieval as a substitute for live state. Retrieval is built for static documents and stale snapshots, so don't use them during a conversation that needs live data.

Model, context, and entry point are where you choose a model tier, a context strategy, and a delivery route.
Check if governance and regulated-industry constraints rule-out a route before considering any cost or latency tradeoffs.


## Claude properties
Next-token prediction, Knowledge, Working memory, Steerability

### Next-token prediction
Tasks built on common patterns: summarizing, reformatting, and explaining well-established concepts.

Limitation: Anything requiring precision on specifics. Claude can produce text that appears accurate but isn't. This risk concentrates around names, dates, citations, and statistics.

Mitigation: Use citations, uncertainty signaling, and generator-verifier loops. Route specific factual lookups through tool calls or authoritative sources rather than relying solely on the model's output.

### Knowledge

Topics in the model's training data that are common, recent, and consistently included where the model can answer reliably from what it learned.

Limitation: Topics that are rare, niche, contested, or frequently changing. The model may present stale or incomplete information with the same confident tone it uses for established facts.

Mitigation: Use web search, retrieval (RAG), tool use, or MCP servers to make an external system the source of truth instead of the model. Instead of checking the model's parametric knowledge, the authoritative answer comes from the retrieved source. When freshness or authority matters, re-introduce the data yourself rather than relying on what was provided in the model's training data.

### Working memory

Anything that fits in the active context window.

Limitation: The context window is a hard edge: once content falls outside the window, the model has no access to it at all. Two different errors happen at the edge, but they can be easy to conflate. One is an oversized request, meaning a prompt or conversation that is already too large to send. When an oversized request is sent it is rejected before generation. If the request exceeds the model's token limit, the API returns a 400 invalid_request_error with a message indicating the prompt is too long. If the raw request body exceeds the API's byte limit, the API returns a 413 request_too_large error with a message indicating the request exceeds the maximum allowed number of bytes. The other error occurs when a prompt fits, but its generation runs into the window ceiling and stops early instead; on current models the response comes back with a model_context_window_exceeded stop reason and truncated output. To avoid hitting the limit, you can check the usage field on every response and the token-counting API before you hit send.

Mitigation: Use progressive context loading, chunking, and front-loading of critical information. For extended work, projects can help manage what stays in scope. Get into the practice of summarizing across turns when context is getting long.

### Steerability

Short, concrete, and verifiable instructions with defined formats, explicit length limits, clear roles.

Limitation: Abstract or ambiguous instructions, long reasoning chains, and tasks requiring precise numerical or logical computation. For high-stakes numerical accuracy, deterministic computation or tool execution should own the answer. The model may follow the letter of an instruction while drifting from the intent.

Mitigation: Use system prompts, structured outputs, and code execution for anything requiring logical precision. When intent and literal instruction might diverge, restate the goal explicitly alongside the instruction.


## How a user reaches Claude

The entry points describe who reaches Claude,
the build-time interfaces all describe how code reaches Claude, 
the route describes where the request is processed.

### Entry point
What a person or system directly interacts with. Entry Points are the wrappers that decide who can talk to Claude and how.
Examples: Claude.ai (web, mobile, desktop), Claude Code, a custom application built on the API.

### Built-time interfaces
How an engineer programs against Claude, the layer the partner's code is written to.
Examples: The direct API, the SDKs, MCP, the Agent SDK.

### Delivery routes
Where API traffic terminates. Delivery routes determine whose infrastructure the request runs on.
Examples: Anthropic directly, AWS Bedrock, GCP Vertex AI, Microsoft Foundry.

Case:
A proposal for a retail banking workflow solution put Claude Code, an engineering entry point, in front of a non-engineering audience because, in the author's words, "it's all Claude." It is all Claude, in the sense that the same model sits underneath every entry point. But the entry point is the wrapper, and Claude Code was built for developers running a terminal, not for bank branch staff following a workflow. Treating the three layers as one erased the distinction that should have ruled the choice out immediately.

Cost: Every entry point carries its own integration cost. Picking the wrong layer because the vocabulary was unclear can lead to paying for the wrong solution, then paying again to replace it.

Complexity: When the three layers are named and discussed precisely, a design review can isolate exactly which decision is contested. When they are blurred, the review argues in circles.

Risk: An entry point chosen before the user is named is a common and avoidable architecture error that is often traceable to collapsing these three distinct layers into one concept.


## Seven primitives

tools - What lets the model take an action or fetch a result from your code, a function the model can call.
mcp - A protocol for exposing a set of tools so multiple Claude clients can reach the same entry points.
subagents - Hand a scoped sub-task to a separate context so work runs in isolation or in parallel.
hooks - Deterministic code that fires on defined events to enforce a rule the model cannot skip.
skills - A versioned, reusable unit (instructions plus optional scripts) that packages a repeatable procedure.
agent teams - Multiple agents working as coordinated peers, each owning part of a larger goal.
dynamic workflows - Assemble the steps of a workflow at runtime rather than fixing them in advance.

Cost: Reaching for a heavier primitive than the job requires is paid for in latency, tokens, and operational surface area. 
E.g., Using a team of agents when a single tool call would suffice.

Complexity: Each primitive added to a design is a part to build, observe, and govern. The discipline is to use the fewest primitives necessary to meet the requirement.


## Deterministic drift

A rule that needs to be right every time was handed to a system that is right most of the time. That tradeoff is easy to miss during scoping because the model handles the clean cases correctly, and clean cases are what you see in demos and early testing. The cost of "most of the time" doesn't reveal itself until you audit and by then the partner is calling.


## Capability packaging
 
Three options sit on a spectrum: 
1. a prompt-only solution (instructions alone), 
2. a direct tool use (the model calls functions in your code),
3. a skills-based architecture (a versioned, reusable Skill that packages the procedure, its instructions, and any scripts as one unit). 

Reach for a Skill when the same procedure runs repeatedly, needs to be distributed across teams or products, or must be versioned and governed.

## Flexibility vs Determinism

Every increase in flexibility comes at the cost of reduced determinism. Only pay that cost when the task genuinely requires it.
Don't optimize for hypothetical future flexibility when today's task can be solved with a deterministic workflow.

Teams often pick agents because a task feels open-ended, but if the execution paths are known or can be enumerated, a deterministic workflow (for example, a router with fixed chains) is simpler, more reliable, and easier to maintain.
Using an agent for predictable work makes it harder to explain, audit, and validate decisions. When auditors ask which step made a decision, pointing to a model interaction is far less effective than pointing to a defined workflow step.
Introducing non-deterministic control flow where deterministic workflows would have met the requirements is a typical mistake.

## Multi-agent systems

Some problems are too large or too varied for a single agent to hold in one context. When that happens, the design moves to multiple agents working together. 

The orchestrator - owns the goal: it decomposes the work, decides what to delegate, and synthesizes the results into a single answer. The orchestrator never does the sub-task work itself; its job is delegation and synthesis.

The subagents - own scoped sub-tasks: each runs in its own context, does one piece, and returns a result.

Three things must be designed, not assumed: 
1. how the work is decomposed into sub-tasks, 
2. how each subagent's result is structured so the orchestrator can combine it, and 
3. how the orchestrator resolves conflicts or gaps when the results come back.

### Error recovery

In a multi-agent system, the architectural question to ask is 'Where is each failure mode recoverable?'.

A subagent failure is usually recoverable: if one unit fails, the orchestrator can retry it, route it elsewhere, or drop it and flag the gap, while the rest of the work proceeds.
An orchestrator failure is usually not recoverable: if the agent that owns the goal and holds the synthesis loses its thread, the whole run fails, and partial subagent work may be stranded.
Design for this asymmetry, make subagent work idempotent and retryable, and protect the orchestrator's state.

| Failure | Where it lands | Design response |
|---|---|---|
| A subagent returns a malformed or empty result	 | Subagent boundary (recoverable)	 | Validate each result; retry or re-route the failed unit; record the gap rather than failing the run. | 
| Two subagents return conflicting results	 | Synthesis step (recoverable) | 	Give the orchestrator an explicit conflict-resolution rule, or escalate the conflict to a human. | 
| The orchestrator loses the goal or its synthesis state | 	Orchestrator (often unrecoverable)	 | Protect orchestrator state; checkpoint progress so a failed run can resume rather than restart. | 
| Traces fragment across orchestrator and subagents	 |  Observability (cross-cutting)	 | Propagate a shared trace identifier so a single run is reconstructable end to end. | 

The dangerous failure is the silent one: a subagent drops a unit and the orchestrator synthesizes a confident, complete-looking answer over incomplete work. 
Validate coverage, do not assume it.

### Human-in-the-loop

A human-in-the-loop checkpoint is a gate that pauses execution for review, positioned by the risk and reversibility of the action about to be taken. 
Place a gate before any irreversible or high-stakes action a subagent would otherwise take autonomously;

### Verification

The orchestrator synthesized over the results it happened to receive, with no rule that the number of results must equal the number of units dispatched. A multi-agent system fails most dangerously when the summary looks complete and is not.
No coverage check at synthesis - completeness was assumed, not verified. 
Confident synthesis over incomplete work. The output's fluency masked the gap.

Watch for places where a failure or a high-stakes action crosses a boundary unobserved.
A recoverable failure was never recovered. A timed-out subagent is the recoverable case, but only if something retries it or flags the gap. Here the failure was silent because nothing was watching the boundary.


## Retrieval vs Tool call

* Retrieval is for stable knowledge: things that were true yesterday and will be true tomorrow. 
* Tool use is for live state: things whose current value is owned by a system and changes independently of your index. 
Conflating them produces answers that are fluent, confident, and wrong in ways that are hard to detect because the system shows no error signal.

Retrieval is the right mechanism for knowledge: FAQs, policies, manuals. 
It's the wrong mechanism for transactional state. 

Order status wasn't failing because retrieval is broken. It was failing because current state had been represented as historical text snapshots in the first place.

Embedding similarity confidently merged two stale snapshots into one answer. A higher similarity score does not mean a truer answer; it means the retrieved text was semantically close to the query, which is not the same thing when the underlying state has changed since the text was written.

Not a better chunker, a shorter refresh interval, or a higher similarity threshold: a tool call to the order-status service. 

The knowledge base keeps the FAQ content. 
The transactional database keeps the orders. 
Two types of data, two access patterns, two mechanisms.

## Model selection

The model eval set is not just a release gate, it's the only thing that makes the model decision defensible during the design conversation.

What to evaluate:
* Model tiers: Opus / Sonnet / Haiku.
* Context strategy: monolithic (Send all available context in one request) / progressive (Provide information incrementally)
* Call volume per day
* Extended thinking (on / off)

Classifier step is using Haiku and confirmed no regression on the eval. 
Mid-pipeline summarization is done by Sonnet.
Opus is used on the final response-composition step.

Once you have a configuration where everything is within budget, ask: which single setting, if relaxed, would breach a budget first?
Then identify what constraint is not compatible with this settings, this is your dominant constraint.
For example if the constraint is latency, then reasoning is the setting to look at.

## Prompting architecture

At enterprise scale the prompt is not a sentence you type, but an asset you design: a system prompt, a reusable template, and the guardrails that keep both safe and consistent. 

A template is a system prompt with parameterized slots (the parts that change per request) and fixed scaffolding around them. The design goal is that the fixed scaffolding carries the consistency and safety guarantees, so that filling a slot cannot accidentally remove a constraint. A well-designed template makes the safe path the default path: the person using it supplies the variable content and inherits the guardrails without having to re-author them.

A well-described prompt names:
The scope: What is in and out of bounds
The format: The exact output contract
The constraints: The rules that must never be violated

Underspecification is a gap the model fills with its own assumption, differently each time, and is the key failure to watch out for.
Where the prompt is silent, the model improvises and improvisation is exactly the non-determinism you do not want in a reused asset. 
The fix is to make the implicit explicit.

Technique:
Zero-shot - Instruction only, no examples. Well-specified tasks the model already handles reliably;
Few-shot - A handful of input/output examples in the prompt. Tasks where the desired format or judgment is easier to show than to describe.
Chain-of-thought - Prompt the model to reason step by step before answering. Tasks where the path matters to the answer.

The key split: 
* instruction-only (zero-shot) when the task is clear and bounded.
* show don't tell (few-shot) when format is hard to specify; 
* step-by-step (chain-of-thought) when the answer depends on a reasoning path; 

The prompt-model pairing is what you are actually shipping.
The same prompt does not behave identically across models. 

## Prompt caching

Promt cache matches on a stable prefix, putting dynamic content first meant the prefix changed on every request and the cache never hit.

Writing to the cache has its own cost. If a prompt is called infrequently or its fixed portion is small, caching can cost more than it saves. 
Caching is a design decision, not a default to switch on everywhere.

A prompt called constantly benefits from a longer-lived cache; one called rarely may never amortize the write.
The economics depend on call frequency and prefix size

### Delivery routes
* If the partner already has a long-term AWS contract, Bedrock is usually the easiest path, the AI spend falls under the same agreement they already have, and the identity system their team uses (IAM) works as-is. 
* The same logic applies to Vertex AI .Partner runs on GCP, the rest of their ML stack lives in Vertex AI, and they want a single billing and audit entry point across foundation models. 
* Partner has a Microsoft enterprise agreement, runs identity through Entra ID, and the rest of their cloud footprint is on Azure. Foundry consolidates AI procurement 
* The direct Anthropic API is the right call when the partner has no strong cloud preference, wants new features the moment they ship, or prefers to keep AI spend with Anthropic directly. 

The Claude model itself is the same regardless of route. Prompting, evaluation strategy, tool use, and context-window behavior all transfer. 

## Common mistakes

1. Choosing the entry point before the user was named. Branch staff need an interface that fits a banking workflow, not a developer tool. Reaching for Claude Code on non-engineering work. It often leads to higher token usage and latency compared to standard chat interfaces or domain-specific workflows that don't need codebase traversal.
2. Defaulting to building an MCP server for every one-off integration or tightly coupled internal pipeline. If an integration is private, bespoke, and serves a single agent or script, standard direct API calls or simple function calling/tool definitions are simpler to implement, debug, and maintain.
3. Compliance should not run on subagents, because they are weakly deterministic. Compliance lives in deterministic server-side code where the guarantees are explicit rather than emergent. Tool calls are audited at the server boundary.

## Governing
### Hooks	

Hooks govern what must happen before or after an action.
Scripts that fire on Claude code lifecycle events (e.g. before/after a tool runs, at session start, on stop), used as deterministic gates the agent cannot skip.

### Permission boundaries and approval flows

Permissions govern what the agent is allowed to touch.
Six permission modes control what Claude Code can do:
* Default - asks before each action. 
* acceptEdits - approves file edits and common filesystem commands (mkdir, touch, rm, mv, cp, sed), though other Bash commands still prompt. 
* Plan mode - locks the session to read-only until the user approves a plan. 
* Auto mode - uses a classifier to approve safe actions and block risky ones;
* dontAsk - auto-denies anything that would prompt and runs only what your allow rules cover, which makes it the mode for locked-down CI. 
* bypassPermissions - skips all checks and is scoped to containers or CI only.

### Sandboxing and restricted execution	
Containment around the workspace in which Claude Code runs, including filesystem boundaries, network egress rules, and constrained command surfaces.

### Regulated industry
Name the governing constraint when you recommend an entry point and let the constraint eliminate options before preferences do.

API or SDK behind the firm's own application, authenticated via SSO, routed through a firm-approved LLM gateway that logs every request.
Delivery routes match the region geographic boundary of the model.
Claude for Government (C4G) - authorized government environments run on a model lag.

