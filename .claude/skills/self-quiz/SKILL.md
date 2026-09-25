---
name: self-quiz
description: Build a reusable self-quiz Artifact (random draw from a question pool, category progress rack, skip/previous/jump navigation, progress saved on refresh) for notes on any topic in this repo. Use when the user asks for a quiz, self-test, or flashcard-style Artifact based on notes they're studying or a source they just read/watched.
---

# Self-quiz artifact

This skill reproduces the quiz design built for `networking/osi/osi-model.md` (the
OSI Stack Quiz). It is a *pattern*, not a one-size-fits-all generator — the payoff is
a proven interaction design and a working file to adapt, not a templating engine.

Each topic in this repo gets its own folder (e.g. `networking/osi/`) holding its
notes file and an `assets/` subfolder for any images. Put a new quiz's supporting
files the same way if it ever needs any.

## When to use it

The user wants a self-test / quiz Artifact over notes they took (or are about to
take) on some topic — a video, an article, a book chapter, a course. If notes don't
exist yet for the topic, write them first (this repo's normal note-taking flow),
then build the quiz from those notes.

## What makes this design work (keep these, they're the point)

- **A pool bigger than a single run.** Write 25–30 questions total, grouped into the
  topic's natural categories (e.g. OSI's 7 layers) plus a "general/cross-cutting"
  bucket for questions that don't belong to one category. Each run randomly draws a
  smaller subset (10 worked well) so the quiz survives being retaken later. Guarantee
  at least one question per category in every draw so the sample stays representative;
  fill remaining slots randomly from the rest of the pool. Shuffle each question's
  answer order per draw too — don't let position become a memorized shortcut.
- **A category "rack".** A persistent sidebar (desktop) / chip strip (mobile) listing
  the topic's categories, each lighting up correct/wrong/skipped as the user answers
  questions tagged with it. This is the single biggest thing that makes the quiz feel
  purpose-built instead of generic — spend real design effort naming and visualizing
  the categories in the subject's own vocabulary (OSI's rack used "L1..L7" boxes; a
  different subject needs its own equivalent, not a renamed copy).
- **Free navigation, not a forced march.** Previous / Skip / a row of jump-to-question
  dots. Skipping *defers* a question (still answerable later); answering *locks* it
  into a read-only review state with the explanation shown. Don't force linear
  progress — people re-check earlier answers.
- **Progress survives a refresh, per-browser only.** Persist the current draw, answers,
  and position to `localStorage` (wrapped in try/catch — it can throw or come back
  empty in some contexts) after every state change, and restore on load if a valid
  session is found. Say explicitly to the user that this is local to their browser/
  device, not synced anywhere — don't imply more durability than it has.
- **Plain, concrete phrasing.** Favor questions grounded in a recognizable scenario
  ("your cable's plugged in and the light's on, but you still can't load a page — what
  next?") over abstract ordering/logic puzzles. If the user says questions read as
  confusing, that's a phrasing problem to fix, not a knowledge gap on their end —
  rewrite for plainer language before adding difficulty.
- **A tiered result screen**, not just a percentage — a few named tiers grounded in the
  subject's own vocabulary (the OSI quiz used "Zero Packet Loss" / "Full Duplex" /
  "Some Retransmits" / "Link Down"). Also show a per-category breakdown so the user
  knows exactly what to re-read.
- **A subject-specific palette and type pairing**, chosen fresh each time — don't
  reuse OSI's copper/fiber-teal colors or JetBrains Mono/Public Sans pairing verbatim
  for an unrelated subject. Load the `artifact-design` skill and follow its process
  (color/type/layout plan grounded in the new subject) before touching the template.
  Both light and dark themes must work — the template's token structure
  (`:root`, the `prefers-color-scheme` block, `:root[data-theme="dark"]`) is reusable
  as-is; only the token *values* and font links should change per subject.

## How to build one

1. **Confirm or write the notes first.** The quiz should test the same notes file the
   user will study from — read it before writing questions, don't invent facts.
2. **List the topic's categories.** These become the rack. For OSI it was the 7
   layers; for a different subject it might be eras, components, phases, or chapters.
   Keep it to a number that fits a sidebar/chip strip (roughly 4–10).
3. **Write the question pool** (25–30 items): 3–4 per category plus 6–8
   general/cross-cutting questions. Each item needs: `category` (or `"general"`),
   `q`, 3–4 `options`, the `correct` index, and a one-sentence `exp` (shown as
   feedback — teach on both right and wrong answers).
4. **Load `artifact-design`**, sketch the color/type/layout plan grounded in the new
   subject, then copy `template.html` from this skill folder and adapt:
   - Rename `LAYERS` → the new categories (id + display name).
   - Replace `POOL` with the new question set.
   - Replace the `:root` palette tokens and the Google Fonts `<link>` per the design
     plan (keep the light/dark/`data-theme` structure intact).
   - Update the header copy, eyebrow text, tier names/messages, and footer credit
     link to the new subject and its notes file.
   - Update the favicon/title/description when publishing via the Artifact tool.
5. **Sanity-check before publishing**: pool size ≥ ~25, every category has ≥3
   questions, `QUIZ_LENGTH` (draw size) is sensible for the pool (10 out of 25–30
   worked well — much larger pools can raise it), and light/dark both render legibly.
6. **Publish with the Artifact tool** and link it back from the topic's notes file,
   same as `networking/osi/osi-model.md` does.

## Reference implementation

`template.html` in this skill folder is the actual, working OSI Stack Quiz — read it
to see the pattern implemented end-to-end (session draw/shuffle logic, rack + dot
navigation, localStorage persistence, results tiering). Copy it, don't rewrite from
scratch.
