# TakeMeter

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, the
> notebook, the baseline, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> head -5 data/practice_labels.csv     # the shape your labels.csv needs
> ```
>
> Then open `takemeter.ipynb` **in this folder** — in VS Code, or with
> `jupyter notebook` if you prefer. Pick the kernel: the `.venv` inside this
> project. Run section 1, which reports the hardware you'll be training on.
> Everything else waits until you have data.
>
> Nothing to upload, nothing to connect, no accounts and no keys. The notebook
> runs on your machine and writes next to your code.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     Unit 5 asks for the first five sections. Unit 6 adds the five below them.

     Everything is pasted as TEXT. No screenshots, no images.

     ⚠️ The confusion matrix especially. The notebook prints one as a markdown
     table, ready to copy. A screenshot of a matrix earns nothing. Paste the
     table.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 5 — THE BUILD ═══════════════════════ -->

## What This Does

TakeMeter sorts comments from r/buildapc, a public subreddit where people ask for and give advice on PC parts. It reads one comment and labels it `suggest` (it points the original poster to something to buy, do, check, or try), `experience` (it is about the commenter's own situation or a sincere reaction), or `other` (questions asking for information, jokes, bare product names, and general claims). The goal is to tell advice apart from personal stories and noise in a help-heavy community. I labelled 200 comments from 19 threads and fine-tuned a small model on them.


---

## Label Taxonomy

### `suggest`

**Definition:** The comment points OP directly to something to buy, do, check, or try, whether stated as an instruction, as soft phrasing ("you can find...", "maybe look for..."), or as a question ("Have you tried a different cable?").

**Example 1:**
> Do NOT get the 730. If you’re on a super low budget you should be able to find another 1660 super for below $100. Otherwise upgrade.

**Example 2:**
> Keep it in the box.

### `experience`

**Definition:** The comment is about the writer's own situation or a sincere reaction (including toward OP), with no instruction to OP.

**Example 1:**
> I replaced my 1660s last year for a 5060

**Example 2:**
> I had it on the desk and that was even worse

### `other`

**Definition:** Anything that is neither of the above: questions that ask OP for information, jokes and sarcasm, one-word answers, bare product names, and general claims about hardware or prices that aren't about the writer.

**Example 1:**
> What's your budget?

**Example 2:**
> lol, welcome back to the world, i hope your time spent under a rock was relaxing.

### The hardest boundary

**Which two labels:** `suggest` vs `experience` (with `suggest` vs `other` close behind)

**The decision rule I used every time:**
Apply these in order:
1. If the comment points OP to something to buy, do, check, or try, it is `suggest`. This covers imperatives ("Buy...", "Avoid..."), "you should/need to," soft phrasing ("you can find...", "maybe look for..."), and question-form suggestions ("undervolting?"). It does not cover a bare product name ("7900 xt"), a statement that only implies a recommendation, or a snarky imperative that isn't advice ("Wait till you need what was a $200 20tb drive...").
2. If a comment is both advice and the writer's own experience, it is `suggest`. Example: "Save for 4K OLED. Best PC purchase I've ever made."
3. A question that asks OP for information ("What's your budget?") is `other`, even though it's a question.
4. If there is no pointer to OP, a comment about the writer's own situation or a sincere reaction is `experience`.
5. Everything else is `other`: jokes and sarcasm, bare product names, and general claims that aren't about the writer.

---

## The Dataset

**Where the posts came from:** 200 public comments from r/buildapc, collected across 19 threads. I chose threads to cover the subreddit's main flairs: Discussion, Build Help, Build Upgrade, and Troubleshooting. Topics included PC upgrade timing, RAM and SSD prices, GPU choices (e.g. RTX 3060 vs 5060), dust and cooling, and a failed GPU. I collected comments only, not original posts, by [r/buildapc](https://www.reddit.com/r/buildapc/) on 10/18/2026.

**How I labelled them:**
1. **Cold labels (20 rows, `note` = `cold`).** I labelled 20 comments first, using only my written definitions, with no AI help. I picked them from comments I hadn't used to draft the rules.
2. **Hand-labelled development set (50 rows, `note` = `dev`).** I labelled 50 comments myself. I used these to find where my definitions were ambiguous, and I revised the rules several times in conversation with Claude, which pointed out contradictions in my labels. The rules were shaped partly by these 50, so I treat them differently from the rest in the next unit.
3. **AI pre-labelled and checked (130 rows, `note` = `prelabel`).** I gave Claude my final definitions and decision rules and asked it to label the remaining 130 comments in three batches. I read every row, decided my own label, and compared it to Claude's. 

**Counts per label:**

| Label | Count | Share |
|---|---|---|
| `suggest` | 72 | 36.0% |
| `experience` | 49 | 24.5% |
| `other` | 79 | 39.5% |
| **Total** | 200 | 100% |

No label is above the 70% cap. `other` mixes several kinds of comment: information-seeking questions, jokes and sarcasm, bare product names, and general claims about hardware or prices. I expect it to be the hardest label for the model.

**Three hard cases**

**1.**
> *The post:* "Save for 4K OLED. Best PC purchase I've ever made."
>
> *Could have been:* `suggest` or `experience`
>
> *I chose* `suggest`*, because* it contains an explicit instruction ("Save for..."), and my rule says a comment that is both advice and the writer's own experience is `suggest`. Without the first sentence it would have been `experience`.

**2.**
> *The post:* "Wait till you need what was a $200 20tb drive..."
>
> *Could have been:* `suggest` or `other`
>
> *I chose* `other`*, because* it has the form of an imperative, but it doesn't tell OP to buy, do, check, or try anything. It's a sarcastic warning, and my rule excludes snarky imperatives that aren't advice.

**3.**
> *The post:* "7900 xt"
>
> *Could have been:* `suggest` or `other`
>
> *I chose* `other`*, because* it names a product and probably answers OP's question, but it contains no instruction, and my rule says a bare product name or implied recommendation doesn't count as `suggest`. The cost is that the model can't tell this apart from a comment that merely mentions a card, and I found it hard to apply this rule consistently to short answers.

---

## The Training Run

**Starting model:** `distilbert-base-uncased`, fine-tuned for three-way text classification (`suggest`, `experience`, `other`).

**Settings (all notebook defaults, nothing changed):**

| Setting | Value |
|---|---|
| Epochs | 3 |
| Learning rate | 2e-5 |
| Batch size | 16 |
| Max tokens per post | 128 |
| Seed | 42 |
| Split | 70/15/15, stratified by label (140 train, 30 validation, 30 test) |

**Hyperparameters I changed and why:** None. I used the defaults because this unit treats hyperparameters as vocabulary, and my first run is the baseline for unit 6.

**What I trained on:** Apple Silicon GPU (`mps`), torch 2.14.1, recorded in `results.json`. Different hardware does the arithmetic in a slightly different order, so someone running this on `cuda` or `cpu` with the same seed could get slightly different numbers.

**Thin label in the test split:** `experience` has only 7 of the 30 test examples (`suggest` has 11 and `other` has 12), which is below the notebook's threshold of about 8. One wrong `experience` row moves its recall by about 14 points, so I expect its metrics to move a lot between seeds. I'll read any `experience` score in unit 6 with that in mind.

**Files this run produced:** `results.json` and `test_split.csv`, both committed. The baseline in unit 6 will be scored on the same 30 posts.

---

## How I Used AI

**Moment 1: Finding out my first labels were broken**

- *What I asked for:* I sorted 40 sampled comments into four buckets from my reading (suggestion/advice, clarifying question, experience/empathy/agreement, disagreement) and asked Claude to check my tally.
- *What came back:* Claude pointed out that my buckets mixed two things (what a comment does, and how it relates to the comment it replies to), that "disagreement" and "clarifying question" were too thin to measure (2 and 4 of 40), and that I had labelled similar comments inconsistently, such as three troubleshooting questions that went two different ways.
- *What I changed:* I dropped the four buckets and tried a two-label version based on whether a comment names a specific detail. When I found that rule was just a keyword test, I switched to labelling by what a comment does, which gave `suggest`, `experience`, and `other`. 

**Moment 2: Testing my rules on comments I hadn't used**

- *What I asked for:* I wrote my label definitions and decision rules, picked 10 comments I had not looked at, labelled them, and asked Claude to label the same 10 using only my written rules, without seeing my answers.
- *What came back:* We agreed on 8 of 10. The two disagreements were "Don't." (I labelled `other`, Claude `suggest`) and "Currently unavailable, 550 for 'newer' model." (I labelled `experience`, Claude `other`). Claude also showed that several rows in my 40 broke my own rules.
- *What I changed:* I decided that "Don't." is a bare directive and should be `suggest`, like "Neither." and "Keep it in the box," and that the "Currently unavailable" comment is a market fact and should be `other`. I kept `experience` narrow rather than widening it. I then re-labelled the 40 under my final rules and wrote the soft-phrasing and question-form rules into my README. [Add anything you disagreed with Claude about, if anything.]

**Pre-labelling disclosure:** I labelled 20 comments cold, using only my written definitions, with no AI or other tools (`note` = `cold`). I labelled another 50 myself (`note` = `dev`), but my rules were revised through discussion with Claude while I did, so those 50 shaped the definitions and I treat them separately from the cold rows. The remaining 130 comments were pre-labelled by Claude in three batches of 40, 40, and 50 (`note` = `prelabel`). For each batch I gave Claude my final definitions and decision rules. Claude returned a label and a confidence level for every row and flagged the ambiguous ones. I then read every comment, made my own call, and compared. I did not change the labels that Claude marked. I also used Claude to review my acceptance-criteria drafts and to suggest a range for each target, and to help draft parts of this README. The criteria, the numbers I committed, and the label definitions are my own decisions.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

<!-- Both models on the same posts. `python baseline.py --trained results.json`
     prints this table for you. -->

| Measure | Baseline | Trained | Difference |
|---|---|---|---|
| Overall accuracy |  |  |  |
| Macro F1 |  |  |  |
| F1 — `label_one` |  |  |  |
| F1 — `label_two` |  |  |  |

**What I predicted before I looked:**
<!-- Milestone 1 asks you to write this BEFORE seeing the trained numbers. A
     prediction made afterwards isn't one. -->

**What the gap actually means:**
<!-- If the baseline matched your trained model, your fine-tuning added
     nothing — and that is a real finding, not a failure. Say it plainly. -->



---

## Run Log — Before

<!-- Five criteria across three seeds. The notebook's section 6 prints the
     spread table; the Target and Verdict columns are yours. -->

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

### Confusion matrix

<!-- ⚠️ TYPED AS A MARKDOWN TABLE. The notebook prints one ready to paste.
     An image of a matrix earns nothing. -->

| true \ predicted |  |  |  |
|---|---|---|---|
| **** |  |  |  |
| **** |  |  |  |
| **** |  |  |  |

**My biggest off-diagonal number, and what it means:**
<!-- Not "the model made mistakes" — WHICH boundary it didn't learn, and which
     direction. "7 real analysis posts were called hot_take and only 3 went the
     other way" is a direction, not just an error rate. -->



---

## Verdicts and Diagnoses

<!-- MET or MISSED against LAST UNIT's target. The target has to hold across
     all three seeds, not turn up sometimes. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**

<!-- For each miss: the cause, and how you know. The four common causes are:
     too few examples for a label, a boundary you applied inconsistently, a
     genuinely hard label pair, and a task the model can't reach from this
     much data.

     ⚠️ Use your agreement report as evidence. It is the only instrument you
     have that can tell a LABELLING problem from a MODEL problem, and this
     section is graded on whether you used it that way. -->



---

## Agreement Report

<!-- Your rate against the staff set, and every disagreement adjudicated.

     Remember you labelled these 30 under the STAFF taxonomy in
     data/staff_taxonomy.md, not your own — so every argument below is made
     from those definitions and those decision rules. -->

**Agreement rate:** ___ / 30 = ___%

<!-- Nobody grades this number. A 60% who argues every disagreement from the
     stated rules beats a 95% who wrote "staff was right" nine times. Several
     of the 30 were chosen because they're genuinely ambiguous — you should be
     winning some of these. -->

**Disagreements**

<!-- Three lines each: the post, both labels, and who you think is right and
     why — grounded in the staff definitions you were both applying.

     Then sort each into one of three piles:
       (a) the rule covered it and I applied it loosely → a consistency problem
       (b) the rule genuinely doesn't say               → a gap in the definitions
       (c) the rule is ambiguous here and my reading is defensible → argue it.
           This is a legitimate win.

     Pile (a) is the one that matters most for your diagnosis: if you applied a
     written rule two different ways on 30 posts, that is direct evidence about
     what you did across your own 200. -->

**1.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**2.**
> *The post:*
>
> *Staff said / I said:*
>
> *My call, and why:*
>
> *Which pile:*

**What the pattern in my disagreements tells me:**



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Run Log — After

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it didn't, say so. Relabelling that didn't help is a genuinely
     interesting result and earns full credit. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped. -->



**The gap between what I meant my labels to capture and what the model
learned:**
<!-- Two sentences. Your confusion matrix is the evidence. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 5

       [ ] criteria.md has five numbered criteria, each naming a NUMBER
       [ ] Each has a reason underneath tied to your data or taxonomy
       [ ] labels.csv: at least 150 rows, text/label/note, ONE file not split
       [ ] No label above 70%
       [ ] All five unit 5 sections have real content
       [ ] Label Taxonomy includes the decision rule for your hardest boundary
       [ ] The Dataset includes three hard cases
       [ ] results.json and test_split.csv committed (the notebook does this)
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 6

       [ ] Baseline vs. Trained table, with your prediction written beforehand
       [ ] Run Log — Before, five criteria across three seeds
       [ ] Confusion matrix TYPED AS A MARKDOWN TABLE
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, using the agreement report as evidence
       [ ] Agreement Report with every disagreement adjudicated
       [ ] One improvement, with Run Log — After
       [ ] What's Still Broken
       [ ] results_three_seeds_before.json, results_three_seeds_after.json,
           baseline_results.json and
           agreement_results.json committed
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
