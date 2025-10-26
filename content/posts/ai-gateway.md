---
title: "Tracing with Databricks Mosaic AI Gateway: A Practical Guide"
date: 2025-10-05
draft: false
tags: ["databricks", "ai", "tracing"]
categories: ["engineering"]
slug: "ai-gateway"
author: "Matthew Norberg"
---

Mosaic AI Gateway is Databricks’ system for managing and governing how your organization uses large language models (LLMs) and AI agents. Out of the box, it includes features like permission and rate limiting, payload logging, usage tracking, AI guardrails, fallbacks, and traffic splitting. These tools give teams tighter control over their AI workloads — making it easier to manage access, monitor performance, and keep costs in check.

Although Mosaic AI Gateway comes with many powerful features, one capability is still missing: MLflow Tracing. Tracing is like logging with context — it doesn’t just capture the request and response, but also the intermediate steps that reveal what happened inside your AI system when something goes wrong. Without it, visibility into model behavior can be limited. As you’ll see, MLflow traces can be an invaluable tool when debugging or optimizing an LLM workflow.

So the question becomes: **how do you build a Mosaic AI Gateway endpoint that captures traces for each request?**

## The Goal of This Guide

My goal is to show you how to build a Mosaic AI endpoint with tracing enabled by default — and to make that process clear enough that you can adapt it to your own setup. Over the past month, I’ve spent a lot of time configuring endpoints, debugging integrations, and figuring out how all the pieces fit together. After plenty of trial and error, I wanted to share what I’ve learned so other developers can get up and running faster.

If you’ve explored Databricks’ newer AI features, you’ve probably noticed that the documentation, while thorough, can feel circular and fragmented. You often need to jump between Databricks, MLflow, and OpenAI documentation to piece things together. It’s easy to lose the thread when trying to connect everything into a working system.

This post is my attempt to bridge those gaps with a concrete, end-to-end example — the kind I wish I’d had when I started. It’s a practical guide to getting tracing working with Mosaic AI Gateway without getting lost in a maze of configuration steps and scattered docs. To save you time, I’ll start by showing the solution up front, then walk you through the process that got me there — including the different methods I tried along the way.

You could argue that I should simply present the final setup and explain how it works. But I think it’s just as valuable to see what didn’t work. By sharing both the process and the result, I hope to give you confidence that this approach isn’t just functional, but tested, reasoned through, and grounded in actual experimentation.

## The Solution: ResponsesAgent

As promised, let’s start with the answer.

The simplest way to enable tracing while still using all the features of Mosaic AI Gateway is to create and deploy a **ResponsesAgent** model in Databricks. This model type has MLflow Tracing enabled by default, and when hosted through the Gateway, you retain the same production capabilities — including rate limiting, logging, guardrails, and more.

In short, a ResponsesAgent gives you the best of both worlds: full Gateway functionality and detailed trace data for every request.

If you’re here just for the implementation, you can skip to the end — I’ll walk through the setup step by step. But if you’re curious how I arrived at this solution, stick around. The next sections cover the other approaches I tried, the dead ends I hit, and how they led me to the ResponsesAgent path.

## Attempt 1: Foundational Models

Like anyone learning a new system, I started at the beginning — the [Mosaic AI Gateway Introduction](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/) page. It includes a table showing which features each model type supports:

![Databricks Table](/ai-gateway/ai-gateway-table-edited.png)

At first glance, external model endpoints seemed the most capable. However, for this walkthrough, I focused on foundational models. They’re easier for readers to follow since they don’t require setting up authentication or external service access. Aside from that step, foundational and external models behave almost identically in configuration, serving behavior, and available Gateway features.

As a newcomer, I assumed I could host a foundational model and get tracing automatically — use a proven model and gain observability for free. That seemed like the ideal starting point, so my first goal was to spin up an endpoint and see how far I could get.

### Foundational Model Endpoint Creation

