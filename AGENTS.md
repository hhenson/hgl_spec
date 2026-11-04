# Specification rules

- Before every push, check the branch's PR state on GitHub. If its previous PR
  is merged, fetch `origin/main`, start a fresh `codex/` branch from it, and carry
  forward only the new changes. Do not push follow-up commits to a merged PR's
  branch; open a new PR for the fresh branch.
- Keep documents and discussions terse. Avoid extended dialogue.
- Apply the agreed [authoring rules](spec/specification-rules.md) and
  [chapter structure](spec/language-structure.md).
- This repository contains specification and explanation documents, plus
  repository metadata. Put no executable HGL, native source, or tooling here.
- Use `spec/` for rules, example code/APIs, and useful diagrams. Add explanations
  under `explanations/` only when justification needs a separate document.
- Label agreed decisions and proposed behavior. Keep open questions and decision
  checklists in this repository's issue tracker, outside specification and
  explanation documents. Implementation support does not determine acceptance.
- Define language meaning and runtime concepts here. Reference the mandatory
  [core contract](https://github.com/hhenson/hgl_core) for required API behavior;
  do not duplicate its rules.
- Keep target symbols, build details, implementation status, historical context,
  and experiment records outside this repository. It must stand alone.
- Change a rule and its affected examples together. Preserve stable rule IDs.
