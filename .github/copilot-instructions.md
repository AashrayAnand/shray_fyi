# Skill: Publish Blog Post

## Description

Create and publish a new blog post on shray.fyi. Given post content (inline or from a file), create the properly formatted post directory and push it live.

## When to Use

When the user provides blog post content and wants it published to shray.fyi. The content may come as:
- Inline text in the chat
- A path to a file containing the content

## Steps

### 1. Determine post metadata

Ask the user for anything not provided:
- **Title**: The display title for the post
- **Date**: The publication date (default: today, in `YYYY-MM-DD` format)
- **Slug**: A short kebab-case topic summary for the folder name (derive from title if not given)

### 2. Create the post directory and file

```bash
cd /workspace/personal/shray_fyi
mkdir -p content/YYYY-MM-DD-slug
```

Create `content/YYYY-MM-DD-slug/index.md` with this template:

```
+++
title = "Post Title Here"
date = YYYY-MM-DD
+++

<content goes here>
```

### 3. Content formatting rules

**DO:**
- Format the content as valid markdown (headings, code blocks, blockquotes, lists, links)
- Fix obvious spelling errors
- Ensure code blocks have language annotations (e.g., ` ```rust `)
- Ensure blockquotes use `>` prefix
- Preserve the author's voice, structure, and arguments exactly

**DO NOT:**
- Rewrite, rephrase, or restructure the content
- Add introductions, conclusions, or commentary not in the original
- Change the meaning of any sentence
- Remove or add sections
- Change technical claims or opinions

### 4. Commit and push

```bash
cd /workspace/personal/shray_fyi
git add content/YYYY-MM-DD-slug/
git commit -m "Add post: Post Title Here"
git push origin main
```

### 5. Confirm to the user

Report the post path and that it was pushed. The site auto-deploys from the main branch.

## Post format reference

Existing posts follow this structure — the `+++` TOML frontmatter is required by Zola (the static site generator):

```
+++
title = "A Hyperscaler Developer's Foray into Decentralized Systems"
date = 2026-02-12
+++

Post body in markdown...
```

Posts live in `content/YYYY-MM-DD-slug/index.md`. Some posts may include images in the same directory (e.g., `gfs_architecture.png` referenced as `./gfs_architecture.png` in the markdown).

## Repository

- **Repo**: `AashrayAnand/shray_fyi`
- **Remote**: `origin` → `https://github.com/AashrayAnand/shray_fyi.git`
- **Branch**: `main`
- **Site generator**: Zola
- **Auto-deploy**: Vercel from main branch
