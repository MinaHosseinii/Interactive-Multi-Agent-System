# Interactive-Multi-Agent-System

The setup runs Qwen3.5-0.8B-Q8_0 locally using llama.cpp. We’ll reuse the same loaded model for all agent roles, so no paid cloud API is required.

| Stage | What We build | Main challenge |
|---|---|---|
| 1. Model connection | A small Python client | Distinguishing incomplete output from a usable answer |
| 2. Tools from scratch | Inspection and statistical functions | Validating arguments and handling missing data |
| 3. Agent loop | Model chooses a tool, receives results, and continues | Wrong tool choices, invented answers, repeated calls |
| 4. LangChain integration | Tool definitions and native tool-call messages | Understanding what the framework handles—and what it does not |
| 5. LangGraph coordination | Supervisor, specialists, shared state, conditional edges | Passing the correct data and controlling execution |
| 6. Review and evaluation | Reviewers, bounded revisions, human approval, test cases | Checking correctness without trusting another model blindly |
