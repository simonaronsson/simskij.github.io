---
title: The end of "Just ask Sarah"
date: 2026-04-29
tags:
  - Leadership
  - AI
  - Software Health
  - Writing
cover_partial: covers/just-ask-sarah
summary: >
    Documentation used to back up organizational memory. With agents, it becomes execution context. 
description: >
    Every engineering organization has a Sarah.
    Humans can ask her - agents won't.
---

Sarah is the person who knows why a service was split, why an abstraction exists, why a strange constraint shaped the current design, and why the obvious fix was rejected three quarters ago.

That dramatically changes the importance of documentation. Documentation is no longer a courtesy to future humans. Instead, it becomes our primary way of ensuring durable, available, historical context. The same way [boring code is an organizational tell](https://simme.dev/posts/boring-code-is-an-organizational-tell/), the absence of documentation reveals priorities, and agents will amplify them faster than ever.

Imagine an agent opening a new task. It starts going through the codebase to find patterns to help it decide what to do. It looks through the documentation for an ADR explaining why the service is structured the way it is. But there is none, and no spec describing the intent.

The agent will do what it's built to do: find the dominant pattern and extend it. That's not agents being buggy or overly optimistic, but the lack of persisted context propagating.

A decision **was** made - the code is there after all. It just wasn't written down. So, the agent inherits the output without the reasoning, and fills in the blanks the best it can.

## Humans carry state

Organizational memory has always been a social structure just as much as a technical one. It's been living with the engineer who has been on the team for seven years. It's ingrained in the phrase "just ask Sarah", or in the context handed off in one-on-ones and accumulated across stand-ups. Writing turned into a backup. It was always supposed to be there, but rarely maintained to the point where it could actually stand in for the human who held the original context.

This was survivable as long as the cost was deferred, not permanent. The engineer who made the decision last quarter is hopefully still on the team, allowing you to reconstruct the intent by just walking over to their desk and asking. The penalty of the bus factor only ever incurred through staffing changes. And as changes occured, we'd do handovers which, while incomplete, served as plaster over the cracks. 

This meant that the cost of undocumented decisions, while always real, used to be recoverable. At worst it resulted in slower onboarding or an awkward investigation session before touching a brittle component. We've all been in that 30-minute meeting that could just as well have been a two-paragraph ADR, if anyone would have bothered writing the decision down as it happened.

Organizations learned to run on oral tradition because the alternative required discipline that was hard to reward, and not documenting it properly rarely turned into a real problem - at least not immediately. With agents, on the other hand, the full cost of a missing paper trail is paid every time the session terminates.

## Agents don't have Sarah

Agents have whatever is in the context window, and when that closes, it's gone. The next session starts from zero.

Retrieval and agent memory do not change this dramatically. It's convenient that the agent remembers some context between sessions, but it only helps if the organization has produced something worth retrieving. A vector database full of stale tickets and half-written specs, and decisions that only happened in Slack, is not institutional memory. It's just code archaeology with better search.

A human engineer can accumulate unwritten context over time by carrying it forward, updating it, and passing it on. Agents on the other hand can't form the kind of institutional memory that makes tribal knowledge viable. The knowledge that used to live between sessions now has to live *somewhere* that survives the session ending.

What about the codebase? Well, it carries the *result* of decisions: the code that shipped, the abstractions that stuck, and the patterns that propagated, but it doesn't carry the reasoning.

Why did we pick this module boundary and not the other one? What was evaluated and ruled out before the current approach was chosen? What constraint produced a design that looks strange now because the constraint that made us introduce it in the first place no longer exists?

Without that reasoning, an agent encountering the code only has the output to work from. It won't know whether the pattern is intentional or an accident, so it's only option is to treat it as intentional and extend it. And we can't really blame it - it's the correct behavior given what it has access to, even if it might be total nonsense given what actually matters.

An agent sees that every customer integration has its own service, so it creates another one. What it cannot see is that the pattern came from a scaling constraint that happened two years ago, before the shared integration layer existed. The constraint disappeared but the agent treats it as a pattern and keeps reproducing it.

## Intent should be durable

For agents, the value of ADRs, specs, and playbooks is not completeness but introducing friction. They interrupt the default move from "this pattern exists" to "this pattern should be extended". At the same time, it's useful for humans as well.

We use the ADRs to ensure our intent gets captured. We record not just what was decided, but what was considered and why the alternatives lost. This is similar to asking Sarah why the team went for A over B, but without the imperfections of human memory, and available to agents as well. When an agent reads it, the decision isn't an arbitrary historical fact - it's the conclusion of an argument the agent can now evaluate and extend. 

For the longer time horizon, we use the spec. Without it, an agent optimizing a billing pipeline would be inclined to remove what at a glance looks like redundant retry logic. The code didn't say why they were there - only that they were. If asked, Sarah would have told us the retries were a required consistency guarantee rather than an oversight - assuming she still remembered it. But again, agents can't ask Sarah. 

The same goes for the playbook, but now the time horizon is the incident. An agent without context that responds to a memory spike will likely restart the affected pod. The playbook would have told it to check the upstream rate limiter configuration first. While the restart **would** clear the symptom, the playbook clears the cause.

The human SRE handling a production incident at 3am would find it a much needed safety net, but for an agent it might very well be the only structured reasoning available.

None of the three can easily be captured in code, but are trivial to capture as documentation.

## The unwritten decision is still a decision

The absence of documentation is a tell, and it's not neutral. It answers a question about what the organization treats as valuable output. The incentive structure likely treats the time spent writing as overhead, even if it's unintentional.

These organizations often claim that their engineers don't like writing, but in reality they're revealing a priority order: Shipping is more important than explaining. The increment itself is more important than the decision record. The friction was never **really** in the prose.

## Concentration of cost

The cost of not writing concentrates in exactly the place where it's hardest to recover from: the agent that picked up the task and found no ADR, no spec, and was forced to proceed with what it had.

That means writing is now the primary surface through which organizational intent becomes part of the context. 

Any decision that is worth making is also worth recording. Whether that happens or not is an organizational choice with real, tangible consequences. Not for the agents, but for the organization. Agents will just push forward, assuming full context. After all, they never knew there was a Sarah to ask in the first place.