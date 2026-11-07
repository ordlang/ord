# Ord semantics

Version 0.6.0-draft, 2026-11-03. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D`, `T` or `F` refer to the Spec. This spec's rules start with `M`.

The Spec says how text becomes a tree of nodes and fields. This spec says what that tree means across a workspace: which node a reference points to, what types and states a node has, who owns whom, and what formulas and queries written in files stand for.

## 1. Scope and the line

- **M1.** This spec covers what files mean. Two tools that conform to it give a workspace the same graph: the same nodes, owners, types, IDs, states, field values and references, and the same diagnostics from section 10.
- **M2.** A rule belongs here when it is about something written in a file that a tool interprets, or something a tool writes into a file. Anything that exists only in a tool, such as an index, a view, completion or a heuristic whose result isn't written, is implementation and outside this spec.
- **M3.** These are explicitly implementation-defined, so tools may differ:
  - how value types are inferred, and outlier warnings; only value forms (L21 to L27) are normative
  - diagnostics beyond section 10, which use codes outside this spec's letters, e.g. `ord/outlier`
  - how and when a graph is stored, cached or rebuilt
  - completion, hover, inlay hints, views, CodeLens and results documents
  - fold state and other view state, which is never written into the files: a tool keeps it apart, finding a folded node again by its ID or its place in the tree, and a node it can't find stays open
  - merging and sync
- **M86.** **Files, and what they define.** A workspace has three layers:
  - **The files** are the source of truth: a node has what is written on it (M37). A tool changes a file only to help the person writing it: it writes a one-time formula's result in its place (M85), a type's fields, template and checkbox on a node given the type (M37, M51, M66), and a change to a type's fields on its nodes (M84), and makes the edits the person asks for, such as renaming a node or a field along with what refers to it.
  - **What the files define**: searches (M60), mirrors (M48) and formulas (M39, M87). They are written in the files and evaluated over the graph, and their results are virtual: an editor shows them where they are defined, and a tool never writes them into a file. They are one layer because they read each other: a query can match a calculated field (`=(total > 42)`), a mirror stands for its source in formulas (M49), and a formula can run a query (M76).
  - **What is computed from a search's results**: its `footer` and `computed` settings (M64). They are shown with the results, and nothing else reads them.
- **M4.** A **graph builder** reads a workspace (Spec F4) with label syntax (Spec Part 2) and builds the graph as this spec says, and passes the tests with the `graph` option (Appendix A).

## 2. The graph

- **M5.** The **workspace** is every Ord and CSV file in it, its workspace file included (Spec F1, F3, F5; CSV mapping). Files are taken in code point order of their paths relative to the workspace folder.
- **M6.** Every node in every file is a **graph node**: each file's root (C14), each node and anonymous node (C13), and each row of a CSV file. A graph node has a file and a position, and:
  - a **label**: the L19 label for a node Part 2 reads, the Core label for any other node, and none for anonymous nodes
  - **fields**: a list of (key, value) pairs, from field lines, inline fields and triggers (section 3)
  - an **owner**, section 4
- **M7.** A field value is a text item, a node item (whose node is a graph node), or a value trigger read from an item (L20).

## 3. Fields

- **M8.** A node's fields come from every form a field can take: its field lines (C11), the fields Part 2 reads from its label (D12, `inline` and `embedded` alike), and its CSV cells. Each form adds its values under its key, in the order: label forms first, as written, then field lines in file order.
- **M10.** `_label`, `_id`, `_state` and `_mirror` are **single-valued**: a node has at most one value for each, from any form. A second one is the error `M10`. Every other field holds every value it is given, from every form.
- **M11.** The **system fields** are `_label`, `_name`, `_types`, `_id`, `_state`, `_tags`, `_link`, `_mirror`, `_formula`, `_url`, `_parent` and `_children`. They are defined by **system field definitions**, field definition nodes (M55) built into every workspace (M53), and their meaning is fixed by this spec. They can't be defined again: a field definition node for one of them is the error `M55.2`. A type lists them as it lists any field definition node (M58). `_formula` is the formulas in the label, each an item of a computed field (M39): read in a formula, it is their results, one item each. Other keys that start with `_` are free, and a workspace may define fields with them. `_name` is the node's label without the tags at its end, its name (M23), read from the label: a node with the label `Send proof #mail #teams` has the name `Send proof`, and `Call #anna about budget #projectX` the name `Call #anna about budget`. A formula in the label (L12) is replaced in the name by its result, written as text: a list as its items joined by `; `, an object in one line (M39), and nothing as nothing, with runs of spaces and tabs made one space. So `Parent =("bad")` has the name `Parent bad`; a formula that fails gives nothing. A node without a label has no name. It can't be written: a `_name` value on a node, in any form, is the error `M11.1`, and is ignored.
- **M12.** The settings of a node of a system type (M53), a type definition, tag definition, field definition node or search node, configure it; they are not data, and their child lines aren't graph data either, except a field definition node written under a `fields` setting (M58).

## 4. Ownership

- **M13.** A graph node's owner is the node it is indented under (C9), or, for a node item, the node that has the field (C13). A file's root has no owner.
- **M14.** In CSV, a `_parent` cell names the owner of the row, and a `_children` cell names the rows it owns, in order. Both hold references (section 6). Only a file's top-level node can be re-owned this way; a node that has an owner by indentation keeps it, with the warning `M14.1`. A `_parent` cell and a `_children` cell that give one node different owners is the warning `M14.2`; the first one read wins.
- **M15.** Ownership never loops. Nodes in a `_parent` or `_children` loop get the error `M15` and have no owner.
- **M16.** A node's `_children` are the nodes it owns: by indentation in file order, then by `_children` cells in the order written, then by `_parent` cells in file order.

## 5. Types and IDs

- **M17.** A node with the type `type-def` (M53) is a **type definition**. It defines the type named by its `name` value, a name (L3):

  ```
  Task +type-def
  	name:: task
  ```

  A type definition without a `name` is the warning `M17.3` and defines nothing. A value that isn't a name is the error `M17.1`, and two definitions of one name are the error `M17.2`.
