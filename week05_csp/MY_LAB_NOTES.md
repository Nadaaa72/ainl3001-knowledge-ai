# Week 5 Lab — My Plain-English Notes

These notes explain the keywords, the code, and my answers to every question
in the Week 5 lab (Constraint Satisfaction Problems).

---

## The big picture (read this first)

In Week 4 (local search) we had a full answer and kept **improving** it.

This week is different. We build the answer **one piece at a time**, and
every piece has to follow the rules. If a piece breaks a rule, we undo it
straight away and try something else.

The problem: colour 6 regions of Australia using Red, Green or Blue, so that
**no two neighbours have the same colour**.

---

## Keywords and terms

| Term | What it means (plain English) |
|---|---|
| **CSP** | Constraint Satisfaction Problem. We don't want the "cheapest" answer, just *any* answer that follows all the rules. |
| **Variable** | Something we need to fill in. Here: each region (`WA`, `NT`, `SA`, `Q`, `NSW`, `V`). |
| **Domain** | The values a variable is allowed to take. Here: `["Red", "Green", "Blue"]` for every region. |
| **Constraint** | A rule. Here: the pair `("WA", "NT")` means "WA and NT must be different colours". |
| **Assignment** | A dictionary of choices so far, e.g. `{"WA": "Red", "NT": "Green"}`. |
| **Partial assignment** | Only *some* regions have a colour yet. |
| **Complete assignment** | *Every* region has a colour. |
| **Valid** | No rule is broken (so far). |
| **Backtracking** | Undoing the most recent choice when it leads to a dead end, then trying the next option. |
| **Brute force** | Trying every possible complete assignment one by one (729 of them). |
| **MRV** | Minimum Remaining Values. "Pick the region with the **fewest** colours still allowed next." |
| **Recursion** | A function that calls itself. `backtracking_search` calls itself to colour the next region. |

---

## Task 0 — Manual exploration

```python
{"WA": "Red", "NT": "Red", "SA": "Green", "Q": "Blue", "NSW": "Red", "V": "Green"}
```

1. **Is it valid?** No.
2. **Which pairs break a rule?** Two of them:
   - `WA` and `NT` are both **Red**
   - `SA` and `V` are both **Green**
3. **How to fix it:** change `WA` to Blue and `V` to Blue:
   `{"WA": "Blue", "NT": "Red", "SA": "Green", "Q": "Blue", "NSW": "Red", "V": "Blue"}`.
   I checked this with `is_valid_assignment()` and it returns `True`.
4. **Why detect violations early?** Once WA and NT are both Red, nothing
   we do with the other 4 regions can fix it. If we spot it straight away,
   we skip all 3⁴ = 81 complete assignments that start with that mistake.

---

## Task 1 — Understanding the representation

- **Why a list for variables?** It's just an ordered collection of names.
  The order also decides which region "simple selection" picks first.
- **Why a dictionary for domains?** So we can look up a region's allowed
  colours by its name: `domains["SA"]`. Each region could have a different
  list if we wanted.
- **What does `("WA", "NT")` mean?** WA and NT are neighbours, so they must
  be different colours.
- **How to add a new region (e.g. Tasmania `T`)?** Add `"T"` to `variables`
  (the domains dictionary is built from the list automatically), and add a
  pair to `constraints` for each neighbour. Tasmania is an island, so it
  would have **no** pairs and could be any colour.

---

## Task 2 — `is_valid_assignment(assignment)`

```python
for region_a, region_b in constraints:
    if region_a in assignment and region_b in assignment:
        if assignment[region_a] == assignment[region_b]:
            return False
return True
```

**How it works, line by line:**
1. Go through every neighbouring pair.
2. `in assignment` checks whether the region has a colour yet. We only
   compare when **both** have one, because a region with no colour can't
   break a rule. This is what makes it work for **partial** assignments.
3. If the two colours are the same, a rule is broken, so we stop and
   return `False` immediately.
4. If we got through every pair with no problem, return `True`.

**Results:** the example assignment gives `False`, and
`{"WA": "Red", "NT": "Green"}` gives `True`.

---

## Task 3 — `backtracking_search(assignment)`

