# Speech AI Overview

Everyone reads this page — only Group C goes deep on it.

## Turning Sound Into Something a Model Can Read

Audio arrives as a **waveform** — air pressure over time. A model does not consume that directly; it is converted into a **spectrogram**, a representation of which frequencies are present at each moment, or a more compact variant called **MFCC** (Mel-Frequency Cepstral Coefficients) that emphasises the frequency bands human speech actually varies over. This is the audio equivalent of the pixel-normalisation step from [Deep Learning Core Concepts](../deep-learning/core-concepts.md) — it is not the model itself, but nothing downstream works without it.

``` mermaid
        graph LR
    A[Raw waveform] --> B[Spectrogram / MFCC]
    B --> C["ASR model<br/>(e.g. Whisper)"]
    C --> D[Transcript text]
```

## Automatic Speech Recognition Today

Automatic Speech Recognition (ASR) architectures historically used the same CTC loss mentioned in [OCR & Document AI Overview](../ocr-and-document/overview.md) for the same reason — the model does not know in advance how many audio frames correspond to one output character or word, so the loss has to allow for that alignment ambiguity. Modern systems like **Whisper** are trained end-to-end on very large, diverse audio datasets and are now the practical default for most teams — training an ASR model from scratch is rarely worth it outside of research.

The standard evaluation metric is **Word Error Rate (WER)** — insertions, deletions, and substitutions divided by the reference word count. A WER of 0 is a perfect transcript; anything above roughly 15–20% starts to feel noticeably unreliable to a reader.

## What Still Goes Wrong

Speech is genuinely harder than it looks from a demo. The known failure modes worth watching for directly in this track’s case study:

- **Background noise** — call-centre audio is rarely clean

- **Regional accents** — models trained mostly on one accent distribution degrade on others

- **Overlapping speakers** — two people talking over each other; separating “who said what” is a distinct problem called **diarization**

- **Vietnamese specifically** — tonal languages and diacritics are a common weak point for models trained primarily on English data; check accuracy on Vietnamese audio directly rather than assuming it transfers

## Explore Further

- **Audio representation** — `waveform`, `spectrogram`, `MFCC`, `sample`` ``rate`

- **ASR** — `CTC`` ``loss`, `Whisper`, `Amazon`` ``Transcribe`, `streaming`` ``transcription`

- **Evaluation** — `Word`` ``Error`` ``Rate`` ``(WER)`, `diarization`

Curated starting points:

- [OpenAI Whisper repository](https://github.com/openai/whisper) — the most direct path from this overview to a working transcript, runs locally

- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/) — the managed-service alternative, with built-in diarization and custom vocabulary

- A recent Whisper accuracy benchmark for Vietnamese (search “Whisper Vietnamese WER benchmark” — model versions change quickly enough that a fixed link goes stale)

## For Group C

Your project brief is at [Project: Transcribe Call-Centre Recordings](practice/group-project.md). Read it before your first meeting.
