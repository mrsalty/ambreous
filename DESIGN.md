# Design Document

## Overview

Ambreous is a two-tier hierarchical memory system. Working memory captures raw events at ingestion time. A background consolidator periodically promotes those events into a multi-layer long-term graph. At query time, a task router decides retrieval depth so that fast tasks pay minimal cost and reasoning-heavy tasks get full context.

---

## Memory Tiers

### Working Memory

The working memory is an in-process event buffer — essentially an append-only queue of timestamped facts.

**Properties:**
- Stores raw events as they arrive (no processing at write time)
- Each event carries: `id`, `timestamp`, `content` (str), `metadata` (dict)
- Configurable capacity (max events before forced consolidation)
- Cleared after consolidation (or archived, depending on policy)

**Intent:** Analogous to sensory / short-term memory in cognitive models. Fast writes, no LLM calls on the hot path.

### Long-Term Memory

The long-term memory is a directed graph with N configurable layers (default: 3). Layer 0 is the most abstract (themes); layer N-1 is the most concrete (individual fact clusters close to raw events).

**Node schema:**

```
MemoryNode:
  id:           str           # UUID
  level:        int           # 0 = top (most abstract), N-1 = leaf
  summary:      str           # LLM-generated summary for this node
  content_ref:  str | None    # key into object store (leaf nodes only)
  metadata:     dict          # timestamps, source tags, etc.
```

**Edge types:**

| Type | Direction | Meaning |
|---|---|---|
| `BELONGS_TO` | leaf → parent | Hierarchical containment (N → N-1) |
| `RELATED_TO` | any → any (same level) | Semantic relatedness within a layer |

**Storage split:**
- Graph structure and summaries live in the **graph store** (in-memory NetworkX by default).
- Full raw content lives in the **object store** (local filesystem by default), referenced by `content_ref`.
- Top layers (0 and 1) may have no `content_ref` — their summary *is* the content.

---

## Consolidation

Consolidation is the process of promoting working memory events into the long-term graph. It runs on a configurable background schedule (e.g. every 60 seconds), not on every write.

### Steps

1. **Snapshot** the current working memory buffer (swap to a fresh empty buffer to avoid races).
2. **Cluster** events into thematically related groups. Clustering can use:
   - LLM-based grouping (ask the LLM to assign events to topics)
   - Embedding similarity + k-means (no LLM call, faster)
   - The strategy is pluggable.
3. **Create leaf nodes** (layer N-1): for each cluster, generate a summary via the LLM backend and write raw content to the object store. Add these nodes to the graph.
4. **Propagate upward**: for each layer from N-2 down to 0:
   - Find sibling nodes at the layer below that are semantically related or that exceed a grouping threshold.
   - Generate a summary of the group.
   - Create a parent node at the current layer.
   - Add `BELONGS_TO` edges from children to parent.
   - Add `RELATED_TO` edges between sibling nodes at the same layer.
