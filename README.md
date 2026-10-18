# flutter-riverpod-template

Clean Architecture template for **office/client Flutter projects** — the docs and
the conventions, not a code generator.

Stack: Riverpod, `freezed`, `dio` / `dio_extended`, `go_router`. Canonized from
working projects, so the rules describe code that already survived real features.

## Use it

Start with [`USAGE.md`](./USAGE.md) — it walks through copying the docs into a
new project, filling the pinned versions, and replacing the `<Feature>` /
`<Thing>` placeholders as each feature gets built.

Then copy `AGENTS.md.template` → `AGENTS.md` at the project root and fill in the
app name. That is the part that keeps AI coding agents (Claude Code, Codex,
Cursor) writing code that matches the architecture instead of inventing one.

## What's in here

- `docs/architecture.md` — layers, folder shape, dependency direction.
- `docs/rules.md` — coding conventions.
- `docs/design.md` — visual system: color, type, spacing, components.
- `AGENTS.md.template` — instructions for AI agents working in the project.

Docs only, on purpose: no app scaffold to drift out of sync. Write `design.md`
for the new project's real screens once they exist — not before.

For the Cubit-based equivalent used in personal projects, see
[flutter-cubit-template](https://github.com/indrakurniawan902/flutter-cubit-template).
