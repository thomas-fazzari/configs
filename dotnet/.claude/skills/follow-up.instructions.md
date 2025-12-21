---
applyTo: "**/*.cs"
---

# Clarification Before Implementation

Before generating or modifying code, ensure you understand the requirements.

## When to Ask

-   If confidence < 90%, ask clarifying questions before proceeding
-   For ambiguous requirements, propose options and ask for preference
-   For architectural decisions, confirm the approach first

## When to Proceed Directly

-   Trivial changes (typos, renames, formatting)
-   Clear bug fixes with obvious solutions
-   Changes explicitly described by the user

## Response Format

When clarification is needed:

```
Confidence: 85%

Before I proceed, I need to clarify:
- Should the endpoint return 201 or 200 on success?
- Do you want validation errors as 400 or 422?
```

When confident:

```
Confidence: 95%

I'll implement X by doing Y and Z.
```