When creating infrastructure in Databricks, there are usually multiple paths to the same result — Terraform, Python, SQL, or the UI. For investigative work like this, I prefer the UI. It makes it easy to explore configurations and verify behavior visually, even though in production you’d typically automate the process.

To create an endpoint, go to **Serving → Create Serving Endpoint**, then choose **Foundational Models** in the *Served Entities* section. This opens the endpoint creation menu shown below. Working through it from top to bottom, first give your endpoint a name, then configure the *Served Entities* section.

![Endpoint Options](/ai-gateway/endpoint-menu.png)

Click **Select an Entity**, choose **Foundational Models** from the radio list, then click **Select a foundational model** in the box. You'll see a new pop-up menu listing both foundational and external models.

![Entity Menu Creation Guide](/ai-gateway/endpoint-menu-guide.png)

This can be confusing at first because the pop-up menu is labeled Foundational Models, yet it also lists external providers. I’m calling this out for two reasons:

1. Suppose you’d like to configure an external model instead. Start by selecting your provider and completing the authentication step shown above. Then finish creating your serving endpoint, taking note of the name you assign it. You’ll need that endpoint name later when configuring your ResponsesAgent to call this endpoint, which in turn routes to your external model.
2. It shows that the only real difference between foundational and external models is authentication; the rest of the setup is identical.

Once you’ve chosen a foundational model — in this case, I selected **GPT OSS 20B** for demo purposes — you’ll see the configuration screen below.

![Model UI Options](/ai-gateway/model-ui-options.png)

Here you can configure throughput and scaling options, but there’s **no tracing toggle**. 

> **Note**: When I first started, I saw some models with a tracing toggle in the UI, but those have since disappeared. Databricks evolves quickly, and feature changes often land mid-project. When I began this post, I expected to ask, “What if your model doesn’t support tracing?” — now, none of them do. Fortunately, I still have an answer.

### Searching for the Missing Piece

Not knowing how to proceed, I dug into the documentation. There was plenty about tracing GenAI apps — but little on how to create an endpoint that **automatically generates traces**.

A few helpful but incomplete resources included:

- ["Get started: MLflow Tracing for GenAI (Databricks Notebook)"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/getting-started/tracing/tracing-notebook) -- great for learning how traces work, but only covers tracing single notebook requests.
- ["Tracing OpenAI"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/tracing/integrations/openai) — shows how to trace OpenAI calls, but not for endpoint deployment.

As you'll find if you start to go through the docs as well, most examples show how to trace one request, not how to create an endpoint creates a trace for each request it recieves. Eventually, I found ["Deploy agents with tracing"](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/tracing/prod-tracing), which pointed me in the right direction -- though I was skeptical at first. My initial reaction was: “Why do we need an agent? This seems like such a simple thing to achieve — an agent feels overcomplicated.”

But that article sparked an idea: **what if I created a wrapper model** that calls the underlying model while automatically handling tracing? That realization shaped my next experiment.

## Attempt 2: Custom Python Model

If foundational models couldn’t generate traces directly, then I needed something that could. The answer was to build a wrapper model — a lightweight layer that accepts requests, forwards them to the underlying model (like GPT OSS 20B), and returns the response unchanged. The difference is that the wrapper can be configured to add tracing to each request by default.

Here's the high level plan:

1. Build a small model class that wraps around our foundational model.
2. Configure the class so that tracing is enabled by default.
3. Register this model in Unity Catalog.
4. Deploy it as a Serving Endpoint, just like before — except now the endpoint will support both AI Gateway and tracing.

This approach gives you the same Gateway functionality as before, but with complete trace coverage. If this sounds confusing, hopefully the code examples will help make things concrete.

### Wrapper Options

Once I knew I needed a wrapper, the next question was how to define it with MLflow. There are two main options:

- **Custom Python Model** – Define your own the `PythonModel` class and implement your own prediction functions.
- **Responses Agent Model** – Use the `ResponsesAgent` class to create a agent model that calls your foundational model under the hood.

