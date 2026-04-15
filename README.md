# Frontend System Design

A structured, in-depth knowledge base covering frontend system design concepts -- from architectural patterns and performance optimization to security and API design. Each topic includes clear explanations, real-world code examples across React/Vue/Angular, best practices, and a dedicated Q&A section addressing common beginner doubts.

## Topics

### Frontend Architectural Patterns

| Topic | Description |
|-------|-------------|
| [Component-Driven Development](./Frontend%20Architectural%20Patterns/Component-Driven%20Development.md) | Bottom-up UI development with isolated, reusable components |
| [Smart vs Dumb Components](./Frontend%20Architectural%20Patterns/Smart%20vs%20Dumb%20Components.md) | Separating logic-heavy containers from presentational components |
| [Container vs Presentational Pattern](./Frontend%20Architectural%20Patterns/Container%20vs%20Presentational%20Pattern.md) | Data-fetching containers paired with pure rendering components |
| [Atomic Design](./Frontend%20Architectural%20Patterns/Atomic%20Design.md) | Hierarchical UI composition: atoms, molecules, organisms, templates, pages |
| [State Management Strategies](./Frontend%20Architectural%20Patterns/State-Management-Strategies.md) | Global state (Redux/Context), local state (useState/useReducer), server state (React Query/SWR) |
| [Code Splitting & Lazy Loading](./Frontend%20Architectural%20Patterns/Code-Splitting-Lazy-Loading.md) | On-demand loading to reduce initial bundle size |
| [Modular CSS](./Frontend%20Architectural%20Patterns/Modular-CSS.md) | Scoped styling approaches: CSS Modules, CSS-in-JS, BEM, utility-first |

### Performance Optimization

| Topic | Description |
|-------|-------------|
| [Critical Rendering Path](./Performance-Optimization/Critical-Rendering-Path.md) | How the browser turns HTML/CSS/JS into pixels, and how to optimize each step |
| [Tree Shaking](./Performance-Optimization/Tree-Shaking.md) | Dead code elimination with Webpack, Rollup, and Vite |
| [Web Vitals](./Performance-Optimization/Web-Vitals.md) | Core Web Vitals (LCP, FID, CLS) -- measuring and improving real-world performance |

### Frontend Security

| Topic | Description |
|-------|-------------|
| [Cross-Site Scripting (XSS)](./Frontend-Security/Cross-Site-Scripting-XSS.md) | XSS attack vectors, prevention techniques, and CSP |
| [Cross-Site Request Forgery (CSRF)](./Frontend-Security/Cross-Site-Request-Forgery-CSRF.md) | CSRF attack flows, token-based defenses, and SameSite cookies |

### API Integration & Data Handling

| Topic | Description |
|-------|-------------|
| [REST vs GraphQL](./API-Integration-Data-Handling/REST-vs-GraphQL.md) | Comparison of REST and GraphQL: architecture, implementation, and migration strategies |
| [WebSockets for Real-Time Apps](./API-Integration-Data-Handling/WebSockets-For-Real-Time-Apps.md) | WebSocket protocol, connection management, reconnection patterns, and real-time architecture |
| [Pagination, Filtering, and Sorting](./API-Integration-Data-Handling/Pagination-Filtering-Sorting.md) | Offset vs cursor pagination, client/server filtering and sorting, URL-synced state, infinite scroll |
| [File Upload Patterns](./API-Integration-Data-Handling/File-Upload-Patterns.md) | Upload strategies, chunked/resumable uploads, presigned URLs, progress tracking, and client-side file handling |
| [API Retries, Timeouts, and Backoff Strategies](./API-Integration-Data-Handling/API-Retries-Timeouts-Backoff-Strategies.md) | Retry mechanisms, timeout configuration, exponential backoff with jitter, and circuit breaker patterns |

## Content Format

Every topic follows a consistent structure:

1. **Title & Introduction** -- What the concept is and why it matters
2. **Core Concepts** -- Numbered/bulleted breakdown of key principles
3. **Detailed Sections** -- How it works, benefits, drawbacks, use cases, best practices
4. **Code Examples** -- Practical snippets in JavaScript/TypeScript/JSX/CSS with good-vs-bad patterns
5. **Common Beginner Doubts** -- Q&A section addressing frequent misconceptions

## AI-Assisted Content Generation

This repository includes a `SKILL.md` file that defines a content generation skill for AI coding agents. The skill ensures new topics are generated in the same format and quality as existing content. See [`CLAUDE.md`](./CLAUDE.md) and [`AGENTS.md`](./AGENTS.md) for agent configuration.

## License

This repository is intended for learning purposes. Feel free to use it as a reference for your own frontend system design studies.
