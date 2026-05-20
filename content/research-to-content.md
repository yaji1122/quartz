---
title: Research to Content Workflow
date: 2026-05-20
tags:
  - research
  - quartz
  - content
  - workflow
draft: false
---

## Overview

Use this note when a research task is finished and the findings need to be published inside the Quartz `content/` folder.
The goal is to turn raw research into a page that is easy to browse, easy to link, and easy to keep updated.

Quartz treats Markdown files in `content/` as source material, so the most useful result is usually a small page set rather than a single giant dump.

## When to Use

- You just finished a research deep dive and want to publish the result in Quartz.
- You need to turn notes, comparisons, or implementation findings into a readable page.
- You want the research to appear in folder listings, tags, and backlinks.
- You want a repeatable structure for future research writeups.

## Recommended Folder Layout

- Put each topic in its own folder when the research has multiple pages.
  - Example: `content/devops/k8s-api-gateway/`
- Create an `index.md` file for the folder hub.
  - This page should summarize the series and link to the subpages.
- Split deeper topics into separate files.
  - Example: `overview.md`, `spec.md`, `architecture.md`, `deployment.md`.
- For a small one-off result, a single Markdown file at `content/<topic>.md` is fine.

## Writing Pattern

A good research page usually has:

1. A short title and clear frontmatter.
2. A summary of the question or problem.
3. Key findings in plain language.
4. Supporting links or references.
5. A short next-step section if more work is needed.

### Frontmatter Template

```md
---
title: "<page title>"
date: 2026-05-20
tags:
  - research
  - <topic>
draft: false
---
```

### Suggested Sections

- `## Overview`
- `## Key Findings`
- `## Supporting Details`
- `## References`
- `## Next Steps`

## Linking and Navigation

- Link related pages with Quartz wikilinks like `[[other-note]]`.
- Use a folder `index.md` as the entry page for a research series.
- Add tags when the topic belongs in multiple listings.
- Keep names stable so links do not break when the content grows.

## Publishing Checklist

- Make sure the file lives inside `content/`.
- Confirm the page has frontmatter.
- Keep the summary readable on its own.
- Check that the note links to any sibling pages.
- Commit and push after reviewing the result.

## Common Pitfalls

1. **Dumping raw research into one huge file.**
   Split it into a series or at least use headings.
2. **Skipping `index.md` for a topic folder.**
   Folder listings work better when the folder has a clear hub page.
3. **Forgetting tags and frontmatter.**
   Quartz uses them for organization and discovery.
4. **Placing research outside `content/`.**
   Quartz will not publish it if it is not inside the content tree.
5. **Not committing the final writeup.**
   The site only updates after the repo change is pushed.

## Verification Checklist

- [ ] The note is in `content/`
- [ ] Frontmatter includes `title`, `date`, `tags`, and `draft`
- [ ] The page has a readable summary
- [ ] Related pages are linked with Quartz wikilinks
- [ ] Topic folders have an `index.md` when needed
- [ ] The change is committed and pushed
