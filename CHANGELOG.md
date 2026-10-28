# Changelog

From 0.4.0-draft, the specs in this repository share one version: the [Ord specification](Ord_Specification.md), [Test format](Test_format.md), [Semantics](Semantics.md), [JSON mapping](JSON_mapping.md), [CSV mapping](CSV_mapping.md) and the informative [Grammar](Grammar.md). Before that, each spec had its own version, listed under its name below.

**Retired IDs.** These rule IDs were removed and are never reused: C17, C18, C20, C28, C39, L9, J13, M20.2, M30.1 to M30.4, M32 to M35.

## Unreleased

### Spec

- L16: `#` and a space is a list marker too, as `-`, `*` and `+` are, so `# Heading` has the label `Heading`. Editors may draw it as a heading.
- L29: `[#Ord notes]` is short for the inline field `[tag:: Ord notes]`, for a tag that is more than a name. `[#1]` stays text, and `[#x](target)` is a Markdown link.

### Semantics

- M24, M36, M46: `options:: +name` replaces `target`. An `options` item `+name` names a type, and a field whose options name types is a reference field to their nodes: `target:: person` is now `options:: +person`. Its other options stay text, and a value that is none of them is a reference instead of the warning `M46`. The `target` setting is gone; `ord migrate` in Ordlang rewrites it.
- M47: `claims:: target` is now `claims:: options`, which claims the field's options and the nodes of the types its options name.
- M19: a type definition's `checkbox:: true` gives the type's nodes a checkbox: an editor that gives a node line the type adds `[ ]` when the node has no state.
- M47: claims match only what is written, never a slug. A field that claims nodes (`claims:: options`, with types among its options) claims a node by its label as written, then by its label without tags and sigils, then by its ID: `#ord-notes` refers to `Ord-notes +project` or to a node with the ID `ord-notes`, `#ord` to `Ord #new +project`, and `[#Ord Notes]` to `Ord Notes +project`, which `#ord-notes` no longer claims. `claims:: options` claims each option as written: `#tech-debt` claims `tech-debt`, and `Tech Debt` needs `[#Tech Debt]`.

## 0.5.0-draft (2026-10-27)

### Type replaces kind, `+` marks a type and `#` a tag

Two renames in one version, with one migration (`ord migrate` in Ordlang rewrites a workspace). There is no transition period: the old forms stop working at once.

- The concept **kind** is now **type**, the word Anytype and Capacities use. The reserved field `kind::` is `type::` (M11), so a workspace's own `type::` fields need another name; the migration renames them. In JSON and CSV the `kind` field and column are `type`, in the graph dump (M72) the `kind` lines are `type`, and in formulas (M41) a node's member `kind` is `type`. "Field types" are now "value types" (M3). Rule IDs stay as they are.
- L7, L13: the `type` trigger is `+` and a name, `+task`; the `tag` trigger is `#` and a name, `#urgent`, as hashtags are everywhere else. `+` reads as "is a": `call #anna to decide on budget #projectX +task`.
- L13: a tag is `#` and a name, with `/` between names for a nested tag (`#project/apollo`). It ends after its last name, so trailing punctuation is text: `#projectX.` is the tag `projectX`. Before, a tag took every character up to the next space. `#` and a space, a digit or another `#` is not a tag, nor is `C#`.
- L4, L19, D14: a tag stays in the label where it is written, like a link, and is an `embedded` field in the dump. So `Call #anna about it +task` has the label `Call #anna about it`, and references, slugs and CSV labels keep the `#`. A `+type` is still taken out of the label, and the writer moves it after the inline fields, before the ID. An inline `[tag:: x]` is no longer written as a trigger, since that would change the label.
- L18: the escapes are `\+` for a type and `\#` for a tag.
- M38: a computed label keeps the tags written on the line, after the computed text.
- M23: a label matches a node by its label or by its label without tags, so `[[Buy milk]]` still finds `Buy milk #urgent`; a sigil matches labels that are one word without their tags.
- M62: `#project` in a query matches nested tags too, such as `#project/apollo`.
- M17, M19: in the workspace file, a line that is only `+name` defines a type, and `#urgent [color:: red]` sets a tag's color. M47: `claims:: #bug #feature`. M61 to M63: queries are `+task`, `#urgent`, `!#done`, and archived nodes match only queries that name `#archived`.

### Spec

- C3, S3: a no-break space (U+00A0) is read as a space everywhere: in the indent, around `::` and `;`, and between forms, so `#tag +type ^id` reads as forms whatever spaces a phone keyboard typed between them. A writer writes a space for it. Other whitespace characters stay content.
- Grammar: the parsing strategy replaces no-break spaces with spaces when it splits lines.

### Semantics

- M36, M46: a field definition's `options` setting lists the values the field is meant to hold. A value that is none of them, ignoring letter case, is the warning `M46` and stays. Options are about intended use, not value type.
- M40: a value that fails its `valid` check is now an error, not a warning.
- M36, M47: a field definition's `claims` setting takes tags for the field: on a node of the type, `#bug` is the field's value `bug`, not a tag. A light way to write a value, beside sigils. `claims:: options` claims the field's options, and `claims:: target` its target types' nodes, by slug or ID.
- M19: a type's `color` can name a field, as in `color:: project`: a node's type then shows in the color of its value there, so the strongest visual cue is about the content.

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
