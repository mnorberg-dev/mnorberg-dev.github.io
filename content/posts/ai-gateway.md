---
title: "Databricks AI Gateway Guide"
date: 2025-10-05
draft: false
tags: ["databricks", "ai", "tracing"]
categories: ["engineering"]
slug: "ai-gateway"
author: "Matthew Norberg"
---

Over the past month, I’ve been deep in the weeds experimenting with Mosaic AI Gateway in Databricks — configuring endpoints, debugging integrations, and figuring out how all the pieces fit together. After plenty of trial and error, I wanted to share what I’ve learned so other developers can get up and running faster.

If you’ve explored Databricks’ newer AI features, you’ve probably noticed that the documentation, while thorough, can feel circular and fragmented. You often need to jump between Databricks docs, MLflow, and even OpenAI or Meta references, depending on your base model. It’s easy to lose the thread when trying to connect everything into a working system.

This post is my attempt to bridge those gaps with a concrete, end-to-end example — the kind I wish I’d had when I started.

In short: it’s a practical guide for developers who want to create a Databricks AI Gateway endpoint with tracing enabled, without getting lost in a maze of configuration steps and scattered documentation.

## What is Mosaic AI Gateway?

Mosaic AI Gateway is Databricks’ unified interface for managing access to large language models (LLMs) and AI endpoints. It serves as a control layer between your applications and the models they call — helping teams standardize how they deploy, monitor, and secure their AI workloads.

Out of the box, Mosaic AI Gateway provides several powerful features:

- Permission and rate limiting
- Payload logging
- Usage tracking
- AI Guardrails
- Fallbacks
- Traffic splitting

Together, these capabilities make it easier to build production-ready AI systems that are safe, trackable, and cost-efficient.

But there’s one critical capability missing — and it’s the focus of this post: MLflow Tracing.

If you’re new to tracing, think of it as logging with context. A trace doesn’t just record the request and response; it also captures the intermediate steps and outputs along the way. When something breaks, those traces are invaluable for understanding what actually happened inside your model pipeline.

So the question becomes: how do you build a Mosaic AI Gateway endpoint that captures traces for each request?

That’s exactly the problem I set out to solve — creating a Databricks AI Gateway endpoint that generates a trace for every call.

To save you time, I’ll start by showing the solution up front. But the rest of this post takes a more narrative approach — walking through my learning process, the documentation trails I followed, and the false starts that eventually led to the right path.

You could argue that I should simply present the final setup and explain how it works. But I think it’s just as valuable to see what didn’t work. Sharing the failures, trade-offs, and dead ends helps demonstrate that the final approach isn’t just something that happened to work — it’s the one that stood up to testing and comparison.

That’s especially relevant today, when our first instinct to solve a problem is often to open ChatGPT or another LLM and ask for help. The response might look convincing, but is it the best approach? With fast-moving tools like Mosaic AI Gateway, the underlying models often haven’t been trained on the latest APIs or docs — meaning their answers can be incomplete, outdated, or just don't work.

By sharing both the process and the result, I hope to give readers confidence that this solution is not only functional but well-researched and battle-tested — the one that held up after real experimentation, not just a lucky first try.

## The Solution: ResponsesAgent

As promised, let’s start with the answer.

The simplest way to enable tracing while still using all the features of Mosaic AI Gateway is to create and deploy a ResponsesAgent model in Databricks. This model type has MLflow Tracing enabled by default, and when you host it through the Gateway, you retain the same production capabilities — including rate limiting, logging, guardrails, and more.

In other words, a ResponsesAgent gives you the best of both worlds: full Gateway functionality and detailed trace data for every request.

If you’re just here for the implementation details, feel free to skip ahead to the end of this post — I walk through the setup step by step.

But if you’re curious about how I arrived at this solution, stick around. The next few sections cover the other approaches I tried, the dead ends I hit, and the reasoning that ultimately led to the ResponsesAgent path.

## Attempt 1: Foundational Models

