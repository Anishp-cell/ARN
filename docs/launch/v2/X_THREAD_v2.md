# X thread draft (ARN v2) — DRAFT, not posted

Numbers: `docs/RESULTS_v2.md`. Hero image: `docs/paper_v2/fig_pareto.png` (export of the PDF).

---

**1/7**
A model 60 times smaller beat Google's FunctionGemma on Mobile Actions.

ARN: 4.4M parameters vs 268M. Same training data. 11 ms on one CPU thread.

**2/7**
Most small on-device function callers write the JSON one token at a time.

If the tool list is fixed and most argument values are words already in the request, ARN labels the request instead: which functions apply, and which words are the details.

**3/7**
ARN is a BERT-Tiny encoder, a router that picks the functions, and a tagger that marks the argument words.

Dates and times go through a short rule-based step. Without it, the score drops from 89 to 55.

**4/7**
Google's Mobile Actions test split (961 requests, exact match), 3 seeds:

ARN: 89.32 ± 0.47
FunctionGemma-270M, same training rows: 85.12
LFM2.5-230M, tuned: 92.51

So ARN is ahead of FunctionGemma and behind LFM2.5.

**5/7**
Speed on one CPU thread (median per request):

ARN: 11.4 ms
LFM2.5-230M: 4,316 ms

Measured in our setup, end to end including tokenization.

**6/7**
The caveats, up front:
- I looked at the test set while developing, so these results are exploratory.
- With no rules, on a separate test, ARN reaches 40.9 on MixATIS and 70.3 on MixSNIPS. That is below several published models.
- The full limitations are in the repo.

**7/7**
Model: huggingface.co/AnishPathak2202/arn-v2
Code: github.com/Anishp-cell/ARN
Paper: coming soon.
Feedback welcome, especially where the date rules or the fixed function list break on your data.
