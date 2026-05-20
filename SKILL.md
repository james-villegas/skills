---
name: codebase-wiki
description: >
  A lightweight knowledge base skill for software projects. Use this skill whenever the user wants
  to document, discover, or curate knowledge about their codebase — including feature descriptions,
  business rules, domain logic, or architectural decisions. Trigger when the user says things like
  "document this feature", "generate a wiki entry", "update the knowledge base", "what do we know
  about X", "scan these files and write it up", or "curate the wiki". The wiki lives in
  .claude/wiki/ inside the project. Always use this skill before writing any wiki entries or
  reading from the knowledge base.
---

# codebase-wiki

A lightweight, agent-curated knowledge base that lives in `.claude/wiki/` inside any project
already using Claude Code. The agent reads source files or documents, generates structured wiki
articles, and presents them for human review before committing anything.

---

## Directory layout

```
.claude/
  raw/                 ← unstructured files read as sources; not written to
  wiki/
    INDEX.md           ← table of contents (auto-maintained)
    auth.md
    survey-builder.md
    payment-flow.md
    ...
```

All articles are `.md` files with YAML frontmatter. `INDEX.md` is plain markdown, regenerated after every write session.

---

## Article schema

```markdown
---
name: "Feature or concept name"
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
tags: [auth, backend, api]        # freeform; reuse existing tags when possible
links:
  - "[[other-article]]"           # wikilink-style; use the filename stem
  - "[[another-article]]"
summary: "One or two sentence plain-English summary. What it is, what it does."
sources:
  - path/to/file.ts
  - path/to/other.java
---

## Overview
...

## Business rules
- Rule 1
- Rule 2

## Key behaviors
...

## Open questions
- Anything uncertain or worth verifying with a human
```

---

## Workflow

### Step 1 — Setup check

Before doing anything else:

1. Check whether `.claude/wiki/` exists in the project root. If not, create it and create a blank
   `INDEX.md`.
2. List existing articles so you're aware of current tags and topic coverage.

### Step 2 — Read sources

The human will point you at one or more of:
- Source files (`.ts`, `.py`, `.java`, etc.)
- Test files (often encode business rules more explicitly than implementation)
- Existing documentation (`.md`, `.txt`, `.pdf`)
- API specs or config files
- Jira tickets (use a suitable Jira tool if available, when given a ticket key or URL)

Jira tickets are valuable for capturing the *why* behind a feature — motivation, decisions, 
and historical context that isn't visible in the code. When a ticket is referenced,
fetch it and extract any business context or decisions recorded there.

Read all provided sources. Extract:
- What this code/feature *does* (behavior, not implementation detail)
- Business rules: conditions, constraints, validations, access control
- Domain vocabulary: named concepts, states, enums
- Relationships to other features (look for imports, references, foreign keys, shared types)

### Step 3 — Plan articles

Before writing, decide on article boundaries:

**Split into separate articles when:**
- A section would exceed ~40 lines of body content
- The topic is referenced by multiple other features (candidate for its own node)
- The section covers a clearly distinct domain concept (e.g. "session management" inside an auth
  file deserves its own article)
- Business rules in one area would be confusing without understanding the other area separately

**Keep as one article when:**
- Content is tightly coupled and only makes sense together
- Total body is under ~30 lines
- The concept doesn't stand alone

Tell the human your planned article breakdown *before* writing, e.g.:
> "I plan to generate 3 articles: `auth-tokens`, `session-management`, and `permission-roles`.
> Does that breakdown make sense, or would you like to adjust?"

Wait for confirmation if the breakdown is non-obvious. For straightforward single-article cases
you can proceed without asking.

### Step 4 — Generate articles

Write each article following the schema above. The YAML frontmatter holds metadata; the markdown body holds the content. Guidelines:

- **`name`**: Human-readable title, not a slug. E.g. `"Authentication Tokens"` not `"auth_tokens"`.
- **`tags`**: 2–5 tags. Check existing tags in the wiki first; prefer reusing over inventing.
- **`links`**: Add wikilinks (`[[filename-stem]]`) wherever you reference a concept that has or
  should have its own article. If the linked article doesn't exist yet, still include the link —
  it signals a gap.
