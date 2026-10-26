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
- *What I changed:* I decided that "Don't." is a bare directive and should be `suggest`, like "Neither." and "Keep it in the box," and that the "Currently unavailable" comment is a market fact and should be `other`. I kept `experience` narrow rather than widening it. I then re-labelled the 40 under my final rules and wrote the soft-phrasing and question-form rules into my README. 

**Pre-labelling disclosure:** I labelled 20 comments cold, using only my written definitions, with no AI or other tools (`note` = `cold`). I labelled another 50 myself (`note` = `dev`), but my rules were revised through discussion with Claude while I did, so those 50 shaped the definitions and I treat them separately from the cold rows. The remaining 130 comments were pre-labelled by Claude in three batches of 40, 40, and 50 (`note` = `prelabel`). For each batch I gave Claude my final definitions and decision rules. Claude returned a label and a confidence level for every row and flagged the ambiguous ones. I then read every comment, made my own call, and compared. I did not change the labels that Claude marked. I also used Claude to review my acceptance-criteria drafts and to suggest a range for each target, and to help draft parts of this README. The criteria, the numbers I committed, and the label definitions are my own decisions.

<!-- ═══════════════════════ UNIT 6 — THE TEST ═══════════════════════

     Don't fill these in during unit 5.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Baseline vs. Trained

All three columns are scored on the same 30 posts (`test_split.csv`, seed 42). "Baseline (definitions)" is the zero-shot model prompted with my label definitions. "Baseline (bare names)" is the same model given only the label names. The bare-names run is the stronger baseline, so it is the one I compare against below.

| Measure | Baseline (definitions) | Baseline (bare names) | Trained | Trained minus bare names |
|---|---|---|---|---|
| Overall accuracy | 0.367 | 0.467 | 0.633 | +0.167 |
| Macro F1 | 0.179 | 0.386 | 0.476 | +0.089 |
| F1 — `experience` | 0.000 | 0.200 | 0.000 | -0.200 |
| F1 — `other` | 0.000 | 0.353 | 0.727 | +0.374 |
| F1 — `suggest` | 0.537 | 0.606 | 0.700 | +0.094 |

**What I predicted (and when):**

- **Prediction 1 (seed-42 run):** The trained model will beat the baseline on `other` and on overall accuracy, because the baseline with definitions called every post `suggest`. I do not expect it to beat the baseline on `experience`. *(Written after I had already seen the seed-42 results in the notebook output, so this is informed by that run and not a blind prediction.)*
- **Prediction 2 (three-seed run, written before running section 6):** `experience` will stay my weakest label and will vary more between seeds than `suggest` or `other`, because it has only about 7 test rows. I expect the trained model's accuracy to stay above the bare-names baseline's 0.467 on every seed. This is a rough comparison because the baseline was only scored on the seed-42 split.

**What the gap actually means:**

Fine-tuning added something on overall accuracy. The trained model got 19 of 30 posts right against 14 for the bare-names baseline, a gain of five posts. Most of the gain is on `other` (0.353 to 0.727), with a smaller gain on `suggest`. The much bigger gap against the definitions run (0.367) mostly shows how badly that run collapsed, since it called every post `suggest`. My definitions were not worth anything to this model: the bare-names run scored three posts better, and my guess is that scoring each label independently suits a function-based label like "anything that is neither of the above" badly. I haven't tested that guess.

The training added nothing on `experience`. The trained model's F1 is 0.000, below the bare-names baseline's 0.200, and its confusion matrix shows it never predicted `experience` for any of the 30 posts. Five of the seven `experience` posts went to `other` and two to `suggest`. (I don't yet know why. It could be the small number of `experience` training rows, a boundary my labels didn't capture consistently, or something about the model, and I haven't tested any of these. The three-seed run, the confusion matrix, and the agreement check are what I'll use to find out.)

These are one split of 30 posts, and one post is about 3 points of accuracy, so I treat the +0.167 as suggestive and not settled. The three-seed run will show whether it holds.

---

## Run Log — Before

Trained on Apple Silicon GPU (`mps`), torch 2.14.1, `distilbert-base-uncased`, defaults (3 epochs, lr 2e-5, batch 16, 128 tokens). Each seed has a different 30-post test split. All numbers come from `results_three_seeds_before.json`.

