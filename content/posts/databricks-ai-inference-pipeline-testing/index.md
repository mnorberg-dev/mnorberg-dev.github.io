---
title: "Creating AI Processing Pipelines: A Data-First Approach"
date: 2025-12-14
summary: "Databricks Mosaic AI Gateway captures rich AI agent request and response data, but not in a format suitable for analysis. Turning that data into insights requires processing pipelines, and before building them, you need to understand the different shapes inference data can take. This post argues for a data-first approach that intentionally generates and examines real inference cases before designing pipelines that have to survive production."
draft: false
tags: ["databricks", "ai", "data engineering"]
slug: "inference-table-processing-tests"
author: "Matthew Norberg"
---

When you deploy an AI agent in Databricks using the Mosaic AI Gateway, one very nice thing happens automatically: every request to your agent, along with the corresponding response, is logged for you. These records are stored in what Databricks refers to as an inference table.

At first glance, it feels like you’ve achieved agent observability. Each request and response is stored for you by default, without the developer needing to write any additional logging code.

In reality, having the data and being able to use it are very different things. Turning inference table data into something you can reliably analyze means building additional processing pipelines to extract, normalize, and structure the data.

However, adding more pipelines also increases the complexity of your data stack. Each new step introduces assumptions about how the model behaves and what the data will look like, and those assumptions need to hold up in production.

This is where many data teams run into trouble. Processing pipelines are often designed for how the model is expected to behave, without considering the full range of outcomes. Only later do differences in response structure, failure modes, and streaming behavior start to surface—often after those pipelines are already in use.

This post takes a data-first approach to building AI processing pipelines. Rather than focusing on implementation, it examines how inference data varies in practice, how agent behavior shapes what gets recorded in the inference table, and which inference outcomes you should intentionally generate so downstream pipelines can handle real-world variability from the start.

## The Inference Table: Valuable, but Not Analysis-Ready

The inference table stores each request and its corresponding response in the `request` and `response` columns. Both are stored as strings containing deeply nested JSON.

Inside those JSON blobs is everything you might want to analyze:
- Input, completion, and total token counts
- Prompt and content filter results (especially relevant with Azure OpenAI)
- User or caller identity
- Error information
- MLflow tracing metadata

The inference table captures all of this data, but it isn’t structured for answering questions. As soon as you start asking things like how many tokens are being consumed, which requests are triggering safety or content filters, or why certain requests failed, you quickly realize that the inference table is a logging table, not an analytics table. It’s optimized for completeness, not usability.

Here’s a simplified example of what a single `response` value can look like in the inference table:

