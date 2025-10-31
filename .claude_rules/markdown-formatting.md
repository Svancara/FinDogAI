# Markdown Formatting Standards

This document defines mandatory markdown formatting rules for all markdown files created or edited by AI assistants in this project.

## Rule MD022: Headings Should Be Surrounded by Blank Lines

**Requirement**: All headings MUST be preceded and followed by at least one blank line.

**Rationale**: Some parsers (including kramdown) will not parse headings without blank lines before them.

**Examples**:

```markdown
<!-- WRONG -->
Some text here
## Heading
Content starts immediately

<!-- CORRECT -->
Some text here

## Heading

Content starts here
```

## Rule MD026: No Trailing Punctuation in Headings

**Requirement**: Headings MUST NOT end with punctuation characters: .,;:!。，；：！

**Exception**: Question marks (?) are allowed for FAQ-style headings.

**Rationale**: Headings are not meant to be full sentences.

**Examples**:

```markdown
<!-- WRONG -->
## Installation Steps.
### Quick Start:

<!-- CORRECT -->
## Installation Steps
### Quick Start
### How Do I Install?
```

## Rule MD029: Ordered List Item Prefix

**Requirement**: Ordered lists MUST use consistent numbering (either all "1." or sequential 1, 2, 3...).

**Default Style**: one_or_ordered (accepts either pattern, but must be consistent within a list)

**Examples**:

```markdown
<!-- WRONG -->
1. First item
1. Second item
3. Third item (jumped from 1 to 3)

<!-- CORRECT - Style "one" -->
1. First item
1. Second item
1. Third item

<!-- CORRECT - Style "ordered" -->
1. First item
2. Second item
3. Third item
```

## Rule MD032: Lists Should Be Surrounded by Blank Lines

**Requirement**: Lists MUST be preceded and followed by blank lines (except at document start/end).

**Rationale**: Some parsers fail to parse lists without surrounding blank lines.

**Examples**:

```markdown
<!-- WRONG -->
Some text here
- List item 1
- List item 2
Next paragraph

<!-- CORRECT -->
Some text here

- List item 1
- List item 2

Next paragraph
```

## Rule MD040: Fenced Code Blocks Should Have a Language

**Requirement**: All fenced code blocks MUST specify a language identifier.

**Rationale**: Enables proper syntax highlighting and improves rendering.

**Examples**:

```markdown
<!-- WRONG -->
```
const x = 42;
```

<!-- CORRECT -->
```javascript
const x = 42;
```

<!-- For plain text or no highlighting -->
```text
Plain text content
```
```

## Rule MD036: No Emphasis as Heading

**Requirement**: Do NOT use bold or italic text as section headings. Use proper markdown headings instead.

**Rationale**: Using emphasis instead of headings prevents tools from inferring document structure.

**Examples**:

```markdown
<!-- WRONG -->
**Section Title**

Content here...

<!-- CORRECT -->
## Section Title

Content here...
```

## Quick Checklist

When creating or editing markdown files, ensure:

- [ ] Blank lines before and after all headings (MD022)
- [ ] No trailing punctuation in headings except ? (MD026)
- [ ] Consistent ordered list numbering (MD029)
- [ ] Blank lines before and after all lists (MD032)
- [ ] Language specified for all code blocks (MD040)
- [ ] Proper headings used instead of bold/italic text (MD036)

## Common Violations to Avoid

1. **Missing blank lines**: Most common violation - always add blank lines around headings and lists
2. **Unlabeled code blocks**: Always specify language, use `text` if no syntax highlighting needed
3. **Punctuation in headings**: Remove periods, commas, colons from heading text
4. **Bold as headings**: Replace `**Section**` with `## Section`
5. **Inconsistent list numbering**: Pick either all "1." or sequential numbers, stick with it