As I mentioned before, I was skeptical of the `ResponsesAgent` approach. My goal wasn’t to build a full agent-based system — I just wanted to trace model calls. That made the **Custom Python Model** path seem like the most straightforward solution.

That said, the [Gen AI Apps guide](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/guide/introduction-generative-ai-apps) clearly recommends using response agents over custom python models. However, I still wasn't convinced. I wanted somthing simple, transparent, and easy to debug.

So I built the Python model — and, as you can probably guess, it worked, but not as well as I’d hoped. Once it was running, I compared it to a `ResponsesAgent` implementation and found that the agent approach was cleaner, better aligned with Databricks’ newer OpenAI Responses API, and more future-proof as the platform continues to evolve.

### Implementing a Custom Python Model

Creating a custom Python model in MLflow involves encapsulating your logic in a class that inherits from `mlflow.pyfunc.PythonModel`. The key method is `predict`, which receives input and returns the model’s response. Since this model acts only as a wrapper, our `predict` method simply forwards each request to the underlying foundational model and returns its response.

If you’d like to dig deeper, these are the main references I used:

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

> ⚠️ Note: MLflow’s documentation warns against installing both `mlflow` and `mlflow-skinny`. I haven’t encountered any issues, and several Databricks examples use the same approach — but it’s worth keeping in mind if anything behaves unexpectedly.

**2. Define your model**

In cell 2 of our notebook, we define our model and save it to `model.py`. Let’s walk through the code from top to bottom to understand what each part does and why it matters.

```python
%%writefile model.py
import mlflow
from mlflow.pyfunc import PythonModel, PythonModelContext
from databricks.sdk import WorkspaceClient
from typing import Any, Dict, List, Optional


class ModelWrapper(PythonModel):
    def __init__(self):
        self.client = WorkspaceClient().serving_endpoints.get_open_ai_client()

    def predict(
        self,
        context: PythonModelContext,
        model_input: List[Dict[str, str]],
        params: Optional[Dict[str, Any]] = None,
    ) -> List[str]:
        results = []

        response = self.client.chat.completions.create(
            model="databricks-meta-llama-3-1-8b-instruct", messages=model_input
        )
        results.append(response.choices[0].message.content)

        return results

mlflow.openai.autolog()
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Shared/mn-demo-experiments")
mlflow.models.set_model(ModelWrapper())
```

The `%%writefile` command writes this cell’s contents to the model.py file. This is necessary because the registration step later on will need to read your model from disk. You could, of course, create this file manually and omit this cell entirely, but writing it dynamically keeps the notebook self-contained and repeatable. Aside from a small `requirements.txt` file, this approach ensures everything you need to reproduce the setup lives in one place.

The `%%writefile` command writes this cell’s contents to the `model.py` file. This is required because the model registration step needs to read in the model from a `.py` file. We could have placed this code in the file manually and ommited this cell from the notebook. However, the `%%writefile` command allows us to keep all the code self-contained within a single notebook, aside from a small `requirements.txt` file.

The `ModelWrapper` class inherits from `PythonModel`, the standard interface for custom MLflow models. Inside the constructor, we initialize a `WorkspaceClient`, which gives us access to the serving endpoint client that will call our base model. That means our wrapper model can seamlessly forward requests to any foundation model or custom endpoint you’ve already deployed.

At this point, if you’d like to connect to an **external model** instead of a foundational one, follow these steps below:

1. Follow the steps in Attempt 1 to create a serving endpoint for your external model (e.g., `external-model-endpoint`).
2. Replace the `model` parameter in the `chat.completions.create()` call with the name of your external model.

The next portion of the file is the `predict` method, which defines the inference logic — sending the request to the model and returning its response. You’ll notice that I added type hints to this function. In particular, MLflow will emit a `UserWarning` if the `model_input` parameter does not have a type hint applied. Some of the online examples I found only added type hints to the `model_input` parameter to silence the warning. Personally, I think if you are going to add type hints, you should be concsistent and add them everywhere. This approach prevents the `UserWarning` and aligns the code with Python best practices.