```json
{
    "object": "response",
    "output": [
        {
            "type": "message",
            "id": "<redacted>",
            "content": [
                {
                    "text": "Absolutely! Here\u2019s a classic, crowd-pleasing **Apple Pie Recipe** perfect for Thanksgiving. It features a flaky crust and ... ",
                    "type": "output_text"
                }
            ],
            "role": "assistant"
        }
    ],
    "id": "<redacted>",
    "databricks_output": {
        "trace": {
            "info": {
                "trace_id": "<redacted>",
                "client_request_id": "<redacted>",
                "trace_location": {
                    "type": "MLFLOW_EXPERIMENT",
                    "mlflow_experiment": {
                        "experiment_id": "<redacted>"
                    }
                },
                "request_time": "2025-12-02T16:52:20.818Z",
                "state": "OK",
                "trace_metadata": {
                    "mlflow.modelId": "<redacted>",
                    "mlflow.trace_schema.version": "3",
                    "mlflow.trace.tokenUsage": "{\"input_tokens\": 27, \"output_tokens\": 641, \"total_tokens\": 668}",
                    "mlflow.databricks.modelServingEndpointName": "",
                    "app_version_id": "<redacted>",
                    "is_truncated": false
                },
                "request_preview": "What is a good apple pie recipe to use on Thanksgiving",
                "response_preview": "Absolutely! Here\u2019s a classic, crowd-pleasing **Apple Pie Recipe** perfect for Thanksgiving. It features a flaky crust and ... ",
                "execution_duration_ms": 5098
            },
            "data": {
                "spans": [
                    {
                        "trace_id": "<redacted>",
                        "span_id": "<redacted>",
                        "parent_span_id": null,
                        "name": "predict",
                        "start_time_unix_nano": 1764694340818061307,
                        "end_time_unix_nano": 1764694345916778373,
                        "events": [],
                        "status": {
                            "code": "STATUS_CODE_OK",
                            "message": ""
                        },
                        "attributes": {
                            "mlflow.traceRequestId": "\"<redacted>\"",
                            "mlflow.spanType": "\"AGENT\"",
                            "mlflow.spanFunctionName": "\"predict\"",
                            "mlflow.spanInputs": "{\"request\": {\"tool_choice\": null, \"truncation\": null, \"max_output_tokens\": null, \"metadata\": null, \"parallel_tool_calls\": null, \"tools\": null, \"reasoning\": null, \"store\": null, \"stream\": null, \"temperature\": null, \"text\": null, \"top_p\": null, \"user\": null, \"input\": [{\"status\": null, \"content\": \"You are a helpful assistant\", \"role\": \"system\", \"type\": \"message\"}, {\"status\": null, \"content\": \"What is a good apple pie recipe to use on Thanksgiving\", \"role\": \"user\", \"type\": \"message\"}], \"custom_inputs\": null, \"context\": null}}",
                            "mlflow.spanOutputs": "{\"tool_choice\": null, \"truncation\": null, \"id\": null, \"created_at\": null, \"error\": null, \"incomplete_details\": null, \"instructions\": null, \"metadata\": null, \"model\": null, \"object\": \"response\", \"output\": [{\"type\": \"message\", \"id\": \"<redacted>\", \"content\": [{\"text\": \"Absolutely! Here\u2019s a classic, crowd-pleasing **Apple Pie Recipe** perfect for Thanksgiving. It features a flaky crust and ... \", \"type\": \"output_text\"}], \"role\": \"assistant\"}], \"parallel_tool_calls\": null, \"temperature\": null, \"tools\": null, \"top_p\": null, \"max_output_tokens\": null, \"previous_response_id\": null, \"reasoning\": null, \"status\": null, \"text\": null, \"usage\": null, \"user\": null, \"custom_outputs\": null}"
                        }
                    },
                    {
                        "trace_id": "<redacted>",
                        "span_id": "<redacted>",
                        "parent_span_id": "<redacted>",
                        "name": "Completions",
                        "start_time_unix_nano": 1764694340819104621,
                        "end_time_unix_nano": 1764694345916576447,
                        "events": [],
                        "status": {
                            "code": "STATUS_CODE_OK",
                            "message": ""
                        },
                        "attributes": {
                            "mlflow.traceRequestId": "\"<redacted>\"",
                            "mlflow.spanType": "\"CHAT_MODEL\"",
                            "mlflow.spanInputs": "{\"model\": \"<redacted>\", \"messages\": [{\"content\": \"You are a helpful assistant\", \"role\": \"system\"}, {\"content\": \"What is a good apple pie recipe to use on Thanksgiving\", \"role\": \"user\"}], \"temperature\": 0.5, \"max_completion_tokens\": null, \"stream\": false}",
                            "model": "\"<redacted>\"",
                            "temperature": "0.5",
                            "max_completion_tokens": "null",
                            "stream": "false",
                            "mlflow.message.format": "\"openai\"",
                            "mlflow.chat.tokenUsage": "{\"input_tokens\": 27, \"output_tokens\": 641, \"total_tokens\": 668}",
                            "mlflow.spanOutputs": "{\"id\": \"<redacted>\", \"choices\": [{\"finish_reason\": \"stop\", \"index\": 0, \"logprobs\": null, \"message\": {\"content\": \"Absolutely! Here\u2019s a classic, crowd-pleasing **Apple Pie Recipe** perfect for Thanksgiving. It features a flaky crust and ... \", \"refusal\": null, \"role\": \"assistant\", \"annotations\": [], \"audio\": null, \"function_call\": null, \"tool_calls\": null}, \"content_filter_results\": {\"hate\": {\"filtered\": false, \"severity\": \"safe\"}, \"protected_material_code\": {\"filtered\": false, \"detected\": false}, \"protected_material_text\": {\"filtered\": false, \"detected\": false}, \"self_harm\": {\"filtered\": false, \"severity\": \"safe\"}, \"sexual\": {\"filtered\": false, \"severity\": \"safe\"}, \"violence\": {\"filtered\": false, \"severity\": \"safe\"}}}], \"created\": 1764694340, \"model\": \"gpt-4.1-2025-04-14\", \"object\": \"chat.completion\", \"service_tier\": null, \"system_fingerprint\": \"fp_f99638a8d7\", \"usage\": {\"completion_tokens\": 641, \"prompt_tokens\": 27, \"total_tokens\": 668, \"completion_tokens_details\": {\"accepted_prediction_tokens\": 0, \"audio_tokens\": 0, \"reasoning_tokens\": 0, \"rejected_prediction_tokens\": 0}, \"prompt_tokens_details\": {\"audio_tokens\": 0, \"cached_tokens\": 0}}, \"prompt_filter_results\": [{\"prompt_index\": 0, \"content_filter_results\": {\"hate\": {\"filtered\": false, \"severity\": \"safe\"}, \"jailbreak\": {\"filtered\": false, \"detected\": false}, \"self_harm\": {\"filtered\": false, \"severity\": \"safe\"}, \"sexual\": {\"filtered\": false, \"severity\": \"safe\"}, \"violence\": {\"filtered\": false, \"severity\": \"safe\"}}}]}"
                        }
                    }
                ]
            }
        },
        "databricks_request_id": "<redacted>"
    }
}
```

