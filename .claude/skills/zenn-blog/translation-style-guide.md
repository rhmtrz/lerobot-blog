# Translation Style Guide — EN → JP for zenn.dev

Rules for translating English drafts in `drafts/en/` into the Japanese articles in `articles/`. The audience is Japanese engineers, including beginners. Tone should match Zenn's typical tutorial register: friendly, precise, no fluff.

## Tone

- Use **です・ます調** consistently. Never mix だ・である調 in the same article.
- Direct, instructional, second-person feel — but Japanese rarely uses explicit "you/we." Drop pronouns when possible.
- Avoid unnecessary politeness markers (お, ご) on technical terms — `データを記録する`, not `データをご記録になる`.
- Don't start every paragraph with `まず` / `次に`. Vary openings.

## What to keep in English (do NOT translate)

- Product / library names: `LeRobot`, `Hugging Face`, `PyTorch`, `OpenCV`.
- CLI commands, flags, file paths, URLs, config keys: `python -m lerobot.scripts.train`, `--policy.type=act`, `~/.cache/huggingface`.
- Code identifiers: function names, variable names, class names.
- Domain jargon that Japanese engineers already use in English: `teleop`, `policy`, `dataset`, `checkpoint`, `episode`, `inference`, `fine-tuning`.
- Error messages and log output (verbatim).

## When to use カタカナ

For common-noun usage in prose where the English term feels foreign mid-sentence:

| English          | カタカナ           |
|------------------|--------------------|
| calibration      | キャリブレーション |
| training         | トレーニング       |
| dataset (prose)  | データセット       |
| model            | モデル             |
| robot            | ロボット           |
| controller       | コントローラー     |
| episode (prose)  | エピソード         |

When the same word appears as a code identifier or CLI flag, keep it in English.

## Numbers and units

- Half-width digits: `5回`, `3つ`, `100Hz` — never `５回`, `１００Hz`.
- Japanese counters where natural: `5回`, `3つの手順`, `2台のロボット`.
- Units stay attached without space: `100Hz`, `5GB`, `2.5秒`.

## Punctuation

- Use `。` and `、` (full-width) in Japanese sentences.
- Inside `inline code` and code blocks, leave English punctuation untouched.
- Don't add a `、` before parentheses; Japanese typography handles spacing.
- Use full-width parentheses `()` for asides in pure Japanese prose, half-width `()` when wrapping English/code.

## Headings

- Short noun phrases. No trailing `です`.
- ✅ `はじめに`, `環境構築`, `キャリブレーション手順`
- ❌ `はじめにです`, `環境構築をしましょう`

## Code blocks

- **Never modify** the code itself: commands, paths, identifiers, output stay byte-identical to the English source.
- Comments inside code (`# ...`) MAY be translated if they help comprehension. Keep them short.
- Preserve the `language:filename` annotation on fenced blocks.

## Zenn callout conventions

- `:::message` for notes / common errors / tips.
- `:::message alert` for hardware safety warnings, data-loss risks, irreversible actions.
- `:::details <タイトル>` for long error logs, optional deep dives, full config dumps.

Use them to break up long troubleshooting sections — don't dump everything as plain paragraphs.

## Images

- Alt text describes content for accessibility, in Japanese: `![SO-100 のキャリブレーション画面](./images/...)`.
- Captions in italics on the line below: `*図 1: キャリブレーション完了時の画面*`.
- Width hint when needed: `![](url =500x)`.

## Series cross-links

- Always include the previous and next article in the header callout block.
- Last article in the series links back to the index / summary instead of a non-existent next.

## Pre-publish check (every article)

1. `published: false` until user confirms.
2. `topics` array length ≤ 5.
3. `emoji` is exactly one character.
4. Filename slug matches frontmatter intent and is 12–50 chars of `[a-z0-9-]`.
5. All images live under `images/lerobot/<slug>/`.
6. `npx zenn preview` renders without warnings.
