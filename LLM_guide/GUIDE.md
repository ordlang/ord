# Writing Ord: a guide for LLMs

Version 0.6.0-draft, 2026-11-03.

This guide is informative. It tells a language model, or a tool that writes for one, how to read and write Ord so that its edits look like something a careful person typed. It follows the specs of the same version: [Ord specification](../Ord_Specification.md) (rules C, L, F), [Semantics](../Semantics.md) (rules M), [CSV mapping](../CSV_mapping.md) (rules V) and [JSON mapping](../JSON_mapping.md). Rule IDs are given so you can look them up. Where this guide and a spec disagree, the spec wins, and the guide has a bug.

It is released with the specs: a change to a spec that changes how Ord is written changes this guide and the brief in the same commit. The brief is also published at https://ordlang.org/brief.md, built from this folder. Tools that copy it, such as a Claude skill, pin it by version and check that their copy is unchanged.

Read [Ord in brief](brief.md) first: the syntax on one page, with the basic rules for writing and right and wrong examples. This guide builds on it, and doesn't repeat it: what a workspace means, how to edit files well, formulas, queries and the mistakes models make.

More detail, loaded when needed:
- [references/formulas.md](references/formulas.md): the formula data model, every function, dates in formulas, examples.
- [references/queries.md](references/queries.md): query terms, search nodes, view settings.
- [references/csv.md](references/csv.md): CSV files in a workspace.
- [references/diagnostics.md](references/diagnostics.md): what each diagnostic code means and how to fix it, and mistakes that give none.
- [references/recipes.md](references/recipes.md): worked examples: a workspace, journal days, tasks, meetings, a small database.

## What a workspace means