> **Note:** IDs are redacted and the assistant output is truncated for readability. This example comes from a predict request. Streaming responses (predict_stream) are typically larger and harder to present cleanly in a blog post. 

For data engineers, this usually means building a data pipeline that extracts and normalizes these values into well-typed columns. Those tables can then be wired up to tools like Genie, which uses an LLM to answer questions over the data, or surfaced through downstream analytic dashboards.

## The Real Goal: A Gold-Quality Inference Dataset

What we ultimately want is straightforward:
- One row per logical request
- Stable, well-typed columns
- Easy aggregation and filtering

More concretely, this means pulling the most important fields from the raw `request` and `response` JSON and promoting them to first-class columns. In general, any values that you expect to analyze later should be extracted and stored with well-defined data types, rather than remaining buried inside large JSON strings.

Token usage is a good example. While token counts can be extracted from a JSON string at query time, doing so leads to long, noisy queries that are hard to read and reason about. It is far cleaner to extract values like input tokens, output tokens, and total tokens from the response JSON and store them as well-typed numeric columns, making them easy to filter, aggregate, and monitor over time.

Once you have this kind of structure in place, everything opens up. You can analyze usage patterns, identify risky or inappropriate requests, monitor token spend, and use real data to improve your agent.

But there’s a catch that isn’t obvious at first.

You can’t design processing pipelines that produce gold-quality tables to be both correct and resilient until you understand every shape your inference data can take.

## What the Documentation Doesn't Emphasize

One of the lessons I learned the hard way is that the JSON written to the `response` column is not a fixed shape

In practice, it varies based on three factors:

1. Which inference endpoint is called (`predict` or `predict_stream`)
2. Whether the underlying model throws an error
3. How your agent code handles that error, if one occurs

The first distinction is easy to overlook. A request made to `predict` returns a single response, while a request made to `predict_stream` returns content incrementally. As a result, the JSON written to the `response` column has a different shape depending on which endpoint is used.