5. **Merge into existing graph**: new nodes at higher layers may need to be merged with existing nodes if they cover overlapping topics. This is an open design question (see [Open Questions](#open-questions)).

### Consolidation triggers

- **Primary**: periodic background timer (default interval: configurable, e.g. 60s).
- **Secondary** (optional): capacity-based flush when working memory hits a hard limit.

---

## Retrieval

### Task Router

Before fetching anything from the graph, a query is classified into a retrieval depth:

| Class | Max layer fetched | Heuristic signals | LLM-based signals |
|---|---|---|---|
| `SURFACE` | 0 | Short query, single entity, "what is", "who is" | LLM says: factual / definitional |
| `SHALLOW` | 1 | "summarize", "overview", "recent", "list" | LLM says: summarization / recall |
| `DEEP` | N-1 (all) | "why", "how", "plan", "analyze", "compare", "reason" | LLM says: reasoning / synthesis |

The router has two modes:
- **Heuristic mode**: keyword matching, zero LLM calls, ~0ms latency. Good for latency-critical paths.
- **LLM mode**: ask the LLM backend to classify the task. More accurate, adds one round-trip.

Callers can also pass an explicit depth hint to bypass routing entirely.

### Graph Retriever

Given a query and a resolved depth, the retriever:

1. Finds entry-point nodes at layer 0 by semantic similarity (embedding search or keyword match against summaries).
2. If depth > 0, follows `BELONGS_TO` edges downward to collect nodes at each additional layer.
3. Optionally follows `RELATED_TO` edges at each layer to widen the result set.
4. Returns a ranked list of `MemoryNode` objects (with summaries), and for leaf nodes, dereferences `content_ref` to fetch full content from the object store.

---

## Component Interfaces (Abstract)

All external dependencies are behind abstract base classes so backends are swappable without changing core logic.

### `GraphStore`

Responsible for storing and querying the node/edge structure.

```
Methods:
  add_node(node: MemoryNode) -> None
  add_edge(src_id, dst_id, edge_type) -> None
  get_node(node_id) -> MemoryNode
  get_nodes_by_level(level) -> list[MemoryNode]
  get_children(node_id) -> list[MemoryNode]      # BELONGS_TO outgoing
  get_related(node_id) -> list[MemoryNode]        # RELATED_TO outgoing
  search(query_embedding, level, top_k) -> list[MemoryNode]
```

Default implementation: **NetworkX** (in-process, no external dependency).
Future: Neo4j, Kuzu, LanceDB with graph extensions.

### `ObjectStore`

Stores raw content blobs, keyed by string.

```
Methods:
  put(key: str, content: str) -> None
  get(key: str) -> str
  delete(key: str) -> None
  list(prefix: str) -> list[str]
```

Default implementation: **local filesystem** (one file per key under a configurable directory).
Future: S3, GCS, Azure Blob.

### `LLMBackend`

Drives summarization, clustering, task classification, and optionally embedding.

```
Methods:
  summarize(texts: list[str], context: str) -> str
  classify_task(query: str) -> Literal["SURFACE", "SHALLOW", "DEEP"]
  cluster(texts: list[str], n_clusters: int) -> list[list[int]]  # indices
  embed(texts: list[str]) -> list[list[float]]
```

Reference implementation: **Claude** (Anthropic API).
Any backend implementing this interface (OpenAI, local Ollama, etc.) plugs in without changes to core logic.

---

## Module Layout

```
ambreous/
├── memory/
│   ├── working.py          # WorkingMemory: event buffer
│   ├── longterm.py         # LongTermMemory: graph + object store facade
│   └── system.py           # MemorySystem: top-level entry point
├── graph/
│   ├── base.py             # GraphStore ABC
│   ├── node.py             # MemoryNode dataclass + EdgeType enum
│   └── networkx_store.py   # Default in-memory implementation
├── storage/
│   ├── base.py             # ObjectStore ABC
│   └── local.py            # LocalFileObjectStore
├── llm/
│   ├── base.py             # LLMBackend ABC
│   └── claude.py           # ClaudeBackend (reference implementation)
├── retrieval/
│   ├── router.py           # TaskRouter: classify query → RetrievalDepth
│   └── retriever.py        # GraphRetriever: traverse graph by depth
└── consolidation/
    └── consolidator.py     # AsyncConsolidator: background loop
```

---

## Open Questions

These are design decisions not yet resolved — good candidates for iteration:

1. **Merging vs. extending at consolidation time.** When new leaf nodes overlap topically with existing nodes, should we merge them (update the existing node's summary) or always create new nodes and link them? Merging keeps the graph compact but requires similarity judgment. Always-extending is simpler but risks graph bloat.

2. **Embedding strategy.** Should embeddings be stored on nodes (for fast vector search) or computed on demand? Storing them couples the graph store to embedding dimensionality; on-demand is slower but more flexible.

3. **Layer count configurability vs. dynamic depth.** A fixed number of layers (e.g. 3) is simple but may not fit all corpus sizes. A dynamic approach (RAPTOR-style: keep merging until one root node remains) adapts automatically but complicates the retrieval depth mapping.

4. **Working memory retention policy.** After consolidation, should working memory events be fully deleted or archived (e.g. moved to the object store as a raw log)? Archiving enables replay and debugging; deletion keeps things simple.

5. **Heuristic vs. LLM routing trade-off.** The keyword-based router is fast but brittle. How much accuracy do we sacrifice? Should the default be heuristic-first with LLM fallback only when confidence is low?

6. **Cross-layer `RELATED_TO` edges.** Currently `RELATED_TO` is same-level only. Should nodes at different levels be allowed to relate directly (e.g. a leaf fact relates to a top-level theme)? This could enrich retrieval but complicates graph traversal.

---

## Prior Art Deep-Dive

### MemGPT / Letta
Uses a paging metaphor: main context (fast, in-window), external storage (slow, retrieved on demand). Ambreous differs by making the "external storage" itself hierarchical and by routing based on task classification rather than context overflow.

### RAPTOR
Builds a tree of summaries by recursively clustering and summarizing leaf documents. Ambreous adapts this for streaming/online ingestion (via working memory → consolidation) rather than a one-shot offline indexing pass.

### GraphRAG
Extracts a knowledge graph from a corpus, detects communities at multiple granularities, and generates community summaries. Provides global (top-level community) vs local (entity-level) query modes. Ambreous generalizes the depth decision to N levels and makes it query-driven rather than a binary switch.

### HippoRAG
Models the hippocampal-neocortical memory system: a knowledge graph acts as the hippocampal index; dense retrieval populates it; Personalized PageRank scores nodes for a given query. Compatible with Ambreous's graph layer — PPR could be used inside `GraphRetriever`.

### A-MEM
Maintains a dynamic network of notes (Zettelkasten-style), linking them as new notes arrive. The linking is LLM-driven. Similar to Ambreous's `RELATED_TO` edges but without the hierarchical layer structure or depth routing.
