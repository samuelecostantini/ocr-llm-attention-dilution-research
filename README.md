# OCR Output Optimization via Large Language Models
### An Experimental Study on Prompt Engineering for Hybrid OCR+LLM Pipelines

> **B.Sc. Thesis in Computer Science** — University of Perugia, Department of Mathematics and Computer Science
> Academic Year 2024/2025
> **Author:** Samuele Costantini · **Supervisor:** Prof. Leonardo Mostarda

![Status](https://img.shields.io/badge/status-completed-success)
![Language](https://img.shields.io/badge/thesis-italian-blue)
![Stack](https://img.shields.io/badge/stack-PHP%208.2%20%7C%20Laravel%2011%20%7C%20GPT--4o-orange)
![License](https://img.shields.io/badge/license-MIT-lightgrey)

---

## 📖 TL;DR

I built a benchmarking framework to evaluate how different **prompt engineering strategies** affect data extraction accuracy in **hybrid OCR + LLM pipelines** (AWS Textract → GPT-4o). I tested 5 prompting techniques on a heterogeneous corpus of 20 documents (invoices, receipts, IDs, contracts, newspapers, corporate intelligence) across multiple quality and layout dimensions.

**Main finding — counterintuitive but well-supported:** increasing prompt complexity does **not** correlate with higher accuracy. The minimal **Zero-shot** prompt outperformed Few-shot, Chain-of-Thought, Hybrid (Forensic + CoT + Structural Awareness), and Emotion Prompting on structured extraction tasks, empirically confirming the *"Lost in the Middle"* phenomenon (Liu et al., 2024) on a specific applied domain: OCR post-correction for fiscal documents.

---

## 📊 Key Results

### Overall accuracy by prompting strategy

| Strategy | Avg. Accuracy | Prompt Tokens | Verdict |
|---|---|---|---|
| **Zero-shot** | **87.01%** | 49 | 🥇 Best overall, lowest cost |
| Chain-of-Thought | 85.87% | 221 | 🥈 Best on complex/long docs |
| Few-shot | 83.26% | 1,280 | 🥉 Best on output formatting |
| Zero-shot Emotional | 82.52% | 75 | ⚠️ Induces over-answering |
| Hybrid (Forensic+CoT+SA) | 81.15% | 484 | ❌ Hurts numeric fields |

### Accuracy by field type

| Field Type | Best Strategy | Accuracy | Why |
|---|---|---|---|
| **Numeric values** | Zero-shot | **93.02%** | Minimal output drift, no hallucination of decimals |
| **Strings** (semantic) | Zero-shot | 85.78% | Synthetic output reduces statistical noise |
| **Dates** | Chain-of-Thought | 78.87% | Step-by-step reasoning helps disambiguate formats |

### When complex prompts actually win

CoT outperforms Zero-shot on:
- Long documents (>400 words)
- Scanned PDFs requiring interpretation
- Newspapers and contracts (semantically dense layouts)
- English-language documents

This suggests prompt strategy should be **routed by document characteristics**, not chosen globally.

---

## 🔬 Why This Matters

Most "prompt engineering" blog posts assume that richer prompts → better results. This study provides empirical counter-evidence in a production-relevant domain:

1. **Attention dilution is real.** Few-shot prompts at 1,280 tokens consistently underperform a 49-token Zero-shot prompt. The model trades information density for confusion.

2. **Hallucination correlates with prompt complexity.** The Hybrid prompt — which combines Forensic directives, CoT, and Structural Awareness — produced the worst numeric accuracy (75%), with systematic decimal misplacement on invoice totals (e.g., extracting `31.82225` instead of `31822.25`).

3. **Output drift penalizes verbose strategies.** Few-shot tends to add semantic descriptors (`"Marcus V. Thorne, CEO of OmniCorp Logistics"` vs. ground truth `"Marcus V. Thorne (CEO)"`), inflating string-similarity penalties even when the model "understood" the field correctly. This is documented in §9.2.2 of the thesis.

4. **Emotional stimuli induce false positives.** Adding emotional framing to Zero-shot pushes the model to "find something" even when the ground truth is `Unknown`, generating fabricated extractions on absent fields.

---

## 🏗️ Architecture

```
┌──────────┐    ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
│ Document │───▶│ AWS Textract│───▶│ Parsed JSON  │───▶│   GPT-4o    │───▶ Structured JSON
│ (PDF/IMG)│    │   (OCR)     │    │ cellMap +    │    │ via Prism   │     (validated against
└──────────┘    └─────────────┘    │ formFields + │    │   PHP       │      schema)
                                   │ rawLines     │    └─────────────┘
                                   └──────────────┘
                                          ▲
                                          │
                               ┌──────────┴──────────┐
                               │ Detail Schema (JSON)│
                               │ + Prompt template   │
                               └─────────────────────┘
```

The pipeline mirrors patterns adopted by `google/langextract` and similar Document Intelligence libraries: a deterministic OCR layer followed by an LLM acting as a *semantic reasoning engine* that corrects OCR errors and normalizes output to a target JSON schema.

---

## 🛠️ Tech Stack

**Backend & Orchestration**
- PHP 8.2 · Laravel 11
- Laravel Job Queues (async benchmarking via `BulkRunJob`)
- MySQL (normalized relational schema)

**AI Pipeline**
- AWS Textract (OCR layer, with `TABLES` + `FORMS` feature types)
- OpenAI GPT-4o (semantic layer)
- PrismPHP (structured output API wrapper)

**Frontend**
- Filament PHP 3.2 (admin dashboard)
- Livewire 3.4 (server-side reactivity)
- TALL stack (Tailwind, Alpine, Laravel, Livewire)

**Design Patterns**
- Command/Action pattern for business logic isolation
- Service container + Dependency Injection for provider swapping
- Strict typing (PHP 8.2 union types, return types) for runtime safety

---

## 🧪 Methodology

### Dataset (n=20, intentionally heterogeneous for stress testing)

| Dimension | Distribution |
|---|---|
| Document Type | 5 Invoice, 3 Receipt, 2 Pay slip, 1 Contract, 2 ID, 4 Corporate Intel, 3 Newspaper |
| File Format | 9 Native PDF, 4 Scanned PDF, 7 Picture (PNG/JPG) |
| Quality | 15 HD, 3 Low-def, 6 Distorted, 7 Noisy |
| Layout | 12 Structured, 8 Prose |
| Language | 11 Italian, 8 English |
| Word Count | 7 <100 words, 7 100–400, 6 >400 |

### Evaluation Metrics

Different scoring algorithms per field type, to avoid false positives/negatives common in naïve string-match benchmarks:

- **Strings** → Oliver algorithm (`similar_text`, order-agnostic, O(N³)). Tests *semantic understanding*, not lexical alignment.
- **Dates** → Strict semantic match via Carbon PHP. String similarity would generate false positives (e.g., `23/02/2025` vs `3/02/2025` → 0.9 by similarity, but they're semantically different).
- **Numbers** → Currency sanitization + locale normalization + absolute decimal difference (`score = 1.0 - |Δ|`).

Each extraction receives a score in `[0.0, 1.0]`. Benchmark results are persisted with both `expected_value` and `extracted_value` to enable retrospective analysis if Ground Truth changes.

### Prompting Strategies Tested

1. **Zero-shot** (Radford et al., 2019) — minimal task description
2. **Few-shot** (Brown et al., 2020) — explicit input-output examples with correction rules
3. **Chain-of-Thought** (Wei et al., 2022) — step-by-step reasoning
4. **Hybrid: Forensic + CoT + Structural Awareness** — composite advanced prompt
5. **Emotion Prompting** (Li et al., 2023) — psychological stimuli appended to base prompt

---

## 📁 Repository Structure

```
.
├── Thesis-italian-version.pdf      # Full thesis (54 pages, Italian)
├── README.md                       # You are here
├── app/
│   ├── Actions/                    # RunBenchmarkAction, RunExtractionAction
│   ├── Services/                   # AwsTextractService, OpenAIService, EvaluationService
│   ├── Models/                     # Document, Prompt, Run, GroundTruth, BenchmarkResult, ...
│   ├── Jobs/                       # BulkRunJob (async dispatch)
│   └── Filament/                   # Admin dashboard resources
└── ...
```

---

## 📄 Full Thesis

The complete 54-page thesis (in Italian) is available here:

📥 **[Download Thesis-italian-version.pdf](./Thesis-italian-version.pdf)**

It covers in depth:
- Architectural justifications and design pattern adoption (§3)
- Detailed pipeline walkthrough with PHP code (§6)
- Theoretical background on each prompting strategy (§7)
- Full evaluation methodology and metric design rationale (§8)
- Per-strategy and per-cluster result analysis (§9)
- Critical reflection on metric sensitivity and limitations (§9.2.2)

> **English translation in progress.** Open an issue if you'd like to be notified when it's available.

---

## ⚠️ Honest Limitations

This is a B.Sc. thesis, not a peer-reviewed empirical study. In the interest of intellectual honesty:

- **N=20 is small** for statistical generalization. Findings are presented as **observed trends**, not universal claims. Confidence intervals are not computed.
- **Single model tested (GPT-4o).** Attention dilution patterns may differ on Claude, Gemini, or open-weight VLMs. Replication on other models is the natural next step.
- **No cross-modal experiment.** The pipeline tested is `OCR text → LLM`. The alternative `image → VLM directly` (which avoids OCR errors but introduces visual hallucination on degraded inputs) is not compared.
- **Metric design choice trades fairness for clarity.** The Oliver algorithm penalizes verbose outputs, which may unfairly disadvantage strategies that legitimately enrich responses. This is discussed explicitly in §9.2.2.

These are signposted as opportunities for future work, not weaknesses being concealed.

---

## 🚀 Reproducing the Results

> 🚧 **Setup instructions coming soon.** For now, the codebase is structured for reproducibility:
> - All prompts are stored in the database (entity: `Prompt`), versioned with `slug`
> - All Ground Truth values are manually validated and stored in `GroundTruth`
> - Each extraction creates a `Run` record, ensuring full traceability
> - `BulkRunJob` allows re-running any subset of (document × prompt) pairs

If you'd like to run the framework locally, open an issue and I'll prioritize writing the setup guide.

---

## 📚 Citation

If you reference this work, please cite as:

```bibtex
@thesis{costantini2025ocr,
  author       = {Costantini, Samuele},
  title        = {Sviluppo di un framework per l'ottimizzazione dell'output OCR
                  tramite Large Language Models: un'analisi sperimentale
                  sull'efficacia del Prompt Engineering},
  type         = {Bachelor's Thesis},
  institution  = {University of Perugia, Department of Mathematics and Computer Science},
  year         = {2025},
  supervisor   = {Mostarda, Leonardo},
  url          = {https://github.com/samuelecostantini/ocr-llm-attention-dilution-research}
}
```

---

## 📬 Contact

- 🐙 GitHub: [@samuelecostantini](https://github.com/samuelecostantini)
- 💼 LinkedIn: [add your link]
- ✉️ Open an issue on this repo for technical questions

---

## 🤝 Acknowledgments

Special thanks to **Prof. Leonardo Mostarda** for supervision, and to the company where this research originated as a real production challenge during my internship.

---

*If you found this work useful, a ⭐ on the repo helps with visibility. Thanks for reading.*
