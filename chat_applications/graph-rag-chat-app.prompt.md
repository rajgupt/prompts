# Prompt: Multi-Page Streamlit Chat + Graph Explorer App

Generate a Python-only multi-page Streamlit application using the file structure, specifications, and backend function contracts defined below. All backend integration functions must be implemented as dummy stubs that `raise NotImplementedError`. The stubs serve as the integration contract — implementers fill them in with real API calls. Do not add any real credentials, API keys, or live connections anywhere in the generated code.

---

## Section 1 — App Overview & Constraints

- **Framework**: Streamlit (Python only, no JavaScript)
- **Pages**: Two pages — a main chat page and a graph explorer page
- **Backend stubs**: Every function in `backend/` must have full type annotations, a docstring describing its contract, and `raise NotImplementedError` as the body. No mock return values.
- **Persistence**: SQLite via Python's built-in `sqlite3` module. Call `init_db()` once at app startup in `app.py`.
- **Session state**: Use `st.session_state` for all cross-widget and cross-rerun state.
- **Navigation**: Use `st.switch_page()` for programmatic page transitions.

**Dependencies (add to `requirements.txt`):**
```
streamlit
openai
openai-agents
neo4j-graphrag
streamlit-agraph
```

---

## Section 2 — File Structure

Generate exactly these files:

```
app.py                    # Streamlit entry point — Page 1 (main chat)
pages/
    graph_explorer.py     # Page 2 — graph node explorer
backend/
    __init__.py           # empty
    llm.py                # OpenAI + OpenAI Agents SDK stubs
    graph_rag.py          # neo4j-graphrag stubs
    db.py                 # SQLite persistence stubs
requirements.txt          # dependencies listed in Section 1
```

---

## Section 3 — Page 1 Spec: Main Chat (`app.py`)

### Layout

Use `st.columns([1, 3, 2])` to create three panels side by side.

#### Left Panel — Thread List
- Display a "New Thread" button at the top.
  - On click: derive a default title from the first 40 characters of the next user input (use a placeholder title `"New conversation"` until the first message is sent), call `create_thread("chat", title)`, store the returned ID in `st.session_state["active_thread_id"]`, and rerun.
- Below the button, call `list_threads("chat")` and render each thread as a button showing its title and creation date.
- Clicking a thread button sets `st.session_state["active_thread_id"]` to that thread's ID and reruns.
- Highlight the active thread visually (e.g. bold text or different button style).

#### Center Panel — Chat Messages
- If no `active_thread_id` is set in session state, show a prompt asking the user to create or select a thread.
- Otherwise, call `load_thread_history(st.session_state["active_thread_id"])` and render each message using `st.chat_message(role)`.
- For assistant messages: render the `text` field as markdown. If the message has a non-empty `nodes` list, render each node as a small clickable button below the message text, labelled `[node["label"]] (node["type"])`. Clicking a node button sets `st.session_state["selected_node"]` to that node dict and reruns.
- Place `st.chat_input("Ask anything...")` at the bottom of the panel.

**On chat input submit:**
1. Set the thread title to the first message text (truncated to 40 chars) if this is the first message in the thread.
2. Call `save_message(thread_id, "user", user_input)`.
3. Call `graph_rag.get_graph_context(driver, user_input)` to get a graph context string.
4. Call `llm.create_agent(client, graph_context)` to get an agent.
5. Build `messages` from `load_thread_history(thread_id)` formatted as `[{"role": r["role"], "content": r["content"]}]`.
6. Call `llm.run_agent(agent, messages)` → `result` dict with keys `"text"` and `"nodes"`.
7. Call `save_message(thread_id, "assistant", result["text"], result["nodes"])`.
8. Rerun.

**Initialisation (top of `app.py`):**
```python
from backend import db, llm, graph_rag

db.init_db()
client = llm.create_openai_client()
driver = graph_rag.get_neo4j_driver()
```

Cache `client` and `driver` using `@st.cache_resource`.

#### Right Panel — Node Info
- If `st.session_state.get("selected_node")` is set, display:
  - A heading: the node's `label` value.
  - Each key-value pair in the node dict as `st.metric` or `st.text` rows.
  - A button "Explore in Graph Explorer". On click:
    - Set `st.session_state["explore_node_id"]` to `selected_node["id"]`.
    - Call `st.switch_page("pages/graph_explorer.py")`.
- If no node is selected, show a muted hint: `"Click a node in a chat message to see details here."`

---

## Section 4 — Page 2 Spec: Graph Explorer (`pages/graph_explorer.py`)

### Layout

Use `st.columns([3, 1])` for main graph area and right info panel.

#### Query Input
- Place a `st.text_input("Enter a query to explore the graph...")` above the columns.
- On page load, check `st.session_state.get("explore_node_id")`:
  - If set, pre-populate the query input with `f"Show node {st.session_state['explore_node_id']}"` and immediately trigger a graph query (treat it as a submitted query).
  - Clear `st.session_state["explore_node_id"]` after consuming it so navigating back doesn't re-trigger.
