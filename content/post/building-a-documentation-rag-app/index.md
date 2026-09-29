+++
author = "Xiaokai Dong"
title = "Building a RAG Application: Indexing, Retrieval, and Generation"
slug = "building-a-documentation-rag-app"
date = "2026-07-26T00:00:00+08:00"
draft = false
description = "A practical walkthrough of a RAG application's architecture, from structural chunking and hybrid retrieval to cited answers, conversation context, and index updates."
image = "header-image-rag.png"
tags = ["AI", "RAG", "Retrieval", "Software Engineering"]
categories = ["Vibe Coding", "Doc Workflow"]
+++

<!--more-->

A RAG app can look simple from the chat window: ask a question, get an answer, open a source. Behind that interaction, several components have to agree on what content to search, which passages to use, and how to connect the answer back to its evidence.

Retrieval-augmented generation, or RAG, supplies selected reference material to a language model alongside a question. It does not retrain the model on that material. The application builds a searchable index, retrieves relevant passages at request time, and includes them in the model's context.

This walkthrough examines a RAG architecture, from prepared content and indexing to retrieval, generation, and conversation handling. The API names in the examples are invented.

## System architecture: indexing and answering

The application uses a Next.js chat interface, a Python/FastAPI backend, and PostgreSQL with [pgvector](https://github.com/pgvector/pgvector). PostgreSQL holds content records, chunks, vectors, and conversation state. Two local models handle embeddings and reranking; a hosted language model generates the answer.

![Component architecture of a documentation RAG app: a chat UI connects to an API containing retrieval and answer components; the API uses PostgreSQL, local retrieval models, and a hosted language model. A separate index builder prepares new searchable generations.](rag-architecture.png)

*The arrows show component dependencies, not a request timeline. Content preparation begins outside the scope of this diagram.*

There are two execution paths. The index builder prepares searchable content outside the request path: it parses text, creates chunks, computes embeddings, and stages an index generation for validation. The API handles incoming questions, retrieves evidence from the active generation, and sends that evidence to the generator.

The backend controls these steps. The language model does not independently search the database or choose tools. Keeping retrieval separate from generation also gives debugging a useful starting point: inspect the supplied passages before changing the prompt.

## Building the index: structure, context, and embeddings

A chunk needs enough context to make sense on its own. A paragraph under “Limitations” may be hard to interpret once its parent heading disappears. A table row is worse without the column names.

The parser identifies Markdown structure: headings, paragraphs, lists, tables, and fenced code. The chunker keeps a heading path and links to adjacent chunks. When a table exceeds the size budget, its split pieces repeat the header. Large code blocks may still need splitting; keeping every block intact is not always practical.

Each chunk has display text and text prepared for embedding, which includes contextual information such as headings. The latter goes through [BGE-M3](https://huggingface.co/BAAI/bge-m3), producing a dense vector. At query time, the same model embeds the question, and pgvector compares it with stored vectors using cosine distance.

BGE-M3 also supports sparse and multi-vector retrieval, but the architecture here uses only its dense output. Keyword search runs through a separate PostgreSQL full-text search path. Both paths operate on the same chunk records, so their results can be merged by chunk ID.

## Index versioning and embedding reuse

Queries should not see an index halfway through a rebuild. New content is therefore processed into a separate index generation. Parsing, chunking, embedding, and validation finish before a transaction changes the active-generation pointer. If the build fails, the previous generation remains available.

Embedding reuse reduces the cost of subsequent builds. The cache key includes the embedding input text, model identifier, and vector dimension. If that combination is unchanged, the vector can be reused. The build still creates a new generation and its chunk records, so this is incremental embedding work, not a completely incremental database update.

Two consistency concerns deserve separate attention. For embedding reuse, the model identifier should resolve to a specific weights revision, and cache invalidation should account for preprocessing changes.

For retrieval, an atomic pointer switch does not by itself give an entire request a consistent snapshot. If separate retrieval queries resolve the active generation independently, a switch between them can mix versions. To avoid that race, resolve the generation once at request start and pass its ID through every retrieval path, including neighboring-chunk lookups.

## Hybrid retrieval and candidate reranking

Consider the question: “How do I change `request_timeout_ms`?”

A semantic search might find a helpful explanation of request timeouts. It might also return material about connection timeouts, which sounds similar but may describe a different setting. The exact identifier matters.

The backend runs four retrieval paths concurrently:

- **Identifier matching** looks for extracted API names, parameters, and other symbols in indexed metadata.
- **Title matching** helps with questions that name a feature or page directly.
- **Full-text search** looks for matching terms in the indexed text.
- **Dense retrieval** finds passages with related meaning, even when the wording differs.

These paths return ranked lists with incompatible scores. Adding a text-search score directly to a vector similarity score would make the result depend on their unrelated scales. The first merge uses weighted [reciprocal rank fusion](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking):

```text
score(chunk) = sum(weight[channel] / (60 + rank[channel, chunk]))
```

Ranks start at one, and a channel contributes nothing for a chunk it did not retrieve. A passage appearing near the top of several lists accumulates more support. The constant softens differences between rank positions; it is not the number of results returned.

The fused candidate pool then feeds a bounded shortlist to [bge-reranker-v2-m3](https://huggingface.co/BAAI/bge-reranker-v2-m3). Unlike the embedding model, this cross-encoder reads a question and passage together and returns a relevance score. It can consider their relationship more directly, but it requires inference for each pair.

To bound that cost, only the head of the fused list is reranked. The remaining candidates stay in fused order before further selection rules are applied. Evidence selection also considers question intent and limits repeated results from one document when choosing the initial passages. It can then add neighboring chunks. A good match sometimes needs the paragraph just before it to be useful.

There are tradeoffs in those limits. A smaller reranking shortlist is cheaper but cannot promote a passage it never sees. A shorter model input can cut off a useful qualification. Both settings need evaluation against retrieval quality as well as latency.

## Answer generation and citation handling

Only content cleared for the intended use should enter the index or be sent to a hosted model. The same boundary applies to user questions and conversation history. Confidential or personal information should be excluded unless that use is explicitly authorized and appropriate safeguards are in place.

Once evidence is selected, the backend assembles the generation input: the user's question, the selected passages, and a bounded window of conversation history. Each passage carries a source ID, and the prompt asks the model to use these IDs when citing evidence.

After generation, the application checks citation markers against the supplied passages and builds links the interface can open. It can normalize supported citation formats and remove references to unknown source IDs.

This is a structural check. A valid source ID shows that the passage was available to the model. It does not establish that the passage supports the sentence beside the citation.

One fallback is to make retrieved passages available under a separate “Retrieved evidence” label when the model omits citations. That keeps the material inspectable, but it does not repair the missing connection between claims and sources. An evaluation should distinguish this fallback from an answer with supported inline citations.

Likewise, an empty retrieval result can trigger a deterministic no-answer response, but a non-empty result does not prove that the question is answerable. The prompt asks the model to acknowledge missing information. That still needs testing with questions the content cannot answer.

The prompt also treats passages as reference material rather than instructions. That distinction is explicit, although prompt wording alone is not a complete defense against malicious content.

## Conversation context and streaming responses

Saving chat history gives the interface continuity. Retrieval still needs a usable query for each turn.

After a question about `request_timeout_ms`, a follow-up such as “What about its default?” is incomplete as a search query. A simple baseline is to detect contextual references and prepend the most recent user question for retrieval. The generator receives the original question, a bounded window of conversation history, and fresh evidence.

This handles some short follow-ups. It will struggle with several intervening topics or ambiguous pronouns. It is not a general conversation-understanding system, and previous assistant answers should not become evidence just because they are in the chat history.

The interface presents this as a continuous conversation and renders Markdown in both answers and source excerpts. Source rendering matters too: opening a citation is much less useful if a parameter table is still displayed as raw Markdown.

The backend streams server-sent events over the response to a POST request, which the browser reads with streaming `fetch`. Status updates, sources, and text can appear before completion. Once generation ends, the backend normalizes citations and sends the final answer, replacing the provisional streamed version.

That last detail is important. The streaming text has not yet passed the final citation-format checks. Streaming improves the experience of waiting; it does not make an unfinished answer verified.

## Evaluating latency, retrieval, and answer quality

Latency is easier to investigate when retrieval time, time to the first generated token, and total completion time are measured separately. They point to different problems. Loading local models affects cold starts; reranking adds work before generation; the hosted model adds its own delay.

Warming the local models before reporting readiness moves initialization out of the first user request. Capping reranking work reduces inference cost. Streaming makes progress visible. These changes help in different ways, and none makes the whole pipeline free of waiting.

Quality needs a similar separation. The retrieval tests pair reviewed questions with relevant document or chunk IDs. Recall@k measures the fraction of those relevant items found in the first k results. MRR averages the reciprocal rank of the first relevant result across questions. nDCG measures ordering against relevance labels, which can be binary or graded. Exact-identifier questions and changed-content cases deserve their own checks.

Answer evaluation checks whether claims are supported, whether citations point to the right passages, and whether the system admits when it lacks enough information. A small smoke-test set helps catch regressions, but it is not proof of broad accuracy.

These checks also suggest a practical build order. Start with inspectable chunks and retrieval results, then add generation and citation handling. Conversation and streaming can build on that working request path. Update tests should exercise the same path against a new index generation, rather than stopping at a successful indexing job.

With those boundaries visible, a poor answer becomes easier to investigate. Missing evidence points back to chunking or retrieval. A misread passage points toward context assembly or generation. An outdated answer calls for checking the content version that the request actually used.
