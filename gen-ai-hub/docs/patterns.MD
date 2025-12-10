# 🧩 Core Design Patterns in the Gen AI Hub

## Purpose & Scope
This document describes the **architectural and UI design patterns** used across the Gen AI Hub.  
These patterns ensure:

- High maintainability  
- Predictable user experience  
- Consistent behavior across 20+ tools  
- Separation of concerns  
- Extensibility for future tools  

This file captures the patterns exactly as implemented in the production codebase.

---

# 1. Page Registry Pattern

The **Page Registry Pattern** centralizes all page metadata in a single dictionary:

```python
PAGES = {
    "homepage": {
        "title": "Home",
        "icon": "🏠",
        "category": "info",
        "module": "home.py",
    },
    ...
}
```

### Benefits

| Benefit | Explanation |
|--------|-------------|
| **Single Source of Truth** | No duplicated routing logic; navigation reads 100% from the registry. |
| **Discoverability** | Adding a new tool is as simple as registering a new entry. |
| **Metadata Co-location** | Icons, titles, visibility rules, and file pointers live together. |
| **Extensible** | Feature flags, user roles, or beta labeling can be added in one place. |

### Execution Model

- `streamlit_app.py` loads `PAGES`
- Passes it to `st.navigation(PAGES)`
- Returns a `Page` object with a `.run()` method
- The chosen module executes in isolation

This pattern supports the Hub’s **Hub-and-Spoke** structure.

---

# 2. Tool Card Registry Pattern

Used in `home.py`.

A **CARDS** dictionary stores metadata for all tools to render “app-store-like” discoverability cards.

```
CARDS = {
  "simple_chat": {
    "title": "Simple Chat",
    "description": "Fast and lightweight GPT chat.",
    "tags": ["chat", "openai"],
    "status": "stable"
  },
  ...
}
```

### Benefits

| Benefit | Explanation |
|--------|-------------|
| **Metadata-driven UI** | The home page is rendered dynamically, not hard-coded. |
| **Supports multiple views** | Category view, workflow view, list view, or featured-view use the same metadata. |
| **Decoupling** | No UI logic inside page modules. Cards act as declarations. |
| **Scalability** | Works for 20+ tools without clutter. |

---

# 3. Authentication Gate Pattern

Every page begins with:

```python
if not getattr(st, "user", None) or not st.user.is_logged_in:
    login_screen()
    st.stop()
```

### Why this is critical

- Prevents accidental execution of expensive operations before authentication  
- Prevents unauthorized access to tools (ISO agent, video generation, etc.)  
- Eliminates UI flicker or partial rendering  
- Centralizes security enforcement  

### Behavioral Guarantees

- **Hard-stop execution** until login is complete  
- **No provider calls** allowed before authentication  
- **Consistent experience** across modules  

This is one of the most important patterns in the entire system.

---

# 4. Stateless vs Stateful Page Pattern

The Hub uses two persistence paradigms depending on workflow needs:

### 4.1 Stateless Pages
Examples:
- ISO Agent (`ISO.py`)
- DALL·E image generation (`generate.py`)
- Reasoning Chat (`reasoning_chat.py`)
- Veo video generation

| Property | Description |
|----------|-------------|
| **No DB writes** | Stateless pages use session only |
| **Faster** | No round-trips to Supabase |
| **Lower risk** | Sensitive data never stored |
| **Simpler UX** | No references to conversation history |

Used for workflows where inputs are either sensitive or short-lived.

---

### 4.2 Stateful Pages
Examples:
- `simple_chat.py`  
- `gemini_simple_chat.py`

| Property | Description |
|----------|-------------|
| **Full DB persistence** | Conversations and messages stored in Supabase |
| **Conversation continuity** | Users can return days later to resume chats |
| **CO₂ tracking continuity** | Carbon model accumulates per user |
| **Autosave model** | Every assistant & user message is captured |

State is stored across:
- `st.session_state`
- Supabase `conversations` table
- Supabase `messages` table  

This dual-layer design allows rich UX without losing session integrity.

---

# 5. Navigation Pattern (`st.navigation`)

Introduced early in the app lifecycle:

```python
pg = st.navigation(PAGES)
pg.run()
```

### Why this pattern matters

| Advantage | Description |
|-----------|-------------|
| **Declarative navigation** | Navigation is not manually coded; it’s driven by the registry. |
| **Automatic refresh** | Streamlit handles re-renders elegantly across reruns. |
| **Decoupled execution** | Business logic lives inside `page.run()`. |
| **Future-friendly** | Supports nested navigation, mobile-friendly UI, and menu expansion. |

---

# 6. Secrets Injection Pattern

Secrets are **never hard-coded**.

Instead:

```python
openai_key = st.secrets["openai"]["api_key"]
supabase_url = st.secrets["supabase"]["url"]
```

### Benefits

| Benefit | Explanation |
|--------|-------------|
| **Security** | AES-256 encrypted secrets; nothing in Git history. |
| **Environment-specific overrides** | Dev, staging, and prod secrets differ. |
| **Provider abstraction** | Switching models/providers does not require code changes in business logic. |

---

# 7. Page-Level State Initialization Pattern

Each page initializes its session state keys **once** at the start:

```python
if "messages" not in st.session_state:
    st.session_state.messages = []
```

This prevents:

- KeyErrors  
- Dirty shared state between pages  
- UI glitches from undefined state  

It also ensures repeatable behavior on reload or rerender.

---

# 8. Error-Resistant Rendering Pattern

This pattern ensures the UI never crashes due to upstream failures.

Concept:

```python
try:
    result = api_call()
except Exception:
    st.error("The provider is temporarily unavailable.")
    return
```

Ensures:

- LLM outages do not break UI  
- User receives helpful messaging  
- Does not leak stack traces or sensitive info  
- Keeps navigation usable even during upstream failures  

---

# 9. Related Documentation

| File | Description |
|------|-------------|
| [**architecture.md**](../architecture.md) | High-level system overview, Hub-and-Spoke architecture, lifecycle diagrams |
| [**chat-systems.md**](./chat-systems.md) | Architecture of all chat systems (OpenAI, Gemini, Reasoning, Transcript) |
| [**agents.md**](./agents.md) | ISO Agent, Guardrail system, RAG pipeline, Packaging Agent |
| [**authentication.md**](./authentication.md) | Entra ID SSO, JWT identity mapping, RLS integration |
| [**persistence.md**](./persistence.md) | Supabase schema, CRUD functions, auto-purge retention logic |
| [**heuristics.md**](./heuristics.md) | Gemini auto-thinking heuristics (temperature, budgets, cost protection) |
| [**provider-routing.md**](./provider-routing.md) | Dual-provider routing between OpenAI & Gemini |
| [**patterns.md**](./patterns.md) | Core design patterns across the Hub |
| [**ui-showcase.md**](./ui-showcase.md) | Screenshots, video demo references, UI patterns |

---