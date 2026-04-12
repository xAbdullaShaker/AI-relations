# User → LLM → Database: The Complete Circle

A visual and code explanation of how a user, an LLM, and a database interact in a full cycle.

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
│                                   decides    │         │
│                                   it needs   │         │
│                                   data       │         │
│                                              │         │
│                                   3. LLM     ▼         │
│                                   ┌──────────────────┐ │
│                                   │    DATABASE      │ │
│                                   │                  │ │
│                                   │  4. DB searches  │ │
│                                   │  & finds data    │ │
│                                   └────────┬─────────┘ │
│                                            │           │
│                                   5. Data  │           │
│                                   returned │           │
│                                   to LLM   │           │
│                                            ▼           │
│                                      (back to LLM)     │
└─────────────────────────────────────────────────────────┘
```

---

## Each Step Explained

### Step 1 — User → LLM
```
User: "What is the price of product #42?"
```
The user sends a natural language question. The LLM receives raw text — it does not know the answer yet.

---

### Step 2 — LLM decides it needs data
The LLM thinks:
```
"I don't have this in my training.
 I need to look it up. I'll query the database."
```
It figures out **what to search for** — this is called "tool use" or "function calling."

---

### Step 3 — LLM → Database (Query)
The LLM generates a structured query:
```sql
SELECT price FROM products WHERE id = 42;
```
It sends this to the database via a tool/API call.

---

### Step 4 — Database searches
The database:
1. Receives the query
2. Scans its tables
3. Finds the matching record

---

### Step 5 — Database → LLM (Result)
```json
{ "id": 42, "name": "Laptop", "price": 999.99 }
```
Raw data returns to the LLM. The LLM now has the facts.

---

### Step 6 — LLM → User (Answer)
The LLM takes the raw data and turns it into a human response:
```
"Product #42 (Laptop) costs $999.99."
```

---

## The Full Circle in Code (Simplified)

```python
# 1. User sends query
user_query = "What is the price of product #42?"

# 2. LLM processes and decides to search
llm_decision = llm.think(user_query)
# → "I need to query the database"

# 3. LLM queries the database
db_query = "SELECT price FROM products WHERE id = 42"
db_result = database.execute(db_query)  # Step 4 happens here

# 5. DB returns result to LLM
# db_result = { "price": 999.99 }

# 6. LLM builds final answer for user
final_answer = llm.respond(user_query, db_result)
# → "Product #42 costs $999.99"
```

---

## Key Concept: Why does the LLM go back?

The LLM has two phases:

| Phase | What happens |
|---|---|
| **Without data** | LLM only knows what it was trained on (static knowledge) |
| **With data** | LLM gets live/real facts from the DB, then answers accurately |

This pattern is called **RAG** (Retrieval-Augmented Generation) — the LLM is "augmented" by real data from your database before it answers.
