# Contributing to Learnings Repo

This repository aims to provide "Masterclass-level" documentation for technical concepts. To maintain this high standard, every new file or update **must** follow the established structure and depth.

## 1. Core Philosophy
- **From Scratch to Advanced**: Don't just explain the "how," explain the "why." Cover the internal physics (architecture, storage, memory) of the tool.
- **"Light English"**: Use professional but simple, beginner-friendly language. Avoid overly academic jargon unless you explain it immediately with an analogy.
- **No Placeholders**: Every concept mentioned must be explained in detail. If you mention "MVCC," don't just link to it—explain it.

## 2. File Structure (The Masterclass Template)
Every new documentation file (HTML) should include:
1.  **Sidebar Navigation**: Fixed navigation for jumping between deep-dive sections.
2.  **Modern CSS Design**: Dark-mode primary, using gradients, glassmorphism-lite, and premium typography (e.g., Outfit or Inter).
3.  **Technical Callouts**: Use border-left boxes to highlight "Internal Deep Dives" or "Advanced Secrets."
4.  **Visual Flows**: Integrate **Mermaid.js** diagrams for lifecycles, sequences, or architectures.
5.  **Code & Schemas**: Provide production-ready SQL/code examples, not just snippets.

## 3. How to Add a New Topic
1.  **Research First**: Study the official documentation thoroughly.
2.  **Draft the Sections**:
    - **Foundation**: Basics and "The Choice" (Why use this?).
    - **Architecture/Engines**: How it works under the hood.
    - **Security/Advanced**: Tokens, Encryption, or Scaling.
    - **Modeling/Design**: How to structure data using this tool.
3.  **Use the Template**: Copy the structure from `db_auth_guide.html` or the `templates/master_template.html`.

## 4. Quality Checklist
- [ ] Does it start from absolute zero?
- [ ] Does it reach an "Advanced/Expert" level?
- [ ] are there analogies for complex topics?
- [ ] Is there a Mermaid diagram for flows?
- [ ] Is the CSS premium and responsive?