Like anyone learning a new system, I started at the beginning — the [Mosaic AI Gateway Introduction](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/) page. It includes a table showing which features each model type supports:

![Databricks Table](/ai-gateway/ai-gateway-table-edited.png)

At first glance, external model endpoints seemed the most capable. However, for this walkthrough, I focused on foundational models. They’re easier for readers to follow along with since they don’t require setting up authentication or access to external services.

Aside from that authentication step, foundational and external models function almost identically in configuration, serving behavior, and available Gateway features.

As a newcomer, I assumed I could host a foundational model and still get traces. That would have been ideal — use a proven model and get observability for free. So my first goal was to spin up an endpoint and confirm whether tracing worked out of the box.

The next section walks through how to create such an endpoint right up until the point where you learn that it does not support tracing. If you’re just here for the tracing solution, feel free to skip ahead to the Wrapper section — otherwise, here’s what I found.

### Foundational Model Endpoint Creation

When creating infrastructure in Databricks, there are usually multiple paths to the same result — Terraform, Python, SQL, or the UI. For investigative work like this, I prefer the UI. It’s quick to explore configurations and verify behavior visually, even though in production you’d typically automate the process.

To create an endpoint, go to Serving → Create Serving Endpoint, then choose Foundational Models in the Served Entities section. This opens the endpoint creation menu shown below. Working through it from top to bottom, first give your endpoint a name, then configure the Served Entities section.

![Endpoint Options](/ai-gateway/endpoint-menu.png)

The Served Entities section is where you select the model you want to serve. Click on the 'Select an Entity' text in the 'Entity details' box. Then choose the 'Foundational Models' option from the radio list.

![Served Entity Menu](/ai-gateway/served-entity-menu.png)

After selecting that option, a second pop-up lists both foundational and external models, as shown below.

![Selection Menu Options](/ai-gateway/selection-menu.png)

This can be confusing at first because the pop-up menu is labeled Foundational Models, yet it also lists external providers. I’m calling this out for two reasons:

1. If you’re following along and want to use an external model instead, this is how you’d do it — select your model, complete the additional authentication step, and you’ll have an external model endpoint. You could then substitute that model’s name wherever a foundational model is used in this post.

2. It highlights that the only real difference between foundational and external model endpoints is authentication; otherwise, the setup is identical.

Once you’ve chosen a foundational model — in this case, I selected GPT OSS 20B — you’ll see the configuration screen below.

![Model UI Options](/ai-gateway/model-ui-options.png)

Here you can configure throughput and scaling options, but notice there’s no tracing toggle. So how do you set up tracing for your model?

> Note: When I first started, I saw some models with a tracing toggle in the UI, but those seem to have disappeared. With Databricks’ rapid feature updates, I’ve seen frequent changes in the platform which often impact what you're working. 
> 
> When I first planned this post, I expected to ask, “What do you do if your model doesn’t support tracing?” — but now, none of them do. Fortunately, I still have an answer.

### Searching for the Missing Piece

Not knowing how, I dug deeper into the documentation. I found plenty of useful material about tracing GenAI apps, but very few that answered the core question: “How do I set up an endpoint that generates traces by default?”

Here are a few of the more relevant but incomplete resources:

- ["Get started: MLflow Tracing for GenAI (Databricks Notebook)"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/getting-started/tracing/tracing-notebook) -- great for learning how traces work, but only covers tracing single notebook requests.
- ["Tracing OpenAI"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/tracing/integrations/openai) — shows how to trace OpenAI calls, but not how to create an endpoint with tracing enabled.

As you'll find if you start to go through the docs as well, most examples show how to trace one request, not how to create an endpoint creates a trace for each request it recieves.

Finally, I stumbled on this artice ["Deploy agents with tracing"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/tracing/prod-tracing). This article finally pointed me in the right direction — though I was skeptical at first. My initial thought was: “Why do we need an agent? This seems like such a simple thing to achieve — an agent feels overcomplicated.”

