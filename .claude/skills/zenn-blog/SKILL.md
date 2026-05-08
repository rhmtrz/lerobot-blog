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

| # | Topic                       | Slug                              | Emoji | Suggested topics                                        |
|---|-----------------------------|-----------------------------------|-------|---------------------------------------------------------|
| 1 | Installation & Calibration  | `lerobot-install-and-calibration` | 🤖    | `["lerobot","robotics","huggingface","python","setup"]` |
| 2 | Teleop                      | `lerobot-teleop-control`          | 🎮    | `["lerobot","robotics","teleop","huggingface","python"]`|
| 3 | Record dataset              | `lerobot-record-dataset`          | 📝    | `["lerobot","robotics","dataset","huggingface","ai"]`   |
| 4 | Train policy                | `lerobot-train-policy`            | 🧠    | `["lerobot","ai","machinelearning","pytorch","robotics"]`|
| 5 | Run trained model           | `lerobot-run-trained-model`       | 🚀    | `["lerobot","ai","inference","robotics","huggingface"]` |

## Scaffold mode

Steps:

1. Identify the slug from the user's request (match against the table above).
2. Refuse if `articles/<slug>.md` or `drafts/en/<slug>.md` already exists — ask whether to overwrite.
3. Read `.claude/skills/zenn-blog/template-en.md` and write to `drafts/en/<slug>.md`. Replace:
   - `<Title in English — replace before commit>` with a sensible English title (user can edit).
   - `emoji` and `topics` with the values from the table.
   - Series part number `X` with the article number, and prev/next slug placeholders.
4. Read `.claude/skills/zenn-blog/template-jp.md` and write to `articles/<slug>.md` with the same frontmatter values (Japanese title placeholder kept).
5. Create `images/lerobot/<slug>/.gitkeep`.
6. Tell the user the two files exist and to fill in the English draft first.

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

- Templates: `.claude/skills/zenn-blog/template-en.md`, `template-jp.md`.
- Style rules: `.claude/skills/zenn-blog/translation-style-guide.md`.
- Don't restate template content in new files — reference these instead.

## What this skill does NOT do

- Does not run `git add` / `git commit` / `git push`.
- Does not flip `published: true`.
- Does not write actual tutorial content from scratch — the user drafts the English source; this skill scaffolds, translates, and lints.
- Does not create new series entries beyond the 5 locked slugs.
