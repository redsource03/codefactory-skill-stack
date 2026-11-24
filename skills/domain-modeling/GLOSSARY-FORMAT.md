# GLOSSARY FORMAT

Each entry is a YAML frontmatter block followed by a plain-text definition:

```markdown
---
term: <term>
category: <category>
---

<definition in plain English, no implementation details>
```

- `term`: the canonical term (title case)
- `category`: one of `domain`, `process`, `actor`, `event`, `state`, `artifact`, `constraint`
- The body is the definition only — no examples, no implementation notes

Add entries in alphabetical order by `term`. If the term already exists, update its entry in place.
