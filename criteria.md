# Acceptance criteria — TakeMeter

Five criteria that say what "working" means for this classifier, written in
unit 5 **before** anything was trained.

**All five are yours this time.** None are given. You've had two projects of
practice.

An acceptance criterion names a number. *"The model is accurate"* is an
opinion. *"Every label has an F1 of at least 0.60 on the held-out set"* is a
criterion.

Under each, write a sentence or two on **why that number**. A reason that says
something about your data or your taxonomy earns credit — *"I picked 0.60 F1
for `reaction` because it's my smallest label and I only have about 50
examples of it"*. A reason that could be attached to any project does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

Not numbers that sound impressive. Three labels means a coin-flip guesser gets
about 33%, so a target of 0.40 is barely a target. Your number should sit
somewhere you'd honestly call useful.

**Cover at least three of these five areas.** They're here as prompts, not as a
form to fill in — a criterion that fits none of them is fine if it names a
number.

| Area | A question it could answer |
|---|---|
| Overall accuracy | How often does it need to be right to be worth using? |
| Per-label performance | Is one label allowed to be much worse than the others? |
| Balance | How lopsided can your label counts get before it's a problem? |
| Consistency | If someone else labelled the same posts, how often should you agree? |
| Confidence | Should a confident prediction be right more often than an unsure one? |

Two things worth knowing before you pick numbers, because both will affect
whether you hit them:

- **Your smallest label will have the jumpiest score.** If a label has 50
  examples, about 8 land in the test split. An F1 computed on 8 examples moves
  a lot between seeds. A target for that label should be looser than one for
  your biggest label, and saying so is a good reason.
- **Unit 6 tests across three seeds, and the target has to hold across all
  three.** A target of 0.65 against results of 0.71, 0.62, 0.68 is a **miss**.
  Pick with that in mind — it is stricter than it first sounds.

---

## 1. Overall accuracy

Overall accuracy on the held-out test set is at least 0.60 on each of the three seeds.

**Why this target:** Always guessing my biggest label (`other`, about 40% of my rows) would get about 0.40, so 0.60 is clearly above guessing. With about 30 test rows, 0.60 is 18 correct. I did not go higher because I train on only about 140 rows and my own labels are fuzzy at the boundaries (I agreed with an AI labeller on 8 of 10 unseen rows), so I can't expect the model to be more consistent than I am.

<!-- Verify by: open results.json, read accuracy for each seed, and check that the lowest is at least 0.60. -->

---

## 2. `other` performance

`other` has an F1 of at least 0.60 on the held-out test set on each of the three seeds.

**Why this target:** `other` is my biggest label, so always guessing it already gets an F1 of about 0.57. This target is only slightly above that. I accept a target this close because `other` mixes jokes, bare product names, general claims about hardware and prices, and questions that ask OP for information, so there is no single pattern for the model to learn.

<!-- Verify by: open results.json, read the per-label F1 for `other` on each seed, and check that the lowest is at least 0.60. -->

---

## 3. `experience` performance

`experience` has an F1 of at least 0.50 on the held-out test set on each of the three seeds.

**Why this target:** `experience` is my smallest label (49 of 200 rows, about 7 in the test split), so one wrong row moves its F1 a lot between seeds. That is why I set it below my 0.60 target for `other`. Always guessing `experience` would get about 0.39, so 0.50 is clearly above guessing. This label also sits on my hardest boundary with `suggest`, where implied-advice comments are easy to confuse.

<!-- Verify by: open results.json, read the per-label F1 for `experience` on each seed, and check that the lowest is at least 0.50. -->

---

## 4. Boundary confusion

Of the test rows whose true label is `experience`, no more than 40% are predicted as `suggest`.

**Why this target:** `suggest` vs `experience` is the hardest boundary in my taxonomy, and implied-advice comments like "I replaced X and it fixed it" sit on it. With about 7 `experience` test rows, 40% means at most 2 rows can be predicted as `suggest`, and a third row fails the target. I made this stricter than criterion 3 implies, because an F1 of 0.50 alone would still allow up to about two-thirds of `experience` rows to be missed.

<!-- Verify by: build a confusion matrix from the model's predictions and test_split.csv, read the `experience` row, and check that the share predicted as `suggest` is at most 40%. -->

---

## 5. Consistency

When I label the 30 staff posts using only the staff definitions, my labels agree with staff labels on at least 70% of them (21 of 30).

**Why this target:** With three labels, chance agreement is about 33 to 40%. One post is about 3 points. The staff instructions say that a rate in the 70s is normal because several of the posts were picked for being ambiguous, so 70% is at the low end of what staff expect. This measures whether I can apply a written rule I didn't write, which is the same skill my own 200 labels depend on.

<!-- Verify by: run `python agreement.py` on my_staff_labels.csv and staff_labels.csv and read the agreement rate. -->

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 6 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Every label performs acceptably

         The model performs well on all labels.

         **Why this target:** ...

         > **Revised in unit 6:** Every label has an F1 of at least 0.60 on
         > the held-out set.
         >
         > **Why revised:** "performs well" gave me nothing to check. I
         > couldn't produce a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "Overall accuracy of at least 0.65" → "at least 0.55", because
            0.65 turned out to be optimistic for 200 examples.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
