 The harder domains were CLAUDE.md rule hierarchy and specific MCP function-calling conventions: these use Claude-specific naming and structure that general developer experience doesn't help with. Next candidate: do high-volume practice exams targeting weak domains, and drill CLAUDE.md scoping (@-import vs. directory, managed vs. local) plus exact JSON/error-handling


 The trap to watch out for
A few questions are structured in a sneaky way. They’ll ask something like “what would you do at this step of the process” and then follow up with “what would you do before this process.” It’s easy to lose track of which process they’re actually asking you to evaluate your options against, since the actual process under discussion is usually stated in the first line of the question, not repeated in the follow-up. Read the setup line twice before you commit to an answer.


Topics that show up a lot
• Tradeoffs and use cases for MCP and other tool integrations, when to use an API directly versus wrapping it
• Agentic orchestration versus single-shot execution, and how to decide between them for a given workflow
• Few-shot versus single-shot prompting, including some scenarios that get fairly nuanced
• System decomposition, this comes up a lot and is worth being genuinely comfortable with
• Prompt design for production systems, not just “write a good prompt” but how prompts are structured and maintained at scale
• AI governance and how to apply it in practice, not just define it
• Architectural tradeoffs specific to highly secure or regulated environments