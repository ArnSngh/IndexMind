# PageIndex Vectorless RAG

Question-answering over PDFs using PageIndex's document structure and locally run Ollama models. The notebook avoids embedding vectors and a vector database, while retaining section-aware retrieval and page citations.

> **Privacy note:** PageIndex indexing in this notebook is hosted. Uploading a PDF sends it to the PageIndex service for processing. Ollama runs locally for query-time model inference. Do not upload sensitive documents unless your organization's policies permit it.

## How it works

1. The notebook sends a PDF to PageIndex, which creates a hierarchical table of contents and provides extracted page text.
2. A local Ollama model selects likely sections from the table of contents.
3. The notebook ranks pages from those sections against the question and supplies the selected text to the model.
4. The model answers from that text. Python checks its quoted evidence against the source text and formats section/page citations.

This is **vectorless retrieval**, not retrieval-free RAG: it uses the document's hierarchy and language-model-guided navigation instead of nearest-neighbor search over embedding vectors.

## Vector RAG vs. this approach

| Aspect | Traditional vector RAG | This project |
|---|---|---|
| Indexing | Split text into chunks and create embeddings | PageIndex builds a hierarchical document tree and page text |
| Retrieval | Embed the question and search a vector index | Ask an LLM to choose sections, then rank their pages |
| Infrastructure | Usually an embedding model and vector store | PageIndex API for indexing; no vector store or embedding index |
| Context | Retrieved chunks can lose surrounding structure | Section titles and page boundaries are retained |
| Citations | Depend on chunk metadata and implementation | Section and PDF page citations are produced from retrieved evidence |
| Main trade-off | Fast semantic search at scale, with chunking/index tuning | Structure-aware navigation, but relies on tree quality, OCR, and LLM selection |

Neither approach is universally more accurate. Vector search can be preferable for large collections of unstructured text or when low-latency semantic search is important. Hierarchical retrieval is useful when document structure—such as policy clauses, report sections, or chapters—helps locate evidence.

## Use it

### Requirements

- Python 3.11 or later
- [Ollama](https://ollama.com/) installed and running
- A [PageIndex API key](https://dash.pageindex.ai/api-keys) for document upload, indexing, and OCR

### Setup

1. Install the notebook dependencies:

   ```powershell
   python -m pip install pageindex python-dotenv
   ```

2. Install a model in Ollama. The notebook currently defaults to Qwen:

   ```powershell
   ollama pull qwen2.5:7b
   ```

   The other models benchmarked here are `mistral:7b` and `llama3.1:8b`.

3. Create a `.env` file in the project root and add your key:

   ```text
   PAGEINDEX_API_KEY=your_pageindex_api_key
   ```

   Keep `.env` private. It is excluded from Git; never commit API keys.

4. Open [PageIndex_Vectorless_RAG.ipynb](./PageIndex_Vectorless_RAG.ipynb) in Jupyter or VS Code and run the cells in order. The sample PDF is `Test/Placement.pdf`. Change `PDF_PATH` in the notebook to use another PDF.

5. Ask a question with the notebook's `vectorless_rag` function. Choose any installed Ollama model per call:

   ```python
   answer = vectorless_rag(
       "What does the placement policy say about multiple offers?",
       pageindex_tree,
       model="qwen2.5:7b",
   )
   ```

The notebook needs a valid PageIndex key to upload and index a PDF. Ollama model calls run against the local Ollama service; set `OLLAMA_BASE_URL` in `.env` only if Ollama is not using its default `http://localhost:11434` address.

## Ollama model benchmark

The benchmark used the single PDF currently present in `Test/`: a scanned, 9-page placement policy. It was OCR'd locally for this comparison. Each model received the same question and the same page excerpt for each of ten policy questions, so this measures answer generation and citation behavior with relevant evidence already supplied; it does **not** measure PageIndex indexing or retrieval recall.

Questions covered eligibility after an internship, revoked offers, PPO/FTE timelines, FTE withdrawal conditions, non-joining penalties, registration consequences, multiple offers, off-campus-offer disclosure, skipping placement processes, and an off-campus internship undertaking.

| Model | Fully correct answers / 10 | Mean generation speed | Observation |
|---|---:|---:|---|
| `mistral:7b` | 6/10 | ~10.1 tokens/s | Fast, but missed the seven-day disclosure deadline and confused adjacent rules in a multi-part FTE question. |
| `llama3.1:8b` | 7/10 | ~7.8 tokens/s | Strong on multi-part answers, but had a page-citation error and missed one required citation. |
| `qwen2.5:7b` | **9/10** | **~11.8 tokens/s** | Best result in this small test: strongest combination of answer coverage, correct page references, and speed. It still confused an FTE rule with a neighboring internship exception once. |

**Recommendation: keep `qwen2.5:7b` as the default.** It was the most accurate on this test and generated tokens fastest on this machine. This is a practical starting choice, not a universal model ranking: the evaluation is small, uses one document, and does not establish accuracy on other subjects or PDFs.

The score is a strict, manually reviewed pass/fail per question: the answer had to include all requested policy facts, avoid applying an unrelated rule, and cite the correct PDF page. Generation speed is the mean Ollama output-token rate reported for the ten responses; it excludes model-loading time. Benchmark results can vary with Ollama version, hardware, quantization, prompt, and model tag.

### Why model choice matters in AI engineering

The model is part of the system's behavior, not a swappable cosmetic detail. It affects whether the system follows instructions, covers every part of a question, grounds claims in evidence, abstains when evidence is missing, and returns usable citations. It also affects response time, memory use, deployment cost, licensing, and privacy constraints. A good engineering choice comes from testing the intended workload with a fixed question set and source evidence—not from model size or popularity alone.

For a production evaluation, expand the gold set across multiple documents and question types. Track fact coverage, citation correctness, unsupported claims, abstention quality, latency, throughput, and memory use; rerun it whenever the model, quantization, prompt, or retrieval pipeline changes.

## Accuracy work in the notebook

- The RAG wrapper now accepts a `model` argument and passes it to both section selection and answer generation.
- Page retrieval considers up to four matching pages instead of only two, which helps with questions spanning multiple clauses.
- Lexical ranking and evidence checks normalize OCR text so merged words and line-break artifacts are less likely to hide a relevant page or reject a valid quote.
- The answer prompt asks the model to address each part of a question and distinguish separate policy scenarios. Evidence is still checked against retrieved source text; a verified quote is not, by itself, proof that every conclusion is correct.

The current test set also found a remaining weakness: models can still conflate adjacent clauses, especially confirmed FTE offers versus six-month internships. Review answers involving exceptions or multiple conditions against the cited policy before acting on them.

## Future directions

- Add a repeatable, versioned evaluation set with human-reviewed expected facts and citation pages.
- Support multiple PDFs and cross-document retrieval with clear document-level source citations.
- Add a local-only indexing/OCR option for documents that cannot be sent to a hosted indexing service.
- Improve query decomposition, clause-aware retrieval, and handling of tables, scanned pages, and OCR quality.
- Compare hierarchical retrieval with optional lexical or vector-based hybrid retrieval on the same evaluation set.
- Add an interactive interface with feedback, abstention/confidence reporting, and traceable retrieval diagnostics.

## Project files

- [PageIndex_Vectorless_RAG.ipynb](./PageIndex_Vectorless_RAG.ipynb) — setup, indexing, retrieval, answering, and examples.
- `Test/Placement.pdf` — sample scanned placement-policy PDF.
- `.gitignore` — excludes local credentials and notebook/Python cache files.
