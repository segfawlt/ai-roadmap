# OCR & Document AI Overview

Everyone reads this page — only Group B goes deep on it.

## Reading Text Is Two Problems, Not One

``` mermaid
        graph LR
    A[Document image] --> B["Text detection:<br/>where is the text?"]
    B --> C["Text recognition:<br/>what does it say?"]
    C --> D[Extracted text]
```

**Text detection** finds regions in the image that contain text — a bounding box per line or word, conceptually similar to the object detection you may hear about from Group A. **Text recognition** takes each of those regions and reads the actual characters. Classic OCR architectures (CRNN) train these jointly using a loss called CTC, which handles the fact that the model does not know in advance how many characters a region contains.

Modern engines (Tesseract, PaddleOCR, EasyOCR) package both steps behind one API call, so in practice you rarely implement detection and recognition separately — but knowing they are two different problems explains why OCR fails in specific, recognisable ways: detection failing looks like missed text entirely; recognition failing looks like garbled characters in a region that was found correctly.

## OCR Is Not Document Understanding

Reading the characters on an invoice is not the same as knowing which characters are the total, which are the vendor name, and which are a line-item description. That second problem — **document understanding** — needs layout information (where things sit relative to each other, not just what they say) and usually a model trained specifically for it, such as LayoutLM, or a managed service like **Amazon Textract**, which is built specifically for structured extraction from invoices, contracts, and forms.

The practical decision most teams face: use a managed extraction API first, and only build a custom pipeline if the API’s accuracy on your specific document type is not good enough. This is the same “build vs. buy” question that comes up across the deep learning areas in this track.

## Explore Further

- **OCR mechanics** — `CTC`` ``loss`, `CRNN`, `text`` ``deskewing`, `binarisation`

- **Evaluation** — `Character`` ``Error`` ``Rate`` ``(CER)`, `Word`` ``Error`` ``Rate`` ``(WER)`

- **Tools** — `Tesseract`, `PaddleOCR`, `EasyOCR`

- **Document understanding** — `LayoutLM`, `key-value`` ``extraction`, `table`` ``extraction`, `Amazon`` ``Textract`

Curated starting points:

- [PaddleOCR documentation](https://github.com/PaddlePaddle/PaddleOCR) — actively maintained, strong on non-Latin scripts including Vietnamese

- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/) — the managed-service reference for invoice/form extraction

- A comparison write-up of open-source OCR engines (search “Tesseract vs PaddleOCR vs EasyOCR benchmark” for a current one — tooling here moves fast enough that a specific link goes stale within a year)

## For Group B

Your project brief is at [Project: Extract Information From Invoices and Contracts](practice/group-project.md). Read it before your first meeting.
