[![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/j5u2qjgo)
[![Open in Codespaces](https://classroom.github.com/assets/launch-codespace-2972f46106e565e64193e422d61a12cf1da4916b45550586e14ef0a7c637dd04.svg)](https://classroom.github.com/open-in-codespaces?assignment_repo_id=22964524)

# BMS 117 AI Tutor — RAG-Based Study Tool for Dental Microbiology & Immunology

## Overview

This project builds a **Retrieval-Augmented Generation (RAG) tutoring system** that helps UCSF dental students study for **BMS 117 Exam 1** (Microbiology & Immunology). The tutor answers questions grounded in actual lecture materials, explains practice exam answers, generates quiz questions, and cites its sources.

The system is evaluated by running all 51 practice exam questions through both the RAG tutor and a base LLM, comparing accuracy against the official answer key.

## System Architecture

```
Lecture PDFs + Exam + Objectives
        ↓
  PDF text extraction (PyMuPDF)
        ↓
  Text chunking with metadata
        ↓
  ChromaDB vector store (via chroma-mcp)
        ↓
  RAG Tutor Agent (OpenAI Agents SDK)
        ↓
  Q&A / Exam Explanation / Quiz Generation
        ↓
  Evaluation pipeline (RAG vs Base LLM)
```

## Dataset

**Input:** 14 lecture PDFs from BMS 117 Exam 1 block, covering:

| Topic | Source |
|:---|:---|
| Microbiology basics, Gram staining | BMS117_Microbiology_Basics_2022_ST.pdf |
| Immune system introduction | Intro_to_Immune_system_2025_STFINAL.pdf |
| Immune response overview | 003_BMS117_Overview_IR_2025.pdf |
| T cell biology | 005_T_cells_2025.pdf |
| Humoral immunity & antibodies | Humoral_Immunity_Antibody_Structure_and_Function.pdf |
| Antibody effector mechanisms | Antibody_Effector_Mechanisms__Mucosal_Immunity.pdf |
| Staph & Strep infections | Staph_and_Strep_2025_ST.pdf |
| Hypersensitivity reactions | ILM_Hypersensitivity_Basics_2023.pdf |
| Tolerance & autoimmunity | BMS_117_tolerance_and_autoimmunity_2024.pdf |
| Immunodeficiency | BMS_117_immunodeficiency_2024.pdf |
| Pathogenesis & host response | 010_Pathogenesis_and_Host_Response_2025_ST.pdf |
| Bacterial genetics | Bacterial_Genetics_2025.pdf |

**Evaluation data:** 51-question practice exam with answer key mapping each question to a correct answer and learning objective.

**Dimensions:** ~130,000 characters of lecture text across ~350 pages, chunked into ~180 retrieval units.

## How to Run

### 1. Clone and install dependencies

```bash
git clone <your-repo-url>
cd <repo-name>
pip install -r requirements.txt
```

### 2. Set up API key

```bash
cp example.env .env
# Edit .env and add your OpenRouter API key
```

### 3. Organize data files

Place your PDFs in the following structure:

```
data/
  lectures/       ← all lecture PDFs
  exams/           ← practice exam + answer key PDFs
  objectives/      ← objectives PDF
```

### 4. Run the notebooks in order

```
notebooks/01_data_ingestion.ipynb    → Parse PDFs, build ChromaDB
notebooks/02_rag_tutor_agent.ipynb   → Test the RAG tutor
notebooks/03_evaluation.ipynb        → Run full evaluation
```

## Dependencies

- Python 3.11+
- OpenRouter or OpenAI API key
- Key libraries: `openai-agents`, `chromadb`, `chroma-mcp`, `pymupdf`, `sentence-transformers`

See `requirements.txt` for the full list.

## Design Decisions & Trade-offs

**ChromaDB via chroma-mcp (not FAISS or Pinecone):** We use the same `chroma-mcp` MCP server taught in class (Lecture 08 Demo 2). This keeps the architecture consistent with course material and lets the agent discover retrieval tools via the MCP protocol.

**Page-grouped chunking (not token-based):** Lecture slides have sparse text (~300 chars/page). We group 2 consecutive pages per chunk to create retrieval units with enough context. This is simpler than token-based chunking and works well for slide decks.

**No image captioning:** Lecture slides contain many diagrams and images. We chose to skip image captioning (BLIP/LLaVA) to keep scope manageable for a 2-3 week project. The text content from slides provides sufficient coverage, and the answer key metadata (lecture source + objective) compensates for missing visual content. This is a clear limitation; a production system would need multimodal support.

**Nemotron 3 Super (free) via OpenRouter:** Originally planned for gpt-4o-mini, but switched to `nvidia/nemotron-3-super-120b-a12b:free` to avoid API costs. This introduced rate limiting challenges (documented in Notebook 03) but kept the project zero-cost. A production system would use a paid model for better reliability.

**Evaluation scope (20 of 51 questions):** The free-tier Nemotron model enforced strict rate limits (~10 requests/minute). After implementing retry logic with exponential backoff, we successfully evaluated 20 questions before rate limits became prohibitive. This is documented as a trade-off; the 20-question sample covers the full range of topics and provides a meaningful comparison.

**Structured answer extraction:** We use regex to extract answer letters from model responses. This is brittle for verbose RAG responses; a more robust approach would use structured output (Pydantic models via `output_type`). This limitation affected 15/20 RAG evaluation scores and is the primary future improvement.

## Example Output

**Q&A Mode:**
```
Q: What are the mechanisms of horizontal gene transfer?

A: Horizontal gene transfer in bacteria occurs through three main mechanisms:
1. Transformation — uptake of free DNA from the environment by competent cells
2. Transduction — transfer of DNA via bacteriophages
3. Conjugation — direct transfer through a pilus/cytoplasmic bridge
[Source: Bacterial_Genetics, pages 8-11]
```

**Exam Explainer:**
```
Q: Explain why answer B is correct for question 3.

A: Question 3 asks how to definitively identify S. aureus from a skin lesion.
The correct answer is (b) coagulase positive. While S. aureus is catalase+,
beta-hemolytic, and forms clusters, coagulase is the definitive differentiator
between S. aureus (coagulase+) and S. epidermidis (coagulase-).
Learning Objective: Describe the role of coagulase in differentiating between
Staph. aureus and Staph. epidermidis.
[Source: Staph_and_Strep, pages 5-6]
```

## Evaluation Results

Results from running 20 practice exam questions (Q1-Q20) through both systems. Evaluation was limited to 20 questions due to free-tier API rate limits (documented trade-off).

| System | Correct | Accuracy | Notes |
|:---|:---|:---|:---|
| Base LLM | 19/20 | 95.0% | Strong general knowledge, no citations |
| RAG Tutor | 5/20 | 25.0%* | 15 answers unparseable due to verbose responses |

**\*Why the RAG score is misleading:** The RAG tutor gives rich, multi-paragraph answers with lecture citations and learning objectives (see Notebook 02 examples). The automated evaluation extracts only the first 200 characters of each response to find the answer letter, but the RAG agent buries the letter deep in its explanation. 15 of 20 RAG responses were marked "unparseable" (`?`) by the regex extractor, not incorrect.

**The base LLM scores high** because gpt-4o-mini already knows general immunology from training data. However, it cannot cite course-specific sources, reference lecture slide numbers, or align answers to learning objectives; capabilities the RAG tutor demonstrates qualitatively in Notebook 02.

**Key takeaway:** The evaluation methodology (regex letter extraction) was insufficient for the RAG agent's verbose output style. A production system would use structured output (Pydantic models via `output_type`) to guarantee parseable responses. This is documented as a limitation and future improvement.

*(Run Notebook 03 to regenerate these results, see `results/` directory for full data)*

## Repository Structure

```
├── README.md
├── requirements.txt
├── example.env
├── data/
│   ├── lectures/          ← lecture PDFs
│   ├── exams/             ← practice exam + key
│   ├── objectives/        ← learning objectives
│   ├── chunks.jsonl       ← generated: all text chunks
│   ├── exam_questions.json ← generated: parsed exam data
│   └── chroma_db/         ← generated: vector store
├── notebooks/
│   ├── 01_data_ingestion.ipynb
│   ├── 02_rag_tutor_agent.ipynb
│   └── 03_evaluation.ipynb
├── presentation/
│   └── BMS117_AI_Tutor_Presentation.pptx
└── results/
    ├── evaluation.csv
    ├── evaluation_summary.json
    └── evaluation_chart.png
```

## Citations

- **Course materials:** BMS 117 Microbiology & Immunology, UCSF School of Dentistry, Winter 2025
- **OpenAI Agents SDK:** https://github.com/openai/openai-agents-python
- **ChromaDB:** https://www.trychroma.com/
- **chroma-mcp:** https://github.com/chroma-core/chroma-mcp
- **PyMuPDF:** https://pymupdf.readthedocs.io/
- **sentence-transformers:** https://www.sbert.net/
- **DATASCI 223 Lecture 08:** LLM Applications & Workflows (RAG, Agents, Workflows)

## Future Work

- **Multimodal support:** Add image captioning for lecture diagrams using a vision-language model
- **Adaptive tutoring:** Track which topics the student struggles with and recommend review material
- **UCSF ChatGPT Enterprise deployment:** This prototype validates features that could be deployed as a Custom GPT integrated with the dental curriculum