- **Files (F1 to F5).** A folder with an `Ord.workspace` file is a workspace. In it, `.txt`, `.ord` and extensionless files are Ord, and `.csv` files are nodes, one row each. Name new files `.txt`. The workspace file usually holds the definitions, by convention only; a definition works in any file.
- **Type definitions (M17, M19, M53).** A `+type-def` node with a `name` defines a type:
  ```
  Task +type-def
  	name:: task
  	color:: blue
  	checkbox:: true
  	state:: review [r] > in-progress | shipped > done
  	fields::
  		due::
  		owner::
  			options:: +person
  		project::
  			options:: +project
  			tags:: true
  ```
  Settings: `name`, `fields`, `state`, `color` (a color name, `#rrggbb`, or a field's name such as `color:: project`), `checkbox`, `nodes` (a template copied under new nodes of the type, M51), `icon` and `description`. Prose children are documentation. There is no inheritance; a node can have several types, and a type without a definition is allowed.
- **Tag definitions (M59).** `Urgent +tag-def` with `name:: urgent` and `color:: red` under it.
- **Field definitions (M36 to M47, M55 to M58).** Each field line under `fields::` defines a field for the type. Its value is its `value` setting:
  - `=expr` makes a computed field; `auto expr` an auto value, written once when the node gets the type and the field is empty; anything else is a default, shown but never written.
  - `options` lists the values the field is meant to hold (another value is the warning M46). An item `+name` makes it a **reference field** to nodes of that type (M24): `options:: +person`.
  - `tags:: true` lets the options be written as tags (M47): with `options:: bug; feature`, `Crash +issue #bug` has `issue type:: bug`. On a reference field, a tag names a node of the type by its label as written, then without its tags, then by ID: `#acme` on a task refers to `ACME +project`.
  - `valid` is a check, `=expr` with `$value`; a failure is the error M40.
  - A field used by several types can be a `+field-def` node with a `key`, defined once for the workspace; a type then lists it by key with no value (M58).
- **Value types are never declared.** They are inferred from the values.
- **System fields (M11).** `_label`, `_type`, `_id`, `_state`, `_tag`, `_link`, `_mirror`, `_formula`, `_url`, `_parent`, `_children`. `_label`, `_id`, `_state` and `_mirror` hold one value each; two is the error M10. A type may compute `_label` and give `_id` an auto rule, and nothing else about them.
- **IDs (M20).** Unique in the workspace, matched ignoring case. Short and readable: `acme`, `i42`. Nodes only need one when something refers to them by ID.
- **References (M21 to M24).** Links, `_link` and `_mirror` values, mirror values (`attendees:: @Anna Smith`), plain values of a reference field, tags on a reference field with `tags:: true`, and CSV `_parent`/`_children` cells. Plain values of any other field are text, even when they name a node.
- **Resolving (M22, M23).** Text that could be either matches an ID first, then a label, ignoring case and with or without tags. A label that matches no node, or several, doesn't resolve (warnings M23.1, M23.2). `[[^id]]` is ID only, `[["Label"]]` label only, `[[Label^id]]` both, and the ID wins.
- **Mirrors (M48 to M50).** `@ Label` on its own line shows that node, its source, in this place without moving it; the source keeps its one owner. Use it to list a node in a second place, not a line that is only `[[Label]]`. Write nothing under a mirror (M50.2): its children are its source's.
- **States (M26 to M31).** Built in: `new`, `in-progress`, `waiting`, `done`. A type's `state` setting adds custom states, `name [marker] > base`, with `|` between them. A node is done when its base is `done`.
- **Ownership (M13 to M16).** A node's owner is the node it is indented under, or a CSV `_parent` cell. Ownership never loops.
- **Searches (M60 to M65).** A `+search` node with a `query` setting, or a line `? query`. See [references/queries.md](references/queries.md).
- **Journal nodes.** Day nodes are ordinary nodes found by ID, not by place: `Monday 12, 2026 ^journal-2026-10-12`, under a week (`^journal-2026-W42`) or a month (`^journal-2026-10`). Which file holds them is up to the tool; see [references/recipes.md](references/recipes.md).
- **No expansions.** `~today` and the like are typing aids, not syntax. In a file, write the value: `due:: 2026-10-23`.

## Writing and updating

**Edit, don't rewrite (C41).** Change only the lines the change needs. Every other line, blank lines and odd spacing included, stays byte for byte. Never reformat a file the user didn't ask to format.

**Place lines where a writer would (C44, C48).**
- Indent a new line like a sibling next to it, else one step deeper than its parent.
- A new field goes after the node's last field, or right after the node line. A new child node goes after the node's last child and its lines.
- Keep the file's line endings, BOM and final newline. A new file uses `\n`, ends with a newline and has no BOM.

**Quote only when needed (C26, C27).** Quote a value or item when it must stay empty, starts or ends with whitespace, starts with `"` or a backtick, contains a backtick, `;` (outside a formula's parentheses) or a control character, or is a number, date or boolean that must stay text. Quote a key with `::` or `[`, or ending with `:`. Quote a label with `::` outside brackets.

**Choosing the shape:**
- **Field or prose.** Fields are for values you would sort, filter or compare on: a date, an owner, a status, an amount, a project. Notes and details are child lines: under a meeting, `Ask about the budget`, not `note:: Ask about the budget`.
- **Inline or field line.** Short values inline, `[due:: 2026-10-20]`; long values and several items as field lines.
- **Type or tag.** A type (`+task`, "is a task") says what a node is and brings fields. A tag (`#urgent`) is a loose label. A nested tag (`#client/ruukki`) for a family of tags.
- **Tags over fields.** When a field has `tags:: true`, write its value as a tag: `#acme` rather than `[project:: acme]`, `#bug` rather than `issue type:: bug`. If the workspace doesn't have `tags:: true` on that field yet, write the field, and suggest adding the setting.
- **Project tag on typed nodes.** When a type has a `project` reference field with `tags:: true`, give each of its nodes the project's tag, even under the project node: `[ ] Send mapping draft #acme [due:: 2026-10-26] +task`. Use the project's label or ID as written.
- **Define types when fields repeat.** Several nodes with the same fields get a type. Don't invent a type for a single node.
- **References, not copies.** Link to a node, or mirror it, rather than copying its text. Reference by label when it is unique; give the target an ID and reference that when it isn't, or may change.
- **No IDs by default.** Add `^id` only when a reference by label wouldn't resolve or a tool must address the node by ID. Journal nodes always get their `^journal-…` ID.
- **Long text.** One paragraph per child line. Code and tables in a fenced block.

**Never write computed results into files.** Computed fields, search results and inline formula results are shown by tools, not stored. The exceptions are a computed `_label`, which an editor writes into the node line, and `auto` values, written once. `key:: =expr` on an ordinary node is the error M39: formulas belong on definitions.

**Common updates:**
- **Complete a task.** `[ ]` to `[x]`. To add a checkbox, put `[ ] ` after any list marker.
- **Set a custom state.** Its marker (`[r]`) when it has one, else a `_state:: review` field line. Never two states on one node.
- **Archive, never delete,** when the user means "remove from views": add `#archived`. Queries skip archived nodes unless they name `#archived`.
- **Move a node** with its whole subtree, re-indented to its new depth.
- **Rename a node.** References by ID keep working. References by label (links, reference field values, mirrors, tags on reference fields) must be updated: search the workspace for them. Adding or removing a tag doesn't break them.
- **Add or rename an ID.** Pick one that isn't used. Renaming updates every `^id`, `[[…^id]]`, mirror and reference value that uses it.
- **Change a type's fields.** Edit the definition only. Nodes keep their values.
- **Add to a day.** Find the day node by its ID (`^journal-2026-10-22`) and add the line under it. Create it, in date order, if it is missing.

## Formulas in brief

A formula is a JSONata 2.x expression written `=(…)` (M41, L12). Always wrap it in parentheses: `=$sum(x)` is text. `$` is the current node, bare names are its fields, and system fields have their `_` keys: `_children`, `_parent`, `_state`, `_type`, `_label`. Dates are ISO text: compare with `<`, shift with `$date(due, 7)`, count with `$days(a, b)`, never `due + 7`.

```
Invoice +type-def
	name:: invoice
	fields::
		client::
			options:: +company
		net:: =($sum(_children.amount))
		gross:: =(net * 1.25)
		late:: =($exists(due) and due < $today() and _state != "done")
		code:: auto "INV-" & $seq(3)
		vat::
			valid:: =($value in [0, 0.12, 0.25])
```

Full rules and functions: [references/formulas.md](references/formulas.md).

## Searches in brief

A query looks like the node lines it matches: `+task`, `#urgent`, `[ ]`, `[[Ruukki AB]]`, `[due:: 2026-10-10]`, `=(late)`, plain words. Space is and, `|` or, `!` not, parentheses group.

```
Open tasks +search
	query:: +task ![x]
? [ ] +task #acme
```

Views, columns and the rest: [references/queries.md](references/queries.md).

## Gotchas

Things that read fine but mean something else, or that models get wrong. Add to this list, with the version, when a new one turns up.

- `due: 2026-10-10` (one colon) is a node, not a field. (0.4)
- `Call note:: two words` makes the whole line a field with the key `Call note`. Write `Call [note:: two words]`. (0.4)
- `summary:: we agreed X; then Y` is two items. Quote it, or make it a child line. (0.4)
- `+urgent` is a type and `#task` a tag: the reverse of Ord before 0.5. (0.5)
- `[defines:: task]`, `target::`, `claims::`, `sigil::`, `search::` and `?? query` are gone. Write `+type-def` with `name::`, `options:: +type`, `tags:: true`, a link, `+search` with `query::`, and `? query`. (0.6)
- `@anna` in a label is text. `@ Anna` alone on a line is a mirror; `@Anna` as a field item is a reference. (0.6)
- `state::`, `type::`, `id::` and `label::` are ordinary fields. The system fields are `_state` and the rest. In formulas, write `_children`, `_parent` and `_state`, not `children`. (0.6)
- In `Ord.workspace`, a line that is only `+task` defines nothing. (0.6)
- `# Heading` is a node labelled `Heading`, not a tag and not a section by itself. (0.6)
- `days-left` in a formula is `days - left`; write `` `days-left` ``. (0.4)

## Before you finish

- Every new field line has `::` and a space, every inline field is in brackets, and no prose has a bare `::`.
- Indentation matches the file's, and each new line sits under the right parent.
- Node lines keep the fixed order: state, label with its tags, inline fields, types, ID.
- Values with `tags:: true` are written as tags, and typed nodes with a project field carry their project's tag.
- Types are `+name`, tags `#name`; definitions are `+type-def`, `+tag-def` and `+field-def` nodes; nothing from the Gotchas list slipped in.
- New references resolve: the label is unique, or you used an ID. New IDs are unused.
- No formula sits on an ordinary node's field line, and no computed result was written.
- Lines you didn't need to change are unchanged.
- If a checker is available, run it, and fix every error you caused.