- **M18.** A node's **types** are its `_types` values, in order, without repeats. A type without a definition is allowed.
- **M20.** A node's **ID** is its `_id` value. It must be a name (L3), with the error `M20.3`, and IDs match ignoring letter case. Two nodes with one ID is the error `M20.1`; both keep it, and references resolve to the first in file order (M5).
- **M19.** A type definition's settings are those of `type-def` (M53). `color` is how editors show the type, e.g. as a pill: a color name (`red`, `orange`, `yellow`, `green`, `teal`, `blue`, `purple`, `pink`, `brown`, `gray`, `black`, `white`) or `#rrggbb`; or the name of a field, as in `color:: project`, so a node's type shows in the color of its value there: the `color` of the node the value refers to, else a color the editor picks by that node's label, or for a value that refers to no node, by its text. A node without the value shows the type as it would without the setting. A color name is never read as a field's name.
- **M59.** A node with the type `tag-def` (M53) is a **tag definition** for the tags its `name` values name, without `#`, ignoring letter case, as `Urgent +tag-def` with the field lines `name:: urgent` and `color:: red` under it. Each value is a tag as L13 or L29 reads it: it starts with a letter and holds no `]`. A value that isn't is the error `M17.1`, and two tag definitions for one tag are the error `M17.2`. Its settings are those of `tag-def`: `color`, a color name or `#rrggbb` that the tag is shown in, and `icon` and `description` (M52).
- **M66.** `checkbox:: true` (L27) gives the type's nodes a checkbox: when an editor or a tool gives a node line the type and the node has no state, it adds the trigger `[ ]` (L11) before the line's text, after any list marker (L16), so `Buy milk +todo` becomes `[ ] Buy milk +todo`. A node that has a state keeps it, and a node without one is not an error: a reader doesn't add the trigger, and taking it away again is allowed.
- **M51.** A type's `nodes` setting is a **content template**: its node children (C13), with their child lines. When an editor or a tool gives a node the type, on a new node or an existing one, it writes a copy of them under the node, in order, after its other children, with their `_id` fields left out, so each node gets its own content. When a node is given several types at once, their templates follow each other in type order (M18). Field lines under `nodes::` aren't part of the template; a type's fields are written from its `fields` setting (M37). Only adding the type writes the template: a reader never adds it, and a node that already has the type doesn't get it again.

  ```
  Meeting +type-def
  	name:: meeting
  	nodes::
  		Agenda
  		Notes
  		Action items
  ```

  Creating `Weekly sync +meeting`, or adding `+meeting` to an existing `Weekly sync`, writes `Agenda`, `Notes` and `Action items` under it.
- **M52.** The `icon` setting of a type, a tag definition, a field definition (M56) or a search node (M60) is an emoji, as in `icon:: 📅`, and its `description` setting is text about it, quoted (C21) when it holds `;`. Both are informative: this spec gives them no meaning, and tools don't check them. An editor may show the icon with the type, tag, field or search, and the description where it is chosen.

### Built-in types (informative)

A workspace usually starts with this definition in its workspace file. It is an ordinary type definition, so its fields and settings can be changed like on any type.

```
Task +type-def
	name:: task
	fields::
		due::
```

Journal nodes, where tools put things for a date, are ordinary nodes found by their ID: `journal-` and the date, such as `Monday 12, 2026 ^journal-2026-10-12` and `October, 2026 ^journal-2026-10`. They have no type. Tools create them when something goes to a date.

## 5a. System types and fields

- **M53.** `type-def`, `tag-def`, `field-def` and `search` are **system types**, built into every workspace. This spec defines them, as if these definitions were in the workspace file, and a file that defines one of them is the error `M17.2`. Their color is `white`, so editors show them apart from a workspace's own types. Their settings are local field definitions (M36), and are settings only on their own nodes: on any other node, `name::`, `key::`, `color::` and the others are ordinary fields, and a node that has them without the system type defines nothing. The system field definitions (M11) are built in the same way.

  ```
  Type definition +type-def
  	name:: type-def
  	color:: white
  	fields::
  		name::
  			description:: The name of the type it defines.
  		fields::
  			description:: The fields of the type's nodes, as field lines or field definition nodes.
  		state::
  			description:: The type's custom states, each a name with an optional marker and base.
  		color::
  			description:: A color name, a hex color, or the name of a field whose value gives the color.
  		checkbox::
  			options:: true; false
  			description:: Whether the type's nodes get a checkbox.
  		nodes::
  			description:: Nodes copied under a node when it is given the type.
  		icon::
  			description:: An emoji shown with it.
  		description::
  			description:: Text about it.
  Tag definition +type-def
  	name:: tag-def
  	color:: white
  	fields::
  		name::
  			description:: The names of the tags it defines.
  		color::
  			description:: A color name or a hex color.
  		icon::
  			description:: An emoji shown with it.
  		description::
  			description:: Text about it.
  Field definition +type-def
  	name:: field-def
  	color:: white
  	fields::
  		key::
  			description:: The key of the field it defines.
  		value::
  			description:: What is written on the nodes of the types that list it: a formula, a one-time formula, or text.
  		valid::
  			description:: A formula that each value of the field must make true.
  		options::
  			description:: The values the field is meant to hold, and the types of the nodes it refers to.
  		tags::
  			options:: true; false
  			description:: Whether the field's options can be written as tags.
  		icon::
  			description:: An emoji shown with it.
  		description::
  			description:: Text about it.
  Search +type-def
  	name:: search
  	color:: white
  	fields::
  		query::
  			description:: The query: terms joined by spaces for and, and by a bar for or.
  		view::
  			description:: How the results are shown.
  		fields::
  			description:: The fields shown for each result.
  		sort::
  			description:: The order of the results.
  		group::
  			description:: The field the results are grouped by.
  		footer::
  			description:: What is shown under the results.
  		computed::
  			description:: What is computed from the results.
  		icon::
  			description:: An emoji shown with it.
  		description::
  			description:: Text about it.
  Label +field-def
  	key:: _label
  	description:: The node's label, its line without the forms that are not part of it.
  Name +field-def
  	key:: _name
  	description:: The node's label without the tags at its end (M23), read from the label; it can't be written.
  Types +field-def
  	key:: _types
  	description:: The node's types.
  ID +field-def
  	key:: _id
  	description:: The node's ID, a name that references can use.
  State +field-def
  	key:: _state
  	description:: The node's state, a built-in state or a custom state of its types.
  Tags +field-def
  	key:: _tags
  	description:: The node's tags.
  Link +field-def
  	key:: _link
  	description:: A reference to a node, by its label, its ID or both.
  Mirror +field-def
  	key:: _mirror
  	description:: The node that this node is a mirror of, its source.
  Formula +field-def
  	key:: _formula
  	description:: The formulas in a label, evaluated where they are written; read in a formula, their results. A formula in a field's value is a formula item of that field (M39).
  URL +field-def
  	key:: _url
  	description:: A URL or a path.
  Parent +field-def
  	key:: _parent
  	description:: The node's owner: the node it is indented under, or the node a CSV cell names.
  Children +field-def
  	key:: _children
  	description:: The nodes it owns, in order: by indentation, then by CSV cells.
  ```

