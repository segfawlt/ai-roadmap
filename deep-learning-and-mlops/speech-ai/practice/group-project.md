# Project: Transcribe Call-Centre Recordings

## Group Responsible

Group C — Speech AI.

## Minimum Deliverable

Run Whisper (local, open-source, free) on several sample call-centre-style recordings and produce transcripts. Deliverable: a working demo that transcribes at least three audio clips, plus a written analysis of the model’s limitations on your specific samples — where does it get things wrong, and why?

Use publicly available call-centre-style audio or customer-service recording samples (search for public ASR benchmark datasets, or record realistic short mock calls yourselves) — do not use real customer recordings.

## Extensions (Not Required)

If the group finishes early, in order of value:

1.  Compute Word Error Rate against a manually-transcribed reference for at least one clip.

2.  Add diarization — separate “agent” from “customer” in the transcript.

3.  Compare Whisper’s output against Amazon Transcribe on the same clip and note the differences.

## Environment

Run Whisper **locally** — the smaller Whisper model sizes run on CPU at a usable speed for short clips, and this avoids burning Colab GPU hours on something that doesn’t strictly need them. If transcription is too slow locally, fall back to Colab.

## Review Schedule

- **Mid-week checkpoint** — show one clip transcribed end to end, even if the transcript has obvious errors.

- **Final demo** — transcribe a clip live, and walk through at least one specific error in the output and your best explanation for it (noise, accent, overlapping speech, Vietnamese-specific issue).

## Questions to Expect From the Other Groups

- Where in the pipeline did an error most likely originate — the audio representation, the model, or something about the specific speaker?

- What would Word Error Rate actually measure here, and what would it not capture (e.g., a transcript that’s 95% correct but garbles the one number that mattered)?

- Why open-source Whisper instead of a managed service, or vice versa?

## Hints

- Start with a short, clean clip to confirm your pipeline works before testing on noisy or accented audio.

- Read [Speech AI Overview](../overview.md) again once you have a real transcript in front of you — the failure modes described there will make more sense against your own output.

- If you attempt diarization, treat it as a genuinely separate problem from transcription accuracy — a transcript can be word-perfect and still mislabel who said it.

## References

- [Speech AI Overview](../overview.md)

- [Deep Learning Core Concepts](../../deep-learning/core-concepts.md)

- [MLflow Basics](../../mlops/mlflow-basics.md)
