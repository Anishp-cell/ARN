# Model Card: ARN-v2 (Aether Routing Network v2)

ARN-v2 is a small model (4.44M parameters) that picks which function to call from a fixed list, and marks the words in the request that are its arguments. It does not write text. It labels the request.

This card covers v2. The first version (v1) is withdrawn. Its evaluation split leaked near-copies of examples between training and test, so its headline numbers do not hold. The v1 history is in git.

---

## 1. Model details

- **Name:** ARN-v2, BERT-Tiny + FAR-v2 router (`arn_v2_tiny_far`)
- **Parameters:** 4,440,019 (checked against the saved checkpoints)
- **Architecture:**
  - Encoder: `google/bert_uncased_L-2_H-128_A-2` (2 layers, hidden size 128), fine-tuned end to end.
  - Tool router: FAR-v2, a factored attention router with a per-tool bias and a [CLS] residual. It outputs one score per tool.
  - Argument tagger: a linear layer plus a linear-chain CRF over tool-qualified BIO tags, such as `B-send_email.subject`. Invalid tag transitions are masked.
  - Decoder: constrained joint decoding. It scores candidate tool sets against the CRF and keeps the best tool set and span assignment.
  - Dates and times: a rule-based canonicalizer. Its conventions were fitted on the training split only.
- **Checkpoints (FP32 PyTorch, about 17.8 MB each):** `arn_v2_tiny_far_s42_newdec.pt`, `..._far_s1.pt`, `..._far_s2.pt`. Each checkpoint stores its own tool and tag registry.
- **Version:** v2, November 2026
- **License:** MIT (code and checkpoints). The training data is Google's Mobile Actions dataset, which has its own terms. Check its dataset card before reuse.

---

## 2. Intended use

- Routing a short user request to one or more functions from a **fixed, known list** (the registry), and extracting the argument values.
- Research on small, non-generative function calling and on how much of the result depends on rules.
- Running on a CPU with no network access. It takes about 11 ms per request on one thread.

### Out of scope
- **Open-ended chat.** The model does not generate text.
- **Unknown tools.** It can only pick from the registry it was trained with. A new tool needs new training data.
- **The same tool twice in one request.** No evaluation row needs this, so the model is not designed for it.
- **Autonomous high-impact actions.** Destructive or sensitive functions (for example, deleting files or sending money) need a human confirmation step in the app that calls the model.
- **Non-English requests.** Training and evaluation are in English only.

---

## 3. Training and evaluation data

- **Train:** Google Mobile Actions, `metadata == "train"`. Split into 90% fit and 10% validation with a fixed seed.
- **Test:** Google Mobile Actions, `metadata == "eval"`, 961 requests. Scored with strict exact match.
- **Important:** the test split was inspected during development. Decoder changes were made while looking at test errors, and each change was then checked on validation and on train. The Mobile Actions numbers are therefore an **exploratory** result. The pre-registered MixATIS and MixSNIPS results (Section 4.3) are the confirmatory evidence.

---

## 4. Results

All ARN-v2 numbers are averages over three training seeds (42, 1, 2), with the spread shown as ± standard deviation. The per-seed predictions are saved in `results/xmodel/`. Re-scoring them with `src/arn/eval/mobile_actions_scorer.py` reproduces the per-seed numbers: 89.28, 89.80, and 88.87.

### 4.1 Google Mobile Actions (test, 961 requests, strict exact match)

| System | Parameters | Trained on train split | Strict exact match |
|---|---|---|---|
| **ARN-v2, BERT-Tiny + FAR-v2** | **4.44M** | yes | **89.32 ± 0.47** |
| ARN-v2, BERT-Mini trunk | 11.38M | yes | 89.91 ± 0.46 |
| FunctionGemma-270M, fine-tuned by us, same rows and recipe | 268M | yes | 85.12 |
| LFM2.5-230M, fine-tuned by us, default recipe | 230M | yes | 91.57 |
| LFM2.5-230M, fine-tuned by us, best of a 5-setting sweep (selected on validation) | 230M | yes | 92.51 |

Paired bootstrap over test rows, seed-matched: ARN-v2 minus fine-tuned FunctionGemma is +4.16, +4.68, and +3.75 points across the three seeds. Each 95% interval is above zero. ARN-v2 minus LFM2.5 (default recipe) is −2.29, −1.77, and −2.71. Those intervals are below zero, or touch it.