## 6. References

- **M21.** A **reference** points from a field value to a graph node. These values are references:
  - a link (L10), in any field
  - a plain text value of the `_link` field
  - a plain text value of the `_mirror` field, read as inside `[[` and `]]` (L10), and a mirror value (L31)
  - a plain text value of a reference field (M24), quoted or not in a field line, and unquoted in an inline field (L5)
  - a `_parent` or `_children` cell (M14)
- **M22.** A reference names a node by ID, by label, or both, as in L10. A plain text value that is a reference (M21), in a field line, an inline field or a cell, is read as inside `[[` and `]]`, as a `_mirror` value is: `follows:: Send proof^p1` names the node by ID and label, as `[[Send proof^p1]]` does, and `follows:: ^p1` by ID alone. `$node()` (M74) reads its text so too. It resolves:
  - by ID alone (`^x1`, or ID and label): the node with that ID. If none has it, the error `M22.1`. The label is never a fallback; when it no longer matches the node (M23), the warning `M22.2`.
  - by label alone (`"Anna Smith"`, or a quoted link): the node with that label, as in M23.
  - by text that could be either (`[[anna]]`, `client:: ruukki`): the node with that ID, or else the node with that label. When it is one node's ID and another node's label, the ID wins, with the warning `M22.3`.
- **M23.** A label matches a node when their names are equal, ignoring spaces and tabs at either end and letter case. A label's **name** is the label without the tags (L13) at its end: each tag followed only by spaces, tabs and other tags is removed, with the spaces and tabs before it. Tags elsewhere are part of the name, and a label that is only tags is its own name. So `Buy milk #urgent` is named `Buy milk`, and `[[Buy milk]]`, `[[Buy milk #urgent]]` and `[[Buy milk^x1]]` name it; `Call #anna about budget #projectX` is named `Call #anna about budget`, which `[[Call about budget]]` doesn't match. A label with formulas (L12) matches by its name as written and by its name with each formula's result in its place (M11): `Parent =("bad")` matches both `[[Parent =("bad")]]` and `[[Parent bad]]`. The results are those of the last evaluation; when they change a name, references are resolved again and the formulas evaluated again, at most twice, so a name that depends on the references it resolves settles. An editor that writes a reference to a node writes its name. Letter case is compared by mapping both to lowercase with Unicode's default case mapping (not locale-specific); there is no other normalization, so `é` written as one character or as `e` and a combining accent are different labels. In a reference field (M24), only nodes of its types match. No match is the warning `M23.1`; several matches is the warning `M23.2`, and the reference doesn't resolve.
- **M24.** A field is a **reference field** when its field definition, a type's (M36) or a field definition node (M55), has an `options` setting (M46) that names one or more types, each as an item `+name`, as in `options:: +person` or `options:: +person; +company`. A plain value of a reference field is then a reference to a node of those types, unless it equals one of the setting's other options, ignoring letter case, which stays text: with `options:: nobody; +person`, `owner:: nobody` is text and `owner:: Anna` a reference. Any other field's plain values are text, even when they name a node; a tool may suggest adding `options:: +name`, but doesn't resolve them.
- **M25.** References may form cycles. They are links, not ownership or dependencies.

## 6a. Mirrors

- **M48.** A node with a `_mirror` field, from a mirror line `@ Label` ([Ord specification](Ord_Specification.md) L30), an inline field or a field line, is a **mirror** of the node its reference resolves to (M22), its **source**. A mirror adds nothing to its source: the source keeps its one owner (M13), its fields and its children. The mirror is a graph node of its own, owned where it is written, with an empty label. A mirror whose reference doesn't resolve has that reference's diagnostic and no source. A mirror can't itself be a source, since it has no label or ID to be named by.
- **M49.** **A mirror stands for its source.** A mirror that is a field's node item (C13) is a reference (M21) from the field's node to the source, so `attendees::` with `@ Anna Smith` under it means what `attendees:: @Anna Smith` and `attendees:: [[Anna Smith]]` do, and a reference field's options (M24, M46) check the source. A mirror that is a child stands for its source in its owner's `_children` (M16), in formulas (M41) too: a project's `$sum(_children.hours)` counts the tasks it mirrors. A source mirrored more than once under one owner is one child, at its first place.
- **M50.** A mirror under its own source, at any depth, is the warning `M50.1`. It stands for nothing in `_children`, and an editor shows the source's line but doesn't unfold it. Lines under a mirror are the warning `M50.2`, since a mirror's children are its source's. They are kept as the mirror's own, and an editor may offer to move them into the source.

## 7. States

- **M26.** A node's **state** is its single `_state` value (M10), from its trigger (L11) or a field. It is a **built-in state**, `new`, `in-progress`, `waiting` or `done`, or a custom state (M27): one that a custom marker stands for (M29), or else the first in type order (M18) of the node's types that defines it. A value that is none of these is the warning `M26`, and the node keeps it as its state, with no base.
- **M27.** A type's `state` setting defines its **custom states**: `|`-separated parts, each a name (L3), an optional marker in brackets, and an optional `>` and **base**, a built-in state, as in `triage [t] > new | review [r] > in-progress | shipped > done | someday`. Spaces and tabs around the parts don't count. A part in another form, with a base that isn't a built-in state, or with the name of a built-in state or of a custom state the type already has, is the warning `M27.1`, and the part is left out.
- **M28.** A custom state's marker is one character. It can't be a built-in marker (` `, `-`, `?`, `x`), with the warning `M28.1`, and a marker can belong to only one custom state across all types, with the warning `M28.2` on each later one. Such a custom state has no marker.
- **M29.** A custom marker stands for the custom state it belongs to, on any node: `[r] Fix login` has the state `review`. A marker no custom state has (L11) gives the node no state.
- **M30.** A node's **base** is its state's base. A built-in state is its own base. A custom state without a base, and a state that is the warning `M26`, give the node no base.
- **M31.** `done` is true for a node exactly when its base is `done`.