Finally, look closely at the last four lines — they’re easy to overlook but absolutely essential:

```python
mlflow.openai.autolog()
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("/Shared/mn-demo-experiments")
mlflow.models.set_model(ModelWrapper())
```

These lines enable tracing and logging.
- `mlflow.openai.autolog()` enables detailed trace collection. Without it, you’d only get partial trace data through manually placed decorators.
- `set_tracking_uri()` and `set_experiment()` specify where to store traces Databricks. If you skip this step, traces will only appear when you call the endpoint interactively from a Databricks notebook — not when hitting it via API.
- `set_model()` sets the model object that is going to be logged.

**3. Restart the Python Library**

This next step might seem odd, but it’s crucial. Right after creating the `model.py` file, restart the Python library using `dbutils`:

```python
dbutils.library.restartPython()
```

If you skip this step, you’ll likely run into an import error the first time you reference `model.py`. The reason is subtle: when your compute session starts, Databricks appears to take a snapshot of your working directory. At that point, `model.py` doesn’t exist yet — so even after you write it, the environment doesn’t see it until you restart the interpreter. Restarting refreshes the session state so the new file becomes visible.

This same issue also applies if you modify `model.py` later. Suppose you rerun the `%%writefile` cell to overwrite the file with new code — unless you restart the Python library again, Databricks will continue using the old, cached version. It’s an easy mistake to make, but if you notice that your updates aren’t showing up, this is probably why.

**4. Register the model**

Once your `model.py` file is defined, the final step is to register it in Unity Catalog.

```python
import mlflow
from mlflow.models.resources import DatabricksServingEndpoint

import model

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
        pip_requirements=["databricks-openai"],
        resources=[DatabricksServingEndpoint(endpoint_name="databricks-gpt-oss-20b")],
    )
```

Since the previous `%%writefile` cell only wrote your model code to disk rather than executing it, you’ll need to re-import MLflow (and any dependencies) here. More importantly, notice the `import model` statement — this is a critical step. When Python imports the `model` module, it executes the last four lines we added earlier (`autolog`, `set_tracking_uri`, `set_experiment`, and `set_model`). This ensures that your experiment setup runs **before** `mlflow.start_run()` is called.

If you skip the import, MLflow will create two separate “Experiments”: one in your `/Shared` directory (as intended) and another automatically tied to your notebook. Only one of them will contain trace data, which makes debugging and cleanup a headache. Running `import model` keeps everything in sync and ensures that both your experiment metadata and your traces land in the right place.

You might be wondering why those four setup lines live inside the `model.py` file instead of the registration cell. I wondered the same thing and tried refactoring the code so the setup happened before the `start_run()` call. Surprisingly, the tracing functionality didn't work correctly anymore. It seems that MLflow requires those configuration calls to live inside the Python file that defines the model in order to properly associate the tracing context with the model object.

If you’ve seen Databricks examples that don’t include this `import model` step, that’s usually because they test the model in an earlier cell by calling its `predict()` method directly. Those tests implicitly execute the four setup lines, so the import isn’t needed later. However, if you skip the test cell and go straight to registration, you must import the module explicitly — otherwise, you’ll again end up with two experiments and inconsistent logging. It’s a small but important detail that saves a lot of confusion down the line.

Finally, the `example` variable defines a simple input that MLflow uses to validate your model during registration.

**What to Expect When You Run Model Registration Code**

When you execute the registration cell, MLflow will confirm that your run completed successfully and that a new model version has been created. During this process, you’ll also see a short message that says something like *“Running the predict function to generate output based on input example.”* This step validates that your model can execute end-to-end without errors by running a quick inference using the `example` input you defined earlier.