**Comparison caveat.** The FunctionGemma row is our fine-tune on the same rows. It is not Google's released checkpoint. A third-party paper reports 90.32 for Google's released Mobile Actions checkpoint, in a different harness, so the two are not directly comparable.

### 4.2 Ablations (3 seeds)

| Variant | Mean ± SD | Change vs full |
|---|---|---|
| Full system | 89.32 ± 0.47 | — |
| No datetime canonicalizer | 55.22 ± 0.37 | about −34 points, all seeds |
| Keyword call ordering replaced by attention ordering | 85.81 ± 6.12 | −10.5 points on seed 42, no change on seeds 1 and 2 |
| No multi-span join | 88.17 ± 0.42 | about −1 point |
| No registry mask in decoding | 89.21 ± 0.47 | about −0.1 point |

The canonicalizer and keyword ordering are rule-based. The system is a neural model plus rules, and the rules matter a lot.

### 4.3 Pre-registered benchmarks (no rules; test read once per seed)

| Benchmark | Overall accuracy (3 seeds) |
|---|---|
| MixATIS (828 utterances) | 40.94 ± 2.62 |
| MixSNIPS (2,199 utterances) | 70.29 ± 0.46 |

These are below several published multi-intent models, including some BiLSTM models that are smaller than ARN-v2 (about 0.6 to 0.9M parameters). These benchmarks test whether the model works without the Mobile Actions rules, and it does not work as well on them.

### 4.4 Speed

One request at a time, FP32 PyTorch, CPU on a Windows laptop, warm-up discarded. p50 is the median time per request. p90 is the 90th percentile.

| System | Threads | p50 | p90 |
|---|---|---|---|
| ARN-v2, end to end (including tokenization) | 1 | 11.40 ms | 18.22 ms |
| ARN-v2, end to end | 4 | 4.10 ms | 6.15 ms |
| LFM2.5-230M | 1 | 4,316 ms | 9,250 ms |
| FunctionGemma-270M | 1 | 4,613 ms | 8,299 ms |

The generative-model numbers are not tuned. Optimized runtimes would be faster. Full table: `results/latency_v2.json`.

---

## 5. How to reproduce

Re-score the saved predictions (no training needed): use `src/arn/eval/mobile_actions_scorer.py` on `results/xmodel/arn_v2_tiny_far_s42_newdec.json`, with the eval rows of the Mobile Actions dataset.

Re-decode a checkpoint with the current decoder (no retraining):

```
python scripts/v2_redecode.py --data <path to Mobile Actions dataset.jsonl> --ckpt checkpoints/v2/arn_v2_tiny_far_s42_newdec.pt
```

Training: `scripts/v2_train_mobile.py`. Pinned environment: `results/environment_v2.json`. The PyTorch build is a nightly, so pin the exact version to reproduce.

---

## 6. Limitations and failure modes

1. **Rules do a lot of the work.** Without the datetime canonicalizer the score drops to 55%. Without keyword ordering it can drop by up to about 10 points, depending on the seed.
2. **Closed registry.** A tool outside the training registry cannot be called, and a new tool needs new training data.
3. **Test split used during development.** See Section 3.
4. **Single-turn requests only.** Multi-turn context and pronoun references are not handled.
5. **Narrow coverage.** The model has only seen the Mobile Actions tool set and phrasings. It has not been tested on real users' speech or real device data. On MixATIS and MixSNIPS it scores lower.
6. **Confidence is not calibrated.** Do not treat a score as a reliability guarantee.

Full disclosures and the robustness checks (train/test overlap, nearest-neighbour splits): `docs/RESULTS_v2.md`, Section 4.

---

## 7. Ethical and safety notes

- **Local by design.** The model runs without a network connection, and no request text leaves the device.
- **Confirmation gate.** Any function that can cause harm must require user confirmation in the calling app. The model does not check this.
- **Bias.** The training data is short, English, assistant-style requests. Accents, dialects, and other languages are not covered.

---

## 8. Contact and status

This is a student research project. It has not been peer-reviewed. Please report errors, failed reproductions, or evaluation concerns as GitHub issues. Corrections to published numbers will be made in this card and in `docs/RESULTS_v2.md`, with the date of each change.
