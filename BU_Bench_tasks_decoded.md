# BU Bench — Decoded Tasks & Prompt Flow

Summary
- I decoded `BU_Bench_V1.enc` and inspected how the benchmark loads tasks and constructs prompts for the LLM/agent.
- This document explains, step‑by‑step, how tasks are decrypted, the task JSON shape, how `run_eval.py` hands a task to the `Agent`, how the agent builds system+user messages, how the model is invoked, and how judging works.

1) Where tasks live and how they are decoded
- Encrypted tasks file: [BU_Bench_V1.enc](BU_Bench_V1.enc)
- The benchmark decodes the file using Fernet with a key derived from the string `BU_Bench_V1` in `run_eval.py`.
  - See the loader in [run_eval.py](run_eval.py#L63-L66):

```py
def load_tasks() -> list[dict]:
    key = base64.urlsafe_b64encode(hashlib.sha256(b"BU_Bench_V1").digest())
    encrypted = base64.b64decode(TASKS_FILE.read_text())
    return json.loads(Fernet(key).decrypt(encrypted))
```

2) Task JSON shape (what each task contains)
- Typical keys you will see in each task object:
  - `confirmed_task` (string): the final user-facing instruction the agent must satisfy.
  - `task_id` (uuid string): unique id for the task.
  - `category` (string): bucket like `WebBenchREAD`, `InteractionTests`, `GAIA`, `OM2W2`, etc.
  - Optional: `answer` (string) — a ground-truth answer used for verification on closed questions.

Example (fake task — not from the benchmark):

```json
{
  "confirmed_task": "Find the next three public 5K races in Cityville and list their names, dates, and registration links.",
  "category": "WebBenchREAD",
  "task_id": "fake-task-0001"
}
```

