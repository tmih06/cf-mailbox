# Agent guidelines

## Code design

- **KISS:** choose the simplest design that satisfies the current requirements. Prefer explicit control flow, small cohesive functions, and existing repository patterns.
- **DRY:** keep each business rule and source of truth in one place. Extract shared logic when its callers share the same meaning and reason to change; keep coincidentally similar code separate.
- **YAGNI:** implement the requested behavior and its required edge cases. Add abstractions, dependencies, configuration, and extension points only for demonstrated needs.

## Function contracts

Document every new or modified function and method with a comment immediately above its declaration, using the language's documentation-comment format. Document anonymous callbacks at their definition when they carry behavior beyond a simple forwarding expression.

Write comments as contracts for future refactoring and optimization:

- State the function's purpose and why the caller needs it.
- Describe input constraints, output guarantees, and meaningful boundary cases. Explain parameter semantics that names and types cannot express.
- Record applicable side effects, state mutations, failure behavior, and ordering or concurrency requirements.
- Identify invariants and the reasons for non-obvious choices, including security or performance constraints that a replacement must preserve.

Cover every applicable part of the contract; a trivial function may need only one precise sentence. Describe observable behavior and rationale rather than narrating implementation steps or repeating types. Keep algorithm-specific explanations beside the relevant code. Update comments with behavior changes; for behavior-preserving refactors, use the contract to check that intent and guarantees remain intact.

## Conventional Commits

Use Conventional Commits for every commit message:

```text
<type>[optional scope][!]: <description>
```

- Choose the type by intent: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, or `revert`.
- Use a short, imperative description; use a scope when it identifies the affected area.
- Mark breaking changes with `!` before the colon or a `BREAKING CHANGE:` footer, and explain the incompatibility and required migration in the message.
- Keep each commit focused on one coherent change. Use the body for rationale and context that the subject cannot convey.

<!--VITE PLUS START-->

## Tooling and verification

Use the project-local Vite+ toolchain through `bun x vp` and the scripts defined in `package.json`. Consult `package.json` and `vite.config.ts` for the commands and tasks relevant to the change.

`vp <name>` selects a built-in command; `vp run <name>` selects a project script or task. These can differ even when the names match. Use `vp run <name>` for project-defined scripts and tasks.

Before handing off code changes, run the relevant checks and exercise the changed behavior. Report the commands run, their results, and any checks blocked by the environment. Completion requires every requested behavior to be accounted for and every changed function's contract to match its implementation.

For Vite+ command usage, consult `bun x vp <command> --help`; for toolchain behavior or configuration, read `node_modules/vite-plus/docs` or https://viteplus.dev/guide/.

<!--VITE PLUS END-->

## Workflow references

- **Issues:** before reading, creating, or updating tickets or specs, or managing wayfinder maps and dependencies, read [the issue-tracker workflow](docs/agents/issue-tracker.md).
- **Triage:** before assigning issue readiness or disposition labels, read [the label mapping](docs/agents/triage-labels.md).
- **Domain:** before exploring a subsystem, naming domain concepts, or proposing architectural changes, read [the domain-documentation workflow](docs/agents/domain.md).
