# Using this template

This folder is a **reusable architecture template** — Clean Architecture +
Riverpod + `freezed` + `dio`/`dio_extended` + `go_router` — not documentation
tied to one specific app. To start a new Flutter project with it:

1. Copy `architecture.md`, `rules.md`, and `design.md` into the new
   project's `docs/` folder.
2. Update each doc's opening blockquote (app name/description, if you add
   one) to match the new project.
3. Fill in `architecture.md §1 Stack` with the actual pinned package
   versions from the new project's `pubspec.yaml`, and drop the optional
   rows (Firebase, realtime) the project doesn't use.
4. Replace `<Feature>`/`<Thing>` placeholders in the code samples with real
   names as each feature gets built — they aren't meant to stay as literal
   identifiers.
5. Write `design.md` for the new project's actual visual system (colors,
   typography, spacing, components) once real screens exist — don't leave
   it blank past that point.
6. Copy `AGENTS.md.template` to the new project's root as `AGENTS.md`, and
   fill in the app name at the top.

For the Cubit-based equivalent of this template, see the `harverst_moon_guide/docs/`
folder.