![Model Registration](/ai-gateway/new-model-registration.png)

In most runs, you’ll also see the model’s output (from the LLM) appear directly in the notebook output area. When I re-ran the notebook to capture the screenshot above, however, the output didn’t appear — so don’t panic if the same thing happens to you. I have found the tracing during the model registration can be inconsistent. If you re-run the cell later, you'll probably see the trace of the input example without any changes to the code.

Even if the model’s output doesn’t display inline, you can still confirm that the trace was generated successfully. Open the **Experiments** page in Databricks, and you should see a new experiment created under the location you specified in your model setup — in my case, `/Shared/mn-demo-experiments`, since that’s the path I set in the `mlflow.set_experiment()` call inside `model.py`.

I’ve added a short demo below showing how that looks in practice:

<div class="video-wrapper">
    <video controls playsinline>
        <source src="/ai-gateway/first-trace.mp4" type="video/mp4">
        Your browser does not support the video tag.
    </video>
</div>

As you can see in the video, the resulting trace contains much more than just the input prompt and model response — it includes structured metadata about the request, timestamps, token usage, model configuration, and more. These details are what make MLflow Tracing so valuable when debugging or tuning model behavior.

Once your model is registered, you can also interact with it directly from any Databricks notebook without re-importing the `model.py` file. Simply load it from Unity Catalog using `mlflow.pyfunc.load_model()` and call `predict()` with your desired input. Because tracing is baked into the model itself (via the `autolog()` configuration in `model.py`), each inference you run from a notebook will automatically produce a new trace entry — like the example shown below.

![Notebook Trace](/ai-gateway/notebook-trace.png)

### Creating the Endpoint

With your model now registered in Unity Catalog, the next step is to deploy it as a serving endpoint.

From the Databricks workspace, navigate to the **Serving** page and click **Create Endpoint**. Select your newly registered Python model from the list — you should see that there's now an option to enable tracing. Make sure this setting is turned on if it isn't already.

![Endpoint Creation Menu With Python Model](/ai-gateway/endpoint-creation-python-model.png)

Scroll down to the **AI Gateway** section to configure additional settings like the **Inference Table**, which records all requests and responses. This table is useful for auditing and performance tracking, though it doesn’t include the same level of detail as MLflow Traces. (Keep in mind that inference tables aren’t available on the free Databricks tier.)

Once you’ve configured your settings, click **Create** and wait for your model’s serving endpoint to finish deploying. When the status shows as **Active**, the endpoint is live and ready to accept REST API calls.

![Endpoint Creation Screen](/ai-gateway/ai-gateway-screen.png)

You can now test it using `curl` or your preferred REST client — I’ve been using the REST Client extension in VS Code. After sending a few requests, check your **Experiments** page under the shared experiment path — you’ll see fresh traces appear there. Here's a short demo showing how to send a request to the endpoint and then see the associated trace:

<div class="video-wrapper">
    <video controls playsinline>
        <source src="/ai-gateway/rest-demo.mp4" type="video/mp4">
        Your browser does not support the video tag.
    </video>
</div>

> Note: I am not sure why the trace in the video looks unusual. I have encountered this before and days later, it was fixed. So no need to worry. The main thing is that the trace was logged.

At this point, you’ve successfully built an endpoint with full Mosaic AI Gateway functionality and detailed tracing — all through a custom Python model. You're probably wondering why I am not recommending the python model. The issue is that I had a difficult time setting up streaming responses with the custom python model. If streaming had worked seamlessly, this might have been my final recommendation. But it didn’t — and that’s what led me to explore the ResponsesAgent next. 

### What Went Wrong with Streaming Requests

If you’ve looked at the documentation I referenced earlier, you may have noticed that the PythonModel class in MLflow also defines a `predict_stream` function that you can override to support streaming requests.

Here’s the basic idea: we implement the `predict_stream` function inside our model class. When a REST request includes a streaming parameter, MLflow will call `predict_stream` instead of predict, allowing the model to return partial results as they arrive.

