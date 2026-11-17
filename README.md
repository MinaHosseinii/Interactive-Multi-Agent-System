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

The intended behavior is to inspect the dataset, calculate the mean, and then answer. Either tool order is acceptable here.

What makes this a loop? After each tool result, Python calls the model again with the accumulated observations. The model chooses its next action; Python limits execution to five decisions.

Two limitations remain deliberately visible:
- A successful tool call does not prove that every part of the question was answered.
- The model’s final explanation can still misreport correct tool results.
We’ll evaluate those separately rather than treating "ok": true as proof of correctness.

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

The calculation is correct, but the agent is stuck repeating the same action. It never requested the order count or produced a final answer. The five-step limit correctly stopped execution.

This is a decision-loop failure, not a calculation error. Our prompt said “don’t repeat,” but Python did not enforce that rule.

Let’s add duplicate-call protection and give the model explicit feedback about completed work.

I addressed the repeated tool calls by adding a record of successful requests and their results. Before executing a request, Python checks whether the exact same tool and arguments have already succeeded. If they have, it blocks execution and returns feedback containing the previous result, asking the model to choose another action or finish. I also included completed calls in the model’s instructions so it can see what has already been done, while keeping the five-step limit to prevent an endless loop. This prevents duplicate calculations, but we still need to test whether the model uses the feedback to complete the task.

