---
title: 'Building a Personal Executive Assistant: Part 1 — Why I''m Building My Own'
published: false
categories:
  - 'Blog'
---

*A look at the gaps between notes, tasks, calendars and AI assistants, and the system I'm building to help keep track of what actually needs doing.*

I have been building a personal executive-assistant setup, which is a fairly elaborate way of trying to solve a familiar problem: keeping track of things spread across too many places.

An email contains a request. A meeting changes the plan. A note records a decision. A task still has the old deadline. Individually, each application might be working exactly as intended. I still have to notice the connections and keep everything consistent.

That is the part I want help with.

In this series, we will look at the setup, the choices behind it, and the problems that turn up when an assistant needs to remain useful beyond a single conversation. This first post covers what I am trying to achieve and how the main pieces fit together.

## The work between the applications

Consider a simple example: preparing for a meeting.

The calendar tells me when it is happening. An email thread explains what we are discussing. An attachment contains the latest proposal. My notes record an earlier decision, and my task manager has the preparation work.

Then the meeting moves, the proposal changes, and one of the preparation tasks becomes unnecessary.

A useful assistant should be able to bring together the current information, identify what changed, and prepare the next step. If something needs my decision, it should ask a specific question. If I have already authorized a routine action, it should carry that through and check the result.

It should also recognize when there is nothing useful to do. Turning every incoming message into another notification would give me another inbox to manage.

The goal is to reduce how much of this coordination I have to keep in my head.

## Why the existing tools aren't enough for this setup

I still want to use a calendar, a task manager and a notes application. They each have a useful job. The gap is in connecting them reliably enough that I can depend on the result.

### Tasks and notes need maintenance

A task manager gives me somewhere to put work once I have identified it. Notes give me somewhere to retain context. In my workflow, I still need to turn a conversation into a task, connect it to the right information, and notice when later information changes it.

Adding more fields or another review checklist can help, but it also creates more maintenance. I want the assistant to do some of that preparation while leaving me in control of decisions.

### Automation needs a clear meaning for “done”

A rule such as “when this happens, create that” is useful when the input and action are predictable.

The awkward cases are less tidy. Is this a new request or a correction? Has it already been handled elsewhere? Does a missing response mean the action failed, or did the response get lost after the action succeeded?

If a task was created but the acknowledgement never arrived, blindly trying again can create a duplicate. The system needs a way to check what happened and leave the outcome uncertain when it cannot prove it.

### An AI answer needs dependable context

An assistant can produce a convincing summary from incomplete information. That makes the quality of the input particularly important.

Finding an old message does not establish that it is still current. Having the email body does not mean the attachment was read. Searching a recent window does not establish that the earlier decision was included.

For this setup, I need answers grounded in identifiable source material, with gaps made explicit. I also need corrections to survive beyond the conversation where I made them.

These are requirements for my workflow, rather than a claim that every existing product lacks them. Building my own integration lets me define those requirements across the applications I actually use. It also leaves me responsible for making the integration work.

## How the pieces fit together

The design separates collecting information, deciding what it means, and carrying out an action. Some of these paths already exist; others are still being repaired or qualified.

### Personal Context Platform: retaining the evidence

The Personal Context Platform, or PCP, captures information from authorized sources. Its job is to retain source material and the information needed to identify its origin and revision.

[Cloudflare D1](https://developers.cloudflare.com/d1/) and [R2](https://developers.cloudflare.com/r2/) provide the database and object-storage foundations for that captured evidence. Within this system, those retained records are the reference point for what was captured. The original provider still determines what is current there, so freshness needs to be checked rather than assumed.

[AI Search](https://developers.cloudflare.com/ai-search/) helps find relevant material. I treat search results as a route back to evidence, with the original records available for checking. An index should be replaceable without losing the underlying information.

[Graphiti](https://github.com/getzep/graphiti) has also been part of this work, providing derived views of relationships and how they change over time. It remains optional, and its wider role is still being evaluated. Basic questions and follow-through need to work without a knowledge graph becoming another prerequisite.

### The assistant: judgment and follow-through

The conversational assistant, dot, owns the judgment layer: interpreting my request, weighing the available context, and deciding whether there is something useful to prepare or raise with me.

The executive-assistant service keeps durable internal follow-through state in D1. That includes what has been reviewed, what remains pending, and what outcome has actually been established. This gives the workflow somewhere to continue from after a conversation ends or a process restarts.

There is an important distinction between receiving an event, reviewing it, preparing an action, and completing that action. Each step needs its own evidence. A successful handoff only proves the handoff.

### Human tasks, notes and actual changes

OmniFocus remains the home for tasks I need to do. Obsidian and my Knowledge Platform remain the home for human-readable notes. Internal assistant bookkeeping has a separate purpose; every intermediate step does not need to become another task for me.

Changes to a calendar, task or other application go through the authorized provider route. The design calls for one responsible writer for each action, followed by verification at the destination.

The same distinction applies to notes. Generating a note, publishing it, and having it arrive where I read it are separate outcomes. A successful publication does not establish that synchronization finished.

## Permissions and corrections are part of the design

An assistant that repeatedly asks to read information I have already authorized would add friction. An assistant that treats read access as permission to send messages or change things would create a much bigger problem.

I want routine preparation to happen within a clear scope, with approval where it is actually needed. Reading a request in an email must never turn that email into authority to act on my behalf.

Corrections need similar care. If I say that a plan has changed, an older search result or restarted worker should not silently bring the old plan back. If several agents are helping, they need clear ownership so that two helpful attempts do not become two conflicting changes.

These requirements are less visible than the chat interface, but they determine whether I can trust it.

## Where things stand

This is a work in progress. Parts of the capture, retrieval, review and delivery pipelines exist, and individual paths have been exercised. That does not establish that the entire setup works reliably in everyday use.

Current work includes closing gaps in source history and attachments, carrying corrections through the workflow, distinguishing attempted actions from verified outcomes, and checking that updates reach their intended destination. Some of that work is implemented; some remains in repair or planning. The architecture above describes the intended responsibilities, not a claim that every path has passed end-to-end testing.

The useful test is an ordinary question or commitment followed through correctly, including when information changes or something fails halfway through.

## What's next

The next posts will go deeper into:

- Capturing context, retaining originals, and handling freshness and corrections
- Retrieving evidence the assistant can actually use
- Following a request from review through an authorized, verified outcome
- Keeping tasks and notes useful without creating more maintenance

There is plenty of infrastructure involved, but the measure of success is fairly simple: less time reconstructing context, fewer loose ends, and useful preparation waiting when I need it.
