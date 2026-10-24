# Changelog

From 0.4.0-draft, the specs in this repository share one version: the [Ord specification](Ord_Specification.md), [Test format](Test_format.md), [Semantics](Semantics.md), [JSON mapping](JSON_mapping.md), [CSV mapping](CSV_mapping.md) and the informative [Grammar](Grammar.md). Before that, each spec had its own version, listed under its name below.

**Retired IDs.** These rule IDs were removed and are never reused: C17, C18, C20, C28, C39, L9, J13, M20.2, M30.1 to M30.4, M32 to M35.

## Unreleased

### Spec

- C3, S3: a no-break space (U+00A0) is read as a space everywhere: in the indent, around `::` and `;`, and between forms, so `+tag #kind ^id` reads as forms whatever spaces a phone keyboard typed between them. A writer writes a space for it. Other whitespace characters stay content.
- Grammar: the parsing strategy replaces no-break spaces with spaces when it splits lines.

## 0.4.0-draft (2026-10-23)

The specs move to their own repository, `ordlang/ord`, as Markdown files, with one shared version.

- Appendices A and B of the Spec (the AST dump and the test format) are the companion spec [Test format](Test_format.md), with their sections and the rule IDs `D` and `T` unchanged.
- Appendix E of the Spec (files) is Part 3, section 21. The rule IDs `F` are unchanged.
- Appendices C and D of the Spec (grammar and parsing strategy) are the informative [Grammar](Grammar.md).
- The changelogs of all the specs are this file.
- S2: a removed rule is deleted, and its ID is listed above as retired. S10: the specs share one version, which can have a pre-release suffix.
- Semantics: its open questions are no longer part of the spec.

### Spec

- C5: the separator is the first `::` outside spans and brackets, whatever follows it. `due::2026-10-10` is a field line; a line no longer turns into a node when the space after `::` is missing.
- C51: brackets, `[` to its matching `]`, hide `::`. C26: a key with `[` and a label with a `::` outside spans and brackets are quoted.
- L5: an inline field is written `[key:: value]`, in a bracket. The value runs to the `]`, so it can hold spaces, and a datetime can be written with a space. `key::value` on a node line is a field line now.
- F1, F3: `.txt` is the usual extension. In a workspace, files without an extension are read as Ord too; `.ord` stays the way to mark a file as Ord outside one.
- L28: a label starting with `??` and a space or tab, after any list marker, is an inline search, short for `search::` with the rest of the label. L4: such a line is written as it is.
- C42: a node with an empty label is written `""`. `::` is written only for a label that is empty and not quoted, which tools no longer create except for the JSON mapping's `null`.
- F5: the workspace file is named `Ord.workspace`, in any letter case, and there is no hidden variant. The `F5` warning for a folder with both names is gone.
- L11, D11: custom markers belong to custom states, which replace statuses in [Semantics](Semantics.md) 0.3.0. Wording only.
- L15, D11: no sigil is bound by default. `@` is no longer bound to `person` when no kind binds it; a kind binds it with `sigil:: @`.

### Semantics

- M17, M19, M61, M62: inline fields and field terms in queries are written `[key:: value]`, after [Ord specification](Ord_Specification.md) L5.
- M19: a kind's `color` setting, and tag settings in the workspace file: a line that is only `+tag`, e.g. `+urgent [color:: red]`.
- M60: a search node can be written `?? query` ([Ord specification](Ord_Specification.md) L28). Several `search` fields on one node are combined with and, and a search without a label is named by its query.
- M5, M17: the workspace file is `Ord.workspace` and is part of the workspace; in it, a line that is only `#name` defines the kind `name`.
- M43: the random ID generator for auto expressions is `$id()`, as `~id` is (Expansions), not `$rid()`.
- M19: no sigil is bound by default; `@` is no longer bound to `person` without a definition.
- Built-in kinds: the `person` ID rule is `auto $slug(label)`, with the `$` every JSONata function call needs (M41). Wording only.
- Built-in kinds: no `sigil` on `person`, and no date kinds. Journal nodes are ordinary nodes found by their ID, `journal-` and the date.

### JSON mapping

- J3, J8, J14: an object without `$ord:label` and a nested array become a node `""` instead of `::`, and an empty label is left out of `$ord:label`. `::` stays for `null` (J4, J11) and an empty nested array, following Spec C42.

### CSV mapping

- V8: an empty label from a cell, or a row without a `label` column, is quoted, so it is written `""`, not `::` (Spec C42).

## Spec, before 0.4.0-draft

### 0.3.0 (2026-10-16)

Simpler Core: fewer quote characters, fewer rules, one block syntax.

