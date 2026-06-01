---
title: "What happens if you give an AI agent a home instead of a session?"
date: 2026-04-11
tags:
  - AI
  - Autonomous Agents
cover_partial: covers/ai-agent-home
summary: |
  My design goals for building an autonomous AI agent, rather than limiting it to
  interactive prompting session-by-session.
description: |
  Up until about a month ago, my interactions with AI had been the same as most others: you open 
  a window, type something, get a response and some action, and then you iterate until the work is 
  done for the time being and you close the window. Next time you launch the agent, it won't 
  remember you. It won't have any opinions about what you should work on. It waits for instructions, 
  and does (in the best case) what you ask of it.
---

I wanted to accomplish something different. I wanted an agent that delivered value not only when I wake it up with a query, but also on its own.

## Giving agents autonomy

The question of what happens if you give an AI agent agency and anatomy is intriguing, and completely horrifying at the same time. Things that I would previously have to do myself, now being executed by an agent - without my supervision.

Being pinged by my personal agent, telling me that it sees clear signs that I should buy a certain stock, or telling me it solved a bug in one of the projects we collaborate on really is exciting.

It's by no means unique or novel: OpenClaw has made the headlines for months now, and rightfully so - what they've managed to accomplish is truly groundbreaking. It's impressive - it truly is - it also has the problem of most projects that have the goal of being generalized - it's ineffective, to say the least.

You see people all over the internet blowing through a full Claude Max subscription in a few minutes without actually accomplishing anything. To be fair, that's likely more a testament to their inefficiency than the system itself, but it also made me motivated enough to want to build something on my own.

## Traditional systems vs. all-in agentic workflows

In a traditional system you would model a system's behavior as code. You'd create concrete execution paths, conditional branches and so on. Essentially, `if X then y`. If something goes wrong, you put a break point in place and you run again - explicit and easy to inspect.

In OpenClaw, as an example, it's the exact opposite. A minimal agent, pi, sits at the center. It's intentionally bare-bones, lacking MCP support, and even basic tools for searching the code base or showing the git history.

Instead, the expectation is that most of the heavy lifting is handled by the model, using bash. This is really cool, especially from a philosophical standpoint, but it also has clear disadvantages: when the abstraction breaks, you'll find yourself debugging two systems rather than one. And you'll realize that the bigger of the two is completely opaque.

## Conduit

```
2026-03-05 22:00:14 [INFO] Embla just joined the server.
```

A late night, about a month ago, as I was sitting and working on something with my AI agent - Embla. Yes, I'm silly like that: I give inanimate things names, even statistical models, and I spend a lot of time infusing them with both personality and traits.

I'd been working on an agent chief of staff for a while, admittedly with limited success until that point. I started thinking about how I could make Embla more proactive, as I was getting much better results from her than from the agent chief of staff.

So we started to sketch out the system that would become Conduit. I had multiple things in mind here:

### Autonomous, always-on behavior

I did not want to have to wake the agent up and tell it what needed to be done. I wanted it to be always on, continuously going over the current state of the land, and proactively suggest improvements, summarize work that had been progressing, and generally just keep track of the things that I want to keep track of.

### Code where it makes sense

Even in the era of agentic workflows, quite a huge portion of work still makes sense to do with traditional, deterministic code, rather than the opaque blackbox that is running a prompt through a statistical model. I wanted to ensure that everything that would make more sense to run as code would be run that way, and for it not to turn into a huge pile of prompts-all-the-way-down. Tasks like classification, ensuring work completion, and state consistency do not really warrant using a model. In my experience it's both less brittle and more predictable when expressed as code.

### Remembers and learns

I also wanted persistent memory, making the agent able to remember previous conversations, refer to prior lessons learned, and adapting to the work we do as more is learned. `CLAUDE.md`, or `SOUL.md`, is a great start, and you'll get quite far with nothing else, but I wanted more.

### Multi-agent workflows

```
2026-03-12 20:30:12 [INFO] Six other agents joined the server.
```

This was a huge deal to me. I wanted the system to utilize multi-agent workflows. Not as sub-agents, which was the cool new thing in Claude code back then, but as independent, autonomous agents with specialized roles and responsibilities - much like a traditional human engineering team.

Instead of the coordinator having to spawn session-scoped sub-agents and wait for them to complete, I wanted the coordinator to remain available while the agents act independently, only interfering when there is need.

These human engineering teams also served as a blueprint for how I wanted to model the workflow. I wanted to use the kanban method, break work down into manageable tightly scoped tasks, distribute them over the team and have them complete their individual piece, reviewing each other and finally committing the work when it met the agreed-upon quality standard and acceptance criteria.

## What's next?

We're still building it. But it's already one of the most interesting pieces of infrastructure I've ever put together — not because of what it can do, but because of the fact that it's built with exactly one user in mind: me. It's already proving useful for my passion projects, research, planning, and just making my digital life smarter in general.

The next time I write about Conduit, I'll walk you through some of the architectural decisions made and the capabilities we've built so far.
