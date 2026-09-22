## Glossary

Champion-per-department rollout - An adoption pattern that enables one champion per team first to prove the workflow, then extend adoption to others.

Runbook - A captured set of known symptom-to-cause-to-action paths that lets a team resolve recurring operational issues without the Architect.

Spend posture - The model defaults, model allowlists and restrictions, effort guidance, and spend, rate, and per-user caps set as part of team configuration keep consumption within bounds.

## Getting people productive with Claude

How to help you get real work done with Claude?

You can configure Claude for yourself in minutes; however, configuring it for a team is different.
The three topics run in order: set up the environment, raise the daily workflow, and keep the system healthy.

Team setup - helps the team get the environment, the reusable assets, and the spend posture right before anyone logs in. 
Developer workflows - raise the bar on how the team works day to day without lowering the bar on quality.
Operational support -is what you do when the team encounters something unexpected within the system.

The four team-setup decisions an Architect owns:
* environment, 
* rollout, 
* skills distribution, 
* and spend.

A project-level baseline: a shared CLAUDE.md, an agreed set of tools and MCP servers, and a permission posture.

Team adoption rarely succeeds as a single all-hands switch-on. The pattern that works best is identifying a champion per department.

The hard part is distribution governance: who can reach, update, and revoke each shared asset. Decide this on a per asset basis.

A shared skill needs versioning, group targeting, or rollback, distribute it inside an organization-managed plugin and identify an owner.

## Skills distribution

Repeated procedures that usually live in people's heads become Skills 
Skill packages are repeatable procedures that appear as versioned, reusable units. 

There are four ways to deploy a Skill to a team, and they differ in who can access it and how much control you retain: 
* An owner-provisioned Skill, uploaded under Organization settings > Skills, becomes available to everyone in the organization at once. Example: A capability that genuinely should be available to every member, with no need for versioning or rollback.
* When a Skill should reach only select members, you bundle one or more Skills into a plugin and assign that plugin to a group. 
Example: A compliance-review procedure every department must run identically, that must be centrally updatable and roll-back-able.
* Claude Code project Skills: filesystem artifacts that live in the project repository (.claude/skills/), so they version with the repository itself and are scoped to the projects that carry them. 
* API Skills, called programmatically by the partner's own products.

A plugin provides org and group targeting plus version-controlled updates and rollback. 
An org-provisioned Skill would reach everyone but offers no versioning or rollback path.

## Spend posture

Admins should set:
* model defaults (which model a session starts on), 
* model allowlists and restrictions (which models the team may switch to), 
* effort guidance (how hard the model works on a task),
* per-user spend caps that keep consumption within bounds.

## Integrate assistance into the workflow

Providing a team with access is not adoption; you need to configure for real enablement within current workflows.

Architect's job is to find where AI assistance has the opportunity to remove real friction and improve the overall process. 
Good practice travels with the tooling rather than depending on who happens to be in the room.

The team maybe using Claude as a question-answering box and never advances to the higher-value workflows such as tool use, repository-aware assistance, packaged skills.

## Diligence

Diligence - taking responsibility for what we do with AI and how we do it.

Developers must take responsibility for verifying the AI generated outputs.

The concrete deliverable that diligence produces is a verification checklist: the explicit set of checks an AI-generated output must pass before it reaches production.
The checklist should include questions that address all four dimensions of verification: 
* Correctness: Tests exist and pass, and the behavior matches the stated requirement including edge cases.
* Security: No secrets in code; inputs are validated; any tools or external calls use least-privilege access.
* Maintainability: The code reads clearly, follows team conventions, and contains no unexplained complexity.
* Human understanding: The developer submitting the change can explain what the code does and why, including how it handles the inputs it was not explicitly tested against.

Can the person merging PR explain what it does and why? 
AI-generated code must be kept to the same review standard as hand-written code.

When speed had quietly replaced understanding, this is exactly the judgment erosion diligence exists to catch.

## Support

When an operational issue lands, the team usually identifies a symptom, not a cause. 
Many operational symptoms trace to a small set of architectural causes. Identifying these allows the team to reason clearly from what they see to where to look.

Examples:
Output quality degraded gradually, but there was no code change	- A model or prompt change, or retrieval drift as the corpus grew.	
Latency spiked - Context size grew, a tool got slow, or a cache stopped hitting.	
Intermittent tool failures - Authorization, rate limits, or an unhandled error path.	
Cost rose without a usage change - Model tier crept up, or caching regressed.

Frequent failure mode is a slow degradation no one connects to a cause.

