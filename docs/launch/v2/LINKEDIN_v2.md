# LinkedIn post draft (ARN v2) — DRAFT, not posted

Numbers: `docs/RESULTS_v2.md`. Image: `docs/paper_v2/fig_pareto.png` (export of `fig_pareto.pdf`). Links go in the first comment, not the post body.

Before posting: the hook compares against FunctionGemma-270M fine-tuned by us on the same training rows. It does not compare against Google's released checkpoint. Keep the "same training rows" phrase in the post.

---

**Part 1: note on my first ARN post (post as a comment on the old post, or edit its first line)**

Update: the numbers in my first ARN post (1.39 ms, 4.28 MB, 16.91% joint exact match) are superseded. The rebuilt ARN v2 results are in the new post.

---

**Part 2: the ARN v2 post**

A model 60 times smaller beat Google's FunctionGemma.

4.4M parameters vs 268M. Same training data. Google's Mobile Actions benchmark.

Before you get excited: this is an early result, and the details matter. Here is how ARN works and where it falls short.

Phones and laptops often get simple requests like "set a timer for 10 minutes" or "text Alex I'm running late." The assistant has to pick one function from a fixed list and pull out the details, like the duration or the name.

Most small models do this by writing the answer one token at a time. ARN reads the request once and does two things:
- It decides which functions apply.
- It marks the exact words in the request that are the details.

Dates and times go through a short rule-based step. The model alone does not format them reliably.

Results on the Mobile Actions test set (961 requests, 3 random seeds):
- ARN: 89.3% exact match
- FunctionGemma-270M, same training setup: 85.1%
- LFM2.5-230M, tuned: 92.5%

So ARN is ahead of FunctionGemma and behind LFM2.5. The full comparison, with confidence intervals, will be in the paper.

Speed: about 11 milliseconds per request on one CPU core. LFM2.5 took about 4 seconds in the same test.

The caveats matter as much as the numbers:
- The date rules do a lot of the work. Without them, the score falls to 55%.
- I looked at the test set while developing, so treat this as an early result, not a final one.
- On a separate test with no rules, ARN reaches 40.9% on MixATIS and 70.3% on MixSNIPS. That is below several published models.

A correction too. My first ARN post had a flaw: random data splits let near-copies of the same examples land in both training and test, which inflated the numbers. I withdrew those results and rebuilt the evaluation with grouped splits. Every number in this post comes from the rebuilt version.

The model is on Hugging Face (link in the first comment). The code is on GitHub (link in the first comment). The paper is coming soon.

If you work on on-device assistants or tool use, I'd like to hear where a fixed list of functions breaks down for you.

#MachineLearning #EdgeAI #OnDeviceAI #NLP

---

**Notes for posting**
- The image is the Pareto chart: size against accuracy, with the models labelled.
- Links go in the first comment. Fill them in only after the Hugging Face model card is live.
- Post on a weekday morning. Reply to every comment in the first 24 hours.
