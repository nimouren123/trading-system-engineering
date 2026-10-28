# 05 — Review methodology: why round 8 found what round 4 missed

After a live incident, the instruction was: keep reviewing until a full pass produces nothing.

Twelve rounds later, the loop terminated. It produced ~38 fixes, three of them critical.

The interesting part is not the count. It is that **round 4 declared convergence, and rounds 5 through 8 each found something worse.**

---

## The rounds

| Round | Lens | What it found |
|---|---|---|
| 1–4 | migration diff, strategy fidelity, engineering baseline | 11 issues — ordering, defaults, exit priority |
| 5 | "is every module *reachable* under live constraints?" | **HIGH** — a module required more bars than the API can return; permanently dead in live |
| 6 | environment / exchange mechanics / chaos sequences | **CRITICAL** — dead reconciliation loop → unhedged position; stale kill switch; stop computed without direction |
| 7 | "what did the previous round's fixes break?" | **CRITICAL** — a new method shadowed by an older property of the same name; **HIGH** — persistence flag never set by in-place mutation |
| 8 | full re-read from zero (all modules) | emergency-stop dead code; drawdown brake reset at the wrong boundary |
| 9 | verification of round 8's fixes | one fix was placed after the branch it needed to precede |
| 10 | convergence check | zero findings |
| 11 | three independent lenses (strategy / engineering / deployment) | 4 fixes, including a second instance of the same empty-string bug |
| 12 | convergence check | zero findings → terminated |

---

## Why convergence was declared too early

Round 4's question was: *"is the migrated code equivalent to the code it replaced?"*

That question has a natural end. Once the diff is accounted for line by line, there is nothing left to ask — **within that scope**. The code had other properties that the question never touched:

- Are all modules reachable given the API's real limits? (round 5 — no)
- What happens when two subsystems fail at the same time? (round 6 — a composition nobody had examined)
- Do the fixes themselves hold up? (round 7 — one didn't)

**Every review has an implicit scope, and defects outside that scope survive every pass.** Repeating the same lens is nearly guaranteed to produce the same "clean" verdict, because the same questions get the same answers.

This is the central failure mode of code review in general: *thoroughness is measured by effort, when it should be measured by the space of questions asked.*

---

## Four practices that produced findings

### 1. Change the lens, not the effort

The effective rounds were not longer — they asked a different question.

| Lens | Question |
|---|---|
| Migration | Is this equivalent to what it replaced? |
| Resource | Can every module actually get the data it needs? |
| Environment | What does the exchange/platform do that the code assumes it won't? |
| Chaos | What if two things fail at once? |
| Regression | What did the last fix break? |
| Fresh read | What does this code do, ignoring what I remember writing it for? |

The last one is the most expensive and the most productive. Reading code as a stranger removes the assumptions that make defects invisible to their author.

### 2. Cross-confirmation for severity

In round 6, two reviewers working independently — one on data-shape edge cases, one on failure sequencing — landed on the same dead function.

Independent convergence is strong evidence. It does not mean the finding is definitely real; it means the finding is not an artifact of one person's reading path. That distinction is what justified blocking a deploy.

### 3. Treat every fix as a new defect candidate

Round 7's critical bug was created by round 5's fix.
Round 9's was created by round 8's fix.

So the review order became: **fixes first, then everything else.** A change made under time pressure, inside a file you just convinced yourself you understand, is the highest-risk code in the repository.

The shadowed-method defect is the canonical example: adding `is_emergency_stopped()` as a method, in a class that already had a property with that name, is invisible unless you read the whole class body — and the diff only showed the addition.

### 4. Define the stop condition before starting

"Review until it looks good" has no natural end and terminates on fatigue. The condition used here:

> **Two consecutive full passes, each using a different lens, producing zero new findings.**

Round 10 satisfied half of it. Round 11 changed the lens again and found four more. Round 12 produced nothing — so the loop ended there, not at round 10.

---

## What did not work

- **Re-reading the same files with the same intention.** Rounds 2–4 were largely this. They were not wasted — they produced 11 real fixes — but they could not have found the round 6 or 7 defects.
- **Checking "does it work."** The system worked. It placed orders, took profit, and ran for days with the unhedged-position bug present. Anything evaluated by "did it run" would have passed.
- **Trusting tests written by the same author as the fix.** Round 7's new method passed its own tests; they called the definition, not the class.

---

## The generalizable part

Three things transferred to every other project since:

1. **A review's conclusion is only as strong as its scope.** "No issues found" is always shorthand for "no issues found *of the kind we looked for*."
2. **Fixes are the highest-risk code in a codebase**, because they are written under confidence, reviewed under time pressure, and deployed with attention directed at the bug they fix.
3. **Convergence needs a defined condition**, or the loop ends because the reviewer is tired — which is indistinguishable, from the outside, from the loop ending because the code is clean.
