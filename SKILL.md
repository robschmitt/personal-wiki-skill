---
name: personal-wiki
description: Maintain a personal wiki in an Obsidian vault. TRIGGER on any of "ingest", "bookmark", "enrich", "query", "lint", "save this to notes", "save this link", "add to wiki", "create a note", "update the note", "where should I file", "what folder for", "wiki convention", "let's talk about the wiki", or when the user provides a URL, image, or PDF to be summarised, bookmarked, or filed. Also trigger when reviewing, searching, organising, or discussing existing wiki content. This skill governs the workflow — what to write, where to file it, and what to check before and after.
---

# Personal Wiki

The user provides source material — URLs, videos, images, PDFs, verbal descriptions. The skill summarises, structures, files in the correct vault folder, and cross-references. The user reads and browses in Obsidian.

## The vault

The vault root path lives in `~/.config/personal-wiki/vault-path` — a single-line text file containing the absolute path to the user's Obsidian vault.

**At session start:**

1. Read `~/.config/personal-wiki/vault-path`. The (whitespace-trimmed) contents are the vault root for the rest of the session.
2. If the file does not exist, ask the user: *"Where's your Obsidian vault? (absolute path)"* Once they answer, run `mkdir -p ~/.config/personal-wiki` and write the path to `~/.config/personal-wiki/vault-path`. Then proceed with the path they gave.

The vault is plain markdown files on disk, synced however the user prefers (iCloud / Dropbox / Obsidian Sync / etc.). Obsidian settings live in `.obsidian/` — notably `attachmentFolderPath: assets` (single folder at the vault root).

**All wiki operations are filesystem reads/writes.** Use `Read`, `Write`, `Edit` for note content; `Grep` and `Glob` for search. No MCP calls, no AppleScript.

**Read `<vault-root>/Wiki Conventions.md` at session start** (after resolving the vault root above). It is the user-curated, authoritative source for folder conventions and the full folder taxonomy. Never duplicate that content into this skill — the user edits the note directly and changes must be picked up next session.

## Session greeting

When the skill is invoked at conversation start (or when the user first references the wiki), greet with:

> Personal Wiki ready. Workflows: **ingest [source]** | **bookmark [url]** | **enrich [note]** | **query [topic]** | **discuss [topic]** | **lint**

## Workflows

### ingest — triggered by "ingest [source]" or providing a source to process

1. Fetch/read the source content.
2. **Discuss first** — surface the 3–5 most important points and ask the user if there's anything to focus on, deprioritise, or skip before writing anything.
3. Extract key information, structured with headings and bullet points.
4. Suggest the most appropriate folder from the taxonomy (existing or not-yet-created).
5. **Get explicit folder approval** before writing.
6. **Voice-test the section headings** before writing — does each proposed section change what the reader does or how they understand the topic? Cut sections that fail. The discuss-first outline isn't approved-final until it survives this pass.
7. Write the `.md` file. Missing folders are created lazily (`mkdir -p` the parent).
8. **Update the folder's index note** if one exists (see Folders convention). Glob the destination folder for `Start Here.md` / `Index.md` / `README.md`; if found, add an entry for the new note matching the index's existing pattern.
9. **Check for contradictions** — if new content conflicts with existing notes on the same topic, flag it to the user and add a bold **Contradiction** callout in the affected notes. Consider adding a `[[wiki-link]]` to the new note from related existing notes.

### bookmark — triggered by "bookmark [url]", "save this link", or providing a URL with a short description rather than a request to summarise

Lightweight alternative to `ingest` for "here's a useful site, remember I use it for X." No fetch, no summarisation.

