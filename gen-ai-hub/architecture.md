![Python](https://img.shields.io/badge/Python-3.13+-blue)
![Streamlit](https://img.shields.io/badge/Streamlit-Easy_Deploy-skyblue)
![Azure](https://img.shields.io/badge/Region-Germany%20West%20Central-darkgreen)
![Security](https://img.shields.io/badge/Security-AES--256-purple)
![GDPR](https://img.shields.io/badge/GDPR-Compliant-khaki)
![Architecutre](https://img.shields.io/badge/Architecture-Hub_&_Spoke-orange)



# 🏗️ Gen AI Hub – System Architecture

> **Portfolio Showcase:** An internal, multipage GenAI platform architected for **cross-functional innovation teams** (Designers, Engineers, Strategists).
> This repository demonstrates an **evolving Hub-and-Spoke architecture** built to strict **enterprise-grade standards** (GDPR, SSO, AES-256), serving 20+ specialized AI tools to streamline high-stakes client workflows.

---

## 🔹 TL;DR

- Multipage **Streamlit** application following a **Hub-and-Spoke** architecture.
- **`streamlit_app.py`** is the **single orchestrator**:
  - Handles authentication, page registration, navigation, and global state wiring.
- 20+ tools organized across **six primary domains**:
  - Info, Audio, Image, Chats, Agents, Experimental/Environmental.
- **Authentication-first execution**:
  - If the user is not logged in → login screen + `st.stop()`.
- **Dual-provider strategy**:
  - OpenAI (GPT-4o, o3, Agents SDK, DALL·E 3) for reasoning + advanced text & image.
  - Gemini (Flash, Veo, Audio) for fast multimodal workloads.
- Global resources:
  - `st.user` (OIDC identity), `st.session_state` (per-session context), `st.secrets` (credentials, keys).

---

## 1. Overview
The **Gen AI Hub** (also referenced internally as the *Indeed AI Toolbox*) is a multipage Streamlit application designed as the central entry point for the organization's AI ecosystem.

**Primary Objectives:**
- **Unified Experience:** Consolidates distinct tools into a single, intuitive UX for Designers, Engineers, and Strategists.
- **Provider Agnosticism:** Centralizes access to top-tier models, implementing a dual-provider strategy that routes complex reasoning to **OpenAI** and multimedia tasks to **Google Gemini**.
- **Rapid Evolution:** Supports a workflow-centric lifecycle where new tools can be prototyped and hardened behind stable navigation abstractions without disrupting the core system.

## 2. Security & Architecture
To ensure compliance and data safety, the platform enforces strict enterprise-grade constraints:

- **Authentication:** Access is gated via **Microsoft Entra ID SSO** (Azure AD) .
- **Data Residency:** All data processing is strictly confined to **Germany West Central**.
- **Encryption:** AES-256 encryption is applied at rest across all application layers.
- **Persistence Strategy:**
  - **Stateless Default:** Most tools operate without long-term storage to minimize the data footprint.
  - **Selective Persistence:** Only specific chat implementations (e.g., `simple_chat.py`) persist conversational history via Supabase(Postgres).

## 3. Functional Scope
The Hub currently hosts 20+ functional pages categorized into six core domains:

| Domain | Description | Key Tech/Models |
| :--- | :--- | :--- |
| **Conversational Chat** | Standard chat interfaces and advanced reasoning chains. | OpenAI |
| **Knowledge Agents** | Specialized agents for ISO standards and compliance. | RAG / Embeddings |
| **Audio Transcription** | Automated speech-to-text pipelines. | Whisper / Gemini |
| **Image Generation** | High-fidelity image creation from prompts. | DALL-E / Gemini |
| **Video Generation** | Content creation pipelines. | Google Veo |
| **Sustainability** | Data exploration interfaces for the IDEMAT database. | Data Visualization |


---

## 4. High-Level Hub-and-Spoke Architecture

At the highest level:

- The Hub:
  - `streamlit_app.py` – the **central orchestrator**.
- The Spokes:
  - Individual page modules like `simple_chat.py`, `gemini_simple_chat.py`, `ISO.py`, `generate.py`, `transcription.py`, etc.
- Shared Infrastructure:
  - Authentication
  - Secrets management
  - Supabase persistence (PostgreSQL)
  - AI provider clients

---

## 5. Application Structure Diagram

The application follows a strict execution flow:  
**Entry → Authentication → Page Registry → Navigation → Page Execution → Global Resources**

```mermaid
graph TD
    subgraph "Entry Point"
        Entry["streamlit_app.py Main<br>orchestrator"]
    end

    subgraph "Authentication Layer"
        Entry --> Gate{"st.user.is_logged_in<br>Gate check"}
        Logout["st.logout Session<br>termination"]
        
        Gate -- false --> LoginUI["login_screen()<br>SSO prompt"]
        LoginUI --> OAuth["st.login<br>OAuth flow"]
        OAuth -.->|Redirect| Gate
    end

    subgraph "Page Registry"
        Gate -- true --> Reg["PAGES dict 6 categories<br>20+ pages"]
        Reg --- Info["Info:<br>home.py, faq.py"]
        Reg --- Audio["Audio:<br>transcribe.py..."]
        Reg --- Image["Image:<br>generate.py..."]
        Reg --- Chats["Chats:<br>simple_chat.py..."]
        Reg --- Agents["Agents:<br>ISO.py..."]
    end

    subgraph "Navigation System"
        Reg --> NavFunc["st.navigation(PAGES)<br>Returns Page object"]
        NavFunc --> Exec["pg.run()<br>Executes selected page"]
    end

    subgraph "Global Resources"
        Exec -.-> User["st.user<br>OIDC identity"]
        Exec -.-> State["st.session_state<br>Per-session storage"]
        Exec -.-> Secrets["st.secrets<br>API keys, JWT secrets"]
    end
```

---

## 6. Application Lifecycle & Initialization

```mermaid
sequenceDiagram
    participant Runtime as Python Runtime
    participant Config as st.set_page_config()
    participant Auth as Authentication Check
    participant Nav as Navigation System
    participant Page as Selected Page Module

    Runtime->>Config: Set page title, icon, layout
    Runtime->>Auth: Check st.user.is_logged_in
    
    alt Not Logged In
        Auth->>Runtime: login_screen()
        Runtime->>Runtime: st.stop()
        Note over Runtime: Execution Halts Here
    else Logged In
        Auth->>Nav: Parse PAGES dict
        Nav->>Nav: st.navigation(PAGES)
        Nav->>Page: pg.run()
        
        activate Page
        Page->>Page: Initialize page-specific keys
        Page->>Page: Render UI Components
        Page->>Page: Update st.session_state
        deactivate Page
    end
```

---

## 7. AI Provider Integration Architecture

```mermaid
graph LR
    subgraph "OpenAI Services"
        GPT4["GPT-4o (Reasoning)"] -->|routes to| Py1["reasoning_chat.py"]
        Agents["Agents SDK"] -->|routes to| Py2["ISO.py, Packaging.py"]
        Dalle["DALL-E 3"] -->|routes to| Py3["generate.py"]
    end

    subgraph "Gemini Services"
        Flash["Gemini 1.5 Flash"] -->|routes to| Py4["gemini_simple_chat.py"]
        Veo["Veo 3.1 (Video)"] -->|routes to| Py5["gemini_video.py"]
        Audio["Gemini Audio"] -->|routes to| Py6["transcription.py"]
    end

    subgraph "Shared Infrastructure"
        Py1 & Py2 & Py3 & Py4 & Py5 & Py6 -->|consumes| Secrets["st.secrets API Keys"]
        Py1 & Py4 -->|persists to| DB["Supabase (PostgreSQL)"]
    end
```

---

## 8. Global Resources

- `st.user` → OIDC identity  
- `st.session_state` → persistent per-session context  
- `st.secrets` → secure API keys + JWT config  

---

## 9. Extensibility Model

Steps to add a new tool:

1. Create a module (e.g., `my_tool.py`)
2. Implement UI + state logic
3. Register in `PAGES`
4. (Optional) Add to tool card registry + docs

This allows independent evolution of spokes without breaking the Hub.

---

## 10. Related Documents

| File | Description |
|------|-------------|
| [**chat-systems.md**](./docs/chat-systems.md) | Architecture of all chat systems (OpenAI, Gemini, Reasoning, Transcript) |
| [**agents.md**](./docs/agents.md) | ISO Agent, Packaging Agent, Guardrail system |
| [**authentication.md**](./docs/authentication.md) | SSO, JWT, RLS identity mapping |
| [**persistence.md**](./docs/persistence.md) | DB schema, CRUD, auto-purge logic |
| [**heuristics.md**](./docs/heuristics.md) | Gemini auto-thinking heuristics |
| [**provider-routing.md**](./docs/provider-routing.md) | Dual-provider model routing |
| [**patterns.md**](./docs/patterns.md) | Page Registry, Auth Gate, Stateful/Stateless paradigms |
| [**ui-showcase.md**](./docs/ui-showcase.md) | Screenshots, demos, UI visualization |
