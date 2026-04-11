---
name: confluence-formatting
description: Use when creating, editing, or formatting Confluence pages with wiki markup, PlantUML diagrams, code blocks, tables, panels, or any structured content in Confluence
---

# Confluence Document Formatting

## Overview

Reference guide for Confluence wiki markup formatting. Covers text styles, lists, code blocks, tables, links, images, panels, PlantUML diagrams, and macros.

**Key principle:** Always use `convert_to_markdown=False` for reading and `content_format="wiki"` for writing to preserve macros and formatting.

## When to Use

- Creating or updating Confluence pages with structured content
- Adding PlantUML diagrams to Confluence
- Formatting documents with wiki markup (panels, code blocks, tables)
- Converting content between formats for Confluence API

## API Patterns

### Read Page

```python
Remote_Confluence_confluence_get_page(
    page_id="12345",
    convert_to_markdown=False  # Preserve macros
)
```

### Update Page

```python
Remote_Confluence_confluence_update_page(
    page_id="12345",
    title="Title",
    content_format="wiki",  # Required for macros
    content=wiki_content
)
```

## Quick Reference

| Element | Wiki Markup |
|---------|------------|
| Bold | `*text*` |
| Italic | `_text_` |
| Strikethrough | `~~text~~` |
| Inline code | `{{code}}` |
| Superscript | `^text^` |
| Subscript | `~text~` |
| H1-H5 | `h1.` to `h5.` |
| Separator | `---` |
| External link | `[text\|url]` |
| Anchor link | `[text\|#anchor]` |
| Image | `!url!` |
| Image + params | `!url\|width=600!` |

## Text Formatting

```wiki
*Bold text*
_Italic_
*_Bold italic_*
~~Strikethrough~~
{{Inline code}}
^Superscript^
~Subscript~
```

## Headings

```wiki
h1. Heading 1
h2. Heading 2
h3. Heading 3
h4. Heading 4
h5. Heading 5
```

## Lists

```wiki
# Ordered item
## Nested ordered
# Another item

* Unordered item
** Nested unordered
* Another item

- Dash item
* Circle item
+ Plus item
```

## Code Blocks

```wiki
{code}
Plain code block
{code}

{code:python}
def hello():
    print("Hello")
{code}
```

Supported languages: `javascript`, `java`, `python`, `ruby`, `php`, `sql`, `groovy`, `csharp`, `cpp`, `css`, `xml`, `html`, `yaml`, `flex`, `scala`, `actionscript`, `coldfusion`, `java-xml`

## Tables

```wiki
| Header 1 | Header 2 | Header 3 |
| Cell 1-1 | Cell 1-2 | Cell 1-3 |
| Cell 2-1 | Cell 2-2 | Cell 2-3 |
```

Cells support inline formatting: `*bold*`, `_italic_`, `{{code}}`, `~~strike~~`

## Links

```wiki
[Display text|https://example.com]
[Internal page|/display/~user/Page]
[Anchor|#section-id]
```

**Repository links:** Always use HTTPS for GitHub/GitLab/Bitbucket repository links (not SSH):

```wiki
# ✅ Correct - https repository
Repository: [https://git.yadro.com/~user/repo|https://git.yadro.com/~user/repo]

# ❌ Wrong - ssh repository
Repository: ssh://git@b.yadro.com:7999/~user/repo.git
```

## Images

```wiki
!https://example.com/image.png!
!https://example.com/image.png|align=center!
!https://example.com/image.png|width=600!
!https://example.com/image.png|align=center,width=600,border=true!
!https://example.com/image.png|align=center,alt=Description|Caption text!
```

## Panels

### Info (blue)
```wiki
{panel:title=Note|borderStyle=solid|borderColor=#0000FF|titleBGColor=#E6F3FF|bgColor=#F0F8FF}
Content
{panel}
```

### Success (green)
```wiki
{panel:title=Success|borderStyle=solid|borderColor=#6B8E23|titleBGColor=#E8F5E9|bgColor=#F1F8E9}
Content
{panel}
```

### Warning (red)
```wiki
{panel:title=Attention|borderStyle=solid|borderColor=#FF0000|titleBGColor=#FFCCCC|bgColor=#FFE6E6}
Content
{panel}
```

### Neutral
```wiki
{panel:title=Info|borderStyle=solid|borderColor=#ccc|titleBGColor=#F7D6C1|bgColor=#FFFFCE}
Content
{panel}
```

## Info Macros

```wiki
{info}Information{info}
{note}Note{note}
{warning}Warning{warning}
{tip}Tip{tip}
```

## Blockquotes

```wiki
{quote}
Quoted text with **formatting** support.
{quote}
```

## PlantUML Diagrams

All PlantUML code goes between `{plantuml}` macros:

```wiki
{plantuml}
@startuml
...
@enduml
{plantuml}
```

### Activity Diagram

```wiki
{plantuml}
@startuml
start
:Participant: Action;
if (Condition?) then (Yes)
  :Action A;
else (No)
  :Action B;
endif

partition "Department" {
  :Action in zone;
}
stop
@enduml
{plantuml}
```

**Rules:**
- Every `if` needs exactly one `endif`
- Use short participant names
- Max 3-4 nesting levels for readability
- `\n` for line breaks inside action blocks

### Sequence Diagram

```wiki
{plantuml}
@startuml
actor User as user
participant System as system
database "DB" as db

user -> system: Request
system -> db: Query
db --> system: Result
system --> user: Response
@enduml
{plantuml}
```

### Component Diagram

```wiki
{plantuml}
@startuml
skinparam componentStyle rectangle

[Component A] as A
[Component B] as B
database "Database" as DB

A --> B : uses
B --> DB : stores
@enduml
{plantuml}
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `convert_to_markdown=True` to read page with macros | Always use `False` |
| Using `content_format="markdown"` with PlantUML | Use `"wiki"` |
| Missing `endif` for `if` in PlantUML | Count `if`/`endif` pairs |
| Long participant names in PlantUML | Use short names |
| Overwriting page without reading first | Always read, modify, then write |
| Using `**` inside `*...*` (nested bold markers) | Avoid nested `**` in bold sections |
| Using `*` inside `**...*` (nested bold markers) | Avoid nested `*` in bold sections |

**Nested formatting issue:** Confluence wiki doesn't support nested markers like `*Text **bold** more*` or `**Text *italic* more**`. This will break rendering.

**Instead of:**
```wiki
*Nested bold:* Text **breaks** formatting
```

**Use one of these options:**

1. Plain label in info panel:
```wiki
{info}*Label:* Text *{bold}{bold} works here{info}
```

2. Just bold, no label:
```wiki
Text **bold** without label markers
```

3. Use code block for technical terms:
```wiki
*Label:* Text {{bold formatting}} using code
```

**Repository links:** Always use HTTPS for GitHub/GitLab/Bitbucket:

```wiki
# ✅ Correct
Repository: [repo-name|https://git.yadro.com/~user/repo]

# ❌ Wrong - SSH breaks web interface access
Repository: ssh://git@yadro.com:7999/~user/repo.git
``` |

## Workflow

1. Read page with `convert_to_markdown=False`
2. Modify wiki content (preserve existing macros/structure)
3. Update with `content_format="wiki"` and same title
4. Each update creates a new version

## Limitations

- `{toggle}`, `{center}`, `{right}` macros may not be available
- Test macros on small documents first
- Back up content before major updates
