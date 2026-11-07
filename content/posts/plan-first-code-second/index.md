---
title: "Plan First, Code Second: How Agentic Coding Changed the Way I Work"
date: 2026-11-07
draft: false
tags: ["AI", "Databricks", "Genie-Code", "Agentic-Coding"]
summary: "I write less code than I used to and spend more time guiding AI tools toward the solution. Here's the workflow I've landed on: start with a conversation instead of code, build up a set of markdown docs that both people and AI tools rely on, and use a few key ideas in the instructions file to tie it all together."
---

My day-to-day work looks very different than it did a year ago. I write a lot less of the code by hand, and more and more, my job is guiding an AI coding tool toward the solution my team and I want to reach.

Every so often, a teammate will look at something Genie Code produced for me and ask how I got it to do that. The honest answer is that there's no trick. It comes down to how I set up the work before any code gets written, and a handful of rules I give the tool about how to use that setup. My workflow is different now than it used to be, and it's certainly different from how a lot of developers I work with do things, so I figured it was worth a blog post.

In my [last post](https://mnorberg-dev.github.io/posts/databricks-orion-app/), I wrote about how context is the bottleneck for AI tools. This post is the everyday version of that idea: how I structure ordinary work so the assistant always has the context it needs.

## Start with a Conversation, Not Code

When I pick up a new body of work, I don't start by writing code. I start by having a conversation with the AI tool.

I open Genie Code and describe what I'm trying to achieve. I lay out the goal, the outcome I want, and anything I already know will be tricky: edge cases, constraints, places where the data is weird, things that have bitten us before. Then I ask it questions. What are the possible approaches? What could go wrong? What am I not thinking about?

Early on, I'll tell it something like: we're not coding yet. Right now we're building a plan.

That plan is the first real artifact of the work. It captures the end goal, the full scope, the context behind it, and the nuances we uncovered while talking it through. Lately my team has been keeping these in a `docs/plans/` folder and committing them to Git, so the plan gets version history like any other file and teammates can review it in a PR before much code even exists.

This approach pays off most when I'm working with something I'm not already an expert in. We get asked to work with unfamiliar tech all the time, and sometimes you just have to figure it out. Starting with a conversation and a plan makes that far easier than it used to be, and it means I can take the work on myself instead of handing it off.

## Why a Plan Matters

The main reason I start with a plan is so the work can survive across multiple sessions.

Anyone who's spent real time with these tools has seen the pattern. A fresh session starts sharp. You haven't used many tokens, the context is clean, and the model is focused. As the conversation grows, quality starts to slip. Eventually the tool compacts the conversation to make room, and after compaction it understands the state of the work noticeably less well.

You can't avoid that, but you can plan around it. If the plan lives in a document instead of in the chat history, then when a session gets long or starts losing the thread, I can open a new one, point it at the plan, and pick up right where I left off. The assistant tracks progress in that document as we go, so the document becomes the memory, not the conversation.

This lines up closely with spec-driven development: describe the end state clearly, then let the agent work toward it. The catch is that it only works with the right amount of context. A vague one-line goal won't get you there. A well-developed plan will.

Plans are working documents, though, so they shouldn't stick around forever. Once the work is done, anything worth keeping, like a new convention or an architecture decision, moves into the doc where it belongs, and the plan gets deleted.

{{< mermaid >}}
flowchart LR
    A["Conversation<br/>with Genie Code"] --> B["Plan in<br/>docs/plans/"]
    B --> C["Build session"]
    C --> D["Update plan<br/>with progress"]
    D -- "context fills up" --> E["Fresh session<br/>reads the plan"]
    E --> C
    D -- "work is done" --> F["Key details move<br/>into the other docs"]
    F --> G["Plan deleted"]
{{< /mermaid >}}

## A Repo Full of Markdown

That last step points to a bigger change. Before agentic coding, most repos had a README, and if you were lucky it was mostly up to date. Today I think a healthy repo has a lot more markdown in it, and each file has a job:

- **README.md**: what the project is and how to get started. Still the front door.
- **DESIGN.md**: the architecture and the reasoning behind it.
- **CONVENTIONS.md**: how we name things, structure code, and handle common patterns. It's what keeps code from several developers looking like it came from one.
- **docs/plans/**: working plans for whatever is in progress.
- **Instructions files**, like CLAUDE.md for Claude Code, AGENTS.md for Codex, or Genie Code's instructions file: the rules the AI tool follows, which I'll get to below.

These get written as we code, not after. And this is the shift I'd most want anyone to take away from this post:

> **Your AI tool is now a consumer of your documentation.** These files aren't docs sitting on a shelf anymore. They're context that gets read at the start of every task.

I think this is one of the keys to good development right now. What an AI tool produces depends heavily on what it reads before it starts, and in a repo like this, what it reads is your docs. For a team, that pays off in two ways.

The first is consistency, even when people use different tools. I lean on Genie Code because it's very good at Databricks, and because it can query the data directly, which gives it a real edge. Some of my teammates prefer Claude Code. That's fine, because the docs are the common ground. Whatever tool someone opens reads the same DESIGN.md and the same CONVENTIONS.md, so the code that comes out looks like it was written by one team instead of five people with five different chat histories.

The second is that the assistant stops tripping over other people's work. When it knows what I'm doing and what my teammates are doing, it's far less likely to add something to my branch that quietly undoes a decision someone else made last week.

## What Goes in the Instructions File

Having the docs isn't enough on its own. The tool also needs to know how to treat them, and that's the job of the instructions file. (If you haven't set one up, I covered the basics of instructions files and skills in an [earlier post](https://mnorberg-dev.github.io/posts/getting-more-out-of-databricks-genie-code/).)

Below are the five ideas I've found make the biggest difference in the quality of what comes out. Think of them as concepts your instructions file should cover, not lines to paste in. The exact wording matters much less than understanding why each one works.

**1. Write the docs for humans, not just for the AI.** Without an explicit instruction, the assistant still produces documents, and they work fine as context for a model. But they're often hard for a person to read: dense, oddly structured, and mostly just taking up space. These docs should be more than something for models to consume. People consume them too, and a new developer should be able to skim them and quickly understand how the code is organized and why. So I tell the assistant to write for a person who is new to the project and doesn't have much context.

**2. Don't say the same thing twice.** Left alone, the assistant will happily explain the same concept in the README, DESIGN.md, CONVENTIONS.md, and a couple of files in `docs/`. Duplicated docs have the same problem as duplicated code: change something in one place and you have to remember to change it everywhere. Developers refactor duplicated code for exactly that reason, and docs are no different. I have a rule telling it to check for overlap, give each topic one home, and link to it from everywhere else.

**3. Keep it short.** This is a newer one for me. I added a line limiting how much the assistant writes, in the docs and in its responses. Without it, I was spending a surprising amount of my day reading markdown. Short output is easier for me to review and easier for the next session to consume.

**4. Read the docs before starting work.** This is the hook that connects everything. At the start of a new body of work, the assistant should read the README and the relevant docs and make sure what it's about to do doesn't contradict decisions that have already been made. It's also what makes CONVENTIONS.md work: the conventions only get applied if the tool knows to read them. Without this rule, all that documentation just sits there.

**5. Keep the docs in sync with the code.** This is the hook going the other direction. When the assistant changes something, it should check whether the docs have drifted and update them if they have. So many projects I've seen started out with good intentions about documentation and then fell behind until the docs were more misleading than helpful. When the tools can write most of the documentation for you, I don't think there's an excuse for that anymore.

Rules 1 through 3 keep the docs worth reading. Rules 4 and 5 form a loop: every session starts by reading the docs and ends by updating them. That loop is what keeps a team of people and a fleet of AI sessions working from the same picture of the project.

Here's an illustration of what this might look like in an instructions file. It isn't my actual file, since mine changes all the time:

```markdown
## Project documentation

Project knowledge lives in markdown: README.md, DESIGN.md,
CONVENTIONS.md, and docs/plans/.

- Before starting new work, read the README, CONVENTIONS.md, and the
  relevant docs. Don't contradict decisions recorded there. If a request
  conflicts with them, say so before writing code.
- Write docs for a human first: a developer new to the project with
  little context. Plain language, short sections, examples.
- Each topic has one home. Before adding content, check whether it
  already exists. Link to it instead of repeating it.
- Be concise. Keep docs and responses as short as they can be while
  still being clear.
- After a change, check whether the docs have drifted from the code and
  update them in the same branch.
- When a plan in docs/plans/ is done, move anything worth keeping into
  the right doc, then delete the plan.
```

My recommendation is not to hand-write a file like this at all. Understand the ideas, then describe them to Genie Code, Claude Code, or whatever you're using, and ask it to write the instructions file for you. You'll end up with something tailored to your own project.

Just keep in mind that the instructions file gets loaded into the context of every request. That makes it valuable real estate, so be selective. Every extra line is something the model carries around on every task, whether it's relevant or not.

## A Databricks-Specific Tip: Map Your Jobs to Your Code

Since most of my development happens in Databricks, there's one more addition to my instructions file that's worked really well.

I keep a small markdown table that maps the jobs and pipelines I own to the Git folder in my home directory where I work on their code. To be clear, this isn't a table in Unity Catalog. It's just a few lines of text in the file. Alongside it is one rule: when a run fails, fix it in my Git folder, never in the deployed copy. Again, this is just an illustration:

```markdown
## Jobs and pipelines I own

| Job / pipeline        | Code (my Git folder)                           |
|-----------------------|------------------------------------------------|
| vendor_ingest_daily   | /Workspace/Users/<me>/data-platform/ingestion  |
| metrics_gold_refresh  | /Workspace/Users/<me>/data-platform/metrics    |

When a job or pipeline run fails, never edit the deployed code owned by
the service principal. Fix it in my Git folder instead: pull the latest
from main, create a branch, make the change, and open a PR.
```

The payoff shows up when something fails. When I ask Genie Code to diagnose the error on a failed run, it knows exactly where the relevant code lives and follows the right process: branch, fix, PR.

Before I added this, I'd seen it go the wrong way. The assistant would notice that the failing job runs as a service principal and try to edit that code directly. If you haven't worked with them, a service principal is a non-human identity that owns and runs jobs in an environment. Like a user, it has its own home directory, and that's where the files actually running in dev, QA, or prod live. Editing those by hand is a software engineering no-no. The fix needs to go through the normal path: push it, merge it, and let your deployment process roll it out to the service principal. Then repair the run.

If your list of jobs grows, this is a good candidate for a skill. Unlike the instructions file, a skill only loads when it's needed. Move the mapping into a skill and leave a single line in the instructions file: when a job or pipeline fails, use the skill to find the project behind it. You get the same behavior without paying for that context on every task.

## How the Work Looks Now

Put it all together and my workflow looks something like this:

1. Start with a conversation, not code. Capture the goal in a plan under `docs/plans/` and commit it.
2. Code against the plan, starting a fresh session whenever the context gets heavy.
3. Let the assistant read the README, DESIGN.md, and CONVENTIONS.md before it changes anything, and update them as it goes.
4. When the work is done, move anything worth keeping into the right doc and delete the plan.

The docs give the assistant context, and the instructions file tells it how to use that context. You need both.

## Wrapping Up

The biggest change in how I work isn't that the AI writes code for me. It's that my job has moved upstream. I spend more of my time thinking through the problem, writing down the plan, and shaping the context, and less time typing out the implementation.

When teammates ask how I got the tool to do something, the answer is almost never a clever prompt. It's that the tool knew what we were building, how we build things, and what everyone else had already done, because it was all written down somewhere it knew to look. Give your assistant a plan, a shared set of docs, and clear rules for using them, and you'll get better code with a team that stays in sync.

---

Here are some resources related to the topics in this post if you want to dig deeper:

- [Genie Code | Databricks Documentation](https://docs.databricks.com/aws/en/genie-code/)
- [Getting More Out of Databricks Genie Code](https://mnorberg-dev.github.io/posts/getting-more-out-of-databricks-genie-code/)