- **`summary`**: Non-technical one-liner in the frontmatter. What would a new team member need to know in 10 seconds?
- **Body sections** — use the structure below, omitting sections that aren't applicable:
    - `## Overview` — what this feature is and why it exists
    - `## Business rules` — explicit constraints, validations, conditions (bullet list)
    - `## Key behaviors` — how it works in practice; state transitions, flows, edge cases
    - `## Integrations` — what other systems/features it depends on or affects
    - `## Open questions` — anything uncertain, underdocumented, or worth asking a human about
- **`sources`**: List every file you read to produce this article.

### Step 5 — Write and index

1. Write each article to `.claude/wiki/<slug>.md` where `<slug>` is a lowercase kebab-case
   version of the name. E.g. `"Authentication Tokens"` → `auth-tokens.md`.
2. Regenerate `INDEX.md` (see format below).
3. Report what was written and invite the human to review and suggest corrections.

---

## INDEX.md format

```markdown
# Wiki index

_Last updated: YYYY-MM-DD_

## Articles

* [auth-tokens](auth-tokens.md) - Short summary here.
* [session-management](session-management.md) - Short summary here.

## Tag cloud

* `auth`
* `api`
* `survey`

```

---

## Reading the wiki

When the human asks "what do we know about X" or asks you to look something up before working on
a feature:

1. Read `INDEX.md` to orient.
2. Read the relevant `.md` file(s).
3. Follow wikilinks one level deep if needed for context.
4. Summarise what the wiki says and note any `Open questions` or orphaned links relevant to the
   task.

---

## Updating existing articles

When the human points you at new source files that relate to an existing article:

1. Read the existing article.
2. Read the new sources.
3. Produce an updated article, changing `updated` date and noting what changed.
4. Present the diff (what was added/changed) rather than the full article if the article is long.
5. Follow the same review → write flow.

---

## Linting the wiki

Triggered when the human asks to "lint", "audit", or "check" the wiki. Read every `.md` file in
`.claude/wiki/` (excluding `INDEX.md`) and `INDEX.md` itself, then run all checks below.

Report findings grouped by check, with specific file paths and a suggested action for each
finding. If a check passes with no issues, note it as clean. Do not auto-fix anything — report
only, then ask the human which issues to address.

### Checks

**Broken wikilinks**
For every `[[slug]]` found in any article, verify that `<slug>.md` exists in `.claude/wiki/`.
Report each broken link with the file it appears in.
_Suggested action: create the missing article, rename the link, or remove it._

**Weakly connected articles**
Flag articles with no wikilinks at all, and note articles with only one. These are isolated nodes
that may be missing relationships.
_Suggested action: review the article for concepts that could link to existing articles._

**INDEX.md out of sync — missing entries**
List every `.md` file in `.claude/wiki/` (excluding `INDEX.md`) that is not linked from the
Articles section of `INDEX.md`.
_Suggested action: add the missing entry to `INDEX.md`._

**INDEX.md out of sync — dead entries**
List every article linked in `INDEX.md` whose file no longer exists.
_Suggested action: remove the entry from `INDEX.md` or restore the file._

**Frontmatter violations**
Every article must have all required frontmatter fields: `name`, `created`, `updated`, `tags`,
`links`, `summary`, `sources`. Flag any article missing one or more fields, naming which fields
are absent.
_Suggested action: open the article and add the missing fields._

**Conflicting claims**
Identify articles that share tags or reference the same named entities (e.g. the same domain
concept, system, or business rule appears in multiple articles). Read those articles and flag
cases where they state different facts about the same thing — e.g. contradictory business rules,
different descriptions of the same flow.
_Suggested action: reconcile the conflict; one article should be canonical and the other should
link to it._

**Oversized articles**
Flag any article whose total line count exceeds 200 lines as a candidate for splitting.
_Suggested action: identify the natural split points and propose new article titles._

**Undeclared tags**
Collect every tag value used across all article frontmatters. Cross-reference against the Tags
list in `INDEX.md`. Flag any tag in use that does not appear in `INDEX.md`.
_Suggested action: either add the tag to `INDEX.md` or replace it with an existing equivalent._

---

## Conventions

- **Never write files without human approval** in the current session.
- **Never invent business rules.** If you're unsure whether something is a rule or an
  implementation detail, put it in `Open questions`.
- **Prefer concrete over abstract.** "Users must verify email before accessing paid features" is
  better than "there is an email verification step."
- **Wikilinks are aspirational.** Link to articles that *should* exist even if they don't yet.
  This makes the index's orphan list useful.
- **Tag discipline.** Before adding a new tag, check if an existing tag covers it. A wiki with
  30 unique tags on 10 articles is noise.