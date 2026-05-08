# lerobot-blog

Engineer-friendly LeRobot tutorial series for [zenn.dev](https://zenn.dev). Drafted in English, published in Japanese.

## Workflow

```
drafts/en/<slug>.md   ──translate──▶   articles/<slug>.md   ──zenn GitHub integration──▶   zenn.dev
   (English source)                       (Japanese, published)
```

1. Write the English draft in `drafts/en/<slug>.md`.
2. Translate to Japanese into `articles/<slug>.md` (zenn-cli only reads `articles/`).
3. Preview locally: `npx zenn preview` → http://localhost:8000.
4. Flip `published: true` in the JP frontmatter and push to the connected zenn GitHub repo.

The `drafts/` directory is invisible to zenn — only `articles/` and `books/` are picked up.

## Using the writing skill

A project-level Claude skill at `.claude/skills/zenn-blog/` automates scaffolding and translation. Invoke from Claude Code with prompts like:

- _"Scaffold the calibration article."_ → creates `drafts/en/lerobot-install-and-calibration.md` + `articles/lerobot-install-and-calibration.md` from the templates with prefilled frontmatter.
- _"Translate the calibration draft."_ → reads the EN draft and writes natural です・ます調 Japanese into the JP article, preserving code blocks verbatim.

See [`.claude/skills/zenn-blog/SKILL.md`](.claude/skills/zenn-blog/SKILL.md) for full rules.

## Planned series

| # | Topic                                | Slug                              | Emoji | Kind     |
|---|--------------------------------------|-----------------------------------|-------|----------|
| 0 | Primer — env, glossary, logs, tips   | `lerobot-primer-and-tips`         | 📚    | primer   |
| 1 | Installation & Calibration           | `lerobot-install-and-calibration` | 🤖    | tutorial |
| 2 | Teleop                               | `lerobot-teleop-control`          | 🎮    | tutorial |
| 3 | Record dataset                       | `lerobot-record-dataset`          | 📝    | tutorial |
| 4 | Train policy                         | `lerobot-train-policy`            | 🧠    | tutorial |
| 5 | Run trained model                    | `lerobot-run-trained-model`       | 🚀    | tutorial |

**Part 0 (the primer)** is a reference/cheat-sheet article — environment versions, hardware quick-reference (USB, motor IDs, cameras), data paths, glossary (episode, policy, dataset…), how to read the CLI logs, and common gotchas. Tutorials 1–5 link back to it instead of re-explaining these basics.

## Local setup

```bash
npm install
npx zenn preview
```

## Publishing

Connect this repo in the [zenn GitHub integration dashboard](https://zenn.dev/dashboard/deploys). After connection, every push to the configured branch syncs `articles/` to zenn. Set `published: true` in the article frontmatter when ready to go live.