Here’s how I initially implemented it inside the `ModelWrapper` class:

```python
def predict_stream(
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

When I imported the module directly in a Databricks notebook and invoked `predict_stream()` manually, everything worked perfectly. I saw the responses stream back in real time, and each run produced a complete MLflow trace showing every chunk of output.

<div class="video-wrapper">
    <video controls playsinline>
        <source src="/ai-gateway/python-streaming.mp4" type="video/mp4">
        Your browser does not support the video tag.
    </video>
</div>

This confirmed that the function worked when called directly from the notebook. I could see all the chunks streaming back one by one, and tracing behaved exactly as expected.

Encouraged by this, I decided to test the same functionality through other methods — and that’s where things started to fall apart.

**Testing Through the Loaded Model and REST API**

After registering the model in Unity Catalog, I tried invoking it again by loading it with `mlflow.pyfunc.load_model()` and calling `predict_stream`. This time, it failed.

![Notebook Trace](/ai-gateway/failed-predict-stream.png)

Interestingly, `predict()` continued to work just fine in this context — but `predict_stream()` didn’t. The same issue appeared when I called the endpoint through the REST API:

![Notebook Trace](/ai-gateway/bad-predict-stream-api.png)

At this point, I was puzzled. The function clearly worked in one context but failed in another. I briefly considered adding a streaming flag to the predict method itself (e.g., `predict(streaming=True)`), but that felt like a hack and went against the intended design of the MLflow API. It also wasn’t as clean as maintaining two clearly defined methods — one for standard predictions and one for streaming.

So I started digging into the root cause.

**Digging into the Cause**

Why didn’t `predict_stream()` work when the model was loaded from Unity Catalog?

The key detial lies in what `mlflow.pyfunc.load_model()` actually returns. According to the [MLflow docs](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#mlflow.pyfunc.load_model), it doesn’t return your `PythonModel` directly — it returns a `PyFuncModel`, a wrapper class that standardizes how models are called.

When you invoke `predict_stream()` on the loaded model, you’re actually calling the wrapper’s version of that function, which then delegates to your implementation. Unfortunately, something in that handoff — specifically in how inputs are validated and passed through — seems incompatible with the OpenAI-style message list I was using.

For anyone interested in exploring further, you can inspect the predict_stream implementation in the [PyFuncModel source code](https://mlflow.org/docs/latest/api_reference/_modules/mlflow/pyfunc.html). 

What frustrated me about this experience is that [MLFlow PythonModel documentation](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#mlflow.pyfunc.PythonModel) states that both predict and predict_stream accept PyFunc-compatible input. Since my input worked perfectly with predict, I expected it to work with predict_stream as well.

The [Inference API docs](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.pyfunc.html#pyfunc-inference-api) also claim that “a list of any type” should be valid input — further suggesting this should have worked.

**Where Things Stand**

To make `predict_stream()` work, I had two main options:

(1) Change its input format to something that MLflow’s wrapper would accept, or
(2) Modify `predict()` to handle streaming requests as well.

Both felt like poor tradeoffs. I didn’t want to maintain separate input schemas for `predict()` and `predict_stream()`. I also didn't like the idea of adding a “streaming” flag to `predict()` just to make it behave differently.

So while the custom Python model approach worked beautifully for standard requests — giving full control, transparency, and seamless Databricks integration — it simply wasn’t reliable for streaming. For most use cases, that limitation might not matter. But for my goal — supporting both standard and streaming completions — it was a dealbreaker.

So it was time to move on to Attempt 3: the ResponsesAgent.

## Attempt 3: Responses Agent

The Databricks documentation includes a “simple” guide for creating a Responses Agent endpoint. It’s a good starting point, but I’ll admit — I wasn’t a huge fan of the sample notebook. The call stack for basic predictions felt unnecessarily complex, and several unused libraries made it tough to tell which parts actually mattered. At one point, I even caught myself wondering if those dependencies had hidden purposes.

That said, the example still illustrates the core concept well. I adapted it into a cleaner, minimal version that focuses on the essentials — the one we’ll walk through here.

For anyone curious, you can find Databricks’ original example notebook here: https://docs.databricks.com/aws/en/notebooks/source/mlflow3/simple-agent-mlflow3.html.

### Implementing Responses Agent

As before, we’ll start our notebook with an installation cell — but this time we’ll add the `databricks-agents` library alongside `databricks-openai`. Following the installation cell, we have the `%%writefile` cell which writes our code to the `model.py` file. 

> Note: In Databricks, .py files can be loaded as notebooks. Cells are seperated by lines with the text `# COMMAND ----------`. As such, the following code cell is representing two notebook cells.