- On query submit:
  - Call `graph_rag.build_retriever(driver)` → `retriever`.
  - Call `graph_rag.query_graph(retriever, query)` → `{"nodes": [...], "edges": [...]}`.
  - Convert to `streamlit-agraph` objects:
    ```python
    from streamlit_agraph import Node, Edge, Config
    nodes = [Node(id=n["id"], label=n["label"], title=n.get("type", "")) for n in result["nodes"]]
    edges = [Edge(source=e["source"], target=e["target"], label=e.get("label", "")) for e in result["edges"]]
    ```
  - Store in `st.session_state["graph_nodes"]` and `st.session_state["graph_edges"]`.

#### Main Area — Interactive Graph
- If `st.session_state.get("graph_nodes")` is set, render:
  ```python
  config = Config(width=900, height=600, directed=True, physics=True, hierarchical=False)
  clicked_node = agraph(nodes=st.session_state["graph_nodes"], edges=st.session_state["graph_edges"], config=config)
  ```
- If `clicked_node` is not None, set `st.session_state["clicked_node_id"]` to `clicked_node` and rerun.
- If no graph data is in session state, show: `"Run a query above to explore the graph."`

#### Right Panel — Node Properties
- If `st.session_state.get("clicked_node_id")` is set:
  - Call `graph_rag.get_node_info(driver, st.session_state["clicked_node_id"])` → `node_info` dict.
  - Display a heading with the node ID.
  - Render each key-value pair in `node_info` as labelled text rows.
- Otherwise show: `"Click a node in the graph to see its properties here."`

**Initialisation (top of `pages/graph_explorer.py`):**
```python
from backend import graph_rag

driver = graph_rag.get_neo4j_driver()
```

Cache `driver` using `@st.cache_resource`.

---

## Section 5 — Backend Function Contracts

### `backend/llm.py`

```python
from __future__ import annotations
from typing import Any
import openai


def create_openai_client() -> openai.OpenAI:
    """
    Create and return a configured OpenAI client.

    Implementer: set the API key and (optionally) a custom base_url here
    to point at any OpenAI-compatible endpoint.

    Returns:
        openai.OpenAI: a ready-to-use client instance.
    """
    raise NotImplementedError


def create_agent(client: openai.OpenAI, graph_context: str) -> Any:
    """
    Create and return an OpenAI Agent using the OpenAI Agents SDK.

    The agent must be configured with a structured output tool named
    `return_response` that accepts `text: str` and `node_ids: list[str]`.
    This forces the model to explicitly cite which graph nodes are relevant
    to its reply rather than embedding them implicitly in prose.

    The `graph_context` string should be injected into the agent's system
    prompt so the model is aware of the available graph vocabulary.

    Args:
        client: the OpenAI client returned by `create_openai_client`.
        graph_context: a text summary of graph nodes/relationships relevant
            to the current user query, produced by `get_graph_context`.

    Returns:
        An agent object compatible with `run_agent`.
    """
    raise NotImplementedError


def run_agent(agent: Any, messages: list[dict]) -> dict:
    """
    Run the agent on the current conversation and return a structured reply.

    Args:
        agent: the agent returned by `create_agent`.
        messages: conversation history as a list of dicts with keys
            `"role"` ("user" | "assistant") and `"content"` (str).

    Returns:
        A dict with exactly two keys:
            "text"  (str):         the assistant's reply in markdown.
            "nodes" (list[dict]):  nodes cited by the agent, each with at
                minimum {"id": str, "label": str, "type": str}.
                Empty list if the agent cited no nodes.
    """
    raise NotImplementedError
```

### `backend/graph_rag.py`

```python
from __future__ import annotations
from typing import Any


def get_neo4j_driver() -> Any:
    """
    Create and return a Neo4j driver instance.

    Implementer: configure URI, authentication, and any driver settings here.
    Use the `neo4j` Python package (installed as a dependency of neo4j-graphrag).

    Returns:
        A Neo4j driver instance.
    """
    raise NotImplementedError


def build_retriever(driver: Any) -> Any:
    """
    Build and return a neo4j-graphrag retriever.

    Implementer: choose the retriever type (e.g. VectorCypherRetriever,
    HybridCypherRetriever), configure the embedder, vector index name,
    and Cypher retrieval template here.

    Args:
        driver: the Neo4j driver returned by `get_neo4j_driver`.

    Returns:
        A neo4j-graphrag retriever instance compatible with `query_graph`.
    """
    raise NotImplementedError


def query_graph(retriever: Any, query: str) -> dict:
    """
    Execute a natural language query through the graph retriever and return
    nodes and edges for visualisation.

    Args:
        retriever: the retriever returned by `build_retriever`.
        query: the user's natural language query string.

    Returns:
        A dict with two keys:
            "nodes" (list[dict]): each node has at minimum
                {"id": str, "label": str, "type": str}
                plus any additional properties.
            "edges" (list[dict]): each edge has at minimum
                {"source": str, "target": str, "label": str}.
    """
    raise NotImplementedError


def get_node_info(driver: Any, node_id: str) -> dict:
    """
    Fetch all stored properties for a single node by its ID.

    Args:
        driver: the Neo4j driver returned by `get_neo4j_driver`.
        node_id: the unique identifier of the node in the graph.

    Returns:
        A flat dict mapping property names to their values.
        Returns an empty dict if the node is not found.
    """
    raise NotImplementedError


def get_graph_context(driver: Any, query: str) -> str:
    """
    Produce a text summary of graph nodes and relationships relevant to `query`.

    This summary is injected into the agent's system prompt on Page 1 so the
    LLM is aware of graph vocabulary when formulating its reply.

    Implementer: use a neo4j-graphrag retriever or a direct Cypher query to
    find relevant subgraph content and serialise it as readable text.

    Args:
        driver: the Neo4j driver returned by `get_neo4j_driver`.
        query: the user's current input message.

    Returns:
        A string describing relevant graph context (nodes, relationships,
        properties). Return an empty string if no relevant context is found.
    """
    raise NotImplementedError
```

