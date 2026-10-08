# VYOM+ Voucher Intelligence: Hybrid Open-Source LLM Voucher Classifier

**Hacktoberfest Hack Day: Qualifier Round**
**Team:** AIS
**Problem Statement:** #4. VYOM+ Intelligent Voucher Classification Using Open-Source LLMs

## Team Members

| Name | Role |
|---|---|
| Alhamda Iqbal Sadiq | Team member |
| Sumit Pandey | Team member |

---
## 1. Project Name

VYOM+ Voucher Intelligence: Hybrid Open-Source LLM Voucher Classifier

## 2. Problem Statement

Accounting software needs every transaction filed under the correct **voucher type**. Today this is done by hand or with brittle keyword rules. The voucher type is determined by the *accounting meaning* of a transaction, not by any one field. Several pairs look almost identical on paper:

- Purchase vs Sales (same fields, opposite party roles)
- Purchase Return / Debit Note vs Sales Return / Credit Note
- Contra vs ordinary Payment / Receipt
- Journal vs conventional Purchase / Sales
- Stock movements (Material In/Out, Stock Journal, Delivery/Receipt Note) vs Purchase / Sales
- Import / Export vs domestic transactions
- Salary / Payroll vs other payments

Given an Excel dataset of structured transactions **with the voucher-type column removed**, our system predicts exactly one of the 27 target categories for each row (Purchase, Sales, Purchase Return / Debit Note, Sales Return / Credit Note, Payment, Receipt, Contra, Journal, Salary / Payroll, Attendance, Purchase Order, Sales Order, Receipt Note, Delivery Note, Rejection In, Rejection Out, Stock Journal, Physical Stock, Material In, Material Out, Job Work In Order, Job Work Out Order, Import, Export, Expense, Advance / Prepayment, Other / Miscellaneous).

## 3. Project Overview

VYOM+ Voucher Intelligence is a hybrid pipeline that reads an Excel file of already-structured transactions (the voucher-type column is intentionally missing) and predicts one appropriate voucher category per row. An open-source LLM is the primary classifier. Lightweight accounting-aware preprocessing and candidate narrowing support it so that it reasons over the complete transaction context and chooses among a few plausible options instead of all 27. The output is structured JSON / Excel that can be evaluated programmatically, with a confidence score and a short explanation for each prediction.

This is not an OCR or invoice-extraction project. It is the classification layer that sits between invoice extraction and automated voucher creation.

## 4. Proposed Solution

A **hybrid pipeline** in which an open-source LLM is the primary classifier. Lightweight accounting-aware preprocessing and candidate narrowing help the LLM and keep inference fast. The LLM reasons over the whole transaction context and returns a structured, machine-evaluable answer.

1. **Ingest and normalize** the Excel file and standardize fields.
2. **Feature enrichment** adds accounting signals the LLM can reason over (for example, which party is the company, whether amounts are negative, return or debit/credit indicators, import/export flags, payroll or stock fields).
3. **Candidate narrowing** uses embeddings plus signal rules to shortlist the most plausible categories (top-k), so the LLM chooses among a few options instead of 27.
4. **LLM classification** uses a fine-tuned open model with schema-constrained JSON output.
5. **Ambiguity handling** runs a second-pass check for low-confidence rows.
6. **Output and evaluation** produces a JSON/Excel result and a reproducible metrics report.

## 5. Objectives
One of 27 categories predicted per transaction.
Open-source LLM reasoning over all fields.
Separating similar categories.
Schema-valid output with confidence and explanation.
Robust handling of missing or ambiguous records.
Reproducible evaluation.

## 6. Target Users

- Accountants, CA firms and bookkeeping teams processing high transaction volumes
- ERP / accounting platforms such as VYOM+ that need automated voucher creation
- Developers building the bridge between invoice extraction (OCR) and automated voucher posting

## 7. Open-Source AI Technology Selected
Primary classifier: open-weight 7B-class instruct LLM (Qwen; Gemma / Llama / Mistral as alternatives)
Adaptation: LoRA / QLoRA fine-tuning
Inference: quantized local inference (llama.cpp / vLLM)
Structured output: grammar / JSON-schema constrained decoding
Candidate retrieval: open embedding model (BGE / E5)

No proprietary API is used as the classification engine.

## 8. Why This Technology Was Selected

| Component | Choice | Why |
|---|---|---|
| Primary classifier | Open-weight 7B-class instruct LLM (**Qwen** family; **Gemma** / **Llama** / **Mistral** as alternatives) | Strong reasoning and structured-output ability at a size that runs locally |
| Adaptation | **LoRA / QLoRA** fine-tuning | Teaches accounting-specific distinctions cheaply on limited hardware |
| Inference | Quantized local inference (**llama.cpp / vLLM**) | Fast, private, no proprietary API |
| Structured output | Grammar / JSON-schema constrained decoding | Guarantees valid, programmatically evaluable output |
| Candidate retrieval | Open-source embedding model (e.g., BGE / E5) | Narrows 27 classes to a shortlist and retrieves similar labeled examples |

No proprietary API is used as the classification engine.

## 9. AI's Role in the System

The LLM is the **core decision-maker**, not an add-on. Keyword rules cannot tell a Purchase from a Sales Return when the fields look alike. The model reads all fields together (parties, amounts, tax, item text, references, flags) and reasons about the accounting meaning. The surrounding rules and embeddings only give it better evidence and a smaller choice set.

## 10. Architecture

```
            +------------------+
            |  Excel (.xlsx)   |   (voucher_type column absent)
            +--------+---------+
                     |
          +----------v-----------+
          | 1. Ingestion &       |  schema mapping, type cleaning,
          |    Normalization     |  missing-value handling
          +----------+-----------+
                     |
          +----------v-----------+
          | 2. Feature           |  party-role, return/debit-credit,
          |    Enrichment        |  import/export, payroll, stock signals
          +----------+-----------+
                     |
          +----------v-----------+
          | 3. Candidate         |  embeddings + rules -> top-k classes
          |    Narrowing         |  + similar labeled examples (few-shot)
          +----------+-----------+
                     |
          +----------v-----------+
          | 4. LLM Classifier    |  LoRA fine-tuned open LLM
          |  (constrained JSON)  |  -> voucher_type, confidence, reason
          +----------+-----------+
                     |
             low confidence?
              /           \
            no             yes
            |               |
            |      +--------v---------+
            |      | 5. Second-pass   |  re-prompt with full context,
            |      |    Resolution    |  fallback: Other / Miscellaneous
            |      +--------+---------+
            |               |
          +-v---------------v-----+
          | 6. Output & Evaluation|  JSON / Excel + metrics report
          +-----------------------+
```

## 11. Component-Level Architecture

| Component | Input | Processing | Output | Technology |
| --- | --- | --- | --- | --- |
| **Ingestion & Normalization** | Excel (.xlsx) | Schema mapping, type cleaning, missing-value handling | Clean transaction rows | `pandas`, `openpyxl` |
| **Feature Enrichment** | Normalized rows | Derives accounting signals and evidence | Enriched rows | Python |
| **Candidate Narrowing** | Enriched rows | Embeddings + rules select top-k classes and examples | Class shortlist + examples | BGE/E5, FAISS |
| **LLM Classifier** | Row + evidence + shortlist | Fine-tuned LLM with constrained JSON output | Voucher type + confidence + reason | Qwen 7B, PEFT, bitsandbytes |
| **Second-Pass Resolution** | Low-confidence rows | Re-evaluates using full context | Resolved / flagged prediction | Same LLM |
| **Output & Evaluation** | Predictions + labels | Export and metric calculation | JSON/Excel + report | `pandas`, `scikit-learn` |
| **Interface (Optional)** | Excel file | Upload and prediction download | Prediction file | Streamlit / CLI |

## 12. Data Flow

1. **Input:** an Excel file in which each row is a transaction (seller, buyer, invoice no. and date, items, quantities, taxable value, GST, discounts, freight, payment info, currency, import/export, payroll, debit/credit, return info, order and delivery references, and so on).
2. **Normalization:** column names are mapped to a canonical schema, types are cleaned, and missing fields are marked explicitly instead of dropped.
3. **Enrichment:** derived signals are attached to each row as short, readable evidence lines.
4. **Narrowing:** the top-k plausible voucher types and a few similar labeled examples are selected.
5. **Classification:** the row, evidence and shortlist are sent to the LLM, which returns schema-valid JSON.
6. **Resolution:** rows below a confidence threshold get a second pass. Rows that remain unresolved are labeled `Other / Miscellaneous` and flagged for review.
7. **Output:** one prediction per row.


## 13. Agentic Workflow

The system uses a **controlled, deterministic LLM workflow** rather than a fully autonomous agent:
1. **Evidence Gathering** – Rules and FAISS retrieval provide accounting signals, candidate classes, and similar examples.
2. **LLM Decision** – The LLM predicts the voucher type using structured JSON output.
3. **Confidence Check** – High-confidence predictions are accepted directly.
4. **Self-Correction** – Low-confidence predictions are re-evaluated with full context.
5. **Fallback & Review** – Unresolved cases are marked **Other / Miscellaneous** and flagged for human review.
6. **Fast Path** – Rule-confident transactions can bypass the LLM to reduce inference cost.

## 14. Technology Stack