```python
%pip install -U -qqqq databricks-openai databricks-agents
dbutils.library.restartPython()

# COMMAND ----------

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

1. **Inheritance**: our class now inherits from `ResponsesAgent` instead of `PythonModel`.

2. **Types**: `predict()` accepts a single parameter of type `ResponsesAgentRequest` and returns a `ResponsesAgentResponse`.

3. **Translation**: before sending messages to the model, it calls `self.prep_msgs_for_cc_llm()` — a helper function that quietly handles a lot of complexity.

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

These schemas define how `ResponsesAgentRequest` and `ResponsesAgentResponse` are structured.  Note that both can include additional parameters (like temperature or max_output_tokens), so it’s worth checking the [full API reference](https://mlflow.org/docs/latest/api_reference/python_api/mlflow.types.html#mlflow.types.responses.ResponsesAgentRequest) for details.

**The Role of `prep_msgs_for_cc_llm()`**

OpenAI recently introduced a new **Responses API** to improve upon their existing **Chat Completions API**. Databricks’ `ResponsesAgent` class and its request/response types are built to align with this newer API. However, the two APIs expect slightly different input formats.

- The Chat Completions API expects a list of messages.
- The Responses API can accept a single string or a structured schema.

This means that a request formatted for the Responses API won’t necessarily work with the Chat Completions API. That’s where the `prep_msgs_for_cc_llm()` (short for prepare messages for chat completion LLM) comes in. It automatically converts input from the Responses format to the Chat Completions format. Fortunately, you don’t have to define it yourself; it’s inherited from the ResponsesAgent base class.

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

Here’s why: the `WorkspaceClient` from the Databricks SDK provides a wrapped client that can access registered models inside your workspace, regardless of where they’re hosted. It’s convenient because you don’t need to configure environment variables for authentication.

My guess is that his SDK client hasn’t been fully updated to support the new Responses API. As a result, calling client.`responses.create()` currently raises an error — even with simple requests.

This theory is further supported by the official Databricks notebooks: all of them use the `ResponsesAgent` class (which matches the Responses API schema) but still call the Chat Completions API using the `prep_msgs_for_cc_llm` function behind the scenes.

**A Note on Alternative Clients**

There is another way to call the Responses API in Databricks — by using the standard OpenAI client instead of the SDK:

```python
from openai import OpenAI

client = OpenAI()
```

This approach works when calling external models that support the Responses API (note that some older models don’t).  However, it requires you to set environment variables for authentication. You also need access to an external model, something I didn't want to set up for this blog since Databricks has so many foundational ones available. 

**Wrapping Up `predict()`**

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

This step ensures the returned object follows the Responses schema, even though the underlying model call still uses Chat Completions. The helper create_text_output_item() builds a properly structured response entry — one of several output types available. You can find the full list in the [ResponsesAgent documentation](https://mlflow.org/docs/latest/genai/serving/responses-agent/#creating-agent-output).

Also don’t worry about losing details from the original response. Even though we return only the generated text, MLflow’s tracing automatically records the full request, response, and metadata — giving you complete visibility into every call.

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
