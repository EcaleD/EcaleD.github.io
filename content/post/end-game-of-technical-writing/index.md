+++
author = "Xiaokai Dong"
title = "The End Game of Technical Writing"
date = "2026-05-12"
description = "A reflection on how AI may shift technical writing from producing pages toward maintaining reliable knowledge for people and machines."
tags = [
    "AI",
    "Technical Writing",
]
categories = [
    "Technical Writing",
    "Career Notes",
]
image = "header-image-end-game-v2.png"
+++

<!--more-->

I keep coming back to the phrase *end game of technical writing*. It sounds dramatic, as if the profession is moving toward one clear ending. I do not think anyone can see that ending yet, including me.

What I can see is a smaller change happening in my own work: producing a decent first draft is no longer the difficult part. AI can turn notes into paragraphs, rewrite awkward sentences, or create something that looks finished surprisingly quickly.

But when I think about the documentation problems that have taken most of my time, the draft was rarely the hardest part. The harder questions were usually about what was true, what had changed, and how I could tell whether the content still worked.

That makes me wonder what happens when writing itself is no longer the main constraint.

## A few numbers that made me pause

I recently came across the latest [U.S. Bureau of Labor Statistics projection](https://www.bls.gov/ooh/media-and-communication/technical-writers.htm). It puts growth for technical writers at 1% between 2025 and 2035, compared with 3% across all occupations. It also expects about 3,200 openings a year, mostly because people will leave or change occupations.

The numbers did not surprise me as much as one sentence on the page: AI may make writers more productive and, as a result, slow employment growth.

I do not read that as a prediction that technical writers will disappear. I read it as a more uncomfortable possibility: documentation may keep growing while the number of people hired mainly to produce it does not grow at the same pace. One person, with AI, may simply be expected to cover more.

Then I looked at the [2025 Stack Overflow Developer Survey](https://survey.stackoverflow.co/2025/ai), and the picture became less tidy. Eighty-four percent of respondents were already using or planning to use AI tools, but more people distrusted their accuracy than trusted it—46% versus 33%.

That feels familiar. I use AI often, and the speed is easy to notice. Confidence is harder. A clean paragraph can still contain the wrong assumption, miss an important condition, or describe a feature that does not quite exist. The time saved in drafting can quietly return as time spent checking.

## The reader I do not see

For a long time, the documentation path in my head was simple:

```text
Writer -> documentation site -> reader
```

That picture now feels incomplete. In early 2026, Google introduced a [Developer Knowledge API and MCP server](https://developers.googleblog.com/introducing-the-developer-knowledge-api-and-mcp-server/) that let AI tools retrieve its official developer documentation directly. What caught my attention was not the API itself, but the way Google described the documentation: a programmatic source of truth.

A developer may still open a page. But an agent may also read the same source, use it while completing a task, and give the developer an answer without the page ever becoming the visible destination.

GitBook offered another glimpse of this change. During one week in April and May 2026, AI agents accounted for [51.8% of what it classified as intentional documentation reads](https://www.gitbook.com/blog/ai-docs-data-april-2026) on its platform. I would not treat one week of GitBook traffic as the future of the entire web. Still, I found the number hard to dismiss.

It suggests a second path:

```text
Documentation -> AI or agent -> user
```

If that path becomes common, my work may be used in ways I cannot see from page views or reader feedback. The structure of a document, the consistency of its terminology, and the freshness of its examples may influence answers produced somewhere else. The prose still matters, but it is no longer working alone.

## Where I see my own work moving

Some of my recent projects already feel like small steps in this direction. I experimented with running documentation tests in CI, built a translation service, and spent time deciding where deterministic software should control an AI-assisted workflow.

I started each project thinking mainly about content. Before long, I was thinking about the system around it:

- What is the source of truth?
- How can a change be detected?
- Which parts should be automated?
- How can the output be checked?
- Who owns the result when the system is wrong?

This has made my own view of a technical writing career less linear. I once imagined progression mostly as writing better content, handling more complex products, and eventually shaping a documentation strategy. I still care about all of those things. But I can also imagine the work spreading into content systems, developer experience, knowledge architecture, testing, or the design of information that AI tools can use.

I do not know whether these become new forms of technical writing or simply adjacent roles with different titles. The answer will probably vary by company. A small product team, a regulated enterprise, and an open-source project are unlikely to arrive at the same model.

## Why I still hesitate over the end game

I can see two forces moving in opposite directions. AI may reduce the time needed to draft and update content. It may also create more content to review, more interfaces that depend on documentation, and more ways for one inaccurate statement to travel beyond its original page.

So I cannot tell whether the future holds fewer technical writers, broader roles, or some mixture of both. I am only becoming less certain that prose will remain the center of the job.

If there is an end game, it may not be the end of technical writers. It may be the end of writing as the unquestioned center of technical writing.

What replaces it is still open. Perhaps more of the work will be about keeping technical knowledge accurate, connected, testable, and usable across both human and machine interfaces. Or perhaps the profession will split into several more specialized forms.

For now, the question I find most useful is not whether AI can write the documentation. It is this:

> When writing is no longer scarce, what part of technical communication still is?