3) How `run_eval.py` hands tasks to the agent
- `run_eval.py` loads tasks via `load_tasks()` (link: [run_eval.py](run_eval.py#L63-L66)).
- For each task the benchmark does roughly:
  1. Create a browser instance (`create_browser`).
  2. Instantiate an `Agent` passing `task=task["confirmed_task"]` and an LLM instance.
     - See the agent creation in [run_eval.py](run_eval.py#L107-L111):

```py
agent = Agent(
    task=task["confirmed_task"],
    llm=llm or ChatBrowserUse(model="bu-2-0"),
    browser=browser,
)
agent_history = await asyncio.wait_for(agent.run(), timeout=TASK_TIMEOUT)
```

- Important: the `task` given to `Agent` is a plain text instruction (the `confirmed_task` string). If a task contains a ground-truth `answer` field, `run_eval.py` later passes that to the judge (see below).

4) What the Agent does with the task (how prompts/messages are built)
- The agent implementation (in the installed `browser_use` package) constructs the LLM input on each step as:
  - One System message (role/instructions) and
  - One State/User message that contains the task and a structured snapshot of agent/browser state.

- System message
  - Built from templates (the `SystemPrompt` logic selects the right template based on model type and flags such as `is_browser_use_model`, `is_anthropic_4_5`, `flash_mode`, and `use_thinking`).
  - The system templates encode rules like "be very picky about task satisfaction", failure conditions, and output format expectations. (These templates are part of the `browser_use` package's `system_prompts` collection.)

- State / User message (exact structured tags inserted)
  - The state message is built by `AgentMessagePrompt` / `MessageManager` and contains a sanitized, structured description with *explicit tags* that the LLM receives, for example:

```
<agent_history>
  ...human-readable history items (previous steps/results)...
</agent_history>

<agent_state>
  <user_request>
    {the original task text}
  </user_request>
  <file_system>
    {todo & available files listing}
  </file_system>
  <step_info>Today:YYYY-MM-DD</step_info>
</agent_state>

<browser_state>
  <page_stats>...links, interactive elements, iframes...</page_stats>
  Available tabs:
  Tab abcd: https://... - Page title
  Interactive elements: ...
</browser_state>

<read_state>
  {extracted text from read actions (if any)}
</read_state>

<page_specific_actions>
  {short page-specific action hints (if any)}
</page_specific_actions>
```

  - When images/screenshots are included, the message is sent as a multipart `UserMessage` composed of `ContentPartTextParam` entries (the state text) and `ContentPartImageParam` entries (base64 PNGs). The agent chooses whether to include screenshots based on `use_vision` and page context.

5) Agent system prompt (flash mode)
- When the agent runs with `flash_mode=True` the `SystemPrompt` selects the `system_prompt_flash.md` template. Below is the exact template used for flash mode in this environment:

```text
You are an AI agent designed to operate in an iterative loop to automate browser tasks. Your ultimate goal is accomplishing the task provided in <user_request>.
<language_settings>Default: English. Match user's language.</language_settings>
<user_request>Ultimate objective. Specific tasks: follow each step. Open-ended: plan approach.</user_request>
<browser_state>Elements: [index]<type>text</type>. Only [indexed] are interactive. Indentation=child. *[=new.</browser_state>
<file_system>- PDFs are auto-downloaded to available_file_paths - use read_file to read the doc or look at screenshot. You have access to persistent file system for progress tracking. Long tasks >10 steps: use todo.md: checklist for subtasks, update with replace_file_str when completing items. When writing CSV, use double quotes for commas. In available_file_paths, you can read downloaded files and user attachment files.</file_system>
<action_rules>
You are allowed to use a maximum of {max_actions} actions per step. Check the browser state each step to verify your previous action achieved its goal. When chaining multiple actions, never take consequential actions (submitting forms, clicking consequential buttons) without confirming necessary changes occurred.
</action_rules>
<output>You must respond with a valid JSON in this exact format:
{{
  "memory": "Up to 5 sentences of specific reasoning about: Was the previous step successful / failed? What do we need to remember from the current state for the task? Plan ahead what are the best next actions. What's the next immediate goal? Depending on the complexity think longer. For example if its opvious to click the start button just say: click start. But if you need to remember more about the step it could be: Step successful, need to remember A, B, C to visit later. Next click on A.",
  "action":[{{"navigate": {{ "url": "url_value"}}}}]
}}</output>
```

6) Example conversation (fake task) — exact message objects and a fake model response
Below is a syntactic example of the exact messages the agent sends to the LLM at step 0, and a plausible structured response the LLM returns. This is a fabricated example to illustrate format; it does not leak any real benchmark tasks or results.

- Messages sent to the LLM (step 0):

```py
# Message list passed to llm.ainvoke(...)
[
  SystemMessage(content=SYSTEM_PROMPT_TEXT),
  UserMessage(content=[
    ContentPartTextParam(text="""
<agent_history>
</agent_history>

<agent_state>
<user_request>
Find the next three public 5K races in Cityville and list their names, dates, and registration links.
</user_request>
<file_system>
No files available
</file_system>
<step_info>Today:2026-04-14</step_info>
</agent_state>

<browser_state>
<page_stats>10 links, 5 interactive, 0 iframes, 120 total elements</page_stats>
Available tabs:
Tab 1: https://example-search.com - Search results
</browser_state>
"""),
    ContentPartImageParam(image_url=ImageURL(url='data:image/png;base64,<screenshot-b64-placeholder>', media_type='image/png'))
  ])
]
```

- Fake LLM response (structured `AgentOutput` the agent expects):

```json
{
  "memory": "Loaded search page showing results. Remember to open each result and gather registration links. Next action: navigate to first result.",
  "action": [
    {"navigate": {"url": "https://example-search.com/result-1"}},
    {"extract": {"selector": "css:.race-details", "attribute": "text"}},
    {"click": {"selector": "css:a.register-button"}},
    {"done": {"success": false, "text": "Found 1/3 races, continuing."}}
  ]
}
```

- Tool invocation and return (what the agent does with the action list):

```py
# Pseudocode of the execution loop
for action in parsed.action:
    result = await tools.act(action=action, browser_session=browser_session, file_system=file_system, page_extraction_llm=page_extraction_llm, sensitive_data=sensitive_data, available_file_paths=available_file_paths)
    # result is an ActionResult object, e.g.:
    # ActionResult(is_done=False, success=True, extracted_content='Race A - 2026-05-01 - https://register.example/A', images=[], error=None)
    # Agent updates history with result and proceeds to next step
```

- Example `ActionResult` (fake):

```json
{
  "is_done": false,
  "success": true,
  "extracted_content": "Cityville 5K - 2026-05-01 - https://register.example/A",
  "images": [],
  "error": null
}
```

7) Judge invocation example (fake)
- After the agent finishes or marks `done`, `run_eval.py` constructs judge messages using `construct_judge_messages(...)`. The judge receives a `SystemMessage` (judge instructions) and a `UserMessage` that contains a textual `<task>` block, `<agent_trajectory>` block (the agent's history), `<final_result>`, and attached screenshots.

```py
judge_messages = construct_judge_messages(
    task=fake_task_text,
    final_result=fake_final_text,
    agent_steps=fake_agent_steps_list,
    screenshots_b64=[...],
    ground_truth=None,
)
# Then: response = await JUDGE_LLM.ainvoke(judge_messages, output_format=JudgementResult)
```

- Example judge response (fake `JudgementResult`):

```json
{
  "reasoning": "Agent found 2 of 3 races and correctly extracted registration links; missed one listing due to pagination.",
  "verdict": false,
  "failure_reason": "Missing one required item (pagination not followed)",
  "impossible_task": false,
  "reached_captcha": false
}
```

8) Notes & next steps
- `SystemMessage`, `UserMessage`, `ContentPartTextParam`, and `ContentPartImageParam` are types from `browser_use.llm.messages` used to compose the final payload sent to the LLM.
- If you want, I can also generate a script that prints the exact step-0 payload for a chosen fake task and saves it to disk so you can paste it to a model for testing.

---
Files referenced in this explanation:
- `run_eval.py` (task loader)  
- `run_eval.py` (Agent instantiation)  
- `run_eval.py` (judge invocation)  
- `judge.py` (construct_judge_messages)  
- `judge.py` (JudgementResult schema)  
- Decoded tasks saved as `BU_Bench_V1.decoded.json` (not included here)