| Criterion | Target | Seed 42 | Seed 7 | Seed 2024 | Verdict |
|---|---|---|---|---|---|
| 1. Overall accuracy | at least 0.60 on every seed | 0.633 | 0.500 | 0.533 | Missed (holds on 1 of 3 seeds) |
| 2. `other` F1 | at least 0.60 on every seed | 0.727 | 0.545 | 0.615 | Missed (seed 7 is 0.545) |
| 3. `experience` F1 | at least 0.50 on every seed | 0.000 | 0.000 | 0.000 | Missed (0.000 on all seeds) |
| 4. True `experience` predicted as `suggest` | at most 40% | 28.6% (2 of 7) | 71.4% (5 of 7) | 57.1% (4 of 7) | Missed (holds on 1 of 3 seeds) |
| 5. Agreement on staff posts | at least 70% (21 of 30) | 63% (19 of 30) | n/a, single measurement | n/a, single measurement | Missed (2 posts short of 21) |

Criterion 5 is measured once, against the fixed 30 staff posts (`agreement_results.json`), so it has no per-seed values.

### Spread across seeds

| Measure | Seed 42 | Seed 7 | Seed 2024 | Spread |
|---|---|---|---|---|
| Overall accuracy | 0.633 | 0.500 | 0.533 | 0.133 |
| Macro F1 | 0.476 | 0.375 | 0.403 | 0.100 |
| F1 — `suggest` | 0.700 | 0.581 | 0.593 | 0.119 |
| F1 — `experience` | 0.000 | 0.000 | 0.000 | 0.000 |
| F1 — `other` | 0.727 | 0.545 | 0.615 | 0.182 |

Accuracy moved 0.133 across seeds, more than 10 points. That is 4 posts out of 30 (19, 15, and 16 correct), so my dataset is too small for a stable measure, and a single seed would have given me a misleading picture. Seed 42 was the friendliest draw.

### Confusion matrix (seed 42)

| true \ predicted | suggest | experience | other |
|---|---|---|---|
| **suggest** | 7 | 0 | 4 |
| **experience** | 2 | 0 | 5 |
| **other** | 0 | 0 | 12 |

**My biggest off-diagonal number, and what it means:**
In seed 42, 5 real `experience` posts were called `other` and 0 real `other` posts were called `experience`, so the model leans toward `other` on this pair, and it never predicted `experience` in any of the three seeds, which means it did not learn that label.

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

