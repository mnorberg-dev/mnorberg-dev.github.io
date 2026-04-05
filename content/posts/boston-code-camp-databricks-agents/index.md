---
title: "Boston Code Camp Recap: Building AI Agents in Databricks"
date: 2026-04-05
summary: "A recap of my session at Boston Code Camp, covering how to build, deploy, and test AI agents in Databricks using the Responses Agent framework. Includes links to the slides and code on GitHub."
draft: false
tags: ["databricks", "ai", "data engineering", "mlflow"]
slug: "boston-code-camp-ai-agents-databricks"
author: "Matthew Norberg"
---

A few weeks ago I gave my first professional talk at Boston Code Camp. The session was called *Your First AI Agent in Databricks: Building, Deploying, and Testing with Confidence*, and it was one of the more rewarding experiences I've had in my career so far.

## What the Session Covered

The talk was aimed at beginner to intermediate practitioners who have probably experimented with AI agents in a notebook but haven't yet figured out how to get one running reliably in production. The core thread running through it: getting an agent to *work* is very different from getting one to *ship*.

We walked through the full lifecycle — defining an agent using the MLflow Responses Agent framework, registering it in Unity Catalog, and deploying it through the Mosaic AI Gateway. From there we looked at two ways to actually use the agent: the AI Playground built into the Databricks UI, and `AI_QUERY`, which lets you call your agent directly from SQL — useful for things like running sentiment analysis on a table of customer reviews as part of a data pipeline.

The second half of the talk was about what happens after you deploy. MLflow tracing gives you full visibility into every request your agent handles, and LLM judges let you evaluate response quality at scale — not just whether the agent ran successfully, but whether it responded appropriately. We also covered the Mosaic AI Gateway's inference tables, where every request gets logged automatically for auditing and analysis.

The final section was on stress testing. The short version: interact with your agent irresponsibly before your users get the chance to. Prompting edge cases, evaluating the results with LLM judges, and tightening your system prompt based on what you find is a much better discovery process than finding out in production.

## Slides and Code

Everything from the talk — the slides and the notebook template — is on GitHub:

**[github.com/mnorberg-dev/first-ai-agent-in-databricks](https://github.com/mnorberg-dev/first-ai-agent-in-databricks)**

The notebook is designed as a starter template, so if you want to follow along with the concepts from the talk you can work through it directly in your own Databricks environment.

---

The conversations after the session were great, and I'm already looking forward to the next Boston Code Camp in November. If you attended and have questions, or you know of a conference where this kind of topic would be a good fit, feel free to reach out.