- **Language:** Python
- **Data Processing:** pandas, openpyxl
- **Models & Training:** Hugging Face Transformers, PEFT (LoRA/QLoRA), bitsandbytes
- **Inference:** llama.cpp / vLLM with constrained structured decoding
- **Retrieval:** sentence-transformers (BGE/E5), FAISS
- **Evaluation:** scikit-learn (Accuracy, Precision, Recall, F1, Confusion Matrix)
- **Interface (Optional):** Streamlit / CLI for Excel upload and prediction download


## 15. Expected Features

- Excel (.xlsx) ingestion with canonical schema mapping
- Classification into **27 voucher categories**
- Accounting-aware feature enrichment
- Embedding-based candidate narrowing and few-shot retrieval
- LoRA/QLoRA fine-tuned open-source LLM with structured JSON output
- Confidence score and explanation for each prediction
- Second-pass resolution for low-confidence cases
- Missing/incomplete field handling with review flags
- JSON and Excel result export
- Reproducible evaluation with accuracy, F1, confusion analysis, speed and memory metrics
- Batched, quantized local inference with a rule-based fast path
- Optional Streamlit / CLI interface


## 16. Implementation Approach

1. **Data Understanding** – Profile the dataset, fields, class balance, and define the canonical schema.
2. **Feature Engineering** – Create accounting-aware signals for difficult category pairs.
3. **Baseline** – Build an embedding + rules classifier for benchmarking and candidate narrowing.
4. **LLM Adaptation** – Apply structured prompting and LoRA/QLoRA fine-tuning.
5. **Robustness** – Handle missing fields, calibrate confidence, and add second-pass resolution.
6. **Evaluation** – Test on held-out data using per-class metrics, speed, and memory measurements.
7. **Packaging** – Provide reproducible scripts, documentation, and an optional upload interface.

 **16.1. Handling Missing and Ambiguous Data**

- Missing fields are passed to the model as explicitly "not provided" rather than silently dropped
- Training includes examples with fields deliberately masked, so the model learns to rely on whatever evidence is available
- Low-confidence rows trigger a second-pass prompt with full context
- Rows that remain unresolved fall back to `Other / Miscellaneous` and are flagged for human review

**16.2. Evaluation Method (Reproducible)**

- Fixed train / validation / test split with a set random seed
- Metrics: accuracy, macro and per-class precision, recall and F1
- Dedicated error analysis on the confusable pairs (Purchase vs Sales, Purchase Return vs Sales Return, Contra vs Payment/Receipt, Journal vs Purchase/Sales, stock movements vs Purchase/Sales)
- Measured inference speed (records per second) and memory use
- A single script reproduces every reported number

## 17. Expected Output

Minimum output, per the challenge format:

```json
{ "invoice_number": "INV-2026-1042", "voucher_type": "Purchase" }
```

Our extended output also includes a confidence score and a short explanation:

```json
{
  "invoice_number": "INV-2026-1042",
  "voucher_type": "Purchase",
  "confidence": 0.93,
  "explanation": "Company is the buyer; taxable value and input GST present; no return or debit-note indicators."
}
```

Results are exported as JSON and as Excel, so they can be scored programmatically.

**17.1. Expected Impact**

A drop-in classification layer for VYOM+ that turns extracted transaction data into correctly typed vouchers automatically. It cuts manual accounting effort and errors, and completes the pipeline from invoice extraction to automated voucher creation using only open-source AI.


## 18. Scalability

- Candidate narrowing keeps LLM prompts short, which cuts per-record cost
- Batched, quantized inference lets large Excel files run on modest hardware
- Rule-confident rows can skip the LLM, saving compute
- The model and its prompts are decoupled, so the model can be swapped for a newer open model without changing the pipeline

## 19. Dependencies

Python 3.10+, PyTorch, Hugging Face Transformers and PEFT, sentence-transformers, FAISS, pandas, openpyxl, scikit-learn, llama.cpp or vLLM. All components are open-source and permissively licensed.

## 20. Expected Challenges and Mitigations

| Challenge | Mitigation |
|---|---|
| Semantically near-identical categories | Enrichment signals, few-shot retrieval, targeted fine-tuning and per-pair error analysis |
| 27 classes with likely imbalance | Class-aware sampling and augmentation, per-class metrics |
| Missing or inconsistent fields | Masking-based training and explicit "not provided" handling |
| Hallucinated or invalid labels | Constrained decoding to the fixed label set and JSON schema |
| Limited compute | QLoRA, quantized inference, candidate narrowing |
| Inference speed | Batching and a rule-confident fast path |


---

*Qualifier submission: this repository intentionally contains only this README. Implementation will be built during the final hackathon.*
