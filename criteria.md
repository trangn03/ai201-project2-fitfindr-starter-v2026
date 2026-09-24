# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**

`search_listings` scores by plain keyword overlap with `description`, not
synonyms or fuzzy matching. A phrasing that doesn't share a token with any
listing's title/description/style_tags scores zero everywhere and the query
looks "impossible" even though a matching item exists. 5 of 5 would require
the search itself to be smarter than a keyword filter — 4 of 5 accepts that
one in five phrasings can miss on wording alone, not on a loop bug.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**

This path doesn't depend on wording, the model, or a score threshold — it's a
single deterministic branch on `len(search_results) == 0`, checked once per
run in `agent.py::run_agent`. There's no fuzziness for it to fail on the way
criterion 1's keyword match can. If this isn't 5 of 5, the branch itself is
missing or wrong, not just unlucky, so anything less than 5 of 5 should be
treated as a bug, not a target to relax.

---

## 3. Something about state

Given a matching query, `session["selected_item"]["id"]` equals the `id` of
the `new_item` dict actually passed into `suggest_outfit` — 5 of 5 tries.

**Why this target:**

Nothing is passed tool-to-tool as a bare variable; everything routes through
`session`. If `search_listings` picks one listing but a stale or
wrong-index item reaches `suggest_outfit`, the fit card will still come back
as a plausible-looking string — the failure hides behind a working-looking
tool call instead of a crash. This is a single dict-copy step with no model
call and no scoring involved, so there's no reason it should ever miss; 5 of
5 is the floor, not a stretch goal.

---

## 4. Something about the fit card

For 5 different matching items, each fit card mentions that item's price and
platform at least once and is 2–4 sentences long — 5 of 5 items.

**Why this target:**

The card's wording is allowed to vary run to run — that's `generate()` doing
its job, not a defect. What isn't allowed to vary is whether the card is
usable as a real post: a caption that drops the price or platform reads like
it forgot what it's describing, and one outside 2–4 sentences reads like
either a fragment or a product blurb, not a caption someone would post. This
checks content and shape, not phrasing, so it stays true regardless of which
exact words the model picks.

---

## 5. Your choice

Given a query with `max_price` set, none of the listings `search_listings`
returns have a `price` above `max_price` — 5 of 5 tries, checked across
queries whose ceiling actually excludes at least one listing in the data.

**Why this target:**

`search_listings` never calls the model — the price filter is a plain
numeric comparison over `data/listings.json`, so nothing about it should be
allowed to be probabilistic. This is also the criterion from the
"single-quote your queries" warning in CLAUDE.md: a PowerShell query like
`"under $30"` silently drops the `$30` and the ceiling stops being enforced
with no error. Checking it at 5 of 5 catches that class of bug directly,
instead of it showing up later as an outfit suggestion for an item that was
never supposed to pass the filter.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
