# Mood Detection Evaluation

Status: keyword fallback evaluated on 2026-11-12. `emotionService.js` is merged; the LLM pipeline run needs a Gemini key in `backend/.env` (`npm run eval:llm`).

## Summary

| Engine | Top-1 accuracy | Macro-F1 | Latency (p95) | Target |
| --- | --- | --- | --- | --- |
| Keyword fallback (rule-based) | **71.5%** | 0.754 | < 3 ms | < 50 ms, never fails |
| LLM pipeline (Gemini + fallback) | pending | pending | pending | **≥ 80%** accuracy, < 3 s |

When the keyword fallback finds at least one mood cue, it is right **95.1%** of the time (98 of 103 sentences). Of its 41 errors, 36 are sentences with no mood word at all, which need the LLM.

## Benchmark

`backend/tests/hinglish_eval.json`: 144 labelled sentences, written the way people text (typos, emojis, code-mixing, indirect phrasing).

| Language | Items | Per mood |
| --- | --- | --- |
| English | 48 | 6 |
| Hindi (Devanagari) | 48 | 6 |
| Hinglish (Romanised) | 48 | 6 |

- Each sentence has one gold mood. A prediction is correct when `targetMoods[0]` equals it; the top-3 hit rate also counts the gold mood anywhere in `targetMoods`.
- The benchmark was frozen on 2026-11-12. Labels are never changed to raise accuracy.
- The LLM prompt's few-shot examples (`promptExamples.js`) are disjoint from the benchmark; `emotion_detection.test.js` enforces this.

## How to run

```bash
cd backend
npm test                                   # unit tests + keyword accuracy floor (65%)
npm run eval                               # keyword fallback report
npm run eval -- --errors                   # ... plus every misclassified sentence
npm run eval:llm                           # full LLM pipeline report (needs LLM_API_KEY)
RUN_LLM_EVAL=1 npm test                    # enforce the 80% target in the test suite
```

`npm run eval:llm` waits 4 s between calls (about 10 minutes for 144 items) to stay under free-tier rate limits; change it with `--delay <ms>`. Answers that fell back to keywords are counted and reported separately.

## Keyword fallback results

| Metric | Value |
| --- | --- |
| Top-1 accuracy | 71.5% |
| Top-3 hit rate | 72.9% |
| Macro-F1 | 0.754 |
| Coverage (sentences with at least one cue) | 71.5% (103 / 144) |
| Accuracy when a cue is found | 95.1% (98 / 103) |

### By language

| Language | Top-1 accuracy | Top-3 hit rate | Accuracy when a cue is found |
| --- | --- | --- | --- |
| English | 64.6% | 64.6% | 100% (29 / 29) |
| Hindi (Devanagari) | 75.0% | 75.0% | 94.6% (35 / 37) |
| Hinglish (Romanised) | 75.0% | 79.2% | 91.9% (34 / 37) |

### By mood

| Mood | Precision | Recall | F1 |
| --- | --- | --- | --- |
| Happy | 80.0% | 66.7% | 0.727 |
| Sad | 100.0% | 77.8% | 0.875 |
| Angry | 100.0% | 72.2% | 0.839 |
| Stressed | 100.0% | 66.7% | 0.800 |
| Relaxed | 31.5% | 94.4% | 0.472 |
| Motivated | 100.0% | 50.0% | 0.667 |
| Romantic | 100.0% | 66.7% | 0.800 |
| Nostalgic | 93.3% | 77.8% | 0.848 |

### Confusion matrix

Rows are the gold mood, columns the predicted mood.

| Gold \ Pred | Hap | Sad | Ang | Str | Rel | Mot | Rom | Nos |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Happy | 12 | · | · | · | 6 | · | · | · |
| Sad | · | 14 | · | · | 3 | · | · | 1 |
| Angry | · | · | 13 | · | 5 | · | · | · |
| Stressed | · | · | · | 12 | 6 | · | · | · |
| Relaxed | 1 | · | · | · | 17 | · | · | · |
| Motivated | · | · | · | · | 9 | 9 | · | · |
| Romantic | 2 | · | · | · | 4 | · | 12 | · |
| Nostalgic | · | · | · | · | 4 | · | · | 14 |

The Relaxed column is the default: a sentence with no cue returns `Relaxed` by contract (README 6.4). That gives Relaxed high recall and low precision, and every other mood near-perfect precision.

## Error analysis

The 41 misclassified sentences fall into four groups.

| Cause | Count | Examples | What fixes it |
| --- | --- | --- | --- |
| No mood word: the feeling is implied by the situation | 36 | "nobody even remembered my birthday" (Sad), "rent is due and my salary hasn't come yet" (Stressed), "leg day today, need something to push through" (Motivated) | World knowledge: the LLM |
| Competing cues | 2 | "kal pehli date hai, thoda nervous thoda excited" (Romantic, predicted Happy) | Context weighting: the LLM |
| A cue that means something else here | 2 | "छत पर बैठकर ठंडी हवा का मज़ा ले रहा हूँ" (Relaxed; "मज़ा" read as Happy), "usse baat karke dil khush ho jata hai" (Romantic; "dil khush" read as Happy) | The LLM |
| Gold label debatable | 1 | "ghar ki bahut yaad aa rahi hai hostel mein" (gold Sad, predicted Nostalgic) | Accept: homesickness is both |

Motivated has the lowest recall (50%) because motivation is usually expressed as a plan ("5am run before class") rather than a feeling word. English has the lowest accuracy (64.6%) for the same reason, yet every English sentence where a cue was found was classified correctly.

### Fixes made during the analysis

The error list exposed four generic bugs. They were fixed without adding any benchmark wording to the lexicon:

| Bug | Example | Fix |
| --- | --- | --- |
| Prefix terms never matched (`frustrat*` lost its `*` during normalisation) | "kitna frustrating hai ye" | Keep the prefix marker when compiling terms |
| Idioms read as negation | "can't stop smiling" read as not happy | `stop`, `help`, `wait` after a negator open an idiom |
| An intensifier inside a phrase broke the match | "mann **bilkul** nahi lag raha" | Allow one intensifier between phrase words |
| Negating a minor mood flipped polarity | "याद नहीं हो रहा" (can't remember) read as Sad | Flip only a cue group's main mood |

Accuracy went from 69.4% to 71.5%. Each fix has a regression test in `keywordFallback.test.js`.

## LLM pipeline results

Pending. After `emotionService.js` is merged, run `npm run eval:llm -- --errors --markdown ../docs/llm-eval.md` and record the numbers in the summary table above. Then:

1. Compare the LLM's per-language accuracy with the fallback's, especially on Hindi (Devanagari).
2. Send the misclassified sentences to the prompt owner for tuning. Never add benchmark sentences to the prompt.
3. Re-run after tuning and report both runs.
