# Publii drafts

A staging area for Hera project-update posts. Draft them here against the current state of
`CHANGELOG.md` / `docs/status.md`, then copy the finished block straight into Publii's post
editor — Title into the title field, Slug into the URL field, Tags into the tags field, Excerpt
into the excerpt field, and Body into the main editor as Markdown.

## How to use this file

- Add a new `##` section per post, in the shape the template below uses.
- Once a post is live in Publii, mark its heading `[posted]` and either keep it here as an
  archive or delete it.
- Source facts from `CHANGELOG.md` and `docs/status.md` — this file is the *public-facing*
  rewrite of that material, not a place to invent new claims. Keep code-level detail out; a
  reader here doesn't need the package name, just what changed and why it matters to them.

## Template

```
**Title:**
**Slug:**
**Tags:**
**Excerpt:** (1–2 sentences, shown in post listings)

**Body:**
(Markdown, blog tone — this is the one place in the docs where prose is the point)
```

---

## v0.2.0 — she remembers now

**Title:** Hera v0.2.0: she remembers now

**Slug:** hera-v0-2-0-she-remembers-now

**Tags:** hera, release, v0.2.0

**Excerpt:** v0.1 proved a message could reach a model and come back — but Hera forgot everything
the moment the tab closed. v0.2.0 is the version where that stops being true.

**Body:**

v0.1.0 was the spine: a message typed into the browser reaching a model and coming back as a
live answer. It worked end to end, and it forgot everything the moment you closed the tab.

v0.2.0 is the deepening pass — the four things that let Hera accumulate instead of just
answering:

- **Projects.** Chats can now be filed under a project with its own instructions, its own pinned
  skills, its own default profile. A project is the work; Settings stays *how she works*.
- **A scratchpad.** Between turns of one conversation, Hera now has somewhere to put a plan, an
  intermediate result, a running list of what's already been checked — so she can pick up where
  she left off instead of re-deriving it from scratch every time.
- **Artifacts.** She can hand you a real file now — a page, a chart, a document, a small program —
  instead of a code fence you copy out by hand. Ask her to change one colour on a 40 KB page, and
  she edits it in place rather than rewriting the whole thing.
- **Memory.** The one people actually asked for. What she remembers about you lives as plain
  markdown files you can open, edit, or take somewhere else entirely. Nothing is hidden behind a
  retrieval step you can't see — if a memory is switched on, it's in her prompt, every time,
  in full.

None of these were built to look impressive. They were built because a chat assistant that
forgets everything between sessions caps out fast, no matter how good any single answer is.

Two things originally planned for this release moved to the next one — deliberately, not
dropped: a broader redesign pass, and "dreaming" (Hera proposing changes to her own behaviour
based on how conversations actually went). Both get their own post when they land.

---

## v0.2.1 — no more mood rings

**Title:** Why Hera stopped having moods

**Slug:** hera-v0-2-1-no-more-mood-rings

**Tags:** hera, release, v0.2.1, design-notes

**Excerpt:** An early build of Hera could show you how she felt about her own answer — curious,
warm, unsure. It sounded like a good idea. It wasn't, and here's what replaced it.

**Body:**

For a while, Hera could tag her own answers with a small emotional read-out — a little card
showing *curious*, or *warm*, or one of a dozen others, drawn next to whatever she'd just said.
It was meant to make her feel less like a black box.

Run against a real model for long enough, it didn't hold up. She'd reach for a stance rarely, and
mostly at random: *curious* attached to an answer that read as completely confident, *warm*
attached to what was really just a correction. Several of the fourteen moods would fire on the
same reply. Shortening the list wouldn't have fixed that — the problem wasn't which fourteen
words were on it.

So the moods are gone, and nothing replaces them. If Hera thinks something in her own answer is
wrong, she says so, in the sentence itself — *I think this is wrong, and here's why* carries the
same information a mood card did, in the place you're already reading.

One piece of that machinery was worth keeping on its own: the ability for Hera to actually stop
and ask you something mid-task, rather than guessing. That's now its own tool, standing on its
own, with exactly three reasons she's allowed to use it — she's unsure, she's genuinely stuck, or
there's a real choice only you can make. No mood attached, no colour-coded guesswork. Just a
question, and a field to answer it in.

It's a small removal by line count and a real one by intent: this project would rather ship
nothing than ship a feature that only looks honest.

---

## Ideas for future entries

Short prompts to turn into full posts once the underlying work lands — delete an item once it's
drafted above.

- **v0.3.0, when it ships:** dreaming (Hera proposing changes to her own mind from how
  conversations went), the sandbox, and `hera-code` — a coding-agent sibling built on the same
  packages.
- **"Why everything lives in one folder on your machine"** — a `~/.hera` explainer post: no
  cloud account, no vendor lock-in, every memory and every skill is a file you already own.
- **"Talking to a model that runs on your own hardware"** — what changes about building an
  assistant when the model is local (Qwen3.6-35B) rather than a hosted API.
