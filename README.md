# An Experimental Study on OCR Output Optimization via Large Language Models and Prompt Engineering

This repository contains the source code, experimental framework, and thesis dataset for a research project focused on automating structured data extraction from heterogeneous documents. The study implements a hybrid pipeline that couples an optical character recognition engine with a state-of-the-art Large Language Model (LLM) to evaluate the semantic correction capabilities of various prompt engineering strategies.

---

## 🎯 Project Overview & Objective

The primary objective of this study is the design, implementation, and experimental validation of a dedicated benchmarking framework engineered to optimize data extraction accuracy within hybrid pipelines (OCR integrated with LLMs).

Traditional OCR processes often struggle with document semantics or non-standard layouts. This project addresses those limitations by isolating the problem within a controlled environment, evaluating how different Prompt Engineering techniques can correct, structure, and normalize raw OCR text into standardized JSON outputs ready for database ingestion.

---

## 💻 Technical Architecture & Stack

The benchmarking platform is built around an action-oriented architecture, strictly implementing the **Command/Action Design Pattern** to ensure high modularity, testability, and decoupling of business logic.

* 
**Backend & Orchestration:** PHP 8.2 and **Laravel 11**.


* 
**Asynchronous Processing:** Powered by **Laravel Job Queues** (`BulkRunJob`). High-volume external API testing is decoupled into background workers operating on a FIFO paradigm to avoid blocking the user interface.


* 
**Data-Centric Frontend:** Developed using **Filament PHP** and **Livewire** (TALL Stack) to build real-time, dynamic dashboards for multi-prompt statistical comparison and dataset management.


* 
**AI Pipeline Integration:** * **OCR Gateway:** **Amazon Textract API** , chosen for its layout analysis capabilities and robust SDK integration. Raw responses are parsed and structured into `cellMap`, `formFields`, and `rawLines`.


* 
**Semantic Engine:** **OpenAI GPT-4o API** , utilized via the **PrismPHP** library for automated context injection and schema-validated structured JSON outputs.




* 
**Database Schema:** A normalized relational MySQL database that physically decouples test definitions (`DetailSet`, `DocumentDetail`, `Prompt`) from executions (`Run`, `ExtractedField`) and evaluations (`GroundTruth`, `BenchmarkResult`).



---

## 🧪 Experimental Methodology & Metrics

The system was evaluated against a diverse dataset of 20 documents classified by type (Invoices, Receipts, ID documents, Corporate Intelligence, Newspapers) , quality (HD, Low-def, Noisy, Distorted) , and layout (Prose vs. Structured).

Five prompt engineering strategies were benchmarked:

1. 
**Zero-Shot Prompting:** Minimal instructions focusing purely on task execution and schema alignment.


2. 
**Few-Shot Prompting:** Context window populated with detailed correction rules and high-quality input-output examples.


3. 
**Chain-of-Thought (CoT) Prompting:** Instructing the model to break down problems into logical intermediate steps.


4. 
**Hybrid Approach:** A highly sophisticated prompt combining Forensic directives, CoT, and Structural Awareness.


5. 
**Emotion Prompting:** Integration of psychological and emotional stimuli phrases to influence generation behaviors.



### Evaluation Metrics

Extracted fields were scored on a rigorous $0.0$ to $1.0$ accuracy scale:

* 
**Strings:** Evaluated using the **Oliver (similar_text)** algorithm to prioritize semantic content over strict lexical alignment (e.g., ignoring token order swapping).


* 
**Dates:** Strict semantic match validation via **Carbon PHP** to eliminate misleading textual similarity false positives.


* 
**Numbers:** Currency sanitization followed by an absolute differential decimal scoring system ($1.0 - |\Delta|$).



---

## 📈 Key Findings & "Attention Dilution"

The experimental results yielded a highly counterintuitive, non-linear trend : **increasing prompt complexity and length does not correlate positively with output accuracy**.

* 
**Zero-Shot Superiority:** The **Zero-Shot prompt achieved the highest overall accuracy (87.01%)**. It proved incredibly resilient for short string and numerical extractions ($93\%$ accuracy) , forcing atomic, noise-free outputs.


* 
**Attention Dilution Phenomena:** Prolix configurations like Few-Shot ($83.26\%$ accuracy) and Hybrid ($81.15\%$ accuracy) introduced context bloat. This caused **Attention Dilution** (validating the premises of the *Lost in the Middle* study by Liu et al.) , which resulted in output drift and format hallucinations where the model tried to match template examples rather than the source text.


* 
**Engineering Implications:** In the domain of structural document intelligence, precision-engineered, concise prompts yield better data extraction reliability while reducing input token consumption, costs, and processing latency.
