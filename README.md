# Contoso-O-Dimensions

# Architecture: Natural Language Data Interaction Patterns

## Overview
This repository documents the architectural patterns for building a **Natural Language Data Interface**. The core challenge addressed here is the tension between **Generative AI** (which is probabilistic and creative) and **Database Operations** (which require deterministic, exact accuracy).

This solution decouples the user interface (Natural Language) from the execution logic, splitting the architecture into three distinct paths based on the *intent* of the user's question: **Insight**, **Exactness**, or **Prediction**.

## The Core Dilemma
Standard "Out-of-the-Box" Copilots often fail at specific data tasks because they attempt to solve every problem with a single architectural pattern (RAG).

* **The Trap:** Asking an LLM to "read" 10,000 log entries to find an error counts as a *token processing* task, which is slow, expensive, and prone to hallucination.
* **The Solution:** The LLM should not *process* the data; it should **write the code** (SQL/Python) that processes the data.

## The "Three Paths" Architecture
The system ingests a large dataset (via Agents, Logic Apps, or APIs) and routes the request through one of three specific architectural flows.

### 1. The Insight Solution (Probabilistic)
* **Goal:** Summarization, sentiment analysis, broad trends, or "chatting" with unstructured documents.
* **Mechanism:** Retrieval Augmented Generation (RAG).
* **Key Tech:** Azure AI Search + Azure OpenAI.
* **Use Case:** *"Summarize the safety incidents reported last month."*

### 2. The Exact Solution (Deterministic)
* **Goal:** Hard numbers, row counts, specific timestamps, and "True/False" validation.
* **Mechanism:** **Text-to-SQL Agent**. The LLM generates a SQL query, validates it, executes it against the DB engine, and returns the *exact* result.
* **Key Tech:** Azure AI Foundry (Agent Logic) + Azure SQL Database + Azure Container Apps (Code Execution).
* **Use Case:** *"How many rows in the 'Orders' table changed status between 2 AM and 4 AM?"*

### 3. The Prediction Solution (Forecasting)
* **Goal:** Categorization, anomaly detection, and future forecasting.
* **Mechanism:** Machine Learning Models.
* **Key Tech:** Azure Machine Learning (AML) + Azure Data Pipelines.
* **Use Case:** *"Which servers are likely to fail next week based on current logs?"*

---

## Technical Design Choices

### Why a Hybrid "Brain & Face" Model?
We utilize a hybrid approach to balance usability with technical power.

#### The Face (UI): Copilot Studio
* Provides the chat interface within Microsoft Teams/Web.
* Handles authentication and basic conversation flow.
* **Decision:** We do *not* use Copilot Studio's native "Generative Answers" for the **Exact Path** because it lacks the granular control needed for complex SQL schema validation.

#### The Brain (Backend): Azure AI Foundry & Container Apps
* Hosts the custom Python/Semantic Kernel logic.
* **Decision:** We chose a custom Azure AI backend to implement a **Validation Loop** (see below) that prevents SQL injection and corrects syntax errors before execution.

### The "Validation Loop" (Text-to-SQL)
For the **Exact Solution**, we do not trust the LLM's first draft. The architecture implements a self-correction loop:
1.  **Draft:** Agent generates SQL based on user prompt + schema definition.
2.  **Validate:** A secondary logic block (or "Judge" agent) checks the SQL for safety (Read-Only enforcement) and syntax.
3.  **Execute:** The query runs against Azure SQL.
4.  **Repair:** If the DB returns an error, the Agent captures the error message and rewrites the SQL automatically.

### Why Azure Data Pipelines?
For the **Exact** and **Prediction** paths, real-time querying of raw logs is performance-prohibitive.
* **Decision:** We use Azure Data Pipelines (ADF) to pre-process raw logs into optimized "Summary Tables" (e.g., `Daily_Changes_Snapshot`).
* **Benefit:** This simplifies the schema the AI has to understand, reducing hallucination rates and query latency.

## Component Stack

| Component | Role | Justification |
| :--- | :--- | :--- |
| **Azure AI Foundry** | Orchestration | Unified platform for managing the Agent lifecycle and prompt flow. |
| **Azure SQL DB** | Source of Truth | Required for deterministic "counting" and math operations. |
| **Azure Container Apps** | Compute | Hosts the Python middleware that connects the LLM to the Database safely. |
| **Copilot Studio** | Frontend | Native integration into the M365 ecosystem (Teams/SharePoint). |
| **Azure Data Factory** | ETL | Optimizes data structure so the AI doesn't have to query raw, messy logs. |

---

## Future Roadmap
* **Agentic Assist UI:** Implementing a dedicated UI panel for "Human in the Loop" verification of generated SQL queries.
* **Knowledge Base Retrieval:** Integrating the "Insight" path more tightly so users can ask hybrid questions (e.g., *"Show me the 5 errors from last night [Exact] and summarize what they mean [Insight]"*).

## Advanced Architecture: Multi-Agent Orchestration & MCP

As an evolution of the "Three Paths" model, this architecture introduces a **Router Pattern** and utilizes the **Model Context Protocol (MCP)** to standardize data access. This design shifts from a static logic flow to a dynamic, agentic workflow.

![Multi-Agent MCP Architecture](https://github.com/user-attachments/assets/placeholder-for-new-diagram)

### 1. The Orchestrator (Router Agent)
Instead of hard-coded logic flows, a top-level **Orchestrator Agent** acts as the front-line interface.
* **Role:** It analyzes the user's natural language request to determine intent.
* **Mechanism:** It views the *Insight*, *Exact*, and *Prediction* capabilities not as separate pipelines, but as **Tools**.
* **Decision Logic:**
    * *User:* "Why did sales drop?" $\rightarrow$ **Tool Selection:** Prediction Agent (Root Cause Analysis).
    * *User:* "How many units sold yesterday?" $\rightarrow$ **Tool Selection:** Exact Data Agent (SQL).
    * *User:* "Summarize the sales meeting notes." $\rightarrow$ **Tool Selection:** Insight Agent (RAG).

### 2. Model Context Protocol (MCP) Server
To solve the "Large Dataset" integration challenge, we replace custom API glue code with an **MCP Server**.
* **What is it?** An open standard that enables the Large Dataset (SQL/Data Lake) to expose its schema and data to the LLM in a standardized format.
* **Benefit:** The Agents don't need hard-coded system prompts explaining the database structure. The MCP Server "broadcasts" available resources (tables, prompts, tools) to the Orchestrator.
* **Flow:**
    1.  The **Exact Data Agent** connects to the **SQL MCP Server**.
    2.  The Server securely exposes the schema for the `Daily_Changes` table.
    3.  The Agent uses this context to generate accurate SQL without manual schema injection.

### Updated Component Stack (Agentic Layer)

| Component | Role | Description |
| :--- | :--- | :--- |
| **Orchestrator LLM** | Router | A high-reasoning model (e.g., GPT-4o) that selects the correct tool based on user intent. |
| **SQL MCP Server** | Data Context | A standardized server layer that exposes database tables and safe query tools to the AI. |
| **Insight Tool** | RAG Agent | Specialized agent with access to Vector Search indexes. |
| **Prediction Tool** | ML Agent | Specialized agent that can trigger Azure ML inference endpoints. |
