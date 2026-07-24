---
title: "Encoder vs. Decoder Today"
description: "Quick notes on a common AI interview question: what encoders and decoders do, when to use them, and what changed in the ChatGPT era."
pubDate: 2026-05-30
heroImage: "../../assets/transformer-attention-hero.png"
---

I found encoder vs decoder in transformers is one of the most popular interview questions for AI engineers.

The basic questions are usually: what is an encoder, what is a decoder, and what do we use each for? So I just wrote down some of my quick notes.

**What are they?**

An encoder reads the whole input and builds a semantic representation of it. Each token becomes a contextual embedding: a vector whose meaning is shaped by the words around it. Because the encoder can see the full sentence, every token can use information from both before and after it. Together, those embeddings capture the meaning of the sentence.

A decoder also turns tokens into contextual embeddings, but each token can only use the context before it. The embedding at each position is then used to predict the next token, so the model generates the output one token at a time.

`Encoder: input -> semantic representation -> label / score / embedding`

`Decoder: prompt -> next token -> next token -> response`

**When do we use each?**

That difference also tells us when to use each one. Some common use cases for encoders are classification, search embeddings, and reranking. Take sentiment classification: the encoder reads the whole review, and its output is pooled into a semantic embedding (a vector). A small classifier then turns that embedding into "positive" or "negative." Nothing needs to be generated.

Use a decoder when the output needs to be generated. Chat, text completion, code generation, and tool calling all fit the same pattern: given what has already been written, what should come next?

An encoder-decoder model puts the two together. This was the original Transformer setup, especially for translation: the encoder read English, and the decoder generated Chinese.

**Decoder-only models today**

Today, many large language models keep only the decoder. A decoder-only model learns one simple job: predict the next token. That job is more flexible than it sounds. It can classify a review by generating "positive," extract fields by generating JSON, or call a tool by generating structured arguments.

ChatGPT is the obvious example of how far this approach can go. In my opinion, this leads to a more interesting question: what can an encoder do that a decoder can't nowadays?

Apparently, not many.

One remaining natural use case that I can think of is creating embeddings for text in our own system. An encoder-based embedding model turns a piece of text into a semantic vector. We can store those vectors in a vector database for search, or compare them to measure semantic similarity.