## 8. Field definitions and formulas

This section is a draft outline. Its rules are numbered so they can be referred to, and need tests before 1.0.

- **M36.** A type definition's `fields` setting holds **field definitions**: **local field definitions**, its field lines under its anonymous node (C13), and field definition nodes (M55, M58). The two are one thing written two ways, with the same settings (M56), and differ only in scope: a local definition applies on the type's nodes, a field definition node on every node (M57). A local definition's key is the field it defines, what is written after it on its line is its `value` setting, and its child field lines are its other settings. A formula must start on the key's line (M87). A child field line whose key isn't a setting (M56) is a **nested field definition**: it defines a field written under that field, as a field line of its anonymous node (C13), and has the same settings and nested field definitions of its own, at any depth.
- **M55.** A node with the type `field-def` (M53) is a **field definition node**. It defines the field whose key is its `key` value, for the whole workspace:

  ```
  Priority +field-def
  	key:: priority
  	options:: high; medium; low
  	tags:: true
  ```

  A field definition node without a `key` is the warning `M55.3` and defines nothing. Where this spec matches a field to its definition, keys match ignoring letter case, as labels do (M23); the Core keeps keys as written, so `Due` and `due` on one node are still two fields. Two field definition nodes for one key are the error `M55.1`, and one for a system field (M11) the error `M55.2`.
- **M56.** A field definition's settings, local or a node, are `value`, `valid`, `options` and `tags`, as in M37 to M47 and, for a calculated field, M87, and `icon` and `description`, as in M52. A field definition node has them as field lines, besides its `key`.
- **M57.** A field definition node applies on every node (M36), so with the definition above, `Crash #high` has `priority:: high` whatever its types. Only its `value` (M37) is limited: it is written on the nodes of the types that list it (M58), or for a formula calculated on them (M87), since only they are meant to have the field.
- **M58.** A type lists a field definition node in its `fields` setting in either of two ways:
  - a field line with its key and no value or child lines. A value or child lines there are the error `M58.1` and are left out, since a type can't change a field definition node. The exception is a `~expr` value for `_id` (M37). So a type can't calculate a system field: `_label:: =(…)` under `fields::` is this error too.
  - the field definition node itself, written under `fields::` as a node item (C13). It is read there although it is under a setting (M12), and defines its field for the whole workspace, as it would anywhere else.

  For a key that no field definition node defines, a type defines its own field (M36).

  ```
  Task +type-def
  	name:: task
  	fields::
  		Owner +field-def
  			key:: owner
  			options:: +person
  		priority::
  		due::
  ```

  Here `owner` and `priority` are defined for the whole workspace, and `due` only for `task`.
- **M37.** **A type's fields are written on its nodes.** When an editor or a tool gives a node a type, on a new node or an existing one, it writes a field line for each of the type's field definitions (M36, M58), other than calculated fields (M87), whose key the node doesn't have, where C48 puts a new field, in definition order. When a node has several types, each type's fields follow in type order (M18), and a key that an earlier type wrote isn't written again. The line holds the definition's `value` as written: `~expr` as a one-time formula, which is then replaced with its result (M85), any other value as text, and no value as `key::`. Nested field definitions (M36) are written as field lines under their field, the same way. `_id` isn't written as a field line: a `~expr` for it gives each node of the type without an ID one, once: the result, when it is a name (L3) that no other node has as its ID, else an ID the tool makes up. A reader never gives a node a field or a value from its types, except a calculated field (M87): a node has what is written on it, and a type's `value` is what tools write.

  ```
  Invoice +type-def
  	name:: invoice
  	fields::
  		amount::
  		vat:: =(amount * 0.25)
  		status:: open
  ```

  Given the type, `March +invoice` becomes the lines below. `vat` isn't written: it is calculated (M87).

  ```
  March +invoice
  	amount::
  	status:: open
  ```
- **M84.** **A type definition edits its nodes.** A change to a type's field definitions is a change to its nodes, as an edit through a mirror is a change to its source (M48), except that it is written into each node's file. When a type gains a field definition, nested ones included, tools write it on each node of the type that doesn't have its key, as M37 does. When a field definition's `value` changes, tools rewrite the field on each node whose value is still the old one as written, ignoring spaces and tabs at either end, and leave it where the node writes another value: that node has changed it. A field definition renamed in place, its key changed and nothing else in the type's fields, renames the field on each node that has it, keeping the node's value. A field definition that is removed leaves its fields on the nodes as they are. A calculated field (M87) isn't written, so a change to its formula changes no file. When a `value` becomes a formula, tools remove the field from each node whose value is still the old one as written; a node that writes another value has the error `M87.1` until it is removed. When a formula becomes a plain value or none, tools write the field on each node as M37 does. Tools write these edits once the change to the type definition is done, not while it is being typed, as one edit of all the files, so one undo takes them back.
- **M87.** **Calculated fields.** A field definition (M36, M55) whose `value` is a formula, `=expr` (L12), defines a **calculated field**. Every node the definition's value applies to (M37, M57) has the field, with the formula's result as its value, evaluated on the node (M41) as a formula item is (M39), so a list gives one item per element and nothing gives none. The field is never written on the nodes, and a file read as plain text doesn't hold it (M86). On a local field definition, the formula is written after the key on its line and must start there; one over several lines (C52) continues under it:

  ```
  Expense +type-def
  	name:: expense
  	fields::
  		line_a::
  		line_b::
  		total:: =(line_a + line_b)
  			valid:: =($value >= 0)
  ```

  An editor shows a calculated field first under the node line, before what is written under the node, as its key, ` = ` and its result, separated by `; ` when there are several: `by a boat +expense`, with `line_a:: 2000` and `line_b:: 50` under it, shows `total = 2050` above them. It is drawn apart from field lines, and can't be edited there; an evaluation error shows in its place (M44). Formulas, queries and mirrors read it as they read any field (M86). Its other settings apply to its result: `valid` checks it (M40), `options` (M46) says what it is meant to hold, `tags` (M47) lets a query match its value by a tag, though a tag on a node never sets it, and `icon` and `description` are as in M52. A diagnostic about a calculated field's result is on the node's line. A node that has a calculated field from several types has the first type's (M18). A nested field definition (M36) with a formula is calculated the same way, on the field's anonymous node (C13) but evaluated on the node (M41), and shown first under that field's line. A field line with nothing on it or under it, as `parts::`, counts as having an anonymous node with no lines, so its nested calculated fields are there too: with `parts::` and `double:: =(line_a * 2)` under it on the type, a node with `line_a:: 10` and an empty `parts::` has `parts.double`, 20, shown under its `parts::` line.

  A node can't write a field it calculates: a value for that key on the node, in any form, is the error `M87.1`, and is ignored. A child field line under a calculated field's definition whose key isn't a setting (M56) is the error `M87.2`, since a calculated field has no nested fields, and so is a formula alone on a child line of a local field definition, which isn't its value; both are left out.