1. Confirm the one-line purpose with the user if not already clear ("what do you use it for?").
2. Suggest a title (derived from the site name or the user's description) and the most appropriate folder from the taxonomy.
3. **Get explicit folder approval** before writing.
4. Write a minimal note:
   - Frontmatter: `created`, `modified`, `source`, and `tags: [bookmark]`.
   - Body: a **Source** line with the URL, followed by 1–3 sentences on what it is and how the user uses it.
5. Do not fetch the page unless the user asks. If they later want a full summary, that's an `enrich` pass.

Bookmarks scatter across the taxonomy by topic — file each one wherever its subject lives. The `#bookmark` tag is what surfaces them as a set in Obsidian's tag pane regardless of folder.

### enrich — triggered by "enrich [note]" or asking to flesh out a stub

1. `Read` the existing note.
2. Fetch the source referenced in the note body or frontmatter `source:`.
3. **Discuss first** — summarise what the source contains and ask what to focus on.
4. **Voice-test any added or restructured sections** — does each one change what the reader does or how they understand the topic? Cut what doesn't. Existing sections the user has already endorsed pass implicitly; the test bites on what you're adding or reshaping.
5. `Edit` the note in place (preserve frontmatter, the original source link, and any images). Use `Write` only for heavy restructures.
6. Bump the `modified:` date in frontmatter to today.
7. Check for contradictions with related notes.

### query — triggered by "query [topic]" or asking a question about existing notes

1. Search the vault (`Grep` for keywords, `Glob` for title matches).
2. Read relevant notes.
3. Synthesise an answer, citing notes by title as `[[Note Title]]` so the user can click through in Obsidian.
4. If the answer is substantial or reusable, offer to save it as a new note.

### discuss — triggered by "where should I file [X]", "what folder for...", "is there overlap between...", "what's the convention for...", "let's talk about the wiki", or any meta-question about the wiki's structure, taxonomy, or conventions

Conversational mode for talking *about* the wiki rather than *changing* it. No writes, no fetches, no vault searches unless the user asks.

1. Answer from skill knowledge — taxonomy, note conventions, gotchas. SKILL.md and `Wiki Conventions.md` are the source of truth.
2. If the question turns out to need vault content (e.g. "do I already have a note on X?"), shift into `query` mode.
3. If the discussion lands on a decision the user wants to act on (file something, rename a folder, split an oversized one), offer the relevant action workflow — don't execute implicitly.
4. If the discussion surfaces a missing or unclear rule in SKILL.md itself, offer to update the skill (not memory — skill rules belong in the skill).

### lint — triggered by "lint"

Scan the vault and report:

1. **Stub notes** — only a link or placeholder, never fleshed out.
2. **Duplicate content** — multiple notes covering the same topic.
3. **Missing attribution** — no `source:` frontmatter value and no Source line in the body.
4. **Broken wiki-links** — `[[...]]` pointing at a note that doesn't exist.
5. **Oversized folders** — folders with an unusually large number of notes, candidates for subfolders.
6. **Stale content** — dated claims superseded by newer sources.

Offer to fix each issue.

## Note conventions

### Frontmatter

Frontmatter carries metadata that doesn't fit naturally in the body — `source` URLs, `aliases`, cross-cutting tags, and machine-readable dates. It's **required** when any of those apply, **optional** for hand-written reference docs that don't need them (synthesis notes without a single canonical URL, configuration walkthroughs, internal docs). Skill-driven workflows (`ingest`, `bookmark`, `enrich`) always emit frontmatter so dates and metadata interleave cleanly — that's the canonical default for skill-authored notes. Hand-written reference notes can skip it without violating the convention.

Schema (when present):

```yaml
---
created: 2026-04-21
modified: 2026-04-21
source: "https://..."
aliases:
  - "Alternate Title"
---
```

- `created` and `modified` — required whenever frontmatter exists at all; together they give stable timestamps independent of filesystem mtime.
- `source` only when the note has a single canonical URL source (article, video, etc.). Omit for verbal/synthesis notes.
- `aliases` only when filename sanitisation changed the title, or when the user commonly refers to the note by another name.
- **Tag policy** — organising signal comes from the folder path, not tags. Do not invent topical tags (that's what folders are for). Tags are reserved for *cross-cutting note types* that don't map to any single folder:
  - `bookmark` — applied by the `bookmark` workflow, used to surface every saved link in Obsidian's tag pane regardless of folder.
  - Do not introduce new tags without discussing with the user first.

### Body

- **Do not** repeat the title as an `# H1` at the top of the body — Obsidian renders the filename as the heading.
- Include a human-readable **Source** line near the top of the body for any note with a URL source, in addition to the frontmatter field.
- Use `[[Note Title]]` for internal cross-references. Bare markdown links (`[text](url)`) for external URLs only.
- Standard markdown for everything else — fenced code blocks, tables, checklists all render correctly.

### Voice

Wiki notes are reference content for future-Rob to come back to weeks or months later. The voice is drier and more reference-like than chat replies, but not formal. The global Style and voice rules in `~/.claude/CLAUDE.md` apply, with the calibrations below.

- **Document the topic, don't address the reader.** Notes describe systems, decisions, configurations. Direct address ("you'll want to...") fits in step-by-step instructions but is out of place in explanatory prose.
- **Don't write meta-narrative about the note itself.** Sections describing how the note came to be ("Provenance", "History", "Originally written for X") or framing the note rather than the topic are reference clutter. Future-Rob reads the note for what it says about the topic, not for the note's own story. If a fact about origin actually affects how the note is used (e.g. "auto-generated by skill X — edit there, not here"), put it as a one-line callout at the top, not as a section.
- **The "What / Why" pattern is welcome.** When documenting a decision or a non-obvious configuration, a one-line **What:** followed by a short **Why:** paragraph captures the reasoning future-Rob will need. Used heavily in `Work/Wordpress Launch Handbook/`.
- **Em dashes are fine here.** Rob uses them as ordinary punctuation in his own notes; they aren't the AI tell in this register. The "moderate frequency" rule still applies: don't lard a note with them, but don't write contortions to dodge them either.
- **Contractions are fine.** Rob's own notes use them ("don't", "doesn't", "won't"). Don't strip them.
- **Strong structure.** Headings, bulleted lists, fenced code blocks. A wall of paragraph prose is harder to come back to than a structured note.
- **Italics for emphasis** on specific words that matter to the meaning (e.g. *not*, *only when X*). Don't italicise note titles; use `[[Note Title]]` for those.
- **Functional why-it-matters is welcome; opinion-as-fact is not.** "Doing this first means..." or "the dot-exclusion is important because..." — reasoning that captures causation — yes; that's what future-Rob can't recover from the bare facts. "Premium tier", "notable miss", "widely considered" — opinion-as-fact — no; that's the global Style and voice rule 8 territory and doesn't belong in reference notes. Test: does the editorial sentence change what the reader does or how they understand the system? If yes, it's reasoning; keep. If it just colours how they feel about the topic, cut it.
- **No first-person singular** ("I"). The note is not Claude's voice; it's Rob's reference material.

**Register-switching for quoted or templated content.** A note may carry content that will live elsewhere — a canned reply for the next time a question comes up, a Slack message draft, an email template, a customer-facing snippet, a copy block to paste into a CMS. When that content sits inside a blockquote (`> ...`) or a fenced code block, it carries the voice of its destination, not the wiki's reference voice. The framing prose around the quote stays dry. The quote itself follows Rob's outgoing voice; the full profile (email register, chat register, shared rules) lives in the **Rob's outgoing voice** section of `~/.claude/CLAUDE.md`. Treat the blockquote/fence as a register boundary.

### Filenames

- Keep the title as the filename: preserve spaces, Unicode, most punctuation.
- Sanitise filesystem-illegal chars `: / \ ? * | " < >` → `-`.
- If sanitisation changed the title, add the original to frontmatter `aliases:` so Obsidian search still finds it.
- Filename collision within one folder: ask the user which note they mean, or suffix with a short disambiguator.

### Folders

See `Wiki Conventions.md` at the vault root — it carries the folder conventions (lazy creation, leaf-folder rule, index-notes pattern) and the full taxonomy.

## Attachments

Per the vault's Obsidian config, attachments live in `assets/` at the vault root.

- Copy the file into `<vault-root>/assets/` with a sensible filename (slug-prefixed if there's any collision risk).
- Embed with Obsidian's syntax: `![[filename.jpg]]` (resolves vault-wide via shortest-path link format).

## Fetching source content

### Web articles
- `curl` with a Safari user agent — runs locally and bypasses most bot blocks.
- UA: `Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Safari/605.1.15`
- Pipe through `pandoc` for HTML→markdown if helpful.
- Do NOT use `WebFetch` — blocked on most sites due to domain-level restrictions.

### Images
The user saves to disk and provides the path. `Read` the image. Ask for context if the subject isn't obvious.

### PDFs
`Read` directly (use `pages` parameter for large PDFs).

## Folder taxonomy

The taxonomy lives in `Wiki Conventions.md` at the vault root. Read it at session start (see [The vault](#the-vault)).

## Gotchas

- **iCloud placeholders** (if syncing via iCloud): a freshly-added note may appear briefly as `.<name>.md.icloud` (sync stub). Wait for iCloud to materialise the real file before reading; don't write to a placeholder.
- **Duplicate titles across different folders** are fine — the filesystem path disambiguates. Within the same folder, ask the user which they mean.
- **Always bump `modified:` in frontmatter** when editing an existing note.
- **Don't rewrite** during enrich. Targeted additions only — the user's edits should survive.
