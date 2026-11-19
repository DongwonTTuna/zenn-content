# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Zenn content repository for technical writing. Zenn is a Japanese technical writing platform for developers.

## Common Commands

### Content Management
- `npm run new:article` - Create a new article
- `npm run new:book` - Create a new book  
- `npm run preview` - Preview content locally (opens at http://localhost:8000)

### Development Workflow
1. Create new content using the above commands
2. Edit markdown files in `/articles` or `/books` directories
3. Preview changes locally with `npm run preview`
4. Commit and push to publish

## Project Structure

- `/articles/` - Individual technical articles in markdown format
- `/books/` - Multi-chapter books (if any)
- `/images/` - Images used in articles, organized by article name
- `package.json` - Contains Zenn CLI dependency and npm scripts

## Article Format

Articles use frontmatter with these fields:
```yaml
---
title: "Article title"
emoji: "📦"
type: "tech" 
topics: ["topic1", "topic2"]
published: false/true
publication_name: "publication_name" (optional)
---
```

## Important Notes

- Image paths in articles should reference `/images/article-name/filename`
- The Zenn CLI handles the publishing workflow
- Articles are written in markdown with Zenn-specific extensions

Zenn.dev의 집필방법은 다음과 같아. 이걸 참고하고 적절히 알아서 잘 사용해줘.
https://zenn.dev/zenn/articles/markdown-guide