- **M39.** A formula (L12) can be written wherever a field's value goes, and is then a **formula item**: on a field line, alone or among other items (`total:: =(a + b)`, `tags:: draft; =(project._tags)`), in an inline field (`[total:: =(a + b)]`), or as a child line of a field line that holds only the formula and has no child lines of its own, which is the same item as the formula written on the field line:

  ```
  total::
  	=(a + b)
  ```

  means what `total:: =(a + b)` does. On an ordinary node, a formula item stands for its result: one item, one item per element when the result is a list, and none when it is empty. A field with formula items is a **computed field**, written on the node, unlike a calculated field (M87): its value is its items in order, each formula item replaced by its result, so formulas and plain values can be mixed, and a field can have several formulas. A result with objects is the field the [JSON mapping](JSON_mapping.md) reads it as (J9 to J11), each member a field: plain elements are items, one object is the field's anonymous node (C13), as `contact::` with `primary:: anna` under it reads as `{"primary": "anna"}` (M41), and several are node lines, each holding its first plain member as an inline field, `[a:: 1]`, with its other members under it (J8). A node with only inline fields and no label of its own reads back as an object of its fields. An editor shows each formula item's result after the formula, where it is written, the items joined by `; `. A result with objects it shows under the formula's line instead, as it shows calculated fields (M87): a row for each field, `key = items`, a field whose value is an object as `key::` with its fields on the rows under it, one level in, and first a row `= items` for the items that aren't objects, so `=({"a": 1, "b": {"c": 2}})` shows `a = 1`, then `b::` with `c = 2` under it. Each object of several shows on a row of its own, its first plain field, with its other fields on the rows under it. Where a result is text in one line, as in a name (M11), an object is its fields as `key:: items`, separated by `, `: `a:: 1, b.c:: 2`. A `=expr` value of a setting of a node of a system type (M12), other than a field definition node's `value` and `valid` and a search node's `footer` (M64), is the error `M39`; the formula isn't evaluated.
