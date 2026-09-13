
Scenario: A corporate support agent has access to a tool named process_refund. Company policy states that any refund over $500 must be approved by a human manager. However, audit logs show that the Claude agent sometimes calls process_refund for amounts over $500 autonomously.  Question: What is the most reliable architectural solution to guarantee policy compliance?  A. Add a line in the system prompt instructing the agent to escalate refunds over $500.  B. Implement an Agent SDK PreToolUse hook on process_refund that programmatically intercepts calls over $500 and triggers human escalation.  C. Remove process_refund entirely from the toolset so humans handle all refunds.  D. Fine-tune the underlying Claude model on past policy compliant logs.  

Correct Answer: B  
Why: Prompt-based instructions have a non-zero failure rate for policy enforcement. The exam heavily tests knowing when to rely on model reasoning vs. when to enforce hard programmatic barriers (like SDK hooks).

---

Scenario: An automated document extraction pipeline uses Claude to parse invoice data. For documents that do not contain a purchase order (PO) number, Claude occasionally invents a realistic-looking PO number instead of omitting it.  Question: What is the best configuration or schema-level fix to resolve this?  A. Lower the temperature parameter to 0.0.  B. Explicitly define the PO field schema as nullable/optional and specify in the tool definition that missing values should return null or a sentinel signal rather than guessing.C. Wrap the system prompt in JSON markdown blocks.D. Force tool_choice: "any" for every extraction pass.  

Correct Answer: B
Why: Temperature adjustments don't eliminate structural hallucinations, but schema design and tool parameter definitions explicitly signal field absence cleanly to the parser.

---

Scenario: You are building an enterprise workflow where the document type is unknown in advance, but you need to guarantee that Claude returns a structured tool call rather than conversational text.  Question: Which tool_choice parameter setting guarantees a tool execution?  A. Set tool_choice: "auto" and instruct the model in the system prompt.  B. Set tool_choice: "any".  C. Set tool_choice: "none" and parse free text.D. Omit tool_choice and increase context size.

Correct Answer: B
Why: Setting tool_choice: "any" forces the model to choose one of the available tools, whereas "auto" allows the model to respond in plain text if it decides to.  

----

Scenario: A multi-step research agent keeps running out of context space during long multi-turn iterations, causing response degradation.Question: What pattern should the architect implement to maintain long-running task context efficiently?A. Use Task tool primitives to spawn isolated subagents that return condensed summary outputs to a main coordinator.  B. Pass the entire context history into every new call using client-side memory expansion.C. Increase system prompt length to retain past step memory.D. Disable tool output logging.

Correct Answer: A
Why: Subagents operate with isolated contexts. Passing condensed structured findings back to the coordinator prevents the main context window from polluting or filling up.  


---

Scenario: An enterprise customer support bot sends a fixed 15,000-token knowledge base and system prompt with every single request. API latency and costs are unacceptably high. The developer wants to leverage Prompt Caching.

Question: How should the API request be structured to maximize cache hits and cost savings?

A. Mark the dynamic user message with cache_control: {"type": "ephemeral"} at the end of the request body.

B. Place the static knowledge base and system instructions near the beginning of the context and set cache_control: {"type": "ephemeral"} on the last static block.

C. Split the knowledge base into 500-token chunks and attach cache markers to each individual chunk.

D. Lower the model temperature to ensure deterministic cache hits across calls.

Correct Answer: B

Why: Prompt Caching works from the beginning of the request prefix downward. Static context (system prompts, large documentation/tools) must sit before variable user input, and the cache_control breakpoint should mark the boundary of the static prefix.


---

Scenario: You are designing a code audit pipeline. Small bug fixes require rapid processing, while complex architectural refactoring requires multi-step evaluation. You want to optimize both cost and latency across requests.

Question: Which architectural pattern best achieves this balance?

A. Run every request through a single large Claude 3.5 Sonnet context using complex step-by-step thinking instructions.

B. Use a fast classifier/router (e.g., Claude 3.5 Haiku) to evaluate incoming code diffs and route routine fixes to a lightweight model and complex refactorings to an orchestrator-subagent workflow.

C. Send all requests to two parallel subagents simultaneously and select the shortest answer.

D. Require human-in-the-loop approval before processing any request over 1,000 tokens.

Correct Answer: B

Why: Dynamic routing using a lighter, faster model as a classifier saves significantly on cost and latency while preserving full reasoning power for high-complexity requests.

---

Scenario: A healthcare enterprise is building an internal assistant that queries sensitive patient records. System requirements strictly prohibit patient PII from being retained by external LLM provider logs or used for model training.

Question: What combination of API configuration and data architecture best complies with these safety requirements?

A. Encrypt patient records client-side using base64 before appending them to the system prompt.

B. Ensure Zero Data Retention (ZDR) agreements/headers are enabled on the API tier, redact PII via local regex/NER models before hitting the API, and restrict retrieval to authorized vector DB collections.

C. Use fine-tuning to permanently store patient data inside model weights so data never travels over API payload calls.

D. Rely entirely on Claude’s built-in safety filters to automatically detect and ignore PII in incoming payloads.

Correct Answer: B

Why: Standard LLM APIs cannot process encrypted text (base64) natively without unencrypting it first, and fine-tuning with PII creates data leakage risks. Client-side sanitization combined with enterprise ZDR policies ensures strict compliance.

---

Scenario: A legal-tech firm needs to evaluate the accuracy of Claude's legal summaries automatically before deployment. They implement an "LLM-as-a-Judge" pipeline.

Question: Which technique reduces bias and improves reliability when using Claude to grade another LLM's outputs?

A. Provide only the generated summary to the judge model without the original source text to save context tokens.

B. Pass clear rubric criteria, explicit scoring guidelines, and swap the positional order of outputs when comparing two options to avoid position bias.

C. Set the judge model’s temperature to 1.0 to encourage diverse, critical feedback.

D. Ask the judge model to respond with a single word ("Good" or "Bad") without giving a rationale.

Correct Answer: B

Why: Position bias and vague criteria are primary causes of failure in automated evaluation systems. Providing concrete rubrics, chain-of-thought rationale, and position swapping delivers consistent, trustworthy grading.

---

Scenario: An automated database agent executes tool calls against a SQL database. Occasionally, Claude generates a SQL query with a syntax error, causing the database wrapper tool to throw an exception.

Question: What is the recommended resilient pattern to handle this tool execution failure?

A. Catch the exception on the server, drop the conversation turn, and start a completely new session from scratch.

B. Return the exact error message and execution stack trace back to Claude as a tool response result, allowing the model to correct its query in the next turn.

C. Fall back to plain text generation and ask the end-user to write the corrected SQL query manually.

D. Retry the exact same API call 3 times without changing the payload.

Correct Answer: B

Why: Claude is designed to self-correct when given execution context. Returning database error messages directly in the tool result block allows the agent to inspect what went wrong and issue a corrected tool call seamlessly.


---

