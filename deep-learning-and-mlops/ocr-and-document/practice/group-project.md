# Project: Extract Information From Invoices and Contracts

## Group Responsible

Group B — OCR & Document AI.

## Minimum Deliverable

A pipeline that takes an invoice or contract image/PDF and extracts a small set of key-value fields (for example: vendor name, invoice date, total amount) using an existing open-source OCR tool. You are **not** required to train a model from scratch — the point of this project is the extraction pipeline, not custom OCR. Deliverable: a working demo on 5–10 sample documents, plus a short write-up of where the pipeline breaks.

Use publicly available sample invoices/contracts (search “sample invoice dataset” or generate a few realistic mock documents yourselves) — do not use real company documents containing anyone’s actual data.

## Extensions (Not Required)

If the group finishes early, in order of value:

1.  Try Amazon Textract on the same documents and compare its extraction accuracy and failure modes against your open-source pipeline.

2.  Handle a multi-page document rather than single-page.

3.  Report Character Error Rate (CER) on a handful of manually-transcribed ground-truth fields.

## Environment

Run locally or on Colab — OCR inference on a handful of documents does not require serious GPU time. If you use a deep document-understanding model directly (rather than a managed API), Colab’s free GPU tier is enough for this scale.

## Review Schedule

- **Mid-week checkpoint** — show OCR reading raw text off at least one real document, even before field extraction works.

- **Final demo** — run the pipeline live on a document the reviewer has not seen, and be ready to show one it gets wrong.

## Questions to Expect From the Other Groups

- What’s the difference between what your OCR step does and what your extraction step does — where does one end and the other begin?

- If your pipeline mislabels a field, is that a detection failure, a recognition failure, or an understanding failure? How do you tell?

- Why did you choose this tool over a managed service (or vice versa)?

## Hints

- Start with the simplest tool that produces raw text reliably before attempting structured extraction — get detection and recognition working before you try to parse key-value pairs.

- Vietnamese documents and diacritics are a real source of recognition errors; check your tool’s language support before committing to it.

- If field extraction from raw OCR text feels brittle, that is expected — it’s exactly the gap [OCR & Document AI Overview](../overview.md) describes between OCR and document understanding.

## References

- [OCR & Document AI Overview](../overview.md)

- [Deep Learning Core Concepts](../../deep-learning/core-concepts.md)

- [MLflow Basics](../../mlops/mlflow-basics.md)
