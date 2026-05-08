---
name: zenn-blog
description: Scaffold or translate LeRobot tutorial blog posts for zenn.dev. Use when the user asks to start a new article in this series, draft in English, translate an English draft into Japanese, or check a post against zenn's frontmatter and markdown rules.
---

# zenn-blog

Project skill for the LeRobot tutorial series at zenn.dev. The user drafts in English under `drafts/en/`, then translates to Japanese in `articles/` (the only directory zenn-cli reads).

## When to invoke

- "Scaffold the <topic> article" / "新しい記事を作って" → **scaffold mode**.
- "Translate the <topic> draft" / "翻訳して" → **translate mode**.
- "Check this article for zenn issues" → **lint mode**.

If the user's request doesn't clearly match a mode, ask once which they want.

## Locked series slugs

Only these slugs are valid for the series. Never invent new slugs without explicit user approval.

| # | Topic                       | Slug                              | Emoji | Kind     | Suggested topics                                         |
|---|-----------------------------|-----------------------------------|-------|----------|----------------------------------------------------------|
| 0 | Primer (env, glossary, logs)| `lerobot-primer-and-tips`         | 📚    | primer   | `["lerobot","robotics","huggingface","ai","python"]`     |
| 1 | Installation & Calibration  | `lerobot-install-and-calibration` | 🤖    | tutorial | `["lerobot","robotics","huggingface","python","setup"]`  |
| 2 | Teleop                      | `lerobot-teleop-control`          | 🎮    | tutorial | `["lerobot","robotics","teleop","huggingface","python"]` |
| 3 | Record dataset              | `lerobot-record-dataset`          | 📝    | tutorial | `["lerobot","robotics","dataset","huggingface","ai"]`    |
| 4 | Train policy                | `lerobot-train-policy`            | 🧠    | tutorial | `["lerobot","ai","machinelearning","pytorch","robotics"]`|
| 5 | Run trained model           | `lerobot-run-trained-model`       | 🚀    | tutorial | `["lerobot","ai","inference","robotics","huggingface"]`  |

**Kinds:**
- `tutorial` — step-by-step doing-the-thing posts (parts 1–5). Use `template-tutorial-en.md` / `template-tutorial-jp.md`.
- `primer` — reference / glossary / tips that other articles link back to (part 0). Use `template-primer-en.md` / `template-primer-jp.md`.

## Scaffold mode

Steps:

1. Identify the slug from the user's request (match against the table above).
2. Refuse if `articles/<slug>.md` or `drafts/en/<slug>.md` already exists — ask whether to overwrite.
3. Pick the template pair based on the slug's **kind** column:
   - `tutorial` → `template-tutorial-en.md` + `template-tutorial-jp.md`
   - `primer` → `template-primer-en.md` + `template-primer-jp.md`
4. Read the EN template and write to `drafts/en/<slug>.md`. Replace:
   - `<English title — replace before commit>` with a sensible English title (user can edit).
   - `emoji` and `topics` with the values from the table.
   - For tutorials: series part number `X` with the article number, and prev/next slug placeholders.
   - For the primer: leave the "Part 0 (prerequisite reference)" callout as-is.
5. Read the JP template and write to `articles/<slug>.md` with the same frontmatter values (Japanese title placeholder kept).
6. Create `images/lerobot/<slug>/.gitkeep`.
7. Tell the user the two files exist and to fill in the English draft first.

Never set `published: true` during scaffolding.

## Translate mode

Steps:

1. Read `drafts/en/<slug>.md` end-to-end before writing anything.
2. Read `.claude/skills/zenn-blog/translation-style-guide.md` and follow it strictly.
3. Read the existing `articles/<slug>.md` to preserve its frontmatter (title may already be set in Japanese — keep it unless the user asks to retranslate).
4. Translate the body section by section. For each section:
   - Localize the heading per the style guide (no trailing です on headings).
   - Translate prose into です・ます調.
   - Keep code blocks byte-identical (commands, paths, identifiers, output, language:filename annotation).
   - Convert plain "Note:" / "Warning:" paragraphs into `:::message` / `:::message alert` blocks.
   - Wrap long error logs in `:::details エラーログ全文`.
5. Confirm the final file: `published: false`, `topics` ≤ 5, emoji is one character, filename matches slug.
6. Report what was translated and remind the user to run `npx zenn preview` before flipping `published`.

Never auto-flip `published: true` — only the user does that, explicitly.

## Lint mode

Quick checks to run on a target article:

- Frontmatter has all required fields: `title`, `emoji`, `type`, `topics`, `published`.
- `type` is `tech` or `idea`.
- `emoji` is exactly one character (count graphemes, not bytes).
- `topics` is a YAML array, length 1–5, all `[a-z0-9]+` (no spaces, no hyphens, no underscores in topic names — zenn restriction).
- Filename slug: 12–50 chars of `[a-z0-9-]`.
- All `![...](path)` image references resolve under `images/lerobot/<slug>/`.
- No `:::message` / `:::details` left unclosed.
- Series header callout includes prev/next links (or notes that this is the first/last).

Report findings as a checklist; fix only what the user approves.

## Prefer existing helpers

- Tutorial templates: `.claude/skills/zenn-blog/template-tutorial-en.md`, `template-tutorial-jp.md`.
- Primer templates: `.claude/skills/zenn-blog/template-primer-en.md`, `template-primer-jp.md`.
- Style rules: `.claude/skills/zenn-blog/translation-style-guide.md`.
- Don't restate template content in new files — reference these instead.

## What this skill does NOT do

- Does not run `git add` / `git commit` / `git push`.
- Does not flip `published: true`.
- Does not write actual tutorial content from scratch — the user drafts the English source; this skill scaffolds, translates, and lints.
- Does not create new series entries beyond the 6 locked slugs (primer + 5 tutorials).
