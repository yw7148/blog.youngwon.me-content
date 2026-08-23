---
name: write-tech-blog-post
description: Turn notes in ideas/ into polished Markdown posts for the Youngwon Tech Blog, or revise and prepare existing posts for publication. Use when Codex drafts, restructures, reviews, illustrates, tags, changes draft status, or publishes technical blog content in this repository.
---

# Write Tech Blog Post

Develop an engineering idea into a readable, technically honest post while preserving this repository's content conventions and the author's preferred voice.

## Establish context

1. Read `README.md`, `SCHEMA.md`, and `templates/post.md` before editing.
2. Read the source note in `ideas/` and any existing target post completely.
3. Inspect `git status -sb` before changing or publishing files. Treat unrelated modifications and untracked files as user-owned.
4. Determine whether the request is to draft, revise, illustrate, publish, or a combination. Do not infer commit, push, or publication from an editing request.

## Draft the post

1. Create `posts/<slug>.md`; remember that the filename becomes the URL slug.
2. Follow `SCHEMA.md`. Default new work to `draft: true` unless the user explicitly asks to publish it.
3. Use the repository template as a starting checklist, not a mandatory outline. Keep only headings that serve the story.
4. Build the explanation around this progression when applicable:
   - the concrete situation and scale;
   - the obvious first solution;
   - the cost or user-experience contradiction it creates;
   - alternatives considered and why they fail;
   - the chosen design and its data flow;
   - operational rules, limitations, and cases requiring a different solution;
   - a concise closing synthesis.
5. Preserve the idea note's original insight, but correct overstatements. For example, explain when an index becomes expensive instead of claiming that indexing a frequently updated field is always wrong.
6. Use concrete queries, records, rankings, or flows when they make the tradeoff easier to understand. Mark conceptual schemas and pseudocode as such.
7. Distinguish user-facing approximations from authoritative results. Never imply that a personalized or eventually consistent view is suitable for rewards, settlement, or winner selection unless the design supports it.

## Shape the opening

Use this default order:

1. Hero image, when one materially helps.
2. Five compact technical keywords on one line, each wrapped in backticks.
3. A question or tension-based hook.
4. One short paragraph promising the problem the post will solve.
5. The first substantive section.

Choose keywords that developers would search for or use to classify the architecture, such as `System Design`, `Database Index`, `Snapshot`, `Score Bucketing`, and `Eventual Consistency`. Do not merely repeat Korean phrases from the title or section headings.

Omit a standalone `한 줄 요약` by default. Add it only when the user requests one or the post is long enough that an executive summary adds information instead of repeating the hook and final `정리`.

Write hooks that create curiosity without making a false promise. Prefer a concrete tension such as accuracy versus database load over generic claims like “고성능 시스템을 만들어보자.”

## Add visuals deliberately

Add a visual only when it clarifies the subject or a relationship.

- Use a generated raster hero for mood and immediate recognition. Make the article's domain unmistakable at thumbnail size; a ranking article should visibly contain an ordered leaderboard, rank markers, or rank movement rather than a generic data pipeline.
- Use a deterministic SVG diagram for architecture, mappings, or exact data flow. Favor accurate labels and a small number of nodes over decorative complexity.
- Keep generated hero images free of prose and pseudo-text. Numerals or universally recognizable markers are acceptable when essential to the concept.
- Give every image descriptive Korean alt text.
- Store project assets under `images/` with stable, descriptive filenames.
- This blog remotely loads Markdown from the content repository rather than copying adjacent assets. Reference committed assets with an absolute raw GitHub URL in this form:

```markdown
![대체 텍스트](https://raw.githubusercontent.com/yw7148/blog.youngwon.me-content/main/images/<filename>)
```

- Remember that a new raw URL returns 404 until its asset is pushed.
- If generating or editing a raster image, invoke the `imagegen` skill and follow its validation and save-path rules. Inspect the result before adopting it.

## Revise with restraint

- Lead with the user's requested outcome; do not rewrite unrelated sections without a reason.
- Remove repeated summaries and duplicated conclusions.
- Prefer natural Korean prose and short paragraphs. Use English for established technical terms when it improves recognition.
- Avoid overstating scale, performance, correctness, or industry practice without evidence.
- When the user challenges a choice, state whether you agree and why. Keep a defensible choice when it materially improves the post; otherwise revise directly.

## Validate

1. Run `git diff --check`.
2. Re-read the opening, transitions, examples, and conclusion as one narrative.
3. Confirm frontmatter field names and the `YYYY-MM-DD` date format against `SCHEMA.md`.
4. Confirm every local asset exists and every Markdown URL uses the final committed filename.
5. Validate SVG files with `xmllint --noout <file>` when available.
6. If the blog application is available and the request warrants integration validation, synchronize or load the content using its documented workflow and run its build. Do not mutate the sibling application merely to answer a writing request.

## Publish safely

Only perform these steps when the user explicitly asks to commit, push, or publish:

1. Set `draft: false` only when the user asks to make the post public. Keep `publishedAt` intentional; do not silently replace the original publication date during later edits.
2. Inspect the final diff and stage explicit post and asset paths. Never use `git add -A` in a mixed worktree.
3. Commit with a terse Korean message describing the outcome.
4. Push the current intended branch and verify `HEAD` matches its remote-tracking ref.
5. Report excluded user changes, the commit hash, branch, and validation result.

If a push appears to finish without updating the remote-tracking ref, inspect the remote state before retrying. A successful first push followed by a stale local ref can make an immediate retry fail with `cannot lock ref`; fetch the branch and compare commit hashes before treating it as a failed publication.
