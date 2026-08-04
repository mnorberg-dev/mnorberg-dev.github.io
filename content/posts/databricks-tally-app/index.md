---
title: "Context Is the Bottleneck: Building a Knowledge Assistant for a Legacy Codebase"
date: 2026-07-29
draft: true
tags: ["AI", "Databricks", "MCP", "Data-Engineering"]
summary: "A keynote line about context sent me down the path of building a Databricks app that gives Genie Code full knowledge of a legacy codebase we're reverse-engineering. Here's why I built it, how it works, and why the same pattern can work for any codebase."
---

For the third year running, I was lucky enough to attend the Databricks Data + AI Summit on behalf of Rightworks. This year, one line from the keynote stuck with me. Databricks CEO Ali Ghodsi put it plainly:

> AI doesn't have an intelligence problem. It has a context problem.

His point was simple. The frontier models are already plenty smart, and only getting smarter. The gap between AI that solves your problem and AI that spins its wheels usually isn't intelligence. It's context. Give a capable model what it needs, and it becomes far more effective.

I sat in the convention hall, turning the idea over against my own work. It explained, almost perfectly, why the big project on my plate had been so painful.

## The Problem: Reverse-Engineering a Black Box

For months, my team has been reverse-engineering a legacy application. I can't name it publicly, so I'll call it Orion.

Orion's job was to collect data from a number of different sources, normalize it, and compute a set of metrics on top of it. Our job has been to rebuild that capability on modern infrastructure: collect the same data, normalize it across sources, and reproduce the metrics exactly, as pipelines that run at scale.

This is exactly the kind of work Databricks is built for. In lakehouse terms, the rebuild falls naturally into a medallion architecture. Raw data lands in a bronze layer, gets conformed and normalized in a silver layer, and the metrics are computed in gold. Each stage is an ETL pipeline written in PySpark and Spark SQL, running on Spark's distributed, in-memory engine, so the work spreads across a cluster instead of grinding through a single machine.

Reverse engineering the application proved to be a very difficult task. Most of the difficulty was in understanding how the original system moved data: following the call stack, tracing how values flowed through memory and got transformed at each step, accounting for every precise operation. The sheer volume of code only made it worse.

The language gap was a challenge of its own. Orion was written in Node.js, a perfectly good language but not a natural fit for large-scale data transformation. Code like that walks through data row by row, holding running state as it goes, and that pattern has no clean equivalent in a set-based world. Picture reproducing a loop that iterates over an array and tracks values along the way, when your target is a single SQL query over the whole set at once. The concepts translate loosely. The approach has to change completely.

## Where the AI Assistant Kept Falling Short

Throughout the project I've leaned heavily on a coding assistant, Databricks' Genie Code, and it's been a real time-saver. But it had one persistent blind spot. It couldn't see Orion.

Genie Code had access to our new Databricks code, but not to the legacy codebase we were trying to understand. When I needed to know how Orion handled a particular calculation, my options were to dig through the source myself or paste a snippet into the chat, which only ever gave the model a keyhole view of one file at a time. Each new case meant hours of painstaking code archaeology.

That's the connection that clicked during the keynote. The Orion codebase was the missing context. If the assistant could search and understand the whole thing, the goals, the original intent, the actual implementation, it could ground its answers in real code instead of guessing. That was the piece I set out to build.

## First Attempt: MCP (and Why It Didn't Work)

My first instinct was the simplest one: connect the codebase to Genie Code over MCP. If Orion lived in GitHub, this would have been close to plug-and-play, since there are GitHub MCP connectors that work with the assistant out of the box.

But Orion lives in Azure DevOps, and there's no MCP connector available for it. The easy path was closed, so I had to get more creative.

## The Solution: A Databricks App to Serve the Codebase Over MCP

Databricks apps ship with MCP support out of the box, and they integrate cleanly with Genie Code. Instead of wiring the assistant to source control directly, I built a small Databricks app to serve the codebase, then connected Genie Code to it over MCP.

The app needed to do a few things.

**Make the codebase searchable.** A keyword search over a large, unfamiliar codebase isn't enough, since you rarely know the exact term to look for. Instead, I set up semantic search with Databricks Vector Search. The files were far too large to embed whole, so each one gets split into smaller chunks, and every chunk becomes a vector produced by an embedding model. Those vectors live in a Vector Search index, each tagged with metadata like the path of the file it came from.

The important detail is that the index holds chunks of text, not whole files. When the assistant asks a question in plain language, the search returns the handful of chunks most relevant to it, wording match or not, each with a pointer back to the file it lives in.

I'll admit I was skeptical this would work. Semantic search is built for natural language, and English doesn't map cleanly onto source code. It worked better than I expected. Part of the reason is that code carries more natural language than you'd think, in its comments, function names, and variable names. The embedding models also turn out to be genuinely good at code, matching snippets that are conceptually similar even when the syntax differs. Whatever the reason, retrieval consistently surfaced the right parts of the codebase.

It did more than point me to the right place, too. I could ask how a particular value was calculated, get the logic back in plain language, then ask the assistant to cite the exact file and lines it drew from. That last step is what made the answers trustworthy. Instead of taking a plausible explanation on faith, I could check it against the source every time.

