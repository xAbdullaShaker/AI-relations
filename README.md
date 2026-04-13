# UoB AI Assistant — Architecture Guide

This repo documents the architecture of the **University of Bahrain (UoB) AI Assistant** — how the user, cache, LLM, and vector database interact in a full cycle to answer institutional questions accurately.

---

## Project Overview

The UoB AI assistant answers student and staff questions using **only** official UoB institutional data. It uses a 3-layer approach before ever reaching the LLM:

1. **Cache** — instant answers for greetings and repeated questions
2. **FAQ layer** — keyword-matched answers for common questions
3. **RAG (Vector DB + LLM)** — semantic search + AI generation for complex questions

The LLM is constrained by a system prompt (`system_prompt.txt`) that enforces:
- JSON-only responses with a confidence score
- Strict use of provided data — no hallucination
- Bilingual support (Arabic & English)
- Prompt injection protection

---

## The Full Flow

```
                        ┌──────────┐
                        │   USER   │
                        └────┬─────┘
                             │
                    1. sends query
                             │
                             ▼
                   ┌──────────────────┐
                   │   CACHE / FAQ    │  ← Layer 1 & 2
                   │                  │
                   │  "hi" → "hello"  │
                   │  "hours?" → ...  │
                   └────────┬─────────┘
                            │
              ┌─────────────┴──────────────┐
              │                            │
          HIT (found)                 MISS (not found)
              │                            │
              ▼                            ▼
      answer instantly              ┌─────────────┐
      skip LLM entirely             │     LLM     │  ← Layer 3
              │                     └──────┬──────┘
              │                            │
              │               2. embed query → vector
              │                    [0.23, 0.87, ...]
              │                            │
              │                            ▼
              │                  ┌──────────────────┐
              │                  │   VECTOR DB      │
              │                  │                  │
              │                  │ 3. cosine search │
              │                  │    find closest  │
              │                  │    meaning match │
              │                  └────────┬─────────┘
              │                           │
              │              4. top matches returned
              │                           │
              │                           ▼
              │                  ┌─────────────┐
              │                  │     LLM     │
              │                  │             │
              │                  │ 5. generate │
              │                  │  JSON answer│
              │                  └──────┬──────┘
              │                         │
              └──────────┬──────────────┘
                         │
                6. answer back to user
                         │
                         ▼
                   ┌──────────┐
                   │   USER   │
                   └──────────┘
```

---

## Each Step Explained

### Step 1 — User sends a query
```
User: "What are the registration deadlines for Spring 2025?"
```
The raw question enters the system. Before anything expensive runs, it hits the cache first.

---

### Cache Layer — Instant answers (no LLM)

Handles greetings and repeated questions immediately:
```python
cache = {
    "hi": "Hello! How can I assist you?",
    "hello": "Hello! How can I assist you?",
    "thanks": "You're welcome! Is there anything else I can help with?",
}

if user_query.lower() in cache:
    return cache[user_query.lower()]  # done — LLM never called
```

---

### FAQ Layer — Keyword matching (no LLM)

Handles common institutional questions with slight wording variations:
```python
faqs = {
    "library hours":        "The UoB library is open Saturday–Thursday, 8am–10pm.",
    "registration deadline": "Spring 2025 registration closes on January 15, 2025.",
    "contact":              "Reach UoB at +973 17437000 or info@uob.edu.bh",
}

for keyword, answer in faqs.items():
    if keyword in user_query.lower():
        return answer  # still no LLM needed
```

---

### Step 2 — LLM embeds the query → vector

Only runs if cache and FAQ both missed. The LLM converts the query into a vector (a list of numbers representing meaning):
```python
query_vector = embedding_model.encode(user_query)
# "registration deadlines for Spring 2025?"
# → [0.23, 0.87, 0.45, 0.61, ...]
```
This captures **meaning**, not just keywords — so similar questions map to similar vectors.

---

### Step 3 — Vector DB similarity search

The vector is searched against all stored UoB institutional data:
```python
results = vector_db.search(query_vector, top_k=3)
```

```
query:   "registration deadlines Spring 2025"  → [0.23, 0.87, 0.45, ...]
stored:  "Spring 2025 reg closes Jan 15"       → [0.22, 0.85, 0.46, ...]
                                                  ↑ very close = high similarity ✓
```
No exact wording needed — meaning-based matching.

---

### Step 4 — Top matches returned to LLM

```json
[
  { "text": "Spring 2025 registration closes on January 15, 2025.", "score": 0.97 },
  { "text": "Late registration fee applies after January 10.", "score": 0.84 }
]
```
The most relevant UoB data chunks are passed back to the LLM as context.

---

### Step 5 — LLM generates a JSON answer

