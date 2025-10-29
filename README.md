# Medical Coding Automation with RAG

> A Python project to automate the extraction of clinical entities and medical codes (ICD-10, CPT) from unstructured medical reports using a hybrid rule-based and retrieval-based system.

This solution implements a pipeline that ingests raw medical reports, extracts key clinical information, and retrieves the most relevant medical codes from a vectorized knowledge base.

##  overview

Manual medical coding is time-consuming, repetitive, and prone to human error. This project provides an automated solution by:

1.  **Parsing** unstructured medical reports.
2.  **Extracting** key entities (Diagnoses, Procedures, Anatomy, Clinical Terms) using a robust rule-based system.
3.  **Retrieving** relevant ICD-10 and CPT codes by comparing the report's clinical context against a vectorized database of medical codes.

This notebook uses **ChromaDB** as a local vector store and **Sentence Transformers** to generate embeddings, creating a powerful retrieval system that can run entirely on a local machine.

## ✨ Key Features

* **Hybrid Entity Extraction:** Combines a large, manually-curated clinical lexicon with regex patterns to identify key entities from structured sections (e.g., "Diagnosis:", "Impression:").
* **Vectorized Code Retrieval:** Uses a `SentenceTransformer` model (`all-MiniLM-L6-v2`) to embed and index hundreds of ICD-10 and CPT codes into a **ChromaDB** vector database.
* **Context-Aware Search:** Generates a query embedding from the most relevant sections of a medical report (like "Findings" and "Diagnosis") to find the most semantically similar codes.
* **Structured Output:** Generates a clean, predictable JSON output for each report, detailing all extracted entities and retrieved codes.
* **Local-First:** Loads all models (Sentence Transformer, LLM) and the database (ChromaDB) locally.
    *Note: The Falcon-7B LLM is loaded in the notebook but is not currently integrated into the final `MedicalCodeExtractor` pipeline. See "Potential Improvements" for how to leverage it.*

## 🚀 Technologies Used

* **Python 3.10+**
* **Core Libraries:**
    * `pandas` & `openpyxl`: For loading medical codes from Excel files.
    * `numpy`: For numerical operations.
    * `re`: For all rule-based and regex-driven extraction.
* **Vector Database:**
    * `chromadb`: For creating and querying the local vector store.
* **NLP & Embeddings:**
    * `sentence-transformers`: For generating text embeddings (`all-MiniLM-L6-v2` model).
* **LLM (Loaded, not fully integrated):**
    * `transformers` & `torch`: For loading the `tiiuae/falcon-7b-instruct` model.

---

## 🔧 System Architecture (Pipeline)

The core logic is encapsulated in the `MedicalCodeExtractor` class. The process works as follows:

### 1. Initialization & Data Ingestion

1.  **Load Embedding Model:** An instance of `SentenceTransformer('all-MiniLM-L6-v2')` is loaded.
2.  **Initialize Vector DB:** A `chromadb.Client()` is started, and any existing collections (`icd_codes`, `cpt_codes`) are cleared.
3.  **Load Medical Codes:**
    * The `ICD_code_Assignment.xlsx` and `cpt_code_assignment.xlsx` files are read using `pandas`.
    * Data is cleaned (stripping whitespace, formatting CPT codes).
4.  **Embed & Index:**
    * Each code and its description are combined into an "enhanced document" (e.g., `"K57.90: Diverticulosis"`).
    * The `SentenceTransformer` model encodes these documents into vectors.
    * The vectors, along with their metadata (code, description, type), are added to their respective `icd_collection` and `cpt_collection` in ChromaDB.

### 2. Report Processing (`process_report`)

When a new medical report is processed, it goes through two parallel steps:

**Step A: Rule-Based Entity Extraction (`extract_entities_rule_based`)**
* The system uses a large, hard-coded `entity_lexicon` (containing terms for `CLINICAL`, `ANATOMY`, `DIAGNOSIS`, `PROCEDURE`) to find matches in the text.
* It also uses regex to find specific sections like `Diagnosis:`, `Pre-operative Diagnosis:`, and `Impression:` to extract listed diagnoses and procedures.
* The results are cleaned, deduplicated, and returned.

**Step B: Medical Code Retrieval (`retrieve_codes_with_rag`)**
* The system first extracts the most relevant clinical context from the report using `extract_clinical_context`. This function uses regex to find and combine the text from `Diagnosis`, `Impression`, `Procedure`, and `Findings` sections.
* This combined context string is then encoded into a single query vector.
* This vector is used to query *both* the `icd_collection` and `cpt_collection` in ChromaDB, retrieving the `top_k=5` most similar codes.
* The top 3 results for each code type (ICD and CPT) are selected based on their retrieval confidence (1 - distance).

**Step C: Final Output**
* The results from Step A (Entities) and Step B (Codes) are combined into a single JSON object.

---
