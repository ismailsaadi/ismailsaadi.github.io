---
name: update-works
description: Refresh the publication list in works.html from Scopus. Use when the user asks to update their papers/publications, sync works.html with Scopus, add newly published articles, or refresh the research page. Pulls journal articles + reviews (no conference papers) for Ismaïl Saadi via the Scopus MCP, rewrites the year-grouped list in works.html, then commits and pushes.
allowed-tools: Read, Edit, Write, Bash, mcp__scopus-assistant__search_scopus, mcp__scopus-assistant__get_abstract_details, mcp__scopus-assistant__get_quota_status
---

# Update works.html from Scopus

Rebuild the "Refereed journal articles" list in `works.html` from Scopus, then commit and push.

Author: **Ismaïl Saadi** — Scopus Author ID `56906820200`.

Only **journal articles and reviews**. **Exclude conference papers** (and any non-journal type).

## Step 1 — Check quota

Call `get_quota_status` first. The full run needs ~3 search calls plus one `get_abstract_details` call per paper (~30 calls). If remaining quota is too low, tell the user and stop before making partial changes.

## Step 2 — Fetch the full publication list

`search_scopus` caps `count` at 25 and has **no offset parameter**, so split the query by publication year to page through everything. Use these queries (sort `coverDate`, count 25):

- `AU-ID(56906820200) AND NOT DOCTYPE(cp) AND PUBYEAR > 2018`
- `AU-ID(56906820200) AND NOT DOCTYPE(cp) AND PUBYEAR < 2019`

If either query returns exactly 25 rows, it may be truncated — split that range further (e.g. add `PUBYEAR > 2021` / `PUBYEAR < 2022`) until each query returns fewer than 25 rows.

Use `NOT DOCTYPE(cp)` (exclude conference papers) rather than `DOCTYPE(ar)`: the list must keep review articles (`re`, e.g. the *Current Environmental Health Reports* paper) and data papers (`da`, e.g. the *Data in Brief* paper), which an article-only filter drops.

Keep only rows where `aggregation_type` is `Journal` (this drops any stray book chapters, editorials, errata, etc.).

## Step 3 — Deduplicate

Scopus sometimes returns the same paper twice with different `scopus_id`/`doi` (e.g. an "article in press" record and the final record, or two DOIs for one IJBIDM paper). Deduplicate by normalized title (lowercase, trimmed). When duplicates exist, prefer the record that has a volume/issue/page range over a bare "in press" record. If both are final, keep the one whose DOI already appears in the current `works.html`.

## Step 4 — Get full metadata per paper

The search response only includes the first author. For every deduped paper, call `get_abstract_details(scopus_id)` to obtain:

- full author list, in order (`authors[].surname` + `initials`)
- official source (journal) title with correct punctuation (`publication_name`)
- publication year (`cover_date`)

**Limitation:** `get_abstract_details` does **not** return volume/issue/page numbers. So:
- For papers already in `works.html`, reuse the existing volume/issue/pages/art.-no. text — don't drop it.
- For a brand-new paper, an Elsevier-style article number can be read from the DOI suffix (e.g. `10.1016/j.cities.2026.107026` → `art. no. 107026`); the volume number is usually unavailable from the MCP. Do **not** invent a volume — list `art. no. XXXXX` without a volume, or treat the paper as **in press** if it has no locator and a current/future cover date. If the user can supply the volume/pages, ask or let them fill it in.

Treat a paper with no volume/pages and a current/future cover date as **in press**.

## Step 5 — Format each entry

Match the exact existing markup. One `<p>` per paper:

```html
<p style="font-size:18px;">AUTHORS (YEAR). <a href="https://doi.org/DOI">TITLE</a>. <i>JOURNAL</i>, VOLUME(ISSUE), pp. START-END.</p>
```

Formatting rules (follow the existing entries in `works.html` exactly):

- **Authors**: `Lastname F.S., Lastname F., ...` (initials with periods, no space between initials). Always render the target author as `<strong>Saadi I.</strong>`. Preserve Scopus author order.
- **Year**: `(2025)`. For in-press papers use `(in press)` instead of the year.
- **Title**: wrapped in the DOI `<a>`. Keep Scopus capitalization. Escape `&` as `&amp;`.
- **Journal**: italic, with proper punctuation/full name (e.g. `Computers, Environment and Urban Systems`, `Sustainability`, `Measurement: Journal of the International Measurement Confederation`) — not the punctuation-stripped `publication_name` from search. Take it from `get_abstract_details`.
- **Locator** (after the journal):
  - article-number papers: `, VOLUME, art. no. XXXXX.`
  - page-range papers: `, VOLUME(ISSUE), pp. START-END.`
  - in-press papers: end after the journal with just `.` (no volume/pages).
- Use the DOI URL `https://doi.org/<doi>` for the link.

When unsure how a field should look, open `works.html` and copy the style of an adjacent entry.

## Step 6 — Rewrite the list in works.html

Edit only the publication list inside `<div class="col-md-12">` (the block under `<h3>Refereed journal articles</h3>`). Leave the rest of the file untouched.

- Group papers under `<h4>YEAR</h4>` headers, **years descending**.
- In-press papers (current or next year, no volume/pages) go in a leading `<h4>YEAR</h4>` block for their cover year with `(in press)`.
- Within a year, keep a stable order (Scopus `coverDate` descending is fine, matching the existing file).
- Preserve the surrounding indentation and the `<h3>`/`<h4>` structure already in the file.

Before overwriting, diff against the current entries: only existing DOIs should disappear if Scopus no longer lists them. Call out any paper that was in `works.html` but is missing from Scopus rather than silently dropping it.

## Step 7 — Commit and push

```bash
git add works.html
git commit -m "update works.html publications from Scopus"
git push
```

Co-author the commit per the repo convention. Report what changed (papers added/updated) in the final summary.
