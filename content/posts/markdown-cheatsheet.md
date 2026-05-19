---
title: "Markdown Cheatsheet for Blog Posts"
date: 2026-05-01
draft: false
tags: ["markdown", "writing", "reference"]
categories: ["Reference"]
description: "A quick reference for Markdown syntax used in Hugo posts — headings, code, tables, and more."
showToc: true
---

This is a quick reference I keep coming back to. All standard Markdown plus some Hugo/PaperMod extras.

## Headings

```markdown
# H1
## H2
### H3
```

## Text formatting

**Bold** — `**bold**`  
*Italic* — `*italic*`  
~~Strikethrough~~ — `~~strikethrough~~`  
`Inline code` — `` `inline code` ``

## Lists

Unordered:

```markdown
- Item one
- Item two
  - Nested item
```

Ordered:

```markdown
1. First
2. Second
3. Third
```

## Code blocks

Fenced code blocks with syntax highlighting:

````markdown
```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

print(greet("world"))
```
````

Renders as:

```python
def greet(name: str) -> str:
    return f"Hello, {name}!"

print(greet("world"))
```

## Links and images

```markdown
[Link text](https://example.com)
![Alt text](/images/photo.jpg)
```

## Blockquotes

```markdown
> This is a blockquote.
> It can span multiple lines.
```

> This is a blockquote.

## Tables

```markdown
| Column A | Column B | Column C |
|----------|----------|----------|
| Value 1  | Value 2  | Value 3  |
| Value 4  | Value 5  | Value 6  |
```

| Column A | Column B | Column C |
|----------|----------|----------|
| Value 1  | Value 2  | Value 3  |
| Value 4  | Value 5  | Value 6  |

## Front matter reference

```yaml
---
title: "Post Title"
date: 2026-05-01
draft: false              # set true to hide from production
tags: ["tag1", "tag2"]
categories: ["Category"]
description: "Short summary shown in post list"
showToc: true             # table of contents
cover:
  image: "/images/cover.jpg"
  alt: "Cover image alt text"
---
```

## Horizontal rule

Three dashes on their own line:

```markdown
---
```

---

That covers most of what you'll need for day-to-day blogging.