But that article sparked an idea: what if I could create a wrapper model that calls the underlying model while automatically handling tracing? That realization set the direction for my next experiment.

## Attempt 2: Custom Python Model

If foundational models couldn’t generate traces directly, then I needed to create something that could. The answer was to build a wrapper — a lightweight layer that calls the real model behind the scenes while also handling tracing. Essentially, requests come into the wrapper, the wrapper calls the underlying model (like GPT OSS 20B), and then it returns the response unchanged. The difference is that the wrapper can log traces automatically.

### The Wrapper Approach

At this point, we knew that while Databricks AI Gateway could host a foundational model directly, it didn’t support tracing in that configuration. Creating a wrapper model will allow us to use our foundational model and attain tracing functionality.

Here’s the high-level plan:

1. Build a small model class that wraps around our foundational model.
2. Configure the class so that tracing is enabled by default.
3. Register this model in Unity Catalog.
4. Deploy it as a Serving Endpoint, just like before — except now the endpoint will support both AI Gateway and tracing.

This approach gives us the same Gateway functionality as before, but with complete trace visibility. If this sounds confusing, just wait till we get to the code examples which I think will help clarify the setup.

### Wrapper Options

Once I knew I needed a wrapper, the next question was how to define it with MLflow.

There are two main ways to do this:

- Custom Python Model – You define your own PythonModel class and control every detail of how predictions are made.
- Responses Agent Model – You use one of Databricks’ prebuilt agent types (like ResponsesAgent) that already integrate tracing, tool use, and prompt orchestration.

As I mentioned before, I was skeptical of the responses agent solution. My goal wasn’t to build a full agent-based system — I just wanted to trace model calls. That left me with one clear option: a Custom Python Model.

That said, the [Gen AI Apps guide](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/guide/introduction-generative-ai-apps) clearly recommends using response agents over custom python models. However, I still wasn't convinced. I wanted somthing simple, transparent, and easy to debug.

So I started by building the Python model — and as you can probably guess, it worked, but not as well as I’d hoped. Once I had it running, I compared it to a ResponsesAgent implementation and found that the agent approach was cleaner and better aligned with Databricks’ newer OpenAI Responses API. It also seemed more future-proof as Databricks expands its generative AI tooling.

### Implementing a Custom Python Model

Creating a custom Python model in MLflow involves encapsulating your logic in a class that inherits from mlflow.pyfunc.PythonModel. The key method is predict, which receives input and returns the model’s response.

Since this model is only a wrapper, our predict method just forwards the request to the underlying foundational model and returns its response.

If you’d like to see the relevant documentation, here are the key references I used:

