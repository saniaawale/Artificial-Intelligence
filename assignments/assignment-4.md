# Assignment 4


## Q1

Fixed-size chunking treats a document as a stream of tokens. It doesnt know what a table is, or what belongs together. It cuts every N tokens and moves on. This creates problems.

**Retrieval fidelity** suffers the most. This is because the table itself is broken up and without context, certain chunks lose their meaning. For example, a fixed chunker might split the header row from the data rows into two seperate chunks. Neitherchunk alone is useful. On the other hand, RAGflows deepdoc engine does layout detection - it identifies tables, headings, captions etc. In doing this, it keeps semantically content together. 

**Index Design** chunks have no metadata, they are just blobs of text.Structure aware chunks come already tagged with things like section titles, page number etc. This lets you filter at query time before vector search even runs, which ultimately improves precision. 

**Preprocessing cost** is one real tradeoff. Deepdoc needs things like vision models for layout detection and table parsing logic.. which can be quite expensive. In situations where documents are rarely fed into the system and queried constantly this can be quite helpful.

Essentially, the failure of naive chunking occurs in how it makes structured content unsearchable.

## Q2
 

**Template-based chunking** uses known document structure, such as headings, section markers etc. in order to define chunk boundaries. It is deterministic and fast. It especially works well when the documents follow a predictable format.

It fails on **loosely structured corpora** like raw web text. There are no reliable delimiters. The chunker either produces chunks that are too large or too small. The output is very noisy and hard to retrieve from.

**Embedding-driven semantic segmentation** uses a model to detect when the topic shifts. It does not rely on structure, it reads the meaning. This handles messy corpora well because it can detect the difference between different parts of a paragraph with no headers in between.

It fails on **highly structured documents**. A table might look semantically similar to a some text discussing the same thing that the table does. The embedding model may not segment them correctly.

## Q3

**Lexical only (BM25)** is exact match based. It scores documents by term frequency and inverse document frequency. It fails on semantic similarity. If a user asks "How do i terminate my subscription" and the document syas "cancellation policy" BM25 scores it low beause the words dont overlap.

**Vector Only Search** finds semantically similar content regardless of exact wording. However, it fails on rare terms, codes and other identifiers.  A query like "error code E-4291" may embed to a generic error-related neighborhood in vector space, pulling back documents about unrelated errors. The embedding model averages out meaning and loses specificity.

**Hybrid retrieval** combines both signals. A document that scores well on BM25 and also has strong vector similarity gets ranked the highest. This covers the failures of each individual model. Hybrid recall is always at least as large as either method alone. Re-ranking then improves precision by scoring the union.

**Hybrid Edge case Failure**: if the query is highly ambiguous. For example, the word "apple" could mean either the fruit or the technology. both BM25 and vector search get confused, and re-ranking may not resolve the ambiguity without some more context.

## Q6

A single pass ANN search retrives the top K nearest vectors and returns them. It is fast, but it trades quality for speed in ways that hurts real world performance. RAGflow follows a multi stage pipeline.

**Recall vs. latency tradeoff:** ANN search is appproximate by design. It does not guarantee finding the true nearest neighbours, it just finds likely ones quickly. A multi stage pipeline uses ANN to generate a candidate set cheaply (high recall, lower precision), then applies a more expensive code re ranker only to that smaller set. The expensive computation occurs on a way smaller set of candidates. This way, you get near optimal precision without paying the full cost at scale.

**Cascading error propagation:** If a candidate misses a relevant document, re-ranking cannot recover it -  it only see what generation passed up. This means the first stage must retrieve more than you need, even at the cost of precision. The re-ranker then cleans it up. If either stage is tuned too conservatively, errors cascade and final output quality drops.


## Q5 

**Elasticsearch-like hybrid stores:** supports both keyword and vector search in a single system. They handle mixed workloads well: full-text search, structured filtering, and approximate vector search together. Best for enterprise document retrieval where queries combine keywords, filters, and semantic similarity.

**Vector-native databases:**  optimized entirely for ANN search. Index structures like HNSW are the right choice when retrieval is almost entirely semantic with no structured filtering or keyword matching needed. They struggle when you need metadata-heavy filtering or exact-match queries alongside vectors.

**Graph-augmented stores** add relationship edges between entities. They support traversal queries, which is valuable for knowledge-graph-backed RAG where reasoning depends on entity relationships, not just similarity.

| Backend | Best For |
|---|---|
| Elasticsearch hybrid | Mixed keyword + vector workloads | 
| Vector-native DB | High-throughput semantic search | 
| Graph-augmented | Entity reasoning | 

## Q6

A user's raw query is often a poor retrieval signal. It may be ambiguous, incomplete, or phrased differently than the indexed content. Query transformation addresses this gap before retrieval runs.

**Static query to retrieval** takes the raw input and embeds it directly. It works when queries are well-formed and vocabulary matches the data. It fails when users use natural, colloquial language that doesn't match document language. It also fails on complex multi-part questions because the embedding of a compound question blurs across all sub-topics and retrieves nothing precisely.

**Query expansion** adds synonyms, related terms, or hypothetical document phrases to the query before retrieval. This improves recall by covering vocabulary gaps. A user asking "how to cancel" gets expanded to also search for "termination," "discontinue," "opt out."

**Query decomposition** is critical for complex questions. "What were the revenue trends in North America compared to Australia over the last two years?" is actually three queries: North America revenue, Australia revenue, and a comparison. Decomposing it before retrieval means each subquery gets relevant chunks, and the LLM synthesizes the answer across all of them.

The key trade-off is latency. Each refinement step adds a round-trip. For low-latency applications, static queries or single-pass expansion are preferable. For high-accuracy question answering over complex data, iterative refinement pays off.

## Q9 

**Vector memory** stores past interactions as embeddings. At query time, semantically similar past exchanges are retrieved and injected into context.  The failure mode is recency blindness, where older but relevant memories may be outscored by recent but less relevant ones. It also can't answer precise questions like "how many times has this user asked about banks?"

**Structured memory** stores facts extracted from conversations as typed records. This handles precise queries well: user preferences, confirmed facts, flagged topics. The cost is extraction. Turning natural language into structured facts reliably is hard and error-prone. It works best for narrow, well-defined domains.
 
**Episodic logs** store the raw conversation history with timestamps. This gives full recall but poor retrieval: finding relevant past context requires scanning or embedding the entire log. These logs are basically backup that makes sure nothing gets lost or distorted when the other memory systems compress and organize this data.

Thus we can see how production systems layer all three : Episodic logic as the source of turth, vector memory for fast semantic retrieval, and structured memory for precise facts.

## Q10 
**Stateless Services:** These take input, produce output and hold no state. Some examples are Parser Workers, Chunking Workers, Embedding Workers, and Re-ranking.

**Stateful services:** Index writer must coordinate state to safely make writes without conflicts, Memory service has to rememeber what a specific user said earlier in the conversation.

**Scaling strategy:**
- Ingestion is slow and heavy. Scale with a worker queue and auto-scale workers based on queue depth.
- Retrieval needs to be fast. So we run multiple copies of the retrieval service and a load balancer spreads incoming queries across them 
- LLM Reasoning is expensive and variable-latency. Therefore, we must limit how many requests come in at once and run it on dedicated GPU machines.

**Failure isolation boundaries:**
- if a document fails to parse during ingestion, it just retries quietly. Users querying the system dont notice anything 
- if re ranking breaks you still return results, just not in the best order.
- If memory breaks, the system still answers, it just wont remember past conversation context.









 


