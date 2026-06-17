# Scientific Literature Retrieval & Summarization Pipeline

This repository contains an automated, multi-agent pipeline designed to intelligently search, retrieve, extract, summarize, and classify scientific articles from the PubMed Central (PMC) Open Access dataset.

## 🏗️ Architecture

The pipeline is divided into four distinct sequential agents/tiers to optimize processing time and memory usage:

1. **Data Ingestion Tier:** Uses the NCBI E-utilities API to find relevant PMCIDs, then anonymously downloads the corresponding full-text `.txt` files directly from the `pmc-oa-opendata` AWS S3 bucket. A text cleaner scrubs academic boilerplate (e.g., copyright notices, funding disclosures).
2. **Extractor Agent:** Embeds the cleaned document paragraphs and the user's query into vector space. It calculates cosine similarity, extracting and retaining only the chunks that meet a strict `>= 0.40` relevance threshold.
3. **Summarizer Agent:** Takes the high-density extracted chunks and generates concise, abstractive scientific summaries (25–75 words). It also includes a term-frequency statistical counter to extract the top 5 keywords.
4. **Verifier Agent:** A zero-shot classification model that categorizes the summarized findings into specific target themes (e.g., "Deep Learning", "Clinical Trial", "Traditional Methods") and assigns a confidence score.

![Pipeline Architecture Diagram](architecture_diagram.jpeg)

## ⚖️ Design & Tradeoffs

The initial client specifications outlined a specific data flow (download 50–100 articles first) and suggested highly lightweight models (`t5-small`, `distilbert-base-uncased`). To balance the constraints of CPU-friendly execution, speed, and scientific accuracy, the following architectural and model decisions were made:

* **Retrieval Strategy & Data Sampling:**
    * *The Baseline:* The initial requirement was to download 50 to 100 articles locally *before* performing any search or classification. 
    * *The Implementation:*  We inverted this to an **API-first approach**. The system queries the NCBI API with the target keyword first to identify up to 30 highly relevant PMCIDs, then downloads *only* those files. 
    * *The Tradeoff:* Downloading 100 dense academic papers blindly wastes massive amounts of bandwidth and CPU cycles on irrelevant text. Our approach saves immense compute overhead, but the tradeoff is that we rely heavily on the NCBI's external search algorithm rather than our own local semantic search over a broader downloaded corpus.
* **Model Selection (Summarization):**
    * *The Baseline:* The client requested `t5-small`.
    * *The Implementation:* Upgraded to `facebook/bart-large-cnn`.
    * *The Tradeoff:* While `t5-small` is faster and smaller, it struggles to generate coherent, abstractive summaries of complex medical text. We traded a higher memory footprint and slightly slower inference for vastly superior, human-readable summaries that still run efficiently on a CPU.
* **Model Selection (Classification):**
    * *The Baseline:* The client requested `distilbert-base-uncased`.
    * *The Implementation:* Upgraded to `typeform/distilbert-base-uncased-mnli`.
    * *The Tradeoff:* The base DistilBERT model requires explicit fine-tuning to categorize text. By swapping to the MNLI variant, we gain powerful **Zero-Shot Classification** capabilities. The tradeoff favors CPU-friendliness and rapid deployment over the nuanced reasoning of a massive LLM, allowing us to accurately map summaries to dynamic themes without any retraining.
* **Embedding Choice:** * *The Implementation:* We retained the client-requested `sentence-transformers/all-MiniLM-L6-v2`.
    * *The Tradeoff:* This model is exceptionally fast, lightweight, and perfect for CPU-based cosine similarity filtering. However, it is trained on general text. For highly technical biomedical literature, a domain-specific model like `PubMedBERT` would yield more accurate semantic matches, but at the cost of being significantly heavier and slower.
* **Agent Architecture, Chunking, & Context:** * *The Implementation:* The pipeline runs sequentially. Documents are chunked by paragraphs `\n\n`, and only passages meeting a strict `0.40` cosine similarity threshold are passed to the Summarizer. Furthermore, inputs are strictly truncated to 1024 characters.
    * *The Tradeoff:* This guarantees the downstream sequence-to-sequence model won't crash from exceeding maximum token limits and ensures rapid processing. The direct tradeoff is that we sacrifice deep, document-wide holistic analysis; any vital context located outside the highest-scoring 1024-character chunk is excluded from the summary.

## ⚙️ Requirements & Installation

To run this notebook, you will need the following libraries:

```bash
pip install boto3 sentence-transformers transformers requests pandas numpy scikit-learn
```
## 🚀 Execution Statements
To execute the pipeline, simply pass your target queries into the orchestrator function. The final cell of the Jupyter notebook utilizes the following statements to process the case studies:

```Python
# The specific test queries mandated by the case study
test_queries = [
    "Adverse events with mRNA vaccines in pediatrics",
    "Transformer-based models for protein folding",
    "Clinical trial outcomes for monoclonal antibodies in oncology"
]

for q in test_queries:
    run_case_study_pipeline(q)
```