```python
if len(assignment) == len(variables):
    return assignment.copy()

variable = select_unassigned_variable(assignment)

for colour in domains[variable]:
    assignment[variable] = colour
    if is_valid_assignment(assignment):
        result = backtracking_search(assignment)
        if result is not None:
            return result
    del assignment[variable]

return None
```

**How it works:**
1. **Base case:** if every region has a colour, we're done, so return a
   copy. (We copy it because we keep changing the same dictionary.)
2. Pick the next region to colour.
3. Try each colour in turn.
4. If the colouring so far breaks no rule, **call ourselves** to colour the
   next region.
5. If that deeper call found a full solution, pass it back up.
6. Otherwise `del` removes the colour. **This is the backtrack step.**
7. If no colour works, return `None` to tell the level above "go back and
   change your choice".

**Simple selection** (`select_first_unassigned`) just returns the first
region in `variables` that has no colour yet.

**Result:**
`WA=Red, NT=Green, SA=Blue, Q=Red, NSW=Green, V=Red`, and
`is_valid_assignment(solution)` returns `True`.

---

## Task 4 — Backtracking vs brute force

**Why 729?** Each of the 6 regions has 3 choices:
3 × 3 × 3 × 3 × 3 × 3 = 3⁶ = **729** complete assignments.

1. **Does brute force waste time on bad assignments?** Yes. It builds all
   6 colours before checking anything, so most of the 729 are invalid.
2. **What does backtracking do when it spots a violation?** It rejects that
   colour straight away and never goes deeper down that path.
3. **Why does this save work?** Rejecting one bad partial assignment rejects
   *every* complete assignment that starts with it in one go. For example,
   rejecting `WA=Red, NT=Red` removes 81 complete assignments.
4. **Does one solution mean we found them all?** No. Backtracking stops at
   the **first** valid solution. Other valid colourings exist, e.g. just
   swap Red and Green everywhere.

---

## Task 5 — MRV (`select_unassigned_variable`)

**Idea:** colour the "most stuck" region first, the one with the fewest
colours still allowed.

**How my code works:**
1. Skip regions that already have a colour.
2. For each remaining region, try each colour **temporarily**, check it with
   `is_valid_assignment()`, then `del` it again. Count how many were legal.
3. Remember the region with the **smallest** count.
4. I use a strict `<`, so on a tie the region that appears first in
   `variables` wins.

**Questions:**
- **Why pick a highly restricted region?** If it only has 1 colour left,
  we must use that colour anyway, so do it now. If it's going to fail, it's
  better to find out early before wasting effort on other regions.
- **What if a region has 0 legal colours?** MRV picks it straight away. The
  `for` loop in backtracking finds nothing valid and returns `None`, so we
  backtrack immediately. This is called **failing fast**.
- **Does MRV always make it faster?** No. It's a rule of thumb (a
  **heuristic**), not a guarantee. It also costs extra work, because it
  checks every colour of every region at every step.

---

## Extension 1 — Counting attempts

I added a global counter `attempts` that goes up by 1 every time
backtracking tries a colour for a region. A `USE_MRV` flag switches between
the two strategies.

| Strategy | Attempted colour assignments | Valid solution found? |
|---|---|---|
| Simple variable selection | 10 | Yes |
| MRV | 10 | Yes |

**Why 10 and not 729?** With simple selection:
- WA: Red ✓ (1 attempt)
- NT: Red ✗, Green ✓ (2)
- SA: Red ✗, Green ✗, Blue ✓ (3)
- Q: Red ✓ (1)
- NSW: Red ✗, Green ✓ (2)
- V: Red ✓ (1)

That's 1 + 2 + 3 + 1 + 2 + 1 = **10**. Every wrong colour was caught
straight away, so the search never had to go back and change an earlier
region.

**Why is MRV the same?** On this map, MRV happens to choose the regions
in the **same order** (WA → NT → SA → Q → NSW → V). At the start all regions
have 3 colours (a tie, so it picks WA). Then NT and SA tie on 2 (it picks
NT). Then SA has only 1 left, and so on. Same order means the same 10
attempts.

**Is this map big enough to show MRV's benefit?** No. It's tiny and easy.
MRV helps much more on big, tightly connected problems like Sudoku or real
timetables, where a bad early choice can cause a lot of backtracking.

---

## Extension 2 — Sudoku as a CSP (4×4 version)