- **M40.** A `valid` setting holds `=expr`, evaluated per node with `value` bound to the field's value, a list when it has several. A result that isn't true is the error `M40`, and the value stays.
- **M41.** **Formula language.** A formula's expression is JSONata 2.x, written `=(…)` so that `$` is allowed (L12). The current node is `$`, the graph root `$root`, and a node is an object with its fields as members, system fields included: `_parent` is its owner (M13) and `_children` the nodes it owns (M16). A reference field's value is the node it refers to, and a field with field children (C13) is an object of them. A formula in a field line under another field, at any depth, is evaluated on the node that owns the outermost field, not on the anonymous node it is written in: with `first:: =(sublevel.second & _tags)` and `sublevel::` with `second:: =(_label)` under it on the node `Test #tag`, `$`, `_label`, `_parent`, `_children`, `_id` and the other fields are the same in both formulas, and `second` is `Test #tag`. Eight system fields also have one-letter aliases: `L` is `_label`, `N` `_name`, `P` `_parent`, `C` `_children`, `S` `_state`, `G` `_tags`, `Y` `_types` and `I` `_id`. A node's own field with the same key as an alias comes before it.
- **M42.** The function library adds `$node(text)`, `$nodes(under?)`, `$query(text, under?)`, `$today()`, `$now()`, `$date(text, days?)`, `$days(a, b?)`, `$format(x, picture)`, `$slug(text)`, `$first(x)` and `$last(x)`. Dates are ISO text. M74 to M83 define them.
- **M74.** `$node(text)` gives the node that `text` names, as a link does: the node whose ID (M20) is `text`, ignoring letter case and spaces and tabs at either end, else the node whose label matches it (M23). Text that starts with `^` names a node by its ID only: `$node("^i42")`, and `Label^id` by its ID, as in a link (M22). Text that is one node's ID and another node's label gives the ID's node, with a warning. Text that names no node, or a label that matches several, gives nothing, with a warning. Unlike a reference field (M24), it matches nodes of any type. A number is read as its text, and a list gives the nodes its items name, leaving out those that name none; any other value gives nothing. Any text works, so `$node("journal-" & $today())` is today's journal node.
- **M75.** `$nodes()` gives every node in the workspace, other than each file's root node (M13), field items (C13) and mirrors (M48): depth first, each node before the nodes it owns, in file order. Archived nodes (M63) are included. `$nodes(under)` gives only the nodes `under` owns, at any depth, in the same order; `under` is a node, or text naming one as `$node()` does, with its warnings, or a list of them, giving each node once. When it names no node, or the node owns none, the result is empty. The result is a list of nodes, filtered with JSONata on any field, system fields included: `$nodes()["task" in _types and _state != "done"]`.
- **M76.** `$query(text, under?)` gives the nodes the query `text` matches, by the query rules (M61 to M63, M65), in file order: `$query("+task #project [ ]")`. With `under`, as in M75, only those `under` owns. A query that doesn't parse, or that holds a formula, is an evaluation error (M44.3); a formula's result is filtered instead: `$query("+task")[due < $today()]`. A formula that uses `$query()` depends on every node's types, tags, states and fields.
- **M77.** `$today()` gives today's date in local time, as a date (L23): `2026-10-22`. `$now()` gives the local date and time, as a datetime (L24) with seconds and without an offset: `2026-10-22T14:30:05`. Local time is the time zone of the tool that evaluates the formula.
- **M78.** A **date name** is one of these, in any letter case: `today`, `tomorrow`, `yesterday` (or `yday`) and `now`; a weekday, `monday` to `sunday`, which is the next one, today included; and `this-`, `next-` or `last-` followed by a weekday, which is that day in this, the next or the last week, or by `week`, `month` or `year`, which is that period (M79). Weeks start on Monday (L25). `now` is a datetime, the periods are lists of dates, and the others are dates.
- **M79.** `$date(text, days?)` reads `text` without spaces and tabs at either end. A date (L23) or a datetime (L24) gives itself. A week or a month (L25), or a year written as four digits, gives a **period**: the list of its dates, in order, so `due in $date("2026-W42")` works. A date name (M78) gives what it names. A number is read as its text, so `$date(2026)` is that year. Anything else, a week that doesn't exist included, gives nothing, with a warning. With `days`, a number, each date is shifted by that many whole days, the number truncated toward zero, and a datetime keeps its time and offset: `$date("2026-10-22T09:00", 7)` gives `2026-10-29T09:00`. A `days` that isn't a number is ignored.
- **M80.** `$days(a, b)` gives the whole days from the date of `a` to the date of `b`, negative when `b` is earlier, ignoring the times of datetimes: `$days("2026-10-20", "2026-10-22")` is `2`. `$days(a)` counts from today to `a`. `a` and `b` are dates or datetimes; anything else, a date name or a period included, gives nothing.
- **M81.** `$format(x, picture)` writes a number or a date with a JSONata picture. A number is written as JSONata's `$formatNumber` does: `$format(1234.5, "#,##0.00")` gives `1,234.50`. A date or a datetime is written as `$fromMillis` does: `$format("2026-10-22", "[D1] [MNn] [Y]")` gives `22 October 2026`. A date is read as its midnight, a datetime without an offset at its time as written, and one with an offset is written in UTC. Any other value, or a `picture` that isn't text, gives the value as text, and nothing gives nothing.
- **M82.** `$slug(text)` gives the slug of the text (M45): `$slug("Åsa Öberg!")` gives `asa-oberg`. A number is read as its text, and a list gives the slug of each item; any other value gives nothing. A slug can be empty: `$slug("2026")` gives `""`.
- **M83.** `$first(x)` and `$last(x)` give the first and the last item of a list, and nothing for an empty list. Any other value gives itself, so a field with one value or several reads the same way: `$first(owner)`.
- **M85.** **One-time formulas.** A one-time formula, `~(…)` or `~expr` (L32), on an ordinary node, in a label or as a field's item, is evaluated as a `=` formula there would be (M39, M41), and a tool writes its result in its place, so the file holds the value, not the formula: in a field, the result's items, written as C26 says (`due:: ~$today()` becomes `due:: 2026-11-06`), and in a label, the result's text. Objects in the result (M39) are written as their field's lines under the field's line, one level deeper, after its other lines: `x:: ~({"a": 1, "b": {"c": [2, 3]}})` becomes `x::` with `a:: 1` and `b::` under it, and `c:: 2; 3` under `b::`, and `x:: ~([{"a": 1}, {"a": 2}])` becomes `x::` with the node lines `[a:: 1]` and `[a:: 2]` under it; the formula goes from the line with the `;` next to it. Objects can't be written in a label, an inline field or a formula alone on a child line, nor one object when the field already has field lines under it, since a field has one anonymous node (C13): the formula stays, with that error. A tool does this once the line is no longer being edited, in a file it edits or shows: a formula that ends where the person is typing is written once a space or a line break follows it, and until then an editor shows its result, as for a `=` formula (M39); one elsewhere is written once edits pause. A result that is an error, or nothing, isn't written: the formula stays, and an editor shows the error after it, as for a `=` formula (M39). Until a tool replaces it, a one-time formula has no value: as a field's item, it adds no item. `$seq(n)` gives the next number, `n` digits wide, after the largest number at the end of a value of the field (of the ID, for `_id`) on the nodes of the node's first type, or of every node when it has none. A `value` of `auto expr`, the form this replaces, is the warning `M85.1`, and isn't evaluated: write `~(expr)`.
- **M43.** Formulas are pure, except `$today()`, `$now()`, `$date()` with a date name (M78) such as `"tomorrow"` and `$days()` with one argument, whose results follow the clock. One-time formulas (M85) may also use `$seq(n)`, `$uuid()`, `$id()` and `$sid()`; elsewhere they are an evaluation error (M44.3).
- **M44.** Dependencies are per node and field, calculated fields (M87) included. Fields in a cycle are empty, with the error `M44.1` naming the whole path; fields that depend on a cycle are empty, with the error `M44.2`. An evaluation error leaves its formula item without a result, with the error `M44.3`; the field's other items stay.
- **M45.** A **slug** is the text in lowercase, accents removed, each run of characters other than letters and digits turned into one `-`, with leading digits and `-` and trailing `-` removed.
- **M46.** An `options` setting lists the values a field is meant to hold, as its items or as the labels of its node items without child lines (C13), in order. An item `+name` names a type instead, whose nodes the field refers to (M24); it is not an option. A value written for the field, other than a formula, on a node the definition applies to (M36), or the result of a calculated field (M87), that equals none of the options of the definitions that apply, ignoring letter case, is the warning `M46`, and the value stays. On a reference field such a value is a reference instead, and not the warning. Options say how a field is meant to be used, not its value type, and tools offer them first when a value is typed.
- **M47.** A `tags` setting of `true` (L27) lets the field's options (M46) be written as tags. On a node the definition applies to (M36), a tag that equals one of the options, ignoring letter case, is a value of that field instead, spelled as the option is, and the node doesn't have that tag: with `options:: bug; feature` and `tags:: true` on the `issue type` field, `Crash +issue #bug` has `issue type:: bug`. On other nodes the tag is a tag. An option is matched as written: `tech-debt` by `#tech-debt`, and `Tech Debt` by `[#Tech Debt]` (L29), never by `#tech-debt`. On a reference field (M24), a tag also names a node of its types, the value then referring to that node: first by its label as written, then by its name (M23), and last by its ID. With `options:: +project` and `tags:: true`, `#ord-notes` on a task refers to `Ord-notes +project ^abc123`, `#ord` to `Ord #new +project`, before a node whose ID is `ord`, and `[#Ord Notes]` (L29) to `Ord Notes +project`. A tag is matched only against what is written, never against a slug: `#ord-notes` doesn't name `Ord Notes +project`. When fields of two of a node's types take one tag, the first type's field has it. The text stays as written.