**Serve the raw files.** The index only holds chunks, so the assistant usually needs the full file behind a promising match, and that full text lives somewhere else. The code effectively sits in two places. The raw source is staged in a Unity Catalog volume that maps an ADLS blob container into our environment, separate from the vector index. The app itself runs as isolated compute and doesn't hold the codebase inside it. When the assistant lands on a relevant file, the app reaches out and fetches it on demand, through its service principal and the Databricks Files API, reading straight from the volume.

**Expose a small set of tools over MCP.** From a coding standpoint this part was deliberately simple, just four or five basic utilities: search the codebase semantically, get an overview of the project, read a raw file by path, and so on. Nothing fancy. Just enough for Genie Code to explore Orion the way a person would.

Here's the shape of it:

{{< mermaid >}}
flowchart LR
    A["Genie Code"] -- MCP --> B["Databricks app<br/>(isolated compute)"]
    B -- "1 · semantic search" --> C["Vector Search index<br/>chunked embeddings + file paths"]
    B -- "2 · fetch full file<br/>service principal + Files API" --> D["Unity Catalog volume<br/>(ADLS blob container)<br/>raw source files"]
    subgraph index ["Indexing (build time)"]
        D -- "chunk + embed" --> C
    end
    subgraph future ["Planned extensions"]
        F["Confluence"]
        G["SharePoint"]
    end
    F -- Lakeflow --> D
    G -- Lakeflow --> D
{{< /mermaid >}}

One decision worth calling out: I deliberately didn't build anything to keep the index or the staged code in sync as the source changes. Normally that would be a glaring omission. But the whole point of this project is to retire Orion, so it's a frozen target. Building refresh pipelines for code we're actively replacing would have been effort spent in the wrong direction.

## Keeping Costs Honest

A Databricks app doesn't need to run around the clock, especially one serving a small internal team. I set up a scheduled job to spin it up and down on a sensible cadence rather than leave it running continuously. It's a small thing, but for an internal tool it keeps the cost proportional to the value.

## Teaching the Assistant When to Use It

Building the app was only half the work. The other half was making Genie Code aware of it, so it knew when to reach out over MCP and how to work the results into its reasoning.

I did that with Genie Code's native configuration: a custom instructions file and skills. If you've built these before, this will feel familiar. If you haven't, I put together a [guide on what they are and how to set them up](https://mnorberg-dev.github.io/posts/getting-more-out-of-databricks-genie-code/). Through trial and error I wrote my own, a set of rules that tell Genie Code what Orion is, what we're trying to do, and when to call the app's tools instead of guessing.

Here's something I didn't appreciate at first: that configuration is never really done. For the first week or so I was tweaking those files constantly. Then it converged. After a while the assistant genuinely knew Orion. It reached for the app on its own when a question called for it, and its answers were grounded and consistent instead of the occasional lucky guess. What started as a promising experiment became something I relied on every day, and it saved us an enormous amount of time on the rebuild.

## Where This Goes Next

This was version one, and it deliberately handled a single source of context: the code itself. That was enough to prove the idea. But code is only the first input.

The obvious next step is to pull in everything around it. Databricks' Lakeflow connectors can ingest from Confluence and SharePoint, where a lot of the written knowledge about Orion lives, in design docs and notes from teammates and the original authors. That material is too sprawling for any one person to hold in their head, but it's exactly what an LLM can sift through to find the right context for a question.

Layer those sources together and the tool changes character. Once the assistant can draw on code, documentation, and tribal knowledge at once, and decide for itself which to consult, it stops being a search tool. It becomes a knowledge assistant agent, something you can hold a real conversation with about the whole system.

Step back, and this stops being about one legacy app at all. It's a reusable pattern: point a coding assistant at a codebase it can't otherwise see, give it the tools to search and read that code on demand, then layer in whatever surrounding knowledge exists. You could stand one up for any codebase in the company, and give a new developer a way to get up to speed in days instead of months. Even after Orion is gone, an assistant like this could stick around as institutional memory, a way to answer "how did the old system actually do this?" long after the source has been retired.

## Wrapping Up

The models are good. What's usually missing is context.

If you're stuck on a problem where an AI assistant should be able to help but keeps coming up short, it's worth asking whether the real bottleneck is the model, or whether you just haven't given it the context it needs. For us, closing that gap was the difference between an assistant that guessed and one that knew.

Ghodsi was right. It's a context problem. The good news is that context is something you can actually do something about.

---

Here are some resources related to the technologies covered in this post that I'd recommend if you want to learn more about these concepts:

Sources:

- [Genie Code | Databricks Documentation](https://docs.databricks.com/aws/en/genie-code/)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [Mosaic AI Vector Search | Databricks Documentation](https://docs.databricks.com/aws/en/ai-search/ai-search)
- [Data engineering with Databricks | Databricks Documentation](https://docs.databricks.com/aws/en/data-engineering/)
- [Add a Unity Catalog volume resource to a Databricks app | Databricks Documentation](https://learn.microsoft.com/en-us/azure/databricks/dev-tools/databricks-apps/uc-volumes)