- **Variables:** the 16 cells, e.g. `(row, col)` from `(0,0)` to `(3,3)`.
- **Domains:** `[1, 2, 3, 4]` for empty cells. A given clue has a domain
  of just its one number.
- **Constraints:** every **row**, every **column** and every **2×2 box**
  must contain all different numbers. Each one can be written as pairs,
  just like the map: `((0,0), (0,1))` means "these two cells must differ".

---

## Extension 3 — Timetabling as a CSP

3 lectures (`L1`, `L2`, `L3`), 2 rooms (`R1`, `R2`), 2 time slots
(`9am`, `10am`).

- **Variables:** `L1`, `L2`, `L3`
- **Domains:** each lecture gets a (room, time) pair, so there are 4 options:
  `(R1, 9am)`, `(R1, 10am)`, `(R2, 9am)`, `(R2, 10am)`
- **Constraints:**
  - **Room clash:** no two lectures can have the same (room, time) pair.
  - **Lecturer clash:** if Dr X teaches `L1` and `L2`, they need
    **different times**.
  - **Student clash:** if the same class attends `L2` and `L3`, they need
    **different times**.

This is the same idea as map colouring: regions become lectures, colours
become (room, time) slots, and "neighbours" become "lectures that can't
clash".

---

## Extension 4 — Degree heuristic

- **What it does:** pick the region that has the **most rules with
  regions that don't have a colour yet**.
- **How it differs from MRV:** MRV looks at how many **colours** are left.
  Degree looks at how many **neighbours** are affected.
- **Most neighbours here:** **SA**, with 5.
- **Useful as a tie-breaker?** Yes. At the very start, every region has 3
  colours, so MRV is tied. The degree heuristic would pick SA first,
  because colouring the most connected region first narrows down the most
  other regions.

---

## Extension 5 — Constraint graph

```
        WA ───── NT
          \     / \
           \   /   \
            SA ──── Q
           /  \     |
          /    \    |
         V ──── NSW ┘
```

(Each line is one of the 9 pairs. SA is joined to WA, NT, Q, NSW and V.)

| Region | Number of neighbours |
|---|---|
| SA | 5 |
| NT | 3 |
| Q | 3 |
| NSW | 3 |
| WA | 2 |
| V | 2 |

1. **Most neighbours:** SA (5).
2. **Fewest:** WA and V (2 each).
3. **How the graph helps:** every line is one rule ("these two must be
   different"), so you can see all the rules at once.
4. **How it supports heuristics:** the number of lines touching a region
   **is** its degree, which is exactly what the degree heuristic uses. A
   region with many lines will usually lose colour options fastest, which
   is what MRV notices.

---

## Reflection questions

1. **What causes backtracking?** Reaching a region where **no colour is
   allowed**, because of earlier choices. The search has to undo the last
   choice and try a different one. Example: WA, NT and SA all touch each
   other (a triangle). If we only had **2** colours, then after colouring WA
   and NT, SA would have no colour left, so the search would have to go back
   and change NT. With 3 colours on our map, every wrong colour was caught
   straight away (Red ✗ → try Green), so it never had to undo an earlier
   region.
2. **Why can it be faster than brute force?** It checks rules **as it
   goes**, so it throws away a bad start before building the rest. In our
   map it found a solution after **10** attempts instead of looking through
   up to **729** complete assignments.
3. **Why might MRV reduce search?** It handles the hardest region first,
   so dead ends show up early when they're cheap to fix. On our small map it
   made no difference (10 vs 10), because it picked the same order.
4. **How could this model represent timetabling?** Lectures are the
   variables, (room, time) slots are the domain, and "can't clash" rules are
   the constraints (see Extension 3). The same `is_valid_assignment` and
   `backtracking_search` code would work. Only the data changes.

---

## Demo cheat sheet (say it in your own words)

- **Constraint check:** "Loop through neighbour pairs. Only compare if both
  are coloured. Same colour means invalid."
- **Backtracking:** "Pick a region, try each colour, go deeper if it's
  valid, undo it if it's not. Return `None` if nothing works."
- **MRV:** "Count the legal colours for each uncoloured region and pick the
  smallest. Ties go to the first one."
- **729:** "3 colours to the power of 6 regions."
- **10 attempts:** "Every wrong colour is rejected straight away, so it
  never builds a full bad map."
