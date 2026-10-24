# Change Proposals

This directory tracks proposed changes to the Xulo Standard Library
Specification in the OpenSpec style. A proposal describes the specification as
it *would* read after the change; it does not amend the chapters themselves.
Until a change is accepted and folded in, the main chapters remain normative:
a construct that appears only in a proposal is not part of the standard
library, and where a proposal and a chapter disagree, the chapter governs.

A change to this repository never adds syntax: grammar and diagnostics belong
to the [language specification](https://github.com/xulo-lang/spec), which is
changed first; the change here then cites it. A proposal whose adoption
depends on a language change MUST name that change in its `proposal.md`.

## Workflow

Every change passes through the same stages:

- **Draft.** The change directory exists with the three required artifacts —
  `proposal.md`, `design.md`, and at least one file under `specs/` — and the
  [index](#index) records it with status `draft`. A draft MAY be edited freely.
- **Review.** The proposal is being evaluated: the alternatives in `design.md`
  are weighed against the chosen design, the deltas in `specs/` are checked
  against the chapters they claim to replace, and the scope is confirmed. The
  index records the status `review`.
- **Accepted.** The deltas are folded into the chapters they name, so that the
  replacement text becomes chapter text; the index entry is dropped and the
  directory is moved to `archive/` with status `accepted`.
- **Rejected.** The directory is moved to `archive/` with status `rejected`,
  recorded by a one-line reason in its `proposal.md`, and is never folded in.

Two rules hold at every stage:

1. **Unaccepted changes MUST NOT contradict the main chapters.** The current
   chapters are normative until acceptance; a proposal states its deltas as
   future text, never as present fact.
2. **A change is folded in whole or not at all.** Partial folding would leave
   the chapters inconsistent with one another, so a change that cannot be
   applied completely stays in draft.

## Directory layout

Each change lives in one directory named after its change ID:

```text
changes/
├── README.md                 workflow, artifact rules, change IDs, index
└── <change-id>/
    ├── proposal.md           required: what the change does and why
    ├── design.md             required: alternatives, choice, rationale
    ├── specs/                required: normative deltas
    │   └── <chapter>.md      one file per affected chapter
    └── tasks.md              optional: the edits an acceptance requires
```

- Change IDs are **kebab-case** (`add-std-glob-with-exclusion`), name the
  change rather than its motivation, and are never reused.
- `proposal.md`, `design.md`, and `specs/` are required, and `specs/` MUST
  hold at least one file. `tasks.md` is optional.
- A file under `specs/` is named after the chapter it modifies — `json.md`
  for [`../modules/json.md`](../modules/json.md) — so that a reader can match
  every delta to its chapter at a glance. A change to the shared conventions
  in [`../modules/README.md`](../modules/README.md) uses `README.md`.

## Artifact rules

| Artifact | Contents |
|----------|----------|
| `proposal.md` | **What** the change adds, removes, or alters; **why** it is wanted, argued from examples; the **scope** — what is in and what is out; any dependency on the language specification. Carries the status metadata and no normative text. |
| `design.md` | The alternatives considered with their costs, the design actually chosen, and the rationale for choosing it, including the signature, semantics, and failure-model consequences. |
| `specs/` | The exact normative deltas: post-acceptance replacement text, one file per affected chapter. |
| `tasks.md` | Optional checklist of the cross-references and neighbouring passages — `signature-index.md`, `modules/README.md`, other module chapters — an acceptance must update. |

Each file under `specs/` follows one convention. Its first heading is the
target chapter's own heading followed by a delta marker, and the line after it
is a blockquote naming the file the delta replaces:

```markdown
# `std::json` — Delta: Preserve Key Order

> **Delta** — replaces the “Object” paragraph in `modules/json.md`.
```

The `> **Delta** — …` blockquote is required and MUST name the target file.
The body then gives the post-acceptance text in full, so that folding the
change in is a matter of copying the body into the chapter it names.

## Change ID conventions

A change ID begins with a verb phrase that states the direction of the edit:

| Prefix | Use |
|--------|-----|
| `add-` | a member, module, rule, or chapter that does not exist yet |
| `remove-` | a member, module, rule, or chapter that is deleted |
| `change-` | a member, module, rule, or chapter that is altered in place |

The remainder of the ID is kebab-case and names the library element touched,
never a version number or an implementation detail.

## Index

| Change | Status | Summary |
|--------|--------|---------|

## Archived

Accepted and rejected changes leave the index above and are kept in
`archive/` with their final status.

| Change | Status | Summary |
|--------|--------|---------|