**Agreement rate:** 19 / 30 = 63%

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
> *The post:* "Ran the same loop in 4C and in 24C within a fortnight, same effort by heart rate. 90 seconds slower in the heat over six miles. People keep saying heat costs you 'a bit' and it's more than that."
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* I labelled it `hot_take` because the last sentence ("People keep saying heat...") sounded like a general claim, but I agreed the first two sentences are checkable facts. Staff is right. Rule 1 asks whether the fact carries the argument, and here the 90-second comparison is the reason for "it's more than a bit." A concluding sentence is part of any argument and doesn't turn it into a `hot_take`. The rule doesn't say whether one runner's own times count as checkable, but its list includes "a time," so I should have followed it.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**2.**
> *The post:* "Went back through the club results going back to 2012. Every single year the number of people who enter the spring marathon and don't start it sits between 18 and 23 percent. It's not a bad-luck year, it's the base rate."
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* I labelled it `hot_take` because "It's not a bad-luck year, it's the base rate" sounded like a general claim, and I let it override the first two sentences. Staff is right. The 18 to 23 percent figure across every year since 2012 is the reason for "it's the base rate," so the fact is load-bearing under rule 1.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**3.**
> *The post:* "The 10K route was remeasured in 2021 and came out 60 metres long. Everyone comparing PBs from before and after that is comparing two different races, which is about 12 seconds at club pace."
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* I labelled it `hot_take` because it sounded like a general claim and not an analysis. Staff is right. The 2021 remeasurement and the 60 metres are why PBs from before and after aren't comparable, so the fact carries the argument (rule 1). I read it as a general claim without checking what supported it.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**4.**
> *The post:* "Three of us ran the same session on the same day, different shoes, within 10 seconds of each other over 5 miles. I'm not saying shoes do nothing, but whatever they do was smaller than the noise between three people."
>
> *Staff said / I said:* `analysis` / `hot_take`
>
> *My call, and why:* I labelled it `hot_take` because the last sentence sounded like a general claim (or possibly a reaction). Staff is right. The result (within 10 seconds over 5 miles, different shoes) is the basis for "smaller than the noise," and rule 3 says that how it reads as a reaction doesn't matter when the post argues from a fact. A sample of three is small, and the rule doesn't say whether that counts, but rule 1 tests whether the fact does the work, not how strong it is.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**5.**
> *The post:* "If you need a gel before 90 minutes you have a fuelling problem, not an energy problem."
>
> *Staff said / I said:* `hot_take` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because I thought the number was the argument supporting why it's not an energy problem. Staff is right. "90 minutes" is a threshold inside the claim, and nothing is offered to support the claim. That's the case rule 1 describes as a `hot_take` with a number in it, like "times are down 3%" as decoration. I mistook a number in the claim for the evidence.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**6.**
> *The post:* "Road racing peaked in about 2015 and everything since has been marketing."
>
> *Staff said / I said:* `hot_take` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because I thought the conclusion ("everything since has been marketing") was derived from a checkable fact ("road racing peaked in about 2015"). Staff is right. "Peaked in about 2015" is itself the claim, and the post offers nothing for it. It has the same shape as staff's example "Super shoes have ruined racing, times are down 3% since 2017," which rule 1 labels `hot_take`. I treated a date inside the claim as a fact the conclusion came from.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**7.**
> *The post:* "Track sessions are wasted on anyone running slower than 8 minute miles. You'd get more out of just running more."
>
> *Staff said / I said:* `hot_take` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because I thought the conclusion was derived from the fact ("wasted on anyone running slower than 8 minute miles"). Staff is right. The 8-minute-mile threshold is part of the assertion, and no fact backs it up. The second sentence is another claim. It's the same mistake as posts 5 and 6.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**8.**
> *The post:* "Four minutes off. FOUR. I've been chasing that time for two years and I'm sat on the kerb outside the finish just grinning at nothing."
>
> *Staff said / I said:* `reaction` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because I treated the result as a fact the conclusion was derived from. Staff is right. The post reports what just happened to the writer and how it felt, with no argument and no conclusion drawn from "four minutes off." That fits rule 2's description of a `reaction`. I treated a time as evidence when it's just the event.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**9.**
> *The post:* "First run back after eight weeks out. Two miles, slower than my warm-up used to be, and I don't care even slightly."
>
> *Staff said / I said:* `reaction` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because the statements sounded like facts. Staff is right. The facts (eight weeks out, two miles) are the writer's own narrative, and "I don't care even slightly" is a feeling. Nothing is argued from them. I mistook facts in a story for facts in an argument.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**10.**
> *The post:* "Wind was so bad on the out section I was leaning at about 20 degrees. Coming back felt like cheating."
>
> *Staff said / I said:* `reaction` / `analysis`
>
> *My call, and why:* I labelled it `analysis` because it captured the moment and led to a conclusion ("Coming back felt like cheating"). Staff is right. "About 20 degrees" is hyperbole and not a checkable fact, and "felt like cheating" is a feeling, not a conclusion derived from the wind. It's the writer's own experience, as rule 2 describes.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**11.**
> *The post:* "Signed up. Card's charged, no going back, immediately regret everything."
>
> *Staff said / I said:* `reaction` / `hot_take`
>
> *My call, and why:* I labelled it `hot_take` because it sounded like a general claim. Staff is right. The post is about the writer's own feeling just after signing up, and it doesn't assert anything general about the world. Rule 2 says a `reaction` reports what happened to the writer and how it felt, and a `hot_take` makes a general claim.
>
> *Which pile:* (a) The rule covered it and I applied it loosely.

**What the pattern in my disagreements tells me:**
All 11 disagreements involve a number or a date, and I got posts with the same structure (a figure plus a conclusion) wrong in opposite directions. In posts 1 to 4, where the data was the reason for the conclusion, I called them `hot_take`. In posts 5 to 7, where the number was part of the claim, I called them `analysis`. In posts 8 to 10, I treated numbers in a personal story as evidence. The common mistake is that I noticed a number was present and didn't ask whether it was doing work in the argument, which is the test in rule 1. Post 11 is the exception, where I read a feeling as a general claim. All 11 fall in pile (a), meaning the staff rules covered each case and I applied them loosely, so the problem was my consistency and not a gap in the definitions. This suggests I may have applied a function-based rule by its surface features, which could also have happened in my own labelling, for example with bare product names or implied advice. That's a hypothesis, and I still need to test it against my own labels.



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

       [X] criteria.md has five numbered criteria, each naming a NUMBER
       [X] Each has a reason underneath tied to your data or taxonomy
       [X] labels.csv: at least 150 rows, text/label/note, ONE file not split
       [X] No label above 70%
       [X] All five unit 5 sections have real content
       [X] Label Taxonomy includes the decision rule for your hardest boundary
       [X] The Dataset includes three hard cases
       [X] results.json and test_split.csv committed (the notebook does this)
       [X] At least four new commits
       [X] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 6

       [X] Baseline vs. Trained table, with your prediction written beforehand
       [X] Run Log — Before, five criteria across three seeds
       [X] Confusion matrix TYPED AS A MARKDOWN TABLE
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