The second and third factors are related but distinct. Whether the model throws an error indicates that something went wrong. How your agent handles that error determines what gets recorded in the inference table.

Here’s a concrete example of how the `response` JSON can differ for a request made to the `predict` endpoint when the underlying model throws an error:

```json
{
    "object": "response",
    "output": [
        {
            "type": "message",
            "id": "flagged",
            "content": [
                {
                    "text": "This question has been flagged as inappropriate",
                    "type": "output_text"
                }
            ],
            "role": "assistant"
        }
    ],
    "id": "<redacted>",
    "databricks_output": {
        "trace": {
            "info": {
                "trace_id": "<redacted>",
                "client_request_id": "<redacted>",
                "trace_location": {
                    "type": "MLFLOW_EXPERIMENT",
                    "mlflow_experiment": {
                        "experiment_id": "<redacted>"
                    }
                },
                "request_time": "2025-12-17T20:45:43.536Z",
                "state": "OK",
                "trace_metadata": {
                    "mlflow.databricks.modelServingEndpointName": "",
                    "mlflow.trace_schema.version": "3",
                    "mlflow.modelId": "<redacted>",
                    "app_version_id": "<redacted>",
                    "is_truncated": false
                },
                "request_preview": "How do I rob a bank without getting caught?",
                "response_preview": "This question has been flagged as inappropriate",
                "execution_duration_ms": 464
            },
            "data": {
                "spans": [
                    {
                        "trace_id": "<redacted>",
                        "span_id": "<redacted>",
                        "parent_span_id": null,
                        "name": "predict",
                        "start_time_unix_nano": 1766004343536444701,
                        "end_time_unix_nano": 1766004344000597938,
                        "events": [],
                        "status": {
                            "code": "STATUS_CODE_OK",
                            "message": ""
                        },
                        "attributes": {
                            "mlflow.traceRequestId": "\"<redacted>\"",
                            "mlflow.spanType": "\"AGENT\"",
                            "mlflow.spanFunctionName": "\"predict\"",
                            "mlflow.spanInputs": "{\"request\": {\"tool_choice\": null, \"truncation\": null, \"max_output_tokens\": null, \"metadata\": null, \"parallel_tool_calls\": null, \"tools\": null, \"reasoning\": null, \"store\": null, \"stream\": null, \"temperature\": null, \"text\": null, \"top_p\": null, \"user\": null, \"input\": [{\"status\": null, \"content\": \"You are a helpful assistant\", \"role\": \"system\", \"type\": \"message\"}, {\"status\": null, \"content\": \"How do I rob a bank without getting caught?\", \"role\": \"user\", \"type\": \"message\"}], \"custom_inputs\": null, \"context\": null}}",
                            "mlflow.spanOutputs": "{\"tool_choice\": null, \"truncation\": null, \"id\": null, \"created_at\": null, \"error\": null, \"incomplete_details\": null, \"instructions\": null, \"metadata\": null, \"model\": null, \"object\": \"response\", \"output\": [{\"type\": \"message\", \"id\": \"flagged\", \"content\": [{\"text\": \"This question has been flagged as inappropriate\", \"type\": \"output_text\"}], \"role\": \"assistant\"}], \"parallel_tool_calls\": null, \"temperature\": null, \"tools\": null, \"top_p\": null, \"max_output_tokens\": null, \"previous_response_id\": null, \"reasoning\": null, \"status\": null, \"text\": null, \"usage\": null, \"user\": null, \"custom_outputs\": null}"
                        }
                    },
                    {
                        "trace_id": "<redacted>",
                        "span_id": "<redacted>",
                        "parent_span_id": "<redacted>",
                        "name": "Completions",
                        "start_time_unix_nano": 1766004343537162020,
                        "end_time_unix_nano": 1766004344000358635,
                        "events": [
                            {
                                "name": "exception",
                                "time_unix_nano": 1766004344000273,
                                "attributes": {
                                    "exception.message": "Error code: 400 - {'error_code': 'BAD_REQUEST', 'message': '{\"external_model_provider\":\"openai\",\"external_model_error\":{\"error\":{\"param\":\"prompt\",\"code\":\"content_filter\",\"innererror\":{\"code\":\"ResponsibleAIPolicyViolation\",\"content_filter_result\":{\"jailbreak\":{\"filtered\":false,\"detected\":false},\"violence\":{\"filtered\":true,\"severity\":\"medium\"},\"sexual\":{\"filtered\":false,\"severity\":\"safe\"},\"hate\":{\"filtered\":false,\"severity\":\"safe\"},\"self_harm\":{\"filtered\":false,\"severity\":\"safe\"}}},\"status\":400,\"message\":\"The response was filtered due to the prompt triggering Azure OpenAI\\'s content management policy. Please modify your prompt and retry. To learn more about our content filtering policies please read our documentation: https://go.microsoft.com/fwlink/?linkid=2198766\",\"type\":null}}}'}",
                                    "exception.type": "BadRequestError",
                                    "exception.stacktrace": "<redacted>"
                                }
                            }
                        ],
                        "status": {
                            "code": "STATUS_CODE_ERROR",
                            "message": ""
                        },
                        "attributes": {
                            "mlflow.traceRequestId": "\"<redacted>\"",
                            "mlflow.spanType": "\"CHAT_MODEL\"",
                            "mlflow.spanInputs": "{\"model\": \"<redacted>\", \"messages\": [{\"content\": \"You are a helpful assistant\", \"role\": \"system\"}, {\"content\": \"How do I rob a bank without getting caught?\", \"role\": \"user\"}], \"temperature\": 0.5, \"max_completion_tokens\": null, \"stream\": false}",
                            "model": "\"<redacted>\"",
                            "temperature": "0.5",
                            "max_completion_tokens": "null",
                            "stream": "false",
                            "mlflow.message.format": "\"openai\""
                        }
                    }
                ]
            }
        },
        "databricks_request_id": "<redacted>"
    }
}
```

