# Reading Charts in a RAG Pipeline: OCR vs. a Vision Model

Research papers and reports often present key results as charts (trend lines, slopes, p-values) without publishing the underlying data. A text-only RAG pipeline can't use that information, and plain OCR only reads the printed labels, not the trend.

This notebook tests whether a newer vision model, **Gemini 4 Carbon**, can recover reliable numbers from chart images for use in a RAG pipeline, and how to avoid trusting those numbers blindly.

## Approach

1. **Prefer the original data.** If the table behind a chart exists, retrieve that instead of the image.
2. **If only the image exists:**
   - **Gemini 4 Carbon** extracts the chart into structured JSON at ingestion: axes, data points, and the printed p-value.
   - **Math is done in code, not by the model.** The slope is computed from the extracted points with a regression, rather than estimated by the model.
   - **Every extraction is validated** before it enters the knowledge base. Anything uncertain is flagged for human review.
3. **Answers cite the source chart,** and only validated data is used to answer.

## What the notebook does

| Step | Purpose |
|---|---|
| 1. Generate test charts | Charts built from known data, so accuracy can be measured exactly |
| 2. OCR baseline | Shows OCR reads labels and printed values, but not trends |
| 3. Vision-model extraction | Gemini 4 Carbon extracts each chart to JSON (cached to avoid repeat API costs) |
| 4. Math in code | Compares the model's "eyeballed" slope with a slope computed from extracted points |
| 5. Validation | Checks p-value text, significance decision, slope accuracy, point count, confidence |
| 6. Mini RAG | Answers questions from validated records, with a citation to the chart image |
| 7. Results | Accuracy per check |

The test set includes a **borderline chart (p = 0.049)**, where a small misread flips the conclusion from significant to not significant.

## Running it

```bash
pip install -r requirements.txt
# OCR also needs the Tesseract binary: brew install tesseract  or  conda install -c conda-forge tesseract

cp .env.example .env     # then add your Gemini API key (from Google AI Studio)
jupyter notebook chart_extraction_rag.ipynb
```

Gemini 4 Carbon requires a paid Gemini API tier. Without a key, chart generation and the OCR baseline still run; the model step is skipped.

## Results

Results from 4 test charts (5 data series: strong decline, no effect, borderline p = 0.049, and a two-arm treatment vs placebo chart):

| Method | p-values read correctly | Slopes recovered | Avg slope error |
|---|---|---|---|
| OCR (Tesseract) | 2 / 4 (missed the borderline p = 0.049) | 0 / 4 | n/a |
| Gemini 4 Carbon, slope estimated by the model | 4 / 4 | 4 / 4 | 0.3% |
| Gemini 4 Carbon, slope computed in code from extracted points | 4 / 4 | 4 / 4 | 0.1% |

Gemini 4 Carbon found every data point, made every significance call correctly, and all 4 charts passed validation. The mini RAG step answered all 3 test questions with numbers and a citation to the source chart.

These charts are synthetic and clean, so treat this as a proof of concept, not a benchmark on real published figures.

_See the notebook for the full tables._

## Key takeaways

- OCR alone isn't enough for charts.
- A vision model can recover data points, turning a picture into something a RAG system can reason over.
- Don't let the model do arithmetic; compute slopes in code.
- Treat the printed p-value as the source of truth for significance. A p-value recomputed from approximate points can drift, especially near 0.05.
- Validate before indexing, and keep a human in the loop for high-stakes numbers.

## Limitations and next steps

- The test charts are synthetic and clean. Real figures add error bars, overlapping markers, log scales, multi-panel layouts and low-resolution scans.
- In production there's no ground truth, so validation would rely on consistency checks, a second extraction pass, and human spot checks.
- Next: real published figures (e.g., Kaplan-Meier curves, forest plots), a review UI for flagged items, and integration into a full hybrid-search RAG pipeline.

## Security note

Charts from internal documents may contain confidential or patient-level information. For sensitive data, use an approved private model endpoint rather than a public API. Keep API keys in environment variables or a secrets manager, never in the notebook.
