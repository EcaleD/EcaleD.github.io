+++
author = "Xiaokai Dong"
title = "The Loop Is the Agent"
date = "2026-06-13"
description = "What finally made agents click for me was the loop that turns one tool result into the next decision."
image = "header-image-agent-loop-mobius-warm.png"
tags = [
    "AI",
    "Agents",
    "Automation",
]
categories = [
    "AI Engineering",
    "Software Design",
]
+++

<!--more-->

For a while, I thought an agent was basically a chatbot with tools. Let the model search the web, read a file, or run a command, and there you have it: an agent.

That was a useful shortcut, but it started to feel incomplete once I paid attention to what happened after a tool ran. The result had to go somewhere. The model had to see it, decide whether the task was finished, and perhaps choose another action. Then the same thing happened again.

That repeating process is the **agent loop**. It is a small idea, but it explains a lot about why agents can handle tasks that do not fit neatly into a fixed sequence.

## A simple way to understand the loop

Say I ask an assistant to find out why a documentation site no longer builds. A single model response might give me a list of likely causes. A working agent can actually follow the evidence:

```text
Run the documentation build
    -> read the error
    -> find the relevant file
    -> inspect the file and nearby resources
    -> make a small change
    -> run the build again
    -> either continue or stop
```

The path changes as new information arrives. A missing-image error may lead to the article bundle. The next build may reveal a front-matter problem instead. There is no point scripting every possible step in advance because the build itself keeps changing what the agent knows.

This is close to the idea behind [ReAct](https://arxiv.org/abs/2210.03629): reasoning and actions are interleaved, so an observation from the outside world can change the next decision.

This also helped me separate an agent from a workflow. A workflow already knows that step A is followed by step B. An agent is given a goal and a set of allowed actions, then chooses its next move from what it has just observed. Anthropic draws a similar line between predefined workflows and systems in which a model directs its own tool use in [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents).

## What sits around the model

On paper, the architecture is surprisingly small:

```text
User goal
   |
   v
Runner -> model -> final response
   ^         |
   |         v
   +-- observation <- tool executor <- tool call
```

The model either answers or asks to use a tool. The runner—the ordinary application code around it—executes an allowed action and feeds the result back into the next model call.

That simple loop still needs some structure around it:

- **Instructions** describe the goal and the limits.
- **State** keeps earlier model output, tool calls, and results.
- **Tools** expose a small set of actions with structured inputs.
- **Policies** decide which requested actions are allowed.
- **Validation** checks the result against something more reliable than the model's own confidence.
- **Stop conditions** keep an unproductive loop from running forever.

I find this useful because it puts the model back in proportion. It makes decisions inside the loop, while the surrounding software decides what it may touch and what counts as success.

## What the loop looks like in code

A framework can hide most of the plumbing, but I wanted to see the loop without that layer. The following sketch uses the current Responses API function-calling pattern from the [official OpenAI documentation](https://developers.openai.com/api/docs/guides/function-calling):

```python
import json
import os

from openai import OpenAI

client = OpenAI()
model = os.environ["OPENAI_MODEL"]

# Defined elsewhere by the application:
# - tools: JSON schemas for the available functions
# - dispatch(name, arguments): validates and executes one function
# - build_has_passed(): returns True only after a successful build

items = [{
    "role": "user",
    "content": "Find and fix the documentation build error."
}]

for turn in range(8):
    response = client.responses.create(
        model=model,
        instructions=(
            "Work only inside the documentation directory. "
            "Make the smallest necessary change. "
            "Do not report success until the build passes."
        ),
        tools=tools,
        input=items,
    )

    # Preserve the model output, including any function calls.
    items += response.output
    calls = [item for item in response.output
             if item.type == "function_call"]

    if not calls:
        if build_has_passed():
            print(response.output_text)
            break

        items.append({
            "role": "developer",
            "content": (
                "Validation failed: the documentation build has not "
                "passed yet. Continue investigating."
            ),
        })
        continue

    for call in calls:
        try:
            arguments = json.loads(call.arguments)
            result = dispatch(call.name, arguments)
        except Exception as error:
            result = {"ok": False, "error": str(error)}

        items.append({
            "type": "function_call_output",
            "call_id": call.call_id,
            "output": json.dumps(result),
        })
else:
    raise RuntimeError("The agent exceeded the turn limit")
```

The missing pieces—tool schemas, file handling, and the actual build command—belong to the application. The loop itself is there: call the model, execute its requested functions, append their outputs, and call the model again. When the application manages the history, it also needs to preserve `response.output`, including the function-call items.

One detail matters more than it first appears. The model can decide that it has no more tools to call, but the application does not accept success until `build_has_passed()` agrees. A fluent final answer is not proof that the site builds.

## A practical application: a documentation build repair agent

For a first version, I would keep the toolset deliberately boring:

```text
run_doc_build()
read_file(path)
search_docs(query)
replace_text(path, old, new)
show_diff()
```

The names look broad, so the implementations should be narrow. `read_file` and `replace_text` reject paths outside the documentation directory. `replace_text` touches only supported text files and fails when the expected old text is missing. `run_doc_build` returns an exit code and a limited amount of output. There is no reason for this agent to commit, push, or publish anything.

Suppose a Hugo article still refers to `header.png` after the file was renamed to `header-v2.png`. A run could look like this:

```text
1. run_doc_build()
   Observation: failed to resolve image resource "header.png"

2. search_docs("header.png")
   Observation: the reference appears in one article's front matter

3. read_file("content/post/example/index.md")
   Observation: image = "header.png"

4. replace_text(..., "header.png", "header-v2.png")
   Observation: one replacement made

5. run_doc_build()
   Observation: exit code 0

6. show_diff()
   Final response: explain the change and present the diff for review
```

The flexible part is deciding where to look next. The less flexible parts are just as important: file access is restricted, the edit is an exact replacement, Hugo produces the exit code, and the final diff stays available for review.

This is still a toy-sized agent. A real version would need timeouts, output limits, logs, protection against repeated actions, and probably an approval step before accepting a patch. Tools with external side effects would need their own permissions rather than inheriting access from the rest of the loop.

## The part I had overlooked

I used to look first at the model when trying to understand an agent. Now I am more interested in the code around it. What does the model get to observe? Which actions can it take? What happens when a tool fails? What evidence is required before the task is considered finished?

Those choices shape the agent at least as much as the prompt does. They also make agent development feel less mysterious to me. The model supplies some flexibility; familiar software practices supply scope, checks, and a way to stop.

Perhaps that is the most useful way to think about an agent loop. It is not a route to unlimited autonomy. It is a controlled way to let each result influence the next step, while keeping enough evidence around to know whether anything has actually been accomplished.
