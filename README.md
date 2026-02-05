# 🛠️ Engineering & Architecture Portfolio

> ### ⭐ Value-First Strategic Overview
>
> My expertise lies in translating workflow bottlenecks into measurable business impact using GenAI and Automation. I engineer systems that are not just technically sound, but designed to drive efficiency and compliance.
>
> * **Time Savings:** Built the internal GenAI Hub from scratch, establishing its vector database and driving user adoption which reduced initial research time by **50%** for design and strategy teams.
> * **Compliance & Security:** Architected the Hub to strict **GDPR** standards with AES-256 encryption and role-based access control.
> * **Workflow Mapping:** All solutions are developed with a **Human-Centric** philosophy, ensuring tools map directly to established business processes.
>
> This repository documents the architecture and design patterns behind these solutions. It is a **living document** that grows alongside my work.

---

---

## 📂 Project Directory

| Project / Module | Domain | Status | Description |
| :--- | :--- | :--- | :--- |
| **[Gen AI Hub](gen-ai-hub/architecture.md)** | **GenAI Architecture** | 🟢 **Active** | A GDPR-compliant, Hub-and-Spoke GenAI platform serving 20+ enterprise-grade internal tools. Includes Agentic Development with Pydantic guardrail and Postgres. |
| **[Product Carbon Footprint Automation](./n8n-workflows/product-carbon-footprint/)** | **Process Automation** | 🟢 **Active** | Documentation of end-to-end product carbon footprint calculation automation |
| **[GCP Data Workflows](./gcp-data-workflows)** | **Data & Analytics** | 🚧 *Planned* | Documentation and code for integrating Google ADK with **BigQuery** for large-scale data synchronization and reporting. |

---

## 🏗️ Design Philosophy
My work is driven by one goal: bridging the gap between **unstructured creative workflows** and **structured engineering systems**.

1.  **Value-First Architecture:** System design is prioritized over raw script volume. This repo documents architecture (Mermaid diagrams, Data Flows) to show *why* solutions were built.
2.  **Security by Design:** Whether building agents or integrations, security (GDPR, Auth) is baked in from day one, not patched on later.
3.  **The Underestimated Cost:** I design systems with **Human-in-the-Loop (HIL)** thinking to address the common flaw of underestimating the human time required for AI verification. Solutions must minimize the "Evaluation Tax"—the manual effort needed to validate and approve AI output for high-stakes professional use.
4.  **The Right Tool for the Layer:** I advocate for a **hybrid architecture**. I use **Python** for complex logic (Agents, RAG, Heuristics) and **Low-Code Orchestrators (n8n)** for efficient data movement and API glue. This portfolio showcases the "Heavy Lift" Python layer that powers the wider automation ecosystem.

---

## 👀 Highlighted Demo
## 👀 Highlighted Demo

**ISOagent — Agentic Compliance Workflow**

**ISO & PPWR Guardrail Agent — Ideation Phase Control**

This demo showcases an agent-driven workflow supporting the **ISO** and **PPWR** ideation phase for engineers and designers.  
A dedicated **Guardrail Agent** actively evaluates user inputs and **blocks requests that fall outside the defined regulatory and design scope**, ensuring relevance, compliance alignment, and reduced cognitive noise during early-stage ideation.

<video src="assets/ISOagent.mp4" controls width="100%">
  Your browser does not support the video tag.
</video>



## License
CC BY-NC 4.0 — non-commercial use only. Contact me for commercial licensing.


*Last Updated: December 2025*
