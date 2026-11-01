# Tasks

- [ ] Bump the `xulo-spec` submodule pointer to `f885d74` (or later) in the
  same commit that folds this change: the proposal cites the accepted
  `change-tostring-to-display` commit, and the working tree currently
  records `M xulo-spec`.
- [ ] `signature-index.md` — no edit: `std::fmt::format`'s signature is
  unchanged.
- [ ] [`modules/README.md`](../../modules/README.md) — no edit: the module
  face and the well-formedness row are unchanged.
- [ ] [`README.md`](../../README.md) § 1.3 — no edit: the `any` dependency
  (item 2) stands as written; the Display/Debug/Format language change this
  proposal relies on has already landed and is not one of the five pending
  items.
- [ ] Fold [`specs/fmt.md`](specs/fmt.md) into `modules/fmt.md` whole or
  not at all (the workflow's rule 2), then archive this directory with
  status `accepted` and drop the index row.
- [ ] After folding, re-read the chapter end to end: the intro's "layout
  vocabulary" sentence, the member table, and the failure-model prose must
  still agree with the new dispatch (they should — only the sections the
  delta names change).