Your question: How many orders are there, and what is their average recorded value?

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
  "ok": false,
  "error": "Duplicate call rejected. This operation already succeeded. Use its result instead of repeating it.",
  "previous_result": {
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
  "ok": false,
  "error": "Duplicate call rejected. This operation already succeeded. Use its result instead of repeating it.",
  "previous_result": {
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
  "ok": false,
  "error": "Duplicate call rejected. This operation already succeeded. Use its result instead of repeating it.",
  "previous_result": {
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
  "ok": false,
  "error": "Duplicate call rejected. This operation already succeeded. Use its result instead of repeating it.",
  "previous_result": {
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
    }
  ]
}

Python calculated the mean only once. But the model still failed to choose a new action. This test shows that feedback alone did not make your small model complete the task.

I addressed the repeated tool calls by replacing the open-ended agent loop with a constrained workflow. The model now classifies the request once—for example, as count_and_mean—and Python maps that classification to a predefined sequence of tools. In this run, Python executed dataset inspection and mean calculation once each, then constructed the final answer directly from their results: five orders and an average recorded value of €125, with one missing value excluded. This reduced the model’s autonomy while giving the application control over execution order, stopping, and numerical reporting.

Your question: How many orders are there, and what is their average recorded value?

MODEL CLASSIFICATION:
{"task": "count_and_mean"}

TOOL: inspect_dataset
{
  "ok": true,
  "result": {
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
}

TOOL: calculate_mean
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
  "ok": true,
  "task": "count_and_mean",
  "answer": "There are 5 orders. The average recorded order value is \u20ac125.00, based on 4 recorded values, with 1 missing value(s) excluded.",
  "results": {
    "inspect_dataset": {
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
    },
    "calculate_mean": {
      "column": "order_value_eur",
      "mean": 125.0,
      "values_used": 4,
      "missing_excluded": 1
    }
  }
}

We implemented a model-driven loop in stage 3.

**Stage 4 of 6: Introduce LangChain tools**
Until now, we manually defined tool metadata and checked function arguments. LangChain’s @tool wrapper can generate an input schema from a function’s type hints and provide a standard .invoke() interface. It does not improve the model’s decisions automatically.

**python -m pip install "langchain-core>=1,<2"**

I wrapped our existing Python functions as LangChain tools using the @tool decorator, keeping the calculations unchanged. LangChain uses each function’s name, description, and type hints to generate an input schema and provides a standard .invoke() method for execution. For example, calculate_mean requires a string argument called column, so a request without that argument is rejected before the calculation runs. The underlying Python function still checks whether the column exists and contains numeric data. This step introduces LangChain’s tool interface; it does not yet involve the model or native tool calling.

python langchain_tools.py
TOOL NAME:
calculate_mean

TOOL DESCRIPTION:
Calculate a numeric column's mean, excluding missing values.

INPUT SCHEMA:
{
  "description": "Calculate a numeric column's mean, excluding missing values.",
  "properties": {
    "column": {
      "title": "Column",
      "type": "string"
    }
  },
  "required": [
    "column"
  ],
  "title": "calculate_mean",
  "type": "object"
}

VALID EXECUTION:
{
  "column": "order_value_eur",
  "mean": 125.0,
  "values_used": 4,
  "missing_excluded": 1
}

INVALID EXECUTION:
ValidationError
1 validation error for calculate_mean
column
  Field required [type=missing, input_value={}, input_type=dict]
    For further information visit https://errors.pydantic.dev/2.14/v/missing

I converted the existing mean-calculation function into a LangChain tool using @tool. LangChain generated its name, description, and input schema, which specifies that a string argument called column is required. Calling .invoke({"column": "order_value_eur"}) successfully executed the Python function and returned a mean of €125 from four recorded values, excluding one missing value. Calling .invoke({}) produced a validation error because the required argument was absent. This verified tool execution and input validation; the model was not involved yet.

Both checks passed: the tool returned €125 for valid input and rejected a missing argument.
**Stage 4: Native tool calling**
Previously, the model printed ordinary JSON that we parsed ourselves. Now we’ll test the API’s structured tool_calls field.

For this first experiment, we explicitly require calculate_mean. We’re testing the protocol—not independent tool selection.

In native_tool_call.py, we connected LangChain to your local Qwen model through llama-server. The script first retrieves the model’s ID, then uses bind_tools() to send the description and input schema of calculate_mean to the model. For this experiment, we explicitly require that tool, so we are testing native tool calling rather than letting the model choose between tools. The script reads the structured request from reply.tool_calls, checks that it contains exactly one valid request for calculate_mean, and executes it in Python using .invoke(call). LangChain returns a ToolMessage containing the calculation result and the matching tool_call_id, which links the result to its request. The script prints that result; it does not yet send it back to the model for a final explanation.

**python -m pip install "langchain-openai>=1,<2"**

python native_tool_call.py
MODEL TEXT:


NATIVE TOOL REQUESTS:
[
  {
    "name": "calculate_mean",
    "args": {
      "column": "order_value_eur"
    },
    "id": "ooe5Dzxsgbb2MEaqRUllyu2rkw5WP2a8",
    "type": "tool_call"
  }
]

RESULT MESSAGE TYPE:
ToolMessage

MATCHING TOOL CALL ID:
ooe5Dzxsgbb2MEaqRUllyu2rkw5WP2a8

TOOL RESULT:
{"column": "order_value_eur", "mean": 125.0, "values_used": 4, "missing_excluded": 1}
(.venv) minahosseini@Minas-MacBook-Air agent_course_v2 % 

Native tool calling worked with your local model.
- Qwen returned a structured request for calculate_mean.
- Python executed the function and calculated €125.
- LangChain returned a ToolMessage with the same call ID, linking the result to the request.
The empty MODEL TEXT is expected here: the response contained a tool request rather than a written answer. Remember, we explicitly required this tool; this test did not measure independent tool selection.

python native_tool_call.py
MODEL TEXT:


NATIVE TOOL REQUESTS:
[
  {
    "name": "calculate_mean",
    "args": {
      "column": "order_value_eur"
    },
    "id": "BJB197EHj1bm4D4Gt2q89mILEI5NuO8o",
    "type": "tool_call"
  }
]

RESULT MESSAGE TYPE:
ToolMessage

MATCHING TOOL CALL ID:
BJB197EHj1bm4D4Gt2q89mILEI5NuO8o

TOOL RESULT:
{"column": "order_value_eur", "mean": 125.0, "values_used": 4, "missing_excluded": 1}

FINAL ANSWER:
The average recorded order value is **125.0 EUR**.
*   **Values used**: 4
*   **Missing values excluded**: 1







