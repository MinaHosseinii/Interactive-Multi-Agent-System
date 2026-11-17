# Interactive-Multi-Agent-System

****Stage 1.** Model Client**
Python reached your local model, received a completed response, and the model correctly acknowledged that it had no dataset.

The setup runs Qwen3.5-0.8B-Q8_0 locally using llama.cpp. We’ll reuse the same loaded model for all agent roles, so no paid cloud API is required.

| Stage | What We build | Main challenge |
|---|---|---|
| 1. Model connection | A small Python client | Distinguishing incomplete output from a usable answer |
| 2. Tools from scratch | Inspection and statistical functions | Validating arguments and handling missing data |
| 3. Agent loop | Model chooses a tool, receives results, and continues | Wrong tool choices, invented answers, repeated calls |
| 4. LangChain integration | Tool definitions and native tool-call messages | Understanding what the framework handles—and what it does not |
| 5. LangGraph coordination | Supervisor, specialists, shared state, conditional edges | Passing the correct data and controlling execution |
| 6. Review and evaluation | Reviewers, bounded revisions, human approval, test cases | Checking correctness without trusting another model blindly |


The finished prototype will support one dataset at a time, inspection, descriptive and grouped analysis, a local dashboard, and bounded review loops. Cleaning will require an explicit policy and human approval. Broader statistics and deployment can come later.

We’ll distinguish a model limitation, a prompt problem, a tool bug, and a workflow bug

if a model prints a JSON request, the function has not executed yet. Indeed, the model has requested a tool call. Python must validate the request, execute the function, and return its result.

Use llama-server, rather than llama-cli, because Python needs the HTTP interface.

**Stage 2. Build tools independently of the model**
A tool begins as an ordinary Python function. We’ll verify its behavior first, then let the model request it.

INSPECTION:
{
  "row_count": 5,
  "columns": [
    "customer_type",
    "order_id",
    "order_value_eur"
  ],
  "missing_counts": {
    "customer_type": 0,
    "order_id": 0,
    "order_value_eur": 1
  }
}

CALCULATION:
{
  "column": "order_value_eur",
  "mean": 125.0,
  "values_used": 4,
  "missing_excluded": 1
}


**Stage 2. Build a tool registry**
A tool registry maps permitted names to Python functions. It lets our application turn a structured request into an actual function call.

REQUEST:
{
  "tool": "calculate_mean",
  "arguments": {
    "column": "order_value_eur"
  }
}
RESULT:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

REQUEST:
{
  "tool": "calculate_mean",
  "arguments": {
    "column": "customer_type"
  }
}
RESULT:
{
  "ok": false,
  "error": {
    "type": "ValueError",
    "message": "Column 'customer_type' is not numeric."
  }
}
The executor calculates a valid request and returns an understandable error for an invalid one.

**Stage 3: Let the model request a tool**
We’ll first make one model decision, then execute its request. Once that works, we’ll add the loop that returns observations to the model.

**Your question: What is the average order value?**

MODEL OUTPUT:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}

APPLICATION RESULT:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

| Component | Responsibility |
|---|---|
| Model | Request `calculate_mean` for `order_value_eur` |
| Parser | Convert the returned JSON text into a dictionary |
| Executor | Validate the tool and arguments |
| Python function | Calculate €125 from four recorded values |

This is manual tool calling: the model generates ordinary text following our JSON convention. We have not introduced native API tool calls or an autonomous loop yet.

The model chose the right tool, and Python calculated the correct result. Now we’ll return that result to the model and let it decide whether to request another tool or finish.
**Stage 3: Add the agent loop**
**question 1:** What is the average order value?

MODEL OUTPUT:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}

APPLICATION RESULT:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}


**Your question 2:** How many orders are there, and what is their average recorded value?

STEP 1 — MODEL:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}
APPLICATION FEEDBACK:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

STEP 2 — MODEL:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}
APPLICATION FEEDBACK:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

STEP 3 — MODEL:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}
APPLICATION FEEDBACK:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

STEP 4 — MODEL:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}
APPLICATION FEEDBACK:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

STEP 5 — MODEL:
{"tool": "calculate_mean", "arguments": {"column": "order_value_eur"}}
APPLICATION FEEDBACK:
{
  "ok": true,
  "result": {
    "column": "order_value_eur",
    "mean": 125.0,
    "values_used": 4,
    "missing_excluded": 1
  }
}

APPLICATION RESULT:
{
  "ok": false,
  "reason": "Maximum number of decisions reached.",
  "observations": [
    {
      "ok": true,
      "result": {
        "column": "order_value_eur",
        "mean": 125.0,
        "values_used": 4,
        "missing_excluded": 1
      }
    },
    {
      "ok": true,
      "result": {
        "column": "order_value_eur",
        "mean": 125.0,
        "values_used": 4,
        "missing_excluded": 1
      }
    },
    {
      "ok": true,
      "result": {
        "column": "order_value_eur",
        "mean": 125.0,
        "values_used": 4,
        "missing_excluded": 1
      }
    },
    {
      "ok": true,
      "result": {
        "column": "order_value_eur",
        "mean": 125.0,
        "values_used": 4,
        "missing_excluded": 1
      }
    },
    {
      "ok": true,
      "result": {
        "column": "order_value_eur",
        "mean": 125.0,
        "values_used": 4,
        "missing_excluded": 1
      }
    }
  ]
}