- C21, C22, C25: `"` and `` ` `` are the quote characters; `'` is ordinary text. Backticks open spans anywhere; `"` opens a span only where a key, label, value or item starts. The letter and digit rule for spans is gone.
- C29, C31, C32: blocks are backtick fences only, closed by a fence at least as long as the opening one. `"""` and `'''` lines are ordinary text.
- C5: the 60-character limit on keys is gone.
- C16, C17, C18, C28, C38 to C40, C42, C43: empty items, the list marker and list nodes are removed. Empty parts of a value add no item, and a line `;` is a node labelled `;`.
- C20: `\;` is removed; a backslash outside `"` strings is ordinary text.
- C15: in a formula item's groups, `"` and `'` strings are skipped, as in JSONata.
- C26: quoting follows the new spans; keys over 60 characters, items ending in `\` and the label `;` are no longer quoted.
- L2, L9, L18, L24, T4: expansions (`~today`) are removed from the spec; they are an editor feature. Tests no longer run at a fixed time.
- D3, D4, D6, D10, A5: the dump has no `listnode`, `empty` or `list`.
- T1: tests are backtick blocks; a test holding a block uses a longer fence.
- F5: a workspace file is `workspace.ord` or `.workspace.ord`.
- Appendix C is informative.
- From Unreleased: C15, C20, C26: in a formula item, one that starts with `=`, a `;` inside parentheses doesn't separate items, and the writer doesn't quote such an item for it, nor for spans that close within it.

### 0.2.0 (2026-10-15)

- C45: `fmt` keeps a file's line endings and BOM, and every line gets the file's line ending.
- L6, L11: the `state` trigger reads as a state name, so `[ ]` is `state::new` and `[x]` is `state::done`.
- L6: a link reads as itself, brackets included, as in `link::[[anna-47xu]]`.
- L9: dynamic references are now called expansions. In a label, `~now` and generators such as `~uuid` expand to their value.
- L10: in a reference field, a quoted value is a reference only in a field line. In an inline field it stays text, as L5 says.
- D10, D12 to D14: fields read from a label are marked `inline` or `embedded`, items of embedded fields have no name, and a `state` item's range is its marker.
- F5: hidden workspace files come after all others.
- S9, T3 to T5, T7: tests for the companion specs, with the `from-json`, `to-json`, `from-csv` and `to-csv` options.
- T3: `⟨U+XXXX⟩` stands for any character in tests.
- T4: `fmt` with `labels` tests L4. Tests run at a fixed time, and `today:` sets another date.
- Tests for every rule in Parts 1 and 2 that a test can show. C48 and C49 can't be shown with these test options.

### 0.1.0 (2026-10-15)

First numbered version. The spec was rebuilt from scratch and has:

- Core: the data model, reading order, blocks, quoting and spans, lines, tree, fields and items, writing in canonical form, and diagnostics (C1 to C50).
- Label syntax: inline fields, triggers, sigils, and value forms for numbers, booleans, dates, datetimes, weeks and months (L1 to L27).
- The AST dump and the test format, with tests for both parts (D1 to D13, T1 to T7).
- A grammar, a parsing strategy, and rules for files and workspaces (F1 to F5).
- Conventions, versioning and conformance (S1 to S10).

The JSON mapping moved to its own companion spec.

## Semantics, before 0.4.0-draft

### 0.3.0 (2026-10-16)

- M11, M19, M26 to M31: statuses are custom states. A kind's `state` setting defines them, each with an optional marker and an optional base, a built-in state. `status` is no longer reserved, and `M30.1` to `M30.4` are removed: a node has one state, so a marker and a field can't disagree, except as the error `M10`.
- M26: the warning covers custom states, not just the four built-in ones.
- M62: a state trigger in a query matches the base too.
- M72: the graph dump shows `base` instead of `status`.

### 0.2.0 (2026-10-16)

- M6, M7, M9, M20, M32 to M35: follow Spec 0.3.0: no list nodes or empty items, and expansions are a typing aid, not a part of the language, so expansion names are no longer reserved IDs (M20.2 removed).
- Built-in kinds, from Workspace structure, as an informative part of section 5.

- M23: letter case compares by Unicode default lowercase, without normalization.
- M24: without `target`, plain values are text, never references.
- M41: JSONata 2.x is the formula language.
- M14.2: a disagreement between `parent` and `children`, not a one-sided column.
- M20: duplicate IDs resolve to the first; M20.3 for an ID that isn't a name.
- M21, M24: plain values of `link`, and of a field named after a sigil's kind, are references.
- M29: a marker no status has gives no state.
- M71: nodes in another file show their path; node items show their field.
- A third test: ownership through CSV, loops, IDs and quoted labels.

### 0.1.0 (2026-10-15)

First draft. Graph, fields, ownership, kinds, IDs, references, states and statuses, expansions and diagnostics as rules; field definitions, formulas and queries as an outline; the graph dump and two tests.

## JSON mapping, before 0.4.0-draft

### 0.3.0 (2026-10-16)

- J2, J5, J6, J10, J13 to J15: the Spec has no list nodes, empty items or list marker any more. Arrays become node children; nested and root arrays become `::` nodes and root lines, and come back as objects. Empty and one-item arrays come back as `null` and the item.

### 0.2.0 (2026-10-15)

- J1 and the rules that use it: `$ord:value` is renamed `$ord:label`, since a node has a label.

### 0.1.0 (2026-10-15)

First numbered version. The mapping moved out of the Spec into this companion spec, in the layout the companion specs share, with tests in place of the tables.

## CSV mapping, before 0.4.0-draft

### 0.3.0 (2026-10-16)

- V8, V9, V14: follow Spec 0.3.0: `~today` and `;` labels are plain, `\;` is gone, and cells have no list marker, so `a; b;` is two items.

### 0.2.0 (2026-10-15)

- V8: label cells are quoted only where they have to be.

### 0.1.0 (2026-10-15)

First numbered version, in the layout the companion specs share.
