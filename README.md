# UoB AI Assistant — Architecture Guide

This repo documents the architecture of the **University of Bahrain (UoB) AI Assistant** — how the user, LLM, and vector database interact in a full cycle to answer institutional questions accurately.

---

## Project Overview

The UoB AI assistant answers student and staff questions using **only** official UoB institutional data. It uses RAG (Retrieval-Augmented Generation) to fetch relevant data from a vector database before generating a response.

The LLM is constrained by a system prompt (`system_prompt.txt`) that enforces:
- JSON-only responses with a confidence score
- Strict use of provided data — no hallucination
- Bilingual support (Arabic & English)
- Prompt injection protection

---

## The Flow

---

## The Flow

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   ┌──────────┐                        ┌──────────────┐ │
│   │          │  1. User sends query   │              │ │
│   │   USER   │ ──────────────────────>│     LLM      │ │
│   │          │                        │              │ │
│   │          │  6. LLM sends answer   │              │ │
│   │          │ <──────────────────────│              │ │
│   └──────────┘                        └──────┬───────┘ │
│                                              │         │
│                                   2. LLM     │         │
│                                   embeds     │         │
│                                   the query  │         │
│                                   → vector   │         │
│                                              │         │
│                                   3. vector  ▼         │
│                                   ┌──────────────────┐ │
│                                   │  VECTOR DATABASE │ │
│                                   │                  │ │
│                                   │  4. similarity   │ │
│                                   │  search (cosine) │ │
│                                   └────────┬─────────┘ │
│                                            │           │
│                                   5. top   │           │
│                                   matches  │           │
│                                   returned │           │
│                                            ▼           │
│                                      (back to LLM)     │
└─────────────────────────────────────────────────────────┘
```

---

## Each Step Explained

### Step 1 — User → LLM
```
User: "How much does the laptop cost?"
```
The user sends a natural language question. The LLM receives raw text — it does not know the answer yet.

---

### Step 2 — LLM embeds the query
The LLM converts the text into a **vector** (a list of numbers that represents meaning):
```python
query_vector = embed("How much does the laptop cost?")
# → [0.23, 0.87, 0.45, 0.11, ...]
```
This is called an **embedding** — it captures the *meaning* of the words, not just the words themselves.

---

### Step 3 — LLM → Vector Database (Search)
The vector is sent to the vector database to find semantically similar content:
```python
results = vector_db.search(query_vector, top_k=3)
```
No exact keyword match needed — it finds results **closest in meaning**.

---

### Step 4 — Vector Database does similarity search
The database:
1. Compares the query vector against all stored vectors
2. Calculates distance (cosine similarity)
3. Returns the top N closest matches

```
query:   "How much does the laptop cost?"  → [0.23, 0.87, 0.45, ...]
stored:  "Laptop price is $999.99"         → [0.21, 0.85, 0.47, ...]
                                              ↑ very close = high similarity
```

---

### Step 5 — Vector Database → LLM (Result)
```json
[
  { "text": "Laptop price is $999.99", "score": 0.97 },
  { "text": "MacBook Pro costs $1299", "score": 0.81 }
]
```
The most relevant chunks of data return to the LLM.

---

### Step 6 — LLM → User (Answer)
The LLM uses the retrieved data to generate a grounded answer:
```
"The laptop costs $999.99."
```

---

## The Full Circle in Code

```python
# 1. User sends query
user_query = "What are the registration deadlines for Spring 2025?"

# 2. Embed the query into a vector
query_vector = embedding_model.encode(user_query)
# → [0.23, 0.87, 0.45, ...]

# 3. Search the UoB vector database
results = vector_db.search(query_vector, top_k=3)
# Step 4 happens inside here (cosine similarity)

# 5. DB returns top matches from UoB institutional data
# results = [{ "text": "Spring 2025 registration closes on Jan 15", "score": 0.97 }]

# 6. LLM generates a JSON response using retrieved UoB data
final_answer = llm.respond(
    system_prompt=open("system_prompt.txt").read(),
    user_query=user_query,
    context=results
)
# → {
#     "ai_interpretation": "User is asking about Spring 2025 registration deadline",
#     "response_confidence": 9,
#     "response": "Spring 2025 registration closes on January 15, 2025."
#   }
```

---

## Why Vector Search Instead of SQL?

| SQL Search | Vector Search |
|---|---|
| Exact keyword match | Meaning-based match |
| `WHERE name = 'laptop'` | finds "laptop", "MacBook", "notebook PC" |
| Fast for structured data | Fast for unstructured text/docs |
| Breaks if wording differs | Works even with different wording |

A user asking *"how much does the laptop cost?"* and *"laptop price?"* and *"what's the price of the MacBook?"* all map to the **same vector region** — vector search finds them all.

---

## Key Concept: RAG

This pattern is called **RAG** (Retrieval-Augmented Generation):

1. **Retrieval** — fetch relevant data from vector DB
2. **Augmented** — add that data as context to the LLM
3. **Generation** — LLM generates a grounded, accurate answer

Without RAG, the LLM only knows its training data. With RAG, it knows **your data** too.