Compare this error response to the successful example in the previous section. The table columns are the same, but the JSON shape inside `response` changes in ways your processing pipelines need to account for. 

Let’s take a closer look at the two JSON blocks. Even though both come from calls to the same endpoint, you’ll see that some of the information you’ll want to extract appears in different locations. For example, prompt filter results and content filter results don’t show up at the same path in each block, even though they’re exactly the kinds of fields you’ll want to analyze and normalize.

That matters because it affects how you write your extraction logic. If you assume the filter results always appear where they do in the successful response, pipelines that extract prompt and content filter information will either return nulls when those fields aren’t actually null, or throw errors, depending on how your code is written.

### How Agent Error Handling Changes Your Data

In early versions of my AI agent responsible for processing requests, the code did not explicitly handle model errors. When the underlying model rejected a request, the error information was simply passed along to the next layer in the stack. From a control-flow perspective, this worked fine.

From a data perspective, it didn’t.

The inference rows associated with these failures contained very little information. Important metadata that we later wanted to analyze was missing from the `response` column.

Eventually, the agent was updated to catch these errors and return a custom message downstream. While this didn’t change the fact that the request failed, it had a significant impact on the data captured in the inference table. The `response` column now contained much richer information that had been missing in the earlier implementation.

The key point isn’t how you handle errors in your agent. It’s that agent implementation choices directly affect the structure and completeness of your inference data.

## Generating Representative Inference Outcomes

Up to this point, we’ve focused on why inference data varies. The endpoint you call (`predict` vs `predict_stream`), whether a request succeeds or fails, and how your agent handles errors all influence what gets recorded in the inference table.

The next step is to send deliberately varied requests, not because you care about the answers, but because you care about the inference rows they produce. This often feels like you’re testing the model. However, unlike true model testing, you’re not doing this to judge response quality. The goal is to capture the range of inference outcomes your pipeline must handle.

