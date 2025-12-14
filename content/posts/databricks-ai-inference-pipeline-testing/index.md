---
title: "Creating AI Processing Pipelines: A Data-First Approach"
date: 2025-12-14
summary: "Write summary"
draft: false
tags: ["databricks", "ai", "inference-tables"]
categories: ["data engineering"]
slug: "inference-table-processing-tests"
author: "Matthew Norberg"
---

When you deploy an AI agent in Databricks using the Mosaic AI Gateway, one very nice thing happens automatically: every request to your agent, along with the corresponding response, is logged for you. These records are stored in what Databricks calls, an inference table.

At first glance, it feels like you’ve achieved agent observability. Each request and response is stored for you by default, without the developer needing to write any additional logging code.

In practice, though, having the data and being able to use the data are two very different things.

This post is about how to think about AI processing pipelines from a data-first perspective, and why the most important work happens before you write a single line of pipeline code.

## The Inference Table: Valuable, but Not Analysis-Ready

The inference table stores each request and its corresponding response in the request and response columns. Both are stored as strings containing deeply nested JSON.

Inside those JSON blobs is everything you might want to analyze:
- Input, completion, and total token counts
- Prompt and content filter results (especially relevant with Azure OpenAI)
- User or caller identity
- Error information
- MLflow tracing metadata

The inference table captures all of this data, but it isn’t structured for answering questions. As soon as you start asking things like how many tokens are being consumed, which requests are triggering safety or content filters, or why certain requests failed, you quickly realize that the inference table is a logging table, not an analytics table. It’s optimized for completeness, not usability.

For data engineers, this usually means building a Bronze → Silver → Gold pipeline that extracts and normalizes these values into well-typed columns that downstream tools like AI/BI or Genie can consume.

That’s where things get interesting.

## The Real Goal: A Gold-Quality Inference Dataset

What we ultimately want is straightforward:
- One row per logical request
- Stable, well-typed columns
- Predictable null behavior
- Easy aggregation and filtering

More concretely, this means pulling the most important fields out of the raw request and response JSON and promoting them to first-class columns. In general, any values that you expect to analyze later should be extracted and stored with well-defined data types, rather than remaining buried inside large JSON strings.

Token usage is a good example. Instead of parsing token counts out of JSON at query time, it’s far more useful to expose them directly as numeric columns—such as input tokens, output tokens, and total tokens—so they can be easily filtered, aggregated, and monitored over time.

Once you have this kind of structure in place, everything opens up. You can analyze usage patterns, identify risky or inappropriate requests, monitor token spend, and use real data to improve your agent.

But there’s a catch that isn’t obvious at first.

You can’t design that gold table correctly until you understand every shape your inference data can take.

## What the Documentation Doesn't Emphasize

One of the lessons I learned the hard way is that the structure of the inference table is not fixed.

In practice, it varies based on three factors:

1. Which inference endpoint is called (predict or predict_stream)
2. Whether the underlying model throws an error
3. How your agent code handles that error, if one occurs

The first distinction is easy to overlook. A request made to predict returns a single response, while a request made to predict_stream returns content incrementally. As a result, the JSON written to the response column has a different shape depending on which endpoint is used.

The second and third factors are related but distinct. Whether the model throws an error determines that something went wrong. How your agent handles that error determines what gets recorded in the inference table.

### How Agent Error Handling Changes Your Data

In early versions of our agent, we didn’t explicitly handle model errors. When the underlying model rejected a request, the error information was simply passed along to the next layer in our stack. From a control-flow perspective, this worked fine.

From a data perspective, it didn’t.

The inference rows associated with these failures contained very little information. Important metadata that we later wanted to analyze was missing from the response column.

Eventually, we changed the agent to catch these errors and return a custom error message downstream. While this didn’t change the fact that the request failed, it had a significant impact on the data captured in the inference table. The response column now contained much richer information that had been missing in the earlier implementation.

The key point isn’t how you handle errors in your agent. It’s that agent implementation choices directly affect the structure and completeness of your inference data.

## Testing for Data Variability, Not Model Correctness

Up to this point, the focus has been on why inference data varies: different endpoints, different failure modes, and different agent implementations all contribute to changes in the structure of what gets written to the inference table.

