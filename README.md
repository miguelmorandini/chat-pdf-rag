# RAG Projects with Google Gemini

[![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://chat-com-pdf.streamlit.app)

This repository brings together complementary approaches for applying **RAG (Retrieval-Augmented Generation)** architecture with **Google Gemini**:

1. **[Chat with PDF](chat_com_pdf.ipynb):** Lightweight implementation for answering questions about PDF documents using PyPDF2 and TF-IDF.
2. **[Databricks Assistant](databricks_rag_assistant.ipynb):** Multi-document RAG platform using official Databricks technical documentation, powered by **ChromaDB** and **Google Embeddings (`gemini-embedding-001`)**.
3. **[Databricks Technical Assistant (Web App)](streamlit_app.py):** Conversational web interface with streaming, persistent chat history, and Lakehouse design, hosted on Streamlit Cloud: **[https://chat-com-pdf.streamlit.app](https://chat-com-pdf.streamlit.app)**.

---

## Live Web Application

- **Real-Time Streaming:** Responses typed progressively on screen using `st.write_stream`.
- **Multi-Turn Conversational Memory:** Retains context from previous questions for smooth technical dialogues.
- **Persistent Sidebar History:** Start new conversations (`+ Nova Conversa`) and navigate across past chats.
- **Dynamic Model Discovery:** Automatic routing across active Google models (`client.models.list()`) with resilient quota protection.

---

## Available Notebooks and Applications

| Project / Notebook | Focus | Retrieval | Storage | LLM | Online Access |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`databricks_rag_assistant.ipynb`** | Multi-Docs (Databricks Knowledge Base) | Semantic Embeddings | Local ChromaDB | Google Gemini | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/miguelmorandini/chat-com-pdf/blob/main/databricks_rag_assistant.ipynb) [![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://chat-com-pdf.streamlit.app) |
| **`chat_com_pdf.ipynb`** | Single PDF (Delta Lake eBook) | TF-IDF + Cosine | In-memory | Google Gemini | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/miguelmorandini/chat-com-pdf/blob/main/chat_com_pdf.ipynb) |
| **`streamlit_app.py`** | Full Web App (Cloud Deploy) | Hybrid RAG + Fallback | ChromaDB / Memory | Dynamic Discovery | [![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://chat-com-pdf.streamlit.app) |

---

## 1. Chat with PDF (Initial Challenge)

- **Environment:** Runnable on Google Colab and locally.
- **Extraction:** Page-by-page text mapping with `PyPDF2`.
- **Chunking:** Overlapping block segmentation to preserve context continuity.
- **Retrieval:** Cosine similarity search via TF-IDF (`scikit-learn`).
- **Anti-Hallucination Prompt:** Explicit rules to prevent fabrication and require page citations.
- **Deliverables:**
  - 3 technical questions answered with page citations.
  - 1 out-of-scope question answered with: *"Nao encontrei essa informacao no documento."*

---

## RAG Pipeline Architecture

```
                 +-----------------------------------------+
                 |              PDF Document               |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 |     1. Page Extraction (PyPDF2)         |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 |    2. Overlapping Chunking (800 / 150)  |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 |   3. TF-IDF Matrix & Vector Index       |
                 +-----------------------------------------+
                                      |
 User Question  --------------------->+ (Cosine Similarity)
                                      |
                                      v
                 +-----------------------------------------+
                 |     Top-k Relevant Chunks + Pages       |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 | 4. Anti-Hallucination Prompt + Context  |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 |    5. Google Gemini (GenAI API)         |
                 +-----------------------------------------+
                                      |
                                      v
                 +-----------------------------------------+
                 |  Answer with Page Citation (Gradio)     |
                 +-----------------------------------------+
```

---

## How to Run

### Option 1: Google Colab
Open the notebook directly in the cloud:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/miguelmorandini/chat-com-pdf/blob/main/chat_com_pdf.ipynb)

> Set your Google AI Studio key in Colab **Secrets** under the name `GEMINI_API_KEY`.

---

### Option 2: Locally

1. **Clone the repository:**
   ```bash
   git clone https://github.com/miguelmorandini/chat-com-pdf.git
   cd chat-com-pdf
   ```

2. **Create and activate the virtual environment:**
   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\activate
   # Linux/Mac:
   source .venv/bin/activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Set the environment variable:**
   ```bash
   # Windows (PowerShell):
   $env:GEMINI_API_KEY="your_key_here"

   # Linux/Mac:
   export GEMINI_API_KEY="your_key_here"
   ```

5. **Run:**
   ```bash
   # Option A: Run the modern Databricks Chat web app (Streamlit):
   streamlit run streamlit_app.py

   # Option B: Run the original Gradio app (Chat with PDF):
   python app.py

   # Option C: Open the interactive notebooks:
   jupyter notebook databricks_rag_assistant.ipynb
   ```

---

## 2. Databricks Assistant & Continuous Ingestion Pipeline

The second project (`databricks_rag_assistant.ipynb`) features an **automated incremental ingestion pipeline** (`src/ingest_pipeline.py`) that keeps the knowledge base synchronized with the official Databricks documentation site:

- **Official Source:** Automatic reading of the official sitemap (`https://docs.databricks.com/sitemap.xml`), covering Lakehouse, Delta Lake, Unity Catalog, Delta Live Tables, Ingestion, Optimizations, PySpark, and SQL.
- **Change Detection (SHA-256):** Compares each page's text hash against `data/ingestion_state.json`. Unchanged pages are skipped (0 tokens consumed).
- **Batch Embeddings:** Sends chunks in batches with `gemini-embedding-001`, reducing API calls by up to 90%.
- **Smart Fallback with Official Grounding (Zero Cost):** When the user asks about topics not yet in the local knowledge base (ChromaDB), the assistant automatically triggers Google Search Grounding restricted to `docs.databricks.com`, ensuring accurate answers and official links in real time at no cost on the free tier.
- **Weekly GitHub Actions Automation:** The `.github/workflows/sync_databricks_docs.yml` workflow runs autonomously every Sunday at 03:00 UTC, updating the vector database and committing changes to the repository.

### How to run the ingestion pipeline manually:
```bash
# Dry-run scan without spending tokens:
python src/ingest_pipeline.py --dry-run --max-pages 20

# Run actual ingestion of 50 pages:
python src/ingest_pipeline.py --max-pages 50
```

---

## Challenge Validation

| Test Type | Question | Expected Behavior |
| :--- | :--- | :--- |
| **Technical Question 1** | *What is Delta Lake and how does it bring reliability?* | Accurate answer citing pages (e.g., pp. 3 and 8) |
| **Technical Question 2** | *What is a Lakehouse and what problems does it solve?* | Conceptual summary citing pp. 5, 6, and 13 |
| **Technical Question 3** | *How does Delta Lake handle ACID transactions?* | Explanation of the transaction log citing pp. 8 and 13 |
| **Anti-Hallucination Test** | *Qual e a escalacao da selecao brasileira em 2002?* | `"Nao encontrei essa informacao no documento."` |
