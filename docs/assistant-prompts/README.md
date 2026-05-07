# Assistant Prompts

This directory stores versioned assistant prompt files that should remain recoverable from Git history.

## Files

- `jerome-academic-coach.md`: stable prompt for Jerome's middle-school academic coach persona (`Gordon`)

## Recovery

Recover the latest committed version of a prompt file:

```bash
git checkout origin/main -- docs/assistant-prompts/<file>.md
```

Inspect prompt history:

```bash
git log -- docs/assistant-prompts/<file>.md
git show <commit>:docs/assistant-prompts/<file>.md
```