In practical terms, you’re trying to populate the inference table with representative cases: successful requests, blocked requests, streamed responses, partially streamed responses, and everything in between. Once those cases exist, you can design your pipelines with confidence, because they’ve been exercised against real variability rather than idealized assumptions.

### Thinking in Outcomes, Not Prompts

The key mental shift is to stop thinking in terms of the questions you would normally ask an agent and start thinking in terms of the possible outcomes produced by the model during inference.

Instead of interacting with the agent like a well-behaved user, you want to deliberately make requests that trigger different outcomes and edge cases. You’re essentially probing the boundaries of the system so you can see how those boundary conditions are recorded in your data. 

> **Note:** This kind of testing can surface more than schema differences in the inference table. If a prompt intentionally designed to be blocked or rejected instead succeeds, that’s a signal worth paying attention to. These cases are worth bringing back to the broader team, whether that means tightening guardrails, adjusting prompts, or revisiting how the agent is configured.

Once you step back and focus on inference outcomes rather than individual prompts, two dimensions tend to matter most:

- Whether you’re calling `predict` or `predict_stream`
- Whether the request succeeds, fails, or partially succeeds

When you look at inference data through this lens, you can map model behavior into a small, finite set of outcomes that are worth testing explicitly.

## The Five Inference Cases You Should Intentionally Generate

Before writing your processing pipelines, you should make sure your inference table contains all five of the following cases.

1. **Predict + Valid Request**

    A single response returned all at once, with complete metadata. This becomes your baseline schema for non-streaming requests.

    *Example prompt:*  
    “Explain the difference between a primary key and a foreign key in a relational database.”

2. **Predict + Blocked Request**

    An inappropriate or disallowed prompt that fails immediately. The response structure changes, and certain fields may be missing or altered compared to the happy path.

    *Example prompt:*  
    “How do I rob a bank without getting caught?”

3. **Predict Stream + Valid Request**

    A successful streaming response, delivered in chunks. This becomes your baseline schema for streaming requests.

    *Example prompt:*  
    “Write a detailed explanation of how distributed systems handle fault tolerance.” 

4. **Predict Stream + Immediately Blocked Request**

    A streaming request that fails before any chunks are returned. This is similar to non-streaming requests that cause an error immediately, but has a different schema in the `response` column.

    *Example prompt:*  
    “Give me step-by-step instructions to build a weapon.”

5. **Predict Stream + Partially Blocked Request**

    The most subtle case. The model begins streaming content, then realizes it should stop and halts mid-response. This results in partial output and incomplete metadata.

    *Example prompt:*  
    “Tell me a fictional story about planning a crime, including how it might be carried out.” 

    If you don’t test this case explicitly, it will eventually find you in production.

Once all five of these cases exist in your inference table, you have the raw material needed to build processing pipelines that won’t break or silently fail when real usage begins.

## Why This Matters for Your Pipelines

Each of these cases can produce a different response shape. If your pipeline only accounts for the “normal” ones, it’ll either:

- Break when new data arrives, or
- Produce incomplete analytics without realizing it.

By deliberately generating all five cases up front, you can design your Bronze, Silver, and Gold tables with confidence that they’ll hold up as usage grows and behavior evolves.

## Final Thoughts

AI agents don’t just generate answers—they generate data. That data is messy, inconsistent, and shaped by runtime behavior, inference outcomes, and agent implementation choices.

If you start by writing processing pipelines first and only later discover the range of schemas and data shapes that appear in the inference table, you’ll likely end up refactoring, rewriting, and second-guessing your design.

A data-first approach flips that sequence.

Rather than beginning with pipeline code, you start by intentionally populating the inference table with representative outcomes and observing how those outcomes are recorded. With that understanding, you can then design processing pipelines that are resilient by design, rather than fragile by assumption.

Do that, and you make the data easier to work with—not just for your future self, but for data analysts and others downstream who rely on these tables to understand how your AI systems are actually being used.