## 9. Queries

This section is a draft outline, like section 8.

- **M60.** A node with the type `search` (M53) is a **search node**, whether written as `+search` or as an inline search `? query` ([Ord specification](Ord_Specification.md) L28). Its settings are `query` and the view settings (M64). Its query is the value of `query`, or, written as `query::` with child lines, each child line's label in parentheses, joined with spaces. When a node has more than one `query` field, such as `?` and a `query::` line under it, the queries are combined with and: each in parentheses, joined with spaces. This is allowed but not recommended. A search with an empty label is named by its query where a name is needed, such as its results document.
- **M61.** A query is terms, joined by spaces for and, `|` for or, and grouped by parentheses; `!` before a term or group negates it. A term is `+type`, `#tag`, a state trigger, a link, a mirror term `@` and a reference (M65), a field `[key:: value]`, a formula `=(…)`, or a word, which matches labels.
- **M62.** A node matches `+type` when it has the type, `#tag` when it has the tag or a tag nested under it, ignoring letter case (`#project` matches `#project/apollo`), or, when a field of one of its types with `tags:: true` (M47) would take the tag, when that field's value is what the tag would give it (with `tags:: true` and `options:: +project` on `project`, `#LU` matches a task with `project:: LU` as well as one written `#LU`), `[x]` and the other triggers when its state or its base is that state, a link when one of its fields refers to the node it resolves to, `[key:: value]` when one of its `key` values equals the text, ignoring letter case, a computed field's (M39) and a calculated field's (M87) values included, as the text their results are written as (a node by its ID or name), a formula when its result is true, and a word when its label contains it, ignoring letter case.
- **M63.** Nodes tagged `archived` match only queries that name `#archived`.
- **M64.** A search node's view settings (`view`, `fields`, `sort`, `group`, `footer`, `computed`) are implementation, like what they show. A `footer` holds `=expr`, evaluated with the list of the results' nodes as `$`. A `sort` holds conditions as items (C15), applied in turn, each a field key, then optionally `asc` or `desc` and then optionally `number` (or `num`), `text` or `date`; the defaults are `asc` and `text`. What `computed` holds is still to be specified here. Like a `footer`, it is shown with the results, nothing else reads it, and it is never written into a file (M86).
- **M65.** A query matches a node in its own place, never a mirror (M48), so a node mirrored in three places is one result. The term `@` and a reference, written as in a mirror line (L30) but with no space after `@`, matches the node the reference resolves to. In a query scoped to a file or to a node's children, that node matches when it or one of its mirrors is in the scope. Scopes are still to be specified.

## 10. Diagnostics

Codes, severities and positions work as in the Spec (C35 to C37). Positions are in the file the diagnostic is about.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `M10` | error | A node has a second `_label`, `_id`, `_state` or `_mirror` value | the second value |
| `M11.1` | error | A node writes a `_name` value | the value |
| `M14.1` | warning | A `_parent` or `_children` cell names a node owned by indentation | the cell |
| `M14.2` | warning | `_parent` and `_children` cells give a node different owners | the later cell |
| `M15` | error | A node is in an ownership loop | the node |
| `M17.1` | error | A type definition's `name` isn't a name, or a tag definition's isn't a tag | the value |
| `M17.2` | error | A type or a tag is defined twice, or a system type in a file | the second `name` field |
| `M17.3` | warning | A type definition has no `name` | the `+type-def` trigger |
| `M20.1` | error | Two nodes have one ID | each ID after the first, in file order (M5) |
| `M20.3` | error | An ID isn't a name | the ID |
| `M22.1` | error | A reference's ID matches no node | the reference |
| `M22.2` | warning | A reference's label no longer matches the node its ID names | the reference |
| `M22.3` | warning | A reference is one node's ID and another node's label | the reference |
| `M23.1` | warning | A label matches no node | the reference |
| `M23.2` | warning | A label matches several nodes | the reference |
| `M26` | warning | A `_state` value is neither a built-in state nor a custom state of the node's types | the value |
| `M27.1` | warning | A custom state isn't `name [m] > base` with a built-in base and a new name | the `state` setting |
| `M28.1` | warning | A custom state uses a built-in marker | the `state` setting |
| `M28.2` | warning | A marker belongs to a custom state already | the `state` setting |
| `M39` | error | A formula on a setting of a node of a system type, other than a field definition node's `value` and `valid` and a search node's `footer` | the value |
| `M40` | error | A `valid` formula isn't true | the value, or the node's line for a calculated field (M87) |
| `M44.1` | error | A computed field is in a cycle | the formula |
| `M44.2` | error | A computed field depends on a cycle | the formula |
| `M44.3` | error | A formula fails to evaluate | the formula |
| `M46` | warning | A value is none of the field's `options` | the value, or the node's line for a calculated field (M87) |
| `M50.1` | warning | A mirror is under its own source | the mirror's `_mirror` value |
| `M50.2` | warning | A line is under a mirror | the line |
| `M55.1` | error | A field is defined by two field definition nodes | the second `key` field |
| `M55.2` | error | A system field (M11) is defined again | the `key` value |
| `M55.3` | warning | A field definition node has no `key` | the `+field-def` trigger |
| `M58.1` | error | A type gives a field definition node's key a value or child lines | the field line |
| `M87.1` | error | A node writes a field that one of its types calculates | the value |
| `M87.2` | error | A calculated field's definition has a nested field, or a local field definition has a formula alone on a child line | the line |
| `M85.1` | warning | A field definition's `value` is `auto expr`, now written `~(expr)` | the value |

---

# Appendix A: Graph dump and tests

Tests work as in the [Test format](Test_format.md), part B, with one more option.

- **M70.** With the `graph` option, the input is a workspace and the expected output is its graph dump. The input holds one or more files, each starting with a line `--- ` and the file's path, e.g. `--- tasks.ord`. A path ending in `.csv` is a CSV file.
- **M71.** The graph dump lists each file, in the order of M5, as `file` and its path, and under it, indented two spaces per level, its nodes by ownership (M16), starting with the root's children. A node is `node` and its label as a string (D8), or `node` alone when it has none, then `@` and its line when it has one, as `@path:line` when it is in another file than the one listed, and for a node item (C13), `in` and the field's key, e.g. `node "Erik" @12 in "contacts"`.
- **M72.** Under a node, before its children, come only the facts this spec decides, in this order, each when present: `id`, `type` (one line per type), `state`, `base` when it differs from the state, and `ref` lines: `ref`, the key, `->`, and the target's file and line as `path:line`, e.g. `ref "client" -> a.ord:1`.
- **M73.** Diagnostics follow, one per line, sorted by file, line, column and code: `!`, the code, and `path:line:column`.

```example graph M8 M18 M20 M21 M22 M23
--- a.ord
Ruukki AB +company ^ruukki
Anna +person
Call [[Anna]] about [[Ruukki AB]]
	client:: ruukki
.
file "a.ord"
  node "Ruukki AB" @1
    id "ruukki"
    type "company"
  node "Anna" @2
    type "person"
  node "Call [[Anna]] about [[Ruukki AB]]" @3
    ref "_link" -> a.ord:2
    ref "_link" -> a.ord:1
```

```example graph M10 M26 M27 M29 M30
--- types.ord
Bug +type-def
	name:: bug
	state:: triage [t] > new | review [r] > in-progress | shipped > done | someday
--- work.ord
[r] Fix login +bug
Ship it +bug
	_state:: shipped
Later +bug
	_state:: someday
[x] Twice +bug
	_state:: shipped
[q] Unknown
Odd
	_state:: shipped
.
file "types.ord"
  node "Bug" @1
    type "type-def"
file "work.ord"
  node "Fix login" @1
    type "bug"
    state "review"
    base "in-progress"
  node "Ship it" @2
    type "bug"
    state "shipped"
    base "done"
  node "Later" @4
    type "bug"
    state "someday"
  node "Twice" @6
    type "bug"
    state "done"
  node "Unknown" @8
  node "Odd" @9
    state "shipped"
! M10 work.ord:7:11
! L11 work.ord:8:1
! M26 work.ord:10:11
```

```example graph M5 M17 M18 M53
--- Ord.workspace
Types
	+task
	Task +type-def
		name:: task
--- a.ord
Write +task
+task
Task +type-def
	name:: task
.
file "Ord.workspace"
  node "Types" @1
    node "" @2
      type "task"
    node "Task" @3
      type "type-def"
file "a.ord"
  node "Write" @1
    type "task"
  node "" @2
    type "task"
  node "Task" @3
    type "type-def"
! M17.2 a.ord:4:2
```

```example graph M13 M14 M15 M20 M21 M24
--- a.ord
Invoice 42 ^i42
Anna +person ^anna
Erik ^anna
Now ^today
Task +type-def
	name:: task
	fields::
		owner::
			options:: +person
Fix +task
	owner:: "Anna"
	contacts::
		Erik
--- lines.csv
_id,_label,_parent
l1,Workshop,i42
l2,Review,l3
l3,Loop,l2
.
file "a.ord"
  node "Invoice 42" @1
    id "i42"
    node "Workshop" @lines.csv:2
      id "l1"
      ref "_parent" -> a.ord:1
  node "Anna" @2
    id "anna"
    type "person"
  node "Erik" @3
    id "anna"
  node "Now" @4
    id "today"
  node "Task" @5
    type "type-def"
  node "Fix" @10
    type "task"
    ref "owner" -> a.ord:2
    node "Erik" @13 in "contacts"
file "lines.csv"
  node "Review" @3
    id "l2"
    ref "_parent" -> lines.csv:4
  node "Loop" @4
    id "l3"
    ref "_parent" -> lines.csv:3
! M20.1 a.ord:3:6
! M15 lines.csv:3:1
! M15 lines.csv:4:1
```

```example graph M23
--- a.ord
Buy milk #urgent
Anna #friend
See [[Buy milk]], [[buy milk #urgent]] and [[anna]]
.
file "a.ord"
  node "Buy milk #urgent" @1
  node "Anna #friend" @2
  node "See [[Buy milk]], [[buy milk #urgent]] and [[anna]]" @3
    ref "_link" -> a.ord:1
    ref "_link" -> a.ord:1
    ref "_link" -> a.ord:2
```

```example graph M21 M48 M49
--- a.ord
Monday ^journal
	Standup
	@ Send mapping draft^t9x1
Kickoff
	attendees::
		@ Anna Smith
	with:: @Anna Smith; @Nobody
Anna Smith +person
Apollo +project
	[-] Send mapping draft +task ^t9x1
		Ask Erik
	@ ^gone
.
file "a.ord"
  node "Monday" @1
    id "journal"
    node "Standup" @2
    node "" @3
      ref "_mirror" -> a.ord:10
  node "Kickoff" @4
    ref "attendees" -> a.ord:8
    ref "with" -> a.ord:8
    node "" @6 in "attendees"
      ref "_mirror" -> a.ord:8
  node "Anna Smith" @8
    type "person"
  node "Apollo" @9
    type "project"
    node "Send mapping draft" @10
      id "t9x1"
      type "task"
      state "in-progress"
      node "Ask Erik" @11
    node "" @12
! M23.1 a.ord:7:22
! M22.1 a.ord:12:2
```

```example graph M48 M50
--- a.ord
Apollo +project
	Tasks
		@ Apollo
	@ Tasks
		Stray note
.
file "a.ord"
  node "Apollo" @1
    type "project"
    node "Tasks" @2
      node "" @3
        ref "_mirror" -> a.ord:1
    node "" @4
      ref "_mirror" -> a.ord:2
      node "Stray note" @5
! M50.1 a.ord:3:3
! M50.2 a.ord:5:3
```
