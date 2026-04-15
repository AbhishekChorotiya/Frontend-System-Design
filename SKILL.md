# Skill: Frontend System Design Content Generator

## Description

Use this skill when creating new frontend system design topic articles for this repository. It ensures every new article matches the format, depth, and quality of existing content.

## When to Use

- Adding a new topic to an existing chapter (directory)
- Creating a new chapter with its first topic
- Expanding an existing article with missing sections

## Content Structure

Every article MUST follow this exact structure in order:

### 1. Title (H1)

```markdown
# Topic Name
```

A single H1 heading with the full topic name.

### 2. Introduction (1-2 paragraphs)

Immediately after the title, write 1-2 paragraphs that:
- Define what the concept is
- Explain why it matters for frontend system design
- Set context for the rest of the article

Do NOT use a heading for the introduction. It flows directly after the H1.

### 3. Core Concepts (H2)

```markdown
## Core Concepts
```

Or a topic-appropriate variant like `## Core Principles of [Topic]:`.

Use a numbered list where each item has:
- **Bold label:** followed by an explanation
- Sub-bullets for additional detail where needed

Example:
```markdown
1.  **Modularity & Encapsulation:** Components are self-contained units with well-defined interfaces (props) and responsibilities.
2.  **Isolation:** Components are developed, viewed, and tested independently from the rest of the application.
```

### 4. Detailed Sections (H2/H3)

Include ALL of the following as H2 sections (adapt headings to fit the topic naturally):

- **How It Works / How It Works in Practice** -- Step-by-step explanation of the mechanism or workflow
- **Benefits / Key Benefits / Advantages** -- Bulleted list of benefits with bold labels
- **Drawbacks / Challenges / Potential Challenges** -- Bulleted list of drawbacks with bold labels
- **Use Cases / When to Use** -- Where and when this concept applies
- **Best Practices / Best Practices Summary** -- Numbered or bulleted actionable recommendations

Optional sections (include when relevant):
- **Tools and Ecosystem** -- Relevant libraries, frameworks, or dev tools
- **Relationship with Other Patterns/Concepts** -- How this topic connects to other topics in the repo
- **Comparison Tables** -- When comparing two or more approaches (use markdown tables)

### 5. Code Examples

Distribute code examples throughout the relevant sections (not in a single block at the end). Follow these rules:

- Use fenced code blocks with language identifiers: `javascript`, `typescript`, `jsx`, `tsx`, `html`, `css`, `graphql`
- Include a file path comment at the top of each code block when showing file-level code:
  ```jsx
  // src/components/Button.jsx
  ```
- Show **good vs bad patterns** with clear markers:
  ```javascript
  // ✅ Good: descriptive explanation
  const result = items.filter(item => item.active);

  // ❌ Bad: descriptive explanation
  const result = [];
  for (let i = 0; i < items.length; i++) { ... }
  ```
- Provide examples across multiple frameworks (React, Vue, Angular) when the topic is framework-agnostic
- Use modern JavaScript/TypeScript patterns (ES6 modules, hooks, async/await)
- Keep examples practical and realistic, not trivial hello-world snippets

### 6. Common Beginner Doubts or Questions (H2)

```markdown
## Common Beginner Doubts or Questions
```

This is the FINAL section of every article. Structure:

- Each question is an **H3 heading** written as a natural question a beginner would ask
- Answer is 1-3 paragraphs directly below the heading
- Include 3-7 questions per article
- Questions should address genuine misconceptions, not restate the article content
- Answers should be thorough and may include short code snippets

Example:
```markdown
### Is it always better to inline all CSS?

No. Inlining all CSS can make the HTML file very large, potentially slowing down the initial HTML download. It's best to inline only the *critical* CSS needed for the above-the-fold content. The rest can be loaded asynchronously.
```

## Formatting Rules

1. **Headings:** Use `#` for title, `##` for main sections, `###` for subsections. Never skip levels.
2. **Lists:** Use `*` for unordered lists and `1.` for ordered lists. Indent sub-items with 4 spaces.
3. **Bold:** Use `**bold**` for key terms, labels, and emphasis within lists.
4. **Italic:** Use `*italic*` for introducing terms or light emphasis.
5. **Code inline:** Use backticks for inline code references: `componentDidMount`, `useEffect`, `<script defer>`.
6. **Line length:** No hard wrapping. Each paragraph is a single line in the markdown source.
7. **Spacing:** One blank line between all sections, list items at the same level have no blank lines between them.

## File Naming

- Use **kebab-case** for file names: `Topic-Name.md` (capitalize each word, hyphens between words)
- Place the file in the appropriate chapter directory
- If a new chapter directory is needed, use **kebab-case**: `New-Chapter-Name/`

## Quality Checklist

Before considering an article complete, verify:

- [ ] H1 title is present and descriptive
- [ ] Introduction defines the concept and its importance (no heading)
- [ ] Core concepts section with numbered, bold-labeled items
- [ ] At least 3 code examples distributed throughout the article
- [ ] Code examples use proper language identifiers and file path comments
- [ ] Good vs bad patterns shown where applicable
- [ ] Benefits section with bold-labeled bullet points
- [ ] Drawbacks/challenges section with bold-labeled bullet points
- [ ] Use cases section
- [ ] Best practices section with actionable items
- [ ] "Common Beginner Doubts or Questions" as the final section with 3-7 Q&A items
- [ ] All formatting rules followed (headings, lists, bold, code)
- [ ] Article is minimum 150 lines, with complex topics being 300-1400+ lines
- [ ] README.md is updated with a link to the new article

## Updating README

After creating a new article, add it to the README.md under the appropriate chapter heading in the Topics section. Use this format:

```markdown
| [Topic Name](./Chapter-Directory/Topic-Name.md) | One-line description of the topic |
```

If creating a new chapter, add a new H3 heading and table in the Topics section.

## Example Workflow

When asked to create a new topic article:

1. Identify the correct chapter directory (or create a new one)
2. Create the `.md` file following the structure above
3. Write all required sections in order
4. Include 3+ code examples with proper formatting
5. End with 3-7 beginner Q&A items
6. Update `README.md` with the new link
7. Run through the quality checklist
