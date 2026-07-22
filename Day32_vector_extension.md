# pgvector, Embeddings & Building a RAG Pipeline — Session Notes

A full theory-then-practice session on `pgvector`: what a vector/embedding actually is, why similarity search is fundamentally different from a normal `WHERE` clause, the four core AI-workflow patterns (Generative AI, RAG, AI Agent, Agentic AI) mapped onto familiar PostgreSQL DBA scenarios, and a live build-up from hardcoded vectors to a working document-upload RAG pipeline with a local LLM.

---
## Table of Contents

- [1. Why Similarity Search Exists — the "Around" Query](#1-why-similarity-search-exists--the-around-query)
- [2. What a Vector/Embedding Actually Is](#2-what-a-vectorembedding-actually-is)
- [3. Dimensions — How Many, and Why Not More](#3-dimensions--how-many-and-why-not-more)
- [4. Installing pgvector and the Distance Operators](#4-installing-pgvector-and-the-distance-operators)
- [5. The Four AI Workflow Patterns, via DBA Analogies](#5-the-four-ai-workflow-patterns-via-dba-analogies)
- [6. Demo 1 — Hardcoded Embeddings and Cosine Distance](#6-demo-1--hardcoded-embeddings-and-cosine-distance)
- [7. Demo 2 — Real Embeddings via an Embedding Model](#7-demo-2--real-embeddings-via-an-embedding-model)
- [8. Demo 3 — Scaling to 1,000 Rows and Adding an Index](#8-demo-3--scaling-to-1000-rows-and-adding-an-index)
- [9. Demo 4 — Document Upload and Chunking](#9-demo-4--document-upload-and-chunking)
- [10. Demo 5 — Bringing In an LLM for a Full RAG Answer](#10-demo-5--bringing-in-an-llm-for-a-full-rag-answer)
- [11. What "Tuning" a Vector Database Actually Means](#11-what-tuning-a-vector-database-actually-means)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. Why Similarity Search Exists — the "Around" Query

**Motivating question, built up step by step with the class:** ordinary SQL answers exact-match or defined-range questions perfectly well —
```sql
SELECT * FROM employee WHERE salary = 300;
SELECT * FROM employee WHERE salary BETWEEN 250 AND 350;
```
— but neither of these can answer a genuinely fuzzy request: **"give me employees whose salary is *around* 300"** without the caller specifying an exact range. **This is exactly the gap similarity search fills** — finding things that are conceptually "near" a query, without a human having to pre-define what "near" numerically means.

---

## 2. What a Vector/Embedding Actually Is

**Definition given directly:** *"A vector, often called an embedding, is a long list of numbers that represents the meaning of a piece of text, image, audio, or video."* Every embedding has both **magnitude** and **direction** (borrowed from the mathematical definition of a vector) — conceptually, similar *meanings* end up positioned close together in this numeric space, and dissimilar meanings end up far apart.

**Visual/spatial analogy built live:** imagine plotting words in a multi-dimensional space — `car`, `bus`, `bike` cluster together (all vehicles); `apple`, `banana`, `mango` cluster together (all fruits); a new word like `pen` gets placed near whichever existing cluster it's conceptually closest to. **Asking "what is a fruit?" is really asking "find me the points closest to the *fruit* region of this space."**

---

## 3. Dimensions — How Many, and Why Not More

**More dimensions = finer-grained distinctions** — with only 3 dimensions, there's a real risk of unrelated concepts accidentally landing close together (e.g. a company that makes both TVs and something else getting confused for a pure "who makes TVs" query); more dimensions reduce this collision risk.

**But more is not simply better — direct guidance given:** you don't need "thousands" of dimensions for most real use cases. **Practical range cited: roughly 384 up to 1500–2000** is the effective sweet spot — `pgvector` itself supports up to **2,000 dimensions**. Going far beyond that adds cost without proportionate benefit for typical text/image/audio similarity use cases.

---

## 4. Installing pgvector and the Distance Operators

```sql
CREATE DATABASE demo;
\c demo
CREATE EXTENSION vector;
CREATE TABLE knowledge (
    text TEXT,
    embedding VECTOR(3)   -- 3 dimensions for this demo; 384+ typical in practice
);
```

**Three distance operators, tied to three underlying index/operator classes:**

| Use case | Operator class | Applies to |
|---|---|---|
| Text, image, video similarity | `vector_cosine_ops` | The default/most common case |
| (an alternate distance metric) | `vector_l2_ops` | General-purpose distance |
| Inner product | `vector_ip_ops` | Specific specialized cases |

**Confirmed directly: for text/image/audio/video work (the overwhelming majority of real use cases), cosine distance is the default choice** — the other two operator classes are relevant mainly for non-text scenarios like geographic (latitude/longitude) similarity.

---

## 5. The Four AI Workflow Patterns, via DBA Analogies

Before any hands-on `pgvector` work, four terms were distinguished conceptually — using two parallel example sets (an HR leave-policy scenario, and a PostgreSQL vacuum/switchover scenario) to make each pattern concrete:

| Pattern | Shape | HR example | PostgreSQL example |
|---|---|---|---|
| **Generative AI** | input → output | "What is casual leave policy?" (generic knowledge, no company data needed) | "What is a dead tuple in PostgreSQL?" (generic textbook definition) |
| **RAG (Retrieval-Augmented Generation)** | input → retrieve knowledge → output | "How many casual leaves does *my company* provide?" (needs your company's actual HR document) | "What are the conditions to clean dead tuples *in my production database*?" (needs your actual configured thresholds) |
| **AI Agent** | input → retrieve knowledge → take one action → output | "Apply for leave tomorrow" (a single concrete action performed on your behalf) | "When can I perform a switchover?" (retrieve current cluster state, then answer/act) |
| **Agentic AI** | input → retrieve knowledge → take *multiple* coordinated actions → output | "Apply for leave tomorrow *and* email my manager" (two chained actions) | "Perform the switchover activity" (the agent actually executes the operation, not just advises on it) |

**The core distinction driving RAG specifically, stated directly:** the difference between the first two rows in each pair is whether the question can be answered from **general/generic knowledge** (Generative AI alone) or requires **your specific, private data** (RAG) — this is exactly the gap `pgvector`-backed similarity search is built to close: giving an LLM access to your own documents/data as retrievable context before it generates an answer.

---

## 6. Demo 1 — Hardcoded Embeddings and Cosine Distance

```sql
INSERT INTO knowledge (text, embedding) VALUES
('Apple is a fruit', '[0.9, 0.9, 0.9]'),
('Banana is a fruit', '[0.9, 0.9, 0.9]'),
('Mango is a fruit',  '[0.9, 0.9, 0.9]'),
('Car is a vehicle',  '[0.95, 0.96, 0.94]'),
-- ... etc, deliberately hand-picked so "similar meaning" rows get similar numbers
```

**Deliberately hardcoded** (not generated by a real model) purely to make the underlying distance math visible and inspectable before introducing real embedding generation.

```sql
SELECT text, embedding <=> '[0.9,0.9,0.9]' AS cosine_distance
FROM knowledge
ORDER BY cosine_distance;
```

**Confirmed live:** rows semantically matching the query embedding returned near-zero cosine distances (e.g. `0.0001`, `0.0004`); unrelated rows returned much larger distances (`0.76`+). **The rule stated directly: the lower the cosine distance, the more similar the result** — this is the entire mechanism similarity search relies on.

---

## 7. Demo 2 — Real Embeddings via an Embedding Model

```python
# import_data.py
from sentence_transformers import SentenceTransformer
model = SentenceTransformer('all-MiniLM-L6-v2')   # open-source embedding model
embedding = model.encode(text)
# INSERT the resulting embedding into the knowledge table
```

**Confirmed directly:** this is the real-world equivalent of the hardcoded values from Demo 1 — an **embedding model** (here, an open-source one, `all-MiniLM-L6-v2`, described informally as "Almondy LM" in the session) converts arbitrary input text into its actual numeric vector representation, which then gets stored in the `VECTOR` column exactly as the hardcoded values were.

```python
# search.py — take a live question, embed it, then search
question = input("Enter your question: ")
question_embedding = model.encode(question)
# SELECT ... ORDER BY embedding <=> question_embedding LIMIT N
```

**Confirmed live:** asking "What is Apple?" correctly returned the fruit-related rows first (distance ~0.2, still not exactly 0 — flagged directly as expected/normal, since a real model's embedding of a full question is naturally not a perfect numeric match even for a clearly related answer) — this is the mechanism by which a plain-English question becomes a database similarity query.

---

## 8. Demo 3 — Scaling to 1,000 Rows and Adding an Index

```bash
# insert ~1000 rows via script, generating a real embedding for each
```
```sql
EXPLAIN SELECT * FROM knowledge ORDER BY embedding <=> '[...]' LIMIT 5;
-- Seq Scan on knowledge
```

**Confirmed live:** without an index, the query plan shows a full **sequential scan** — every row's distance gets computed, exactly the performance problem an index exists to solve at scale.

```sql
CREATE INDEX ON knowledge USING hnsw (embedding vector_cosine_ops);
```

**Two index types available in `pgvector`, distinguished directly:**
- **HNSW** — appropriate for cosine similarity (text/image/video use cases) — used here.
- **IVFFlat** — for other distance types (e.g. geographic latitude/longitude-style searches).

**Confirmed live, post-index:** re-running the same query no longer shows a sequential scan — the index is used, and the plan/behavior changes accordingly, consistent with the general index-usage principle from earlier query-tuning sessions.

---

## 9. Demo 4 — Document Upload and Chunking

```bash
cat document.txt   # contains definitions of B-tree, HNSW, pg_cron, and other terms
```
```python
# upload flow: read document.txt, break it into pieces, embed each piece, insert into knowledge base
```

**A key mechanic introduced directly: "chunking."** A large uploaded document (or long text) isn't embedded as one single giant vector — it's **broken down into smaller pieces ("chunks"), based on size**, and each chunk gets its own embedding, stored as its own row. This is why a similarity search against an uploaded document returns a specific relevant excerpt rather than the whole document at once.

**Confirmed live:** asking "What is pg_cron?" against the uploaded document correctly retrieved the specific chunk defining `pg_cron` as a scheduler, with a low similarity distance (~0.3), while unrelated chunks scored much higher (farther) distances.

---

## 10. Demo 5 — Bringing In an LLM for a Full RAG Answer

**Everything up to this point (Demos 1–4) never involved an LLM at all** — it was pure similarity search: embed the question, find the closest stored chunk(s), return that raw stored text. **Explicitly flagged as the final missing piece:** feeding the retrieved chunk(s) *into* an LLM as context, and asking the LLM to synthesize/format/reason over that retrieved information before returning an answer — this is what actually makes it "RAG" in the fuller sense, versus plain retrieval.

```python
# app.py — a small local web app tying it together:
# 1. Take a user's uploaded text/PDF and a question
# 2. Embed the question, retrieve the closest matching chunk(s) from PostgreSQL
# 3. Pass the retrieved chunk(s) + the original question to an LLM as context
# 4. Return the LLM's synthesized answer
```

**LLM options named directly, in order of accessibility:**
- A hosted API (Claude, ChatGPT) — requires an account/API key.
- **Ollama** — described as a free, locally-runnable LLM option (Microsoft's small local model was named as an example) — installable and runnable entirely on your own machine, well-suited for a self-contained demo/lab environment with no external API dependency.

**Confirmed live, end to end:** uploading a short paragraph about PostgreSQL, then asking a related question, produced a full LLM-generated answer, not just the raw retrieved text — confirmed by then inspecting the underlying `knowledge`/document table directly (`SELECT * FROM document;`) to show the actual stored chunks and embeddings backing that generated answer.

**A prompting nuance mentioned directly:** you can explicitly instruct the LLM, in your prompt, to answer **strictly from the retrieved context** rather than blending in its own general trained knowledge — useful when you specifically want grounded, source-restricted answers rather than the LLM's broader knowledge bleeding into the response.

---

## 11. What "Tuning" a Vector Database Actually Means

**Summarized directly, closing the loop on why any of this matters to a DBA specifically, not just an AI engineer:** improving answer accuracy, response speed, and relevance in an AI/RAG system comes down to the same fundamentals covered throughout this course — **properly maintaining the underlying vector table** (like any other table), **choosing and tuning the right index** (HNSW vs. IVFFlat, as covered), and generally applying the same performance-tuning discipline already learned to this new data type. **Explicit final framing: `pgvector` is fundamentally "PostgreSQL storing embeddings" — everything else (accuracy, speed) rides on the same database fundamentals already covered in this course, not some entirely separate skill set.**

**Also stated directly, as a closing conceptual note:** *"AI is all about similarity search, not accurate/exact search."* — a deliberate reframe for anyone expecting AI-powered search to behave like a precise `WHERE` clause; it fundamentally does not, and that's the point.

---

## 12. Questions Discussed in This Session

**Q1. In the hardcoded-embedding demo, are the specific numeric values (0.9, 0.95, etc.) standardized or meaningful in some universal sense?**

No — confirmed directly: those specific values were arbitrarily hand-picked purely to make the "similar things get similar numbers" mechanic visible for teaching purposes. In a real system, an embedding model determines the actual values based on its own trained understanding of the input's meaning — there's no manually-assigned numeric convention.

---

**Q2. What's the practical difference between using more vs. fewer dimensions for embeddings?**

More dimensions allow finer-grained distinction between concepts (reducing the chance unrelated things land close together by coincidence), but going far beyond the practically useful range (roughly 384–2000) doesn't meaningfully improve results and just adds cost — `pgvector` itself caps out at 2,000 dimensions, which the session characterized as more than sufficient for standard use cases.

---

**Q3. Do HNSW and IVFFlat serve the same purpose, just with different implementations, or are they genuinely for different use cases?**

Genuinely different use cases based on distance type: **HNSW** pairs with cosine similarity (the standard choice for text/image/video), while **IVFFlat** is suited to other distance types (e.g. geographic/coordinate-based similarity) — choosing between them isn't a performance-only decision, it follows directly from which distance metric your actual similarity search needs.

---

**Q4. Does a RAG pipeline always involve an LLM, or can it work with pure retrieval alone?**

The session drew this distinction directly: pure similarity search/retrieval (Demos 1–4) works without any LLM at all — it just returns the closest matching stored text. What makes something a full RAG *answer* (rather than raw retrieval) is passing that retrieved content into an LLM as context so it can synthesize, format, and reason over it before responding — Demo 5 was explicitly framed as "now we're finally involving an LLM," distinguishing it from everything before it.