This is where testing comes in—but not in the way it’s often discussed.

At this stage, you’re not trying to validate whether the agent gives “good” answers. You’re trying to intentionally generate inference data that covers the full range of outcomes your pipelines will need to handle. This is the essence of a data-first approach to building AI processing pipelines.

The goal of testing here is to populate the inference table with representative rows: successful requests, failed requests, streamed responses, partial responses, and everything in between. Once those rows exist, you can design and build your processing pipelines with confidence, knowing they’ve been exercised against real variability rather than idealized assumptions.

This approach flips the usual order of operations. Instead of writing a pipeline and discovering later that it breaks—or quietly produces bad data—you first create a diverse set of inference records, then build a pipeline that can reliably process all of them into the gold-quality dataset described earlier.

When you reach that point, you don’t just have a working pipeline. You have a robust one, shaped by the actual behaviors of the system rather than by guesswork.

## Thinking in Outcomes, Not Prompts

The key mental shift is to stop thinking in terms of the questions you would normally ask an agent and start thinking in terms of the outcomes produced by the model during inference.

At this stage, you’re not trying to exercise the agent the way a well-behaved user would. You’re trying to exercise it the way your data pipeline will experience it when it's populated with real production data. That often means deliberately asking questions designed to fail, to be blocked, or to behave in unexpected ways. In practice, you almost have to pretend you’re a bad actor and intentionally probe the edges of what the model will and won’t do.

In Databricks, two dimensions matter most:

- Whether you’re calling predict or predict_stream
- Whether the request succeeds, fails, or partially succeeds

When you look at inference through this lens, you can map model behavior into a small, finite set of outcomes that are worth testing explicitly.

The key mental shift is to stop thinking in terms of questions you ask the agent and start thinking in terms of outcomes produced by inference.

## The Five Inference Cases You Should Intentionally Generate

Before writing your processing pipelines, you should make sure your inference table contains all five of the following cases.

1. Predict + Valid Request

    A single response returned all at once, with complete metadata. This becomes your baseline schema.

2. Predict + Blocked Request

    An inappropriate or disallowed prompt that fails immediately.
    The response structure changes, and certain fields may be missing or altered.

3. Predict Stream + Valid Request

    A successful streaming response, delivered in chunks. This will be your baseline schema for streaming requests.
    
4. Predict Stream + Immediately Blocked Request

    A streaming request that fails before any chunks are returned. This is similar to non-streaming requests that cause an error immediately, but have a different schema in the response column.

5. Predict Stream + Partially Blocked Request

    The most subtle case.  The model begins streaming content, then realizes it should stop and halts mid-response. This results in partial output and incomplete metadata.

    If you don’t test this case explicitly, it will eventually find you in production.

## Why This Matters for Your Pipelines

Each of these cases can produce a different response shape. If your pipeline only accounts for the “normal” ones, you’ll either:

- Break when new data arrives, or
- Produce incomplete analytics without realizing it

By deliberately generating all five cases up front, you can design your Bronze, Silver, and Gold tables with confidence that they’ll hold up as usage grows and behavior evolves.

## What This Post Is (and Is Not)

This post is about taking a data-first approach to building AI processing pipelines by intentionally populating the inference table with representative test data. The focus is on what kinds of inference outcomes you need to generate so that your pipelines can be designed to handle real-world variability, rather than idealized assumptions. It is not a guide to writing pipeline code.

If this approach resonates, a natural follow-up is a deeper dive into how to design and implement the actual processing pipelines once you understand the shapes and edge cases present in your inference data.

## Final Thoughts

AI agents don’t just generate answers—they generate data. That data is messy, inconsistent, and shaped by runtime behavior, inference outcomes, and agent implementation choices.

If you start by writing processing pipelines first and only later discover the range of schemas and data shapes that appear in the inference table, you’ll likely end up refactoring, rewriting, and second-guessing your design.

A data-first approach flips that sequence.

Rather than beginning with pipeline code, you start by intentionally populating the inference table with representative outcomes and observing how those outcomes are recorded. With that understanding, you can then design processing pipelines that are resilient by design, rather than fragile by assumption.

Do that, and you make the data easier to work with—not just for your future self, but for data analysts and others downstream who rely on these tables to understand how your AI systems are actually being used.