### `backend/db.py`

```python
from __future__ import annotations
import sqlite3


DB_PATH = "chat_history.db"  # change to an absolute path if needed


def init_db() -> None:
    """
    Create the SQLite database and tables if they do not already exist.

    Schema:
        threads  (id, page, title, created_at)
        messages (id, thread_id, role, content, nodes_json, timestamp)

    Call this once at application startup before any other db function.
    """
    raise NotImplementedError


def create_thread(page: str, title: str) -> int:
    """
    Insert a new conversation thread and return its generated ID.

    Args:
        page: identifier for which app page owns this thread
              (e.g. "chat" for Page 1).
        title: display title for the thread (e.g. first 40 chars of the
               first user message).

    Returns:
        The integer primary key of the newly created thread row.
    """
    raise NotImplementedError


def save_message(
    thread_id: int,
    role: str,
    content: str,
    nodes: list[dict] | None = None,
) -> None:
    """
    Persist a single chat message to the database.

    Args:
        thread_id: the ID of the thread this message belongs to.
        role: "user" or "assistant".
        content: the plain text or markdown content of the message.
        nodes: optional list of node dicts returned by `run_agent`.
               Serialised as JSON in the `nodes_json` column.
               Pass None or an empty list if there are no nodes.
    """
    raise NotImplementedError


def load_thread_history(thread_id: int) -> list[dict]:
    """
    Load all messages for a thread in chronological order.

    Args:
        thread_id: the ID of the thread to load.

    Returns:
        A list of message dicts, each with keys:
            "role"    (str):        "user" or "assistant".
            "content" (str):        message text.
            "nodes"   (list[dict]): deserialised node list (empty list if none).
    """
    raise NotImplementedError


def list_threads(page: str) -> list[dict]:
    """
    List all threads for a given page, newest first.

    Args:
        page: the page identifier used when threads were created
              (e.g. "chat").

    Returns:
        A list of thread dicts, each with keys:
            "id"         (int): the thread's primary key.
            "title"      (str): the thread's display title.
            "created_at" (str): ISO-format creation timestamp.
    """
    raise NotImplementedError


def clear_thread(thread_id: int) -> None:
    """
    Delete all messages belonging to a thread.

    Does not delete the thread row itself — the thread remains in the
    left panel but its history is emptied.

    Args:
        thread_id: the ID of the thread to clear.
    """
    raise NotImplementedError
```

---

## Section 6 — SQLite Schema

The `init_db()` function must execute the following SQL:

```sql
CREATE TABLE IF NOT EXISTS threads (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    page       TEXT    NOT NULL,
    title      TEXT    NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS messages (
    id         INTEGER PRIMARY KEY AUTOINCREMENT,
    thread_id  INTEGER NOT NULL REFERENCES threads(id) ON DELETE CASCADE,
    role       TEXT    NOT NULL CHECK(role IN ('user', 'assistant')),
    content    TEXT    NOT NULL,
    nodes_json TEXT,
    timestamp  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

## Section 7 — Dependencies (`requirements.txt`)

```
streamlit
openai
openai-agents
neo4j-graphrag
streamlit-agraph
```

`sqlite3` is part of the Python standard library and does not need to be listed.

---

## Generation Instructions

1. Generate all files listed in Section 2 with the exact paths shown.
2. Every function in `backend/` must match the signature, docstring, and `raise NotImplementedError` body shown in Section 5 exactly — do not add mock return values or placeholder logic.
3. `app.py` and `pages/graph_explorer.py` must import from `backend` using relative package imports (e.g. `from backend import llm, graph_rag, db`).
4. Use `@st.cache_resource` on `create_openai_client` and `get_neo4j_driver` call sites to avoid re-initialising connections on every rerun.
5. Do not hard-code any API keys, URIs, passwords, or model names anywhere. Leave those as clearly commented placeholders inside the stub docstrings only.
6. Keep all UI logic in `app.py` and `pages/graph_explorer.py`. No UI code in `backend/`.