- [MLflow Python Model Guide](https://mlflow.org/docs/latest/ml/model/python_model/)
- [Custom Serving Applications](https://mlflow.org/docs/latest/genai/serving/custom-apps/)
- [MLflow Python Model Class](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#mlflow.pyfunc.PythonModel)

**1. Install dependencies**

In the first notebook cell, install the following libraries using the code below:

```python {style=monokai}
%pip install -U -qqqq databricks-openai
dbutils.library.restartPython()
```

In addition to installing the databricks-openai package, this command upgrades MLflow — which is important because the current default serverless environment typically includes MLflow Skinny 2.x, but the code below requires MLflow 3.x.

| Default Libraries            | Installed Libraries             |
|------------------------------|---------------------------------|
| `mlflow-skinny==2.21.3`      | `mlflow==3.4.0`                 |
| `databricks-connect==16.4.2` | `mlflow-skinny==3.4.0`          |
| `databricks-sdk==0.49.0`     | `mlflow-tracing==3.4.0`         |
|                              | `databricks-ai-bridge==0.8.0`   |
|                              | `databricks-connect==16.4.2`    |
|                              | `databricks-openai==0.6.1`      |
|                              | `databricks-sdk==0.49.0`        |
|                              | `databricks_vectorsearch==0.59` |

> ⚠️ Note: MLflow’s documentation warns against having both mlflow and mlflow-skinny installed simultaneously. I haven’t encountered issues with this setup, and several Databricks examples use the same install pattern — so I decided to trust it. Still, it’s something to watch for in case anything breaks or behaves unexpectedly.

**2. Define your model**

Here’s the cell that defines the model and writes it to a file called `model.py`. Let’s walk through the file from top to bottom.

```python
%%writefile model.py
import mlflow
from mlflow.pyfunc import PythonModel, PythonModelContext
from databricks.sdk import WorkspaceClient
from typing import Any, Dict, List, Optional

class ModelWrapper(PythonModel):
    def __init__(self):
        self.client = WorkspaceClient().serving_endpoints.get_open_ai_client()

    def predict(  # pyright: ignore
        self,
        context: PythonModelContext,
        model_input: List[Dict[str, str]],
        params: Optional[Dict[str, Any]] = None,
    ) -> List[str]:
        results = []
        
        response = self.client.chat.completions.create(
            model="databricks-meta-llama-3-1-8b-instruct",
            messages=model_input
        )
        results.append(response.choices[0].message.content)
        
        return results

mlflow.openai.autolog()
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Shared/mn-demo-experiments")
mlflow.models.set_model(ModelWrapper())
```

The %%writefile command writes this cell’s contents to a file — in this case, model.py. This is required because the model registration step later references this file directly. It also keeps all the code for this example self-contained within a single notebook, aside from a small requirements.txt file.

The ModelWrapper class inherits from PythonModel. In the constructor, we initialize a Databricks WorkspaceClient to access the serving endpoint client that will call our base model.

If you’d like to connect to an external model instead of a foundational one, here’s how:

1. Follow the steps in Attempt 1 to create a serving endpoint for your external model (e.g., external-model-endpoint).
2. In the code above, replace the model parameter in the chat.completions.create call with the name of your external model.
3. That’s it — your wrapper will now route requests to your external model instead.

The next portion of the file is the predict method, which defines the inference logic — sending the request to the model and returning its response. You’ll also notice that I added type hints to this function. This prevents MLflow from emitting log warnings during model creation.

The last four lines at the bottom of the cell are critical:

```python
mlflow.openai.autolog()
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Shared/mn-demo-experiments")
mlflow.models.set_model(ModelWrapper())
```

These lines enable tracing and logging.
- mlflow.openai.autolog() enables detailed trace collection. Without it, you’d only get partial trace data through decorators.
- The next two lines (set_tracking_uri and set_experiment) tell MLflow where to store trace data in Databricks.
  - If you omit the tracking URI and experiment setup, traces will only appear when you call the endpoint interactively from a Databricks notebook — not when hitting it via API.
- Finally, set_model() sets the model object that is going to be logged.

**3. Register the model**

Once your model.py file is defined, the final step is to register it in Unity Catalog. The code below handles that process:

```python
import mlflow
from mlflow.models.resources import DatabricksServingEndpoint

example = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What is the fibonacci sequence"},
]

with mlflow.start_run():
    mlflow.pyfunc.log_model(
        name="mn-ai-demo",
        python_model="model.py",
        input_example=example,
        registered_model_name="workspace.default.mn-ai-demo",
        pip_requirements="requirements.txt",
        resources=[DatabricksServingEndpoint(endpoint_name="databricks-gpt-oss-20b")],
    )
```

Since the %%writefile command in the previous cell writes your model code to disk rather than keeping it in memory, you’ll need to re-import MLflow (and any other dependencies) before running this step.

The example variable defines a simple test input that MLflow uses to validate your model. Inside your requirements.txt, include databricks-openai — this package is required for deployment to work correctly.

**What to Expect When You Run Model Registration Code**

When you execute the registration cell, MLflow confirms a successful run and model version creation. The example input and response will appear inline:

![Model Registration](/ai-gateway/model-registration.png)

Immediately below that, you’ll see your first trace — showing not only the input and output, but detailed metadata such as call hierarchy, timing, and context:

![Model Registration Trace](/ai-gateway/registration-trace.png)

![Model Registration Trace Part II](/ai-gateway/registration-trace-endpoints.png)

![Model Registration Trace Part III](/ai-gateway/registration-trace-endpoints-p2.png)

At this point, you can also test your model directly from a notebook without importing the model.py file. Simply call mlflow.pyfunc.load_model() to load it from Unity Catalog, then invoke predict(). Due to the way we defined our model, you each notebook inference will automatically generate a trace, as shown here:

![Notebook Trace](/ai-gateway/notebook-trace.png)

### Creating the Endpoint

With your model registered, the next step is deployment.

Navigate back to the Serving page, click Create Endpoint, and select your new Python model. You’ll now see an option to enable tracing — it should already be turned on by default:

![Endpoint Creation Menu With Python Model](/ai-gateway/endpoint-creation-python-model.png)

Scroll down to the AI Gateway section to configure settings like the Inference Table, which logs all requests and responses (though not as richly as traces). Note that inference tables aren’t yet available in the free Databricks tier.

Once your endpoint finishes deploying, the interface should show it as Active and ready for REST requests:

![Endpoint Creation Screen](/ai-gateway/ai-gateway-screen.png)

You can now test it using curl or your preferred REST client — I’ve been using the REST Client extension in VS Code. After sending a few requests, check your Experiments page under the shared experiment path — you’ll see fresh traces appear there:

![Rest Client](/ai-gateway/rest-client.png)

![Rest Trace](/ai-gateway/rest-trace.png)

At this point, you’ve successfully built an endpoint with full Mosaic AI Gateway functionality and detailed tracing — all through a custom Python model. You're probably wondering why I am not recommending the python model. The issue is that I had a difficult time setting up streaming responses with the custom python model. If streaming had worked seamlessly, this might have been my final recommendation. But it didn’t — and that’s what led me to explore the ResponsesAgent next. 

### What Went Wrong with Streaming Requests

If you’ve looked at the documentation I referenced earlier, you may have noticed that the PythonModel class in MLflow also defines a predict_stream function that you can override to support streaming requests.

Here’s the basic idea: we implement the predict_stream function inside our model class. When a REST request includes a streaming parameter, MLflow will call predict_stream instead of predict, allowing the model to return partial results as they arrive.

However, this is where things start to get tricky. The predict_stream function is designed to take the same PyFunc-compatible input as predict, so I defined it as follows:

```python
def predict_stream(  # pyright: ignore
        self,
        context: PythonModelContext,
        model_input: List[Dict[str, str]],
        params: Optional[Dict[str, Any]] = None,
    ) -> Iterator[str]:
        response = self.client.chat.completions.create(
            model="databricks-meta-llama-3-1-8b-instruct",
            messages=model_input,
            stream=True,
        )

        full_message = ""
        for chunk in response:
            if chunk.choices and chunk.choices[0].delta.content:
                new_content = chunk.choices[0].delta.content
                full_message += new_content
                yield new_content

        yield full_message
```

**Testing in a Notebook**

When I imported the module directly and invoked predict_stream, it worked perfectly — chunked responses arrived in real time, and the trace appeared in the notebook output.

![Notebook Trace](/ai-gateway/working-predict-stream.png)

![Notebook Trace](/ai-gateway/working-predict-stream-trace.png)

This confirmed that the function worked when called directly from the notebook. I could see all the chunks streaming back one by one, and tracing behaved exactly as expected.

Encouraged by this, I decided to test the same functionality through other methods — and that’s where things started to fall apart.

**Testing Through the Loaded Model and REST API**

After registering the model in Unity Catalog, I tried invoking it again by loading it with mlflow.pyfunc.load_model() and calling predict_stream. This time, it failed.

![Notebook Trace](/ai-gateway/failed-predict-stream.png)

You can see in the image above that predict() still works fine when the model is loaded this way — but predict_stream() doesn’t. The same issue appears when invoking it through a REST endpoint:

![Notebook Trace](/ai-gateway/bad-predict-stream-api.png)

At this point, I was puzzled. The function clearly worked in one context but failed in another. I briefly considered adding a streaming flag to the predict method itself (e.g., predict(streaming=True)), but that felt like a hack and went against the intended design of the MLflow API. It also wasn’t as clean as maintaining two clearly defined methods — one for standard predictions and one for streaming.

So I started digging into the root cause.

**Digging into the Cause**

Why didn’t predict_stream work when the model was loaded from Unity Catalog?

The key detial lies in what mlflow.pyfunc.load_model() actually returns. According to the [MLflow docs](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#mlflow.pyfunc.load_model), it doesn’t return your PythonModel directly — it returns a PyFuncModel, a wrapper class that standardizes how models are called.

When you invoke predict_stream() on the loaded model, you’re actually calling the wrapper’s version of that function, which then delegates to your implementation. Unfortunately, something in that handoff — specifically in how inputs are validated and passed through — seems incompatible with the OpenAI-style message list I was using.

For anyone interested in exploring further, you can inspect the predict_stream implementation in the [PyFuncModel source code](https://mlflow.org/docs/latest/api_reference/_modules/mlflow/pyfunc.html). 

What frustrated me about this experience is that [MLFlow PythonModel documentation](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#mlflow.pyfunc.PythonModel) states that both predict and predict_stream accept PyFunc-compatible input. Since my input worked perfectly with predict, I expected it to work with predict_stream as well.

The [Inference API docs](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#pyfunc-inference-api) also claim that “a list of any type” should be valid input — further suggesting this should have worked.

**Where Things Stand**

To make predict_stream work, I had two main options:

(1) change its input format, or
(2) modify predict so it could handle streaming requests as well.

Both felt like poor tradeoffs. I didn’t want to maintain separate input schemas for predict and predict_stream, and I also didn’t like the idea of adding a “streaming” flag to predict just to make it behave differently.

If you don’t need streaming, the custom Python model approach is still an excellent choice — it’s flexible, powerful, and integrates seamlessly with Databricks. But for my use case — supporting both standard and streaming completions — it wasn’t enough.

So it was time to move on to Attempt 3: the ResponsesAgent.

## Attempt 3: Responses Agent

The Databricks documentation includes a “simple” guide for creating a Responses Agent endpoint. It’s a good starting point, but I’ll admit — I wasn’t a huge fan of the sample notebook. The call stack for basic predictions felt unnecessarily complex, and several unused libraries made it harder to see what was actually going on. At one point, I even caught myself wondering if those extra dependencies had some hidden purpose I’d missed.

That said, it’s still a valuable reference. I adapted their example into a cleaner, more minimal version that’s easier to follow for anyone new to ResponsesAgent. That’s the version we’ll walk through here.

For anyone curious, you can find Databricks’ original example notebook here: https://docs.databricks.com/aws/en/notebooks/source/mlflow3/simple-agent-mlflow3.html.

### Implementing Responses Agent

As before, we’ll start our notebook with an installation cell — but this time we’ll add the `databricks-agents` library alongside `databricks-openai`.

```python
%pip install -U -qqqq databricks-openai databricks-agents
dbutils.library.restartPython()
```

Next, we’ll define our agent and write it to a Python file:

```python
%%writefile model.py
import mlflow

from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import (
    ResponsesAgentRequest,
    ResponsesAgentResponse,
    ResponsesAgentStreamEvent,
)

from databricks.sdk import WorkspaceClient

from typing import Generator


class SimpleResponsesAgent(ResponsesAgent):

    def __init__(self):
        self.workspace_client = WorkspaceClient()
        self.client = self.workspace_client.serving_endpoints.get_open_ai_client()
        self.model = "databricks-gpt-oss-20b"

    def predict(self, request: ResponsesAgentRequest) -> ResponsesAgentResponse:
        messages = request.input

        response = self.client.chat.completions.create(
            model=self.model,
            messages=self.prep_msgs_for_cc_llm(messages),
        )

        return ResponsesAgentResponse(
            output=[
                self.create_text_output_item(
                    text=response.choices[0].message.content, id=response.id
                )
            ],
        )

    def predict_stream(
        self, request: ResponsesAgentRequest
    ) -> Generator[ResponsesAgentStreamEvent, None, None]:
        response = self.client.chat.completions.create(
            model=self.model,
            messages=self.prep_msgs_for_cc_llm(request.input),
            stream=True,
        )

        item_id = 1
        full_message = ""
        for chunk in response:
            if chunk.choices and chunk.choices[0].delta.content:
                new_content = chunk.choices[0].delta.content
                full_message += new_content
                yield ResponsesAgentStreamEvent(
                    **self.create_text_delta(
                        delta=new_content, item_id=f"msg_{item_id}"
                    ),
                )
                item_id += 1

        yield ResponsesAgentStreamEvent(
            type="response.output_item.done",
            item=self.create_text_output_item(  # pyright:ignore
                text=full_message,
                id=f"msg_{item_id-1}",
            ),
        )

        return


mlflow.openai.autolog()
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Shared/mn-demo-experiments-agent")
mlflow.models.set_model(SimpleResponsesAgent())
```

**Understanding the `predict` Function**

At first glance, this looks similar to our earlier custom Python model — but there are three key differences:

1. Inheritance: our class now inherits from ResponsesAgent instead of PythonModel.

2. Types: the predict method takes a single parameter of type ResponsesAgentRequest and returns a ResponsesAgentResponse.

3. Translation: Before sending messages to the model, it calls self.prep_msgs_for_cc_llm() — a helper function that quietly handles a lot of complexity.

In order to understand these differences and fully understand the code, we have to start at the request and response structures in place.

**Request and Response Structure**

Here's a simplified example from the [MLflow Responses Agent docs](https://mlflow.org/docs/latest/genai/serving/responses-agent/#schema-and-types).   

```python
# Example Request schema
{
    "input": [
        {
            "role": "user",
            "content": "What is the weather like in Boston today?",
        }
    ],
    "tools": [
        {
            "type": "function",
            "name": "get_current_weather",
            "parameters": {
                "type": "object",
                "properties": {"location": {"type": "string"}},
                "required": ["location", "unit"],
            },
        }
    ],
}

# Example Response schema
{
    "output": [
        {
            "type": "message",
            "id": "some-id",
            "status": "completed",
            "role": "assistant",
            "content": [
                {
                    "type": "output_text",
                    "text": "rainy",
                }
            ],
        }
    ],
}
```

These schemas define how ResponsesAgentRequest and ResponsesAgentResponse are structured.  Note that both can include additional parameters (like temperature or max_output_tokens), so it’s worth checking the [full API reference](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.types.html#mlflow.types.responses.ResponsesAgentRequest) for details.

**The Role of `prep_msgs_for_cc_llm()`**

OpenAI recently introduced a new Responses API, which replaces the older Chat Completions API used in many legacy examples.  Databricks’ ResponsesAgent class and its request/response types are built to align with this newer API.

However — and this is where things get tricky — the two APIs expect slightly different input formats.
- The Chat Completions API expects a list of messages.
- The Responses API can accept a single string or a structured schema.

This means that a request formatted for the Responses API won’t necessarily work with the Chat Completions API.

That’s where prep_msgs_for_cc_llm() (short for prepare messages for chat completion LLM) comes in — it automatically converts the input into the correct structure.  You don’t have to define it yourself; it’s inherited from the ResponsesAgent base class.

**Why Not Use the Responses API Directly?**

You might wonder: if our input already matches the Responses schema, why not just call the Responses API?

Something like this should work, right?

```python
messages = request.input

response = client.responses.create(
    model=self.model,
    messages=messages,
)
```

In theory, yes — but in practice, not yet within Databricks.

Here’s why: the WorkspaceClient from the Databricks SDK provides a wrapped client that can access registered models inside your workspace, regardless of where they’re hosted. It’s convenient because you don’t need to configure environment variables for authentication.

My guess is that his SDK client hasn’t been fully updated to support the new Responses API. As a result, calling client.responses.create() currently raises an error — even with simple requests.

This theory is further supported by the official Databricks notebooks: all of them use the ResponsesAgent class (which matches the Responses API schema) but still call the Chat Completions API using the `prep_msgs_for_cc_llm` function behind the scenes.

**A Note on Alternative Clients**

There is another way to call the Responses API in Databricks — by using the standard OpenAI client instead of the SDK:

```python
from openai import OpenAI

client = OpenAI()
```

This approach works when calling external models that support the Responses API (note that some older models don’t).  However, it requires you to set environment variables for authentication.

**Wrapping Up predict**

After the messages are translated, the model is called as usual. The final step is returning a response that matches the ResponsesAgentResponse schema:

```python
        return ResponsesAgentResponse(
            output=[
                self.create_text_output_item(
                    text=response.choices[0].message.content, id="msg_1"
                )
            ],
        )
```

Since the ResponsesAgent is designed for the Responses API (while our response object follows the Chat Completions schema), this constructor bridges the two formats.  The helper function create_text_output_item() creates a properly structured output entry — one of several output types available.  You can find the full list in the [Creating Agent Output section](https://mlflow.org/docs/latest/genai/serving/responses-agent/#creating-agent-output) of the ResponsesAgent documentation.

Also don’t worry about losing details from the original response.
Although we only return the message text, MLflow’s tracing automatically captures the full request and response context — including all the metadata — behind the scenes.

**What About Streaming?**

Streaming worked much more smoothly with ResponsesAgent than it did with the custom Python model.

Here’s what’s happening in the code:

1. The call to the model includes the stream=True parameter, which signals that we want token-by-token output.
2. The response arrives in chunks. The code accumulates these chunks into a single message.
3. As new chunks arrive, we yield incremental ResponsesAgentStreamEvent objects so the user can see updates in real time.
4. Finally, we yield a “done” event to signal that the response is complete.

This design allows your application to display streaming responses without blocking — and since it’s built into the ResponsesAgent framework, the setup is minimal.

**Logging and Deployment**

There are two additional notebook cells to complete the setup:

```python
# cell 1
import mlflow
from mlflow.models.resources import DatabricksServingEndpoint

UC_LOCATION = f"workspace.default.agent-demo-endpoint"

mlflow.openai.autolog()

with mlflow.start_run():
    logged_agent_info = mlflow.pyfunc.log_model(
        name="agent-demo-endpoint",
        python_model="agent_model_streaming.py",
        registered_model_name=UC_LOCATION,
        pip_requirements="requirements.txt",
        resources=[DatabricksServingEndpoint(endpoint_name="databricks-gpt-oss-20b")],
    )

# COMMAND ----------

from databricks import agents  # pyright: ignore

agents.deploy(
    UC_LOCATION,
    model_version=logged_agent_info.registered_model_version,
)
```
The first cell logs the model — something you’ve seen earlier in this series.

The second cell deploys the agent directly from code, so you don’t need to use the Databricks Serving UI manually.

If the model is already deployed, it will automatically apply updates — a small but very welcome time-saver.

## Conclusion

It was a long journey to arrive at the Responses Agent approach — but hopefully one that made the reasoning clear.

If you’ve followed along from the beginning, you can see how a newcomer might start with foundational models, explore custom Python models, and eventually land on Responses Agents as the most robust and traceable option.

If you take away just a few things, let them be these:

- You now understand how Mosaic AI Gateway, model serving, Python models, and Responses Agents fit together.
- If you’re building something similar, you can confidently start with Responses Agents, knowing that the alternatives were explored and tested.

Thanks for reading — and for sticking with a deep-dive post!

I wanted to make this guide as thorough as possible so it answers the questions I had when I first started.

If you enjoyed this, stay tuned for more articles on data engineering, AI, and Databricks.
