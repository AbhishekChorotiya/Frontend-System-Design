# AI Agent Reference File for Frontend System Design Learning

This document outlines the roles, responsibilities, and operational guidelines for the AI agent assisting with the "frontendSystemDesign" learning repository. The agent's primary goal is to facilitate the user's learning by generating structured, detailed, and accurate content on frontend system design topics.

## Agent's Core Responsibilities:

1.  **Chapter and Topic Structure Management:**
    *   **Create Chapter Folders:** For each new chapter outline provided by the user (e.g., "Performance Optimization"), the agent will create a new directory. The folder name should be a sanitized, hyphenated version of the chapter title (e.g., `Performance-Optimization`).
    *   **Create Topic Markdown Files:** Within each chapter folder, for every topic specified (e.g., "Code Splitting"), the agent will create a Markdown (`.md`) file. The filename should be a sanitized, hyphenated version of the topic title (e.g., `Code-Splitting.md`).

2.  **Content Generation for Topic Files:**
    *   **Comprehensive Explanations:** Each topic Markdown file must provide a thorough and in-depth explanation of the subject matter. This includes definitions, core concepts, benefits, drawbacks, use cases, and best practices.
    *   **Illustrative Code Examples:** Include relevant, functional, and well-commented code examples (e.g., JavaScript, TypeScript, React, Vue, Angular, HTML, CSS) to demonstrate practical application of the concepts. Code blocks should be appropriately formatted with language identifiers.
    *   **Attention to Detail:** Ensure meticulous coverage of all aspects of the topic, anticipating potential user questions and addressing them proactively.
    *   **Clarity and Readability:** Content should be well-organized, using headings, subheadings, bullet points, and other formatting to enhance readability and understanding.

3.  **README.md Updates:**
    *   After creating new chapter folders and topic files, the agent must automatically update the main `README.md` file of the repository.
    *   The update should include a hierarchical list of all chapters and their respective topics.
    *   Each chapter and topic in the list must be a clickable link pointing to the corresponding folder or Markdown file.
    *   Maintain a consistent format for the table of contents in the `README.md`.

4.  **Version Control (Git Workflow):**
    *   After successfully generating new content and updating `README.md`, the agent will:
        *   Propose the following Git commands to the user:
            *   `git add .` (or specific files/folders if appropriate)
            *   `git commit -m "docs: Add content for Chapter X, Topic Y"` (with a descriptive commit message)
            *   `git push origin main` (or the relevant branch)
        *   Request explicit user confirmation before executing these commands. The agent should not run these commands without approval.

## Expected Input from User:

*   **Chapter and Topic Outlines:** The user will provide a list of chapters and, for each chapter, a list of topics to be covered.
    *   Example:
        *   Chapter: Performance Optimization
            *   Topic: Lazy Loading
            *   Topic: Debouncing and Throttling
            *   Topic: Code Splitting
*   **Clarifications and Feedback:** The user may provide clarifications on requirements or feedback on generated content.

## Quality Standards for Generated Content:

*   **Accuracy:** All technical information, explanations, and code examples must be accurate and up-to-date.
*   **Depth:** Content should go beyond surface-level explanations, providing deep insights and comprehensive coverage.
*   **Practicality:** Focus on real-world applicability, providing examples and scenarios that a frontend developer might encounter.
*   **Originality:** While drawing from a vast knowledge base, the generated content should be synthesized and presented in a unique and instructive way, not a direct copy from other sources.
*   **Consistency:** Maintain consistent terminology, formatting, and style across all generated files.

## How to Use This Context:

In any new chat session or interaction pertaining to the "frontendSystemDesign" repository, this `agent-reference.md` file should be considered the primary source of truth for the agent's tasks and responsibilities. The agent is expected to refer to this document to understand its role without requiring repeated instructions from the user.

## Updating This Context:

The user may update this `agent-reference.md` file to modify, clarify, or expand the agent's responsibilities or guidelines. The agent must always adhere to the latest version of this document.

---
