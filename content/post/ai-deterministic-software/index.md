+++
author = "Xiaokai Dong"
title = "AI Should Not Replace Deterministic Software"
date = "2026-03-23"
description = "A practical way to decide when AI adds value—and when predictable software may be the safer choice."
tags = [
    "AI",
    "Software Engineering",
]
categories = [
    "Vibe Coding",
    "Doc Workflow",
]
image = "header-image-deterministic-ai.png"
+++

<!--more-->

When I built a small AI translation service for our documentation workflow, I kept asking how much of the process the model could take on.

The model could translate content, generate code, explain bugs, and suggest architecture. That made it tempting to also use it to detect changed files, protect Markdown syntax, decide what should be translated, and validate the output.

As the workflow took shape, however, I realized that those tasks did not all benefit from the same kind of system. In translation, variation and judgment could be useful. When preserving a code block or checking whether a file had changed, an unexpected result was simply an error.

That experience led me to a narrower question: not whether AI *can* perform a task, but whether its flexibility improves that particular task. It can be useful when the work involves language, context, or several acceptable answers. When the expected behavior can be described by stable rules, a deterministic approach may be easier to test, explain, and maintain.

## What deterministic software gives us

Deterministic software is predictable. Under defined conditions, the same input should lead to the same output.

That sounds less exciting than an AI agent, but predictability is one of the foundations of reliable systems. It lets us:

- write exact tests
- reproduce failures
- trace how a result was produced
- estimate time and cost
- change one part of a system without guessing what else may change

An LLM-based feature is usually less predictable at the application level. Even with a detailed prompt, the model may interpret an instruction in an unexpected way, add something that was not requested, or handle two similar inputs differently.

That flexibility can be valuable for translation and other language tasks. It becomes less helpful when the acceptable result is narrow and needs to be reproduced exactly.

## A lesson from incremental translation

One requirement for my translation service was incremental translation. I did not want to translate an entire document again whenever a small part changed.

My first definition of the feature was too vague. The generated implementation used a translation-memory-style mechanism, which did not fit the actual workflow. I then had to define the logic more clearly and ask AI to rebuild that part of the service.

The important question was not:

> Can an LLM decide whether this file needs translation?

It was:

> What exact change should trigger translation?

Once the rules were clear, I could express the feature as a normal software problem:

```text
new document         -> translate
source text changed  -> translate changed content
front matter changed -> follow a defined metadata rule
whitespace changed   -> skip
code only changed    -> skip or handle with a separate rule
```

Git can identify changed files. A Markdown parser can separate prose from code blocks, links, and front matter. A rule engine can decide which content enters the translation queue.

In this workflow, I did not see a clear benefit in adding an LLM to these steps. Doing so would have introduced another layer of interpretation into behavior I wanted to test with exact inputs and outputs.

## Where AI may add value

Translation itself is different. A sentence can have several valid translations. The right choice may depend on product context, terminology, audience, and the sentences around it.

This was the part of the workflow where an LLM offered something that fixed rules could not provide as easily.

A practical translation pipeline can therefore combine both kinds of software:

```text
Git diff
   -> Markdown parsing
   -> deterministic content selection
   -> LLM translation
   -> deterministic structure restoration
   -> schema and format validation
```

In this design, the model handles the semantic work in the middle. Traditional code can control what enters the model, what must remain unchanged, and whether the result is structurally valid.

The same separation can be useful in other AI-assisted products:

- use code to check required fields; use AI to summarize valid content
- use access controls to select available data; use AI to answer from that data
- use a parser to extract a document structure; use AI to classify or rewrite the prose
- use tests to verify behavior; use AI to propose a possible fix

For tasks that already have clear and reliable answers, introducing a model may add more complexity than value.

## The tradeoffs of using AI for exact rules

Using AI for a task with an exact expected result can introduce tradeoffs in three areas.

### Reliability

A deterministic rule can be tested against known cases. A model may make a different decision after a prompt update, a model upgrade, or a small change in context.

If a system needs to preserve every code block or reject every invalid file, “usually correct” may not be a sufficient target.

### Debugging

When deterministic code fails, we can inspect the input, follow the execution path, and reproduce the problem.

When an AI decision fails, the investigation is less direct. We may need to examine the prompt, context, model version, tool results, and output. Even then, the same request may not fail in exactly the same way again.

### Cost and speed

A local rule or parser is usually fast and inexpensive. A model call adds latency, usage cost, and another external dependency.

Those tradeoffs can be worthwhile for a difficult semantic task. They may be harder to justify for checking a file extension or validating a JSON structure.

## Deterministic does not mean hard-coded everywhere

There is a risk in taking this argument too far. Not every task can be reduced to a clean set of rules.

Hand-written rules can become brittle when they try to imitate language understanding. A long list of keywords is not necessarily better than a model if the real task is to understand intent or context.

For me, the goal is not to avoid AI. It is to find a useful boundary between interpretation and control.

I find these questions useful when deciding where that boundary should be:

1. Is the acceptable result narrow and objectively testable?
2. Can the decision be expressed with stable rules?
3. Must the behavior be easy to reproduce and audit?
4. Would a wrong answer break data, permissions, formatting, or a release?

If most answers are yes, I would start with deterministic software.

If the task depends on meaning, incomplete information, or multiple acceptable answers, AI may be the better fit. Even then, deterministic checks can still protect the input and validate the output.

## Make the boundary visible

Choosing the right tool for each task is only part of the design. The handoff between deterministic code and the model also needs to be visible.

In a translation workflow, the surrounding software can:

- select the content sent to the model
- protect code, links, and other non-translatable elements
- validate the structure of the returned document
- record which model, prompt, and context produced the result

This makes failures easier to locate. If a code block changes, I can inspect the parsing or restoration logic. If the wording is poor, I can look at the prompt, context, or model output instead. A visible boundary also makes it easier to change the model without redesigning every rule around it.

That boundary does not have to stay fixed. The important part is making it clear enough to test and reconsider as the system evolves.

## Final thoughts

AI has changed what I can build and how quickly I can explore an idea. After working on this service, however, I am less interested in whether an entire workflow can be called AI-powered. I care more about whether each step is understandable and easy to verify.

In this translation service, deterministic tools were a better fit for change detection, parsing, routing, and validation. The model was more useful in the part of the workflow where language and context mattered. That division was not a universal formula; it was the arrangement that made this particular system easier for me to understand and verify.

I expect that boundary to move as models and requirements change. For now, this is the question I want to carry into future AI-assisted projects:

> Which parts genuinely benefit from AI's flexibility, and which are better served by predictable, testable rules?
