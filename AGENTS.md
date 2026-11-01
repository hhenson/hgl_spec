# Specification rules

- Keep documents and discussions terse. Avoid extended dialogue.
- This repository contains specification and explanation documents, plus
  repository metadata. Put no executable HGL, native source, or tooling here.
- Use `spec/` for rules, example code/APIs, and useful diagrams. Add explanations
  under `explanations/` only when justification needs a separate document.
- Label agreed decisions, proposed behavior, and open questions separately.
  Implementation support does not determine acceptance.
- Define language meaning and runtime concepts here. Reference the mandatory
  [core contract](https://github.com/hhenson/hgl_core) for required API behavior;
  do not duplicate its rules.
- Keep target symbols, build details, implementation status, historical context,
  and experiment records outside this repository. It must stand alone.
- Change a rule and its affected examples together. Preserve stable rule IDs.