The LLM uses the system prompt + retrieved data to generate a structured response:
```python
final_answer = llm.respond(
    system_prompt=open("system_prompt.txt").read(),
    user_query=user_query,
    context=results
)
```

Output:
```json
{
  "ai_interpretation": "User is asking about the Spring 2025 course registration deadline",
  "response_confidence": 9,
  "response": "Spring 2025 registration closes on January 15, 2025. A late fee applies after January 10."
}
```

---

### Step 6 — Answer back to user

The JSON is parsed and the `response` field is displayed to the user.

---

## Layer Decision Table

| Question | Layer that handles it | LLM called? |
|---|---|---|
| "hi" / "hello" / "thanks" | Cache | No |
| "library hours?" / "contact?" | FAQ | No |
| "Can I transfer credits from abroad?" | Vector DB + LLM | Yes |
| "What is the GPA requirement for honours?" | Vector DB + LLM | Yes |

---

## Why Vector Search Instead of SQL?

| SQL Search | Vector Search |
|---|---|
| Exact keyword match | Meaning-based match |
| Breaks if wording differs | Works with any phrasing |
| Fast for structured data | Fast for unstructured text/docs |
| `WHERE topic = 'deadline'` | finds "deadline", "due date", "last day to register" |

---

## Key Concept: RAG

This pattern is called **RAG** (Retrieval-Augmented Generation):

1. **Retrieval** — fetch relevant UoB data from vector DB
2. **Augmented** — inject that data into the LLM as context
3. **Generation** — LLM generates a grounded, accurate JSON answer

Without RAG, the LLM only knows its training data. With RAG, it knows **UoB's data** too — and is forbidden from going beyond it.

---

## How the Query Becomes a Vector (and Why It Matters)

This is the core mechanism that prevents hallucination, speeds up search, and reduces token cost.

### Step 1 — Tokenization
The query is first broken into tokens (word pieces):
```
"What are the registration deadlines?"
→ ["What", "are", "the", "registr", "##ation", "dead", "##lines", "?"]
```

### Step 2 — Embedding model converts tokens to a vector
An embedding model (e.g. `text-embedding-3-small`) reads all tokens and outputs a single fixed-size vector of numbers:
```python
query_vector = embedding_model.encode("What are the registration deadlines?")
# → [0.23, -0.87, 0.45, 0.61, -0.12, 0.98, ...]  ← 1536 numbers
```

Each number represents a dimension of meaning. Similar sentences produce vectors that are close together in space.

```
"registration deadline"     → [0.23, 0.87, 0.45, ...]
"last day to register"      → [0.21, 0.85, 0.46, ...]  ← very close ✓
"library opening hours"     → [0.91, 0.12, 0.78, ...]  ← far away ✗
```

### Step 3 — Cosine similarity finds the closest match
The vector DB compares the query vector against every stored document vector using cosine similarity (angle between vectors):
```
similarity = 1.0  → identical meaning
similarity = 0.95 → very close
similarity = 0.50 → loosely related
similarity = 0.10 → unrelated
```

Only the top matches (e.g. top 3) are returned — not the full database.

---

## Why This Prevents Hallucination

Without vectors (pure LLM):
```
User: "What is UoB's GPA requirement for honours?"
LLM:  "Typically 3.5 GPA is required." ← GUESSED from training data, may be wrong
```

With vectors (RAG):
```
User:      "What is UoB's GPA requirement for honours?"
Vector DB: finds → "UoB requires a minimum 3.7 GPA for honours distinction."
LLM:       "UoB requires a minimum 3.7 GPA for honours distinction." ← from actual data
```

The LLM never answers from memory — it only sees the chunks retrieved from the vector DB. The system prompt forbids it from going beyond that data.

---

## Why This Reduces Tokens

Without RAG you would have to send the **entire UoB knowledge base** to the LLM every time:
```
prompt = system_prompt + ALL_UOB_DATA + user_query
# → 500,000+ tokens per request — slow and expensive
```

With RAG you only send the **top 3 relevant chunks**:
```
prompt = system_prompt + top_3_chunks + user_query
# → ~1,000 tokens per request — fast and cheap
```

| Approach | Tokens per request | Hallucination risk | Speed |
|---|---|---|---|
| Full data dump to LLM | 500,000+ | High (LLM gets confused) | Slow |
| RAG (vector search first) | ~1,000 | Low (only relevant facts) | Fast |

---

## Why This Speeds Up Search

Comparing text strings across thousands of documents is slow. Comparing vectors is a pure math operation — calculated in milliseconds even across millions of entries using indexes like **HNSW** (Hierarchical Navigable Small World):

```
Text search across 100,000 docs → scan every word → slow
Vector search across 100,000 docs → one math operation → milliseconds
```
