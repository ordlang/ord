# Ord semantics

Version 0.5.0-draft, 2026-10-27. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D`, `T` or `F` refer to the Spec. This spec's rules start with `M`.

The Spec says how text becomes a tree of nodes and fields. This spec says what that tree means across a workspace: which node a reference points to, what types and states a node has, who owns whom, and what formulas and queries written in files stand for.

## 1. Scope and the line

- **M1.** This spec covers what files mean. Two tools that conform to it give a workspace the same graph: the same nodes, owners, types, IDs, states, field values and references, and the same diagnostics from section 11.
- **M2.** A rule belongs here when it is about something written in a file that a tool interprets, or something a tool writes into a file. Anything that exists only in a tool, such as an index, a view, completion or a heuristic whose result isn't written, is implementation and outside this spec.
- **M3.** These are explicitly implementation-defined, so tools may differ:
  - how value types are inferred, and outlier warnings; only value forms (L21 to L27) are normative
  - typing aids such as expansions (`~today`), which an editor writes out as you type
  - diagnostics beyond section 11, which use codes outside this spec's letters, e.g. `ord/outlier`
  - how and when a graph is stored, cached or rebuilt
  - completion, hover, inlay hints, views, CodeLens and results documents
  - merging and sync
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
- **M9.** A sigil (L15) adds a value under the key of its type: with `%` bound to `project`, `%apollo` adds `project:: apollo`.
- **M10.** `label`, `id` and `state` are **single-valued**: a node has at most one value for each, from any form. A second one is the error `M10`. Every other field holds every value it is given, from every form.
- **M11.** The **reserved fields** are `label`, `type`, `id`, `state`, `tag`, `link`, `formula`, `url`, `parent` and `children`. Their meaning is fixed by this spec. A type can't redefine them, except that a type may compute `label` and give `id` an auto rule (section 9).
- **M12.** The fields of a **type definition** (M17) and of a **search node** (M60) that this spec names as settings configure it; they are not data, and their child lines aren't graph data either.

## 4. Ownership

- **M13.** A graph node's owner is the node it is indented under (C9), or, for a node item, the node that has the field (C13). A file's root has no owner.
- **M14.** In CSV, a `parent` cell names the owner of the row, and a `children` cell names the rows it owns, in order. Both hold references (section 6). Only a file's top-level node can be re-owned this way; a node that has an owner by indentation keeps it, with the warning `M14.1`. A `parent` cell and a `children` cell that give one node different owners is the warning `M14.2`; the first one read wins.
- **M15.** Ownership never loops. Nodes in a `parent` or `children` loop get the error `M15` and have no owner.
- **M16.** A node's `children` are the nodes it owns: by indentation in file order, then by `children` cells in the order written, then by `parent` cells in file order.

## 5. Types and IDs

- **M17.** A node with a `defines` field is a **type definition**. Its value is the type's name, a name (L3). In the workspace file (Spec F5), a node whose line is only a `type` trigger, `+name`, is short for `[defines:: name]`: a type definition with an empty label, not a node of that type. Elsewhere such a line is a node of that type, as usual. A value that isn't a name is the error `M17.1`, and two definitions of one name are the error `M17.2`.
- **M18.** A node's **types** are its `type` values, in order, without repeats. A type without a definition is allowed.
- **M19.** A type definition's settings are its field lines `sigil`, `fields`, `state` and `color`. `color` is how editors show the type, e.g. as a pill: a color name (`red`, `orange`, `yellow`, `green`, `teal`, `blue`, `purple`, `pink`, `brown`, `gray`, `black`, `white`) or `#rrggbb`; or the name of a field, as in `color:: project`, so a node's type shows in the color of its value there: the `color` of the node the value refers to, else a color the editor picks by that node's label, or for a value that refers to no node, by its text. A node without the value shows the type as it would without the setting. A color name is never read as a field's name. In the workspace file, a node whose line is only `tag` triggers, with inline fields or field lines, sets those tags, as in `#urgent [color:: red]`; it isn't tagged itself, and its only setting is `color`. `sigil` binds one character (L15); it can't be whitespace, a letter, a digit or a fixed symbol (`# ^ ~ [ ] + = ! ; " ' ``), with the error `M19.1`, and a character bound twice is the error `M19.2`. No sigil is bound unless a definition binds it.
- **M20.** A node's **ID** is its `id` value. It must be a name (L3), with the error `M20.3`, and IDs match ignoring letter case. Two nodes with one ID is the error `M20.1`; both keep it, and references resolve to the first in file order (M5).

### Built-in types (informative)

A workspace usually starts with these definitions in its workspace file. They are ordinary type definitions, so fields and settings can be changed like on any type.

```
Person [defines:: person]
	fields::
		id:: auto $slug(label)
		email::
Task [defines:: task]
	fields::
		due::
```

Journal nodes, where tools put things for a date, are ordinary nodes found by their ID: `journal-` and the date, such as `Monday 12, 2026 ^journal-2026-10-12` and `October, 2026 ^journal-2026-10`. They have no type. Tools create them when something goes to a date.

## 6. References

- **M21.** A **reference** points from a field value to a graph node. These values are references:
  - a link (L10), in any field
  - a sigil (L15), to a node of its type
  - a plain text value of the `link` field
  - a plain text value of a reference field (M24), as written in a field line, quoted or not, and as written in an inline field when unquoted (L10)
  - a `parent` or `children` cell (M14)
- **M22.** A reference names a node by ID, by label, or both, as in L10. It resolves:
  - by ID alone (`^x1`, or ID and label): the node with that ID. If none has it, the error `M22.1`. The label is never a fallback; when it no longer matches the node, the warning `M22.2`.
  - by label alone (`"Anna Smith"`, or a quoted link): the node with that label, as in M23.
  - by text that could be either (`[[anna]]`, `client:: ruukki`): the node with that ID, or else the node with that label. When it is one node's ID and another node's label, the ID wins, with the warning `M22.3`.
- **M23.** A label matches a node whose label is equal to it, or whose label without its tags (L13) is, ignoring spaces and tabs at either end and letter case: `[[Buy milk]]` matches `Buy milk #urgent`, as does `[[Buy milk #urgent]]`. A label without tags is the label with each tag and the spaces and tabs before it removed. Letter case is compared by mapping both to lowercase with Unicode's default case mapping (not locale-specific); there is no other normalization, so `é` written as one character or as `e` and a combining accent are different labels. In a reference field with target types (M24), and for a sigil, only nodes of those types match, and a sigil matches only labels that are one word without their tags. No match is the warning `M23.1`; several matches is the warning `M23.2`, and the reference doesn't resolve.
- **M24.** A field is a **reference field** when a type's field definition gives it a `target` setting naming one or more types. A plain value of a reference field is then a reference to a node of those types. A field named after a type that a sigil is bound to (M19) is a reference field to that type without a setting, so `person:: anna` means what `@anna` does. Any other field's plain values are text, even when they name a node; a tool may suggest adding `target`, but doesn't resolve them.
- **M25.** References may form cycles. They are links, not ownership or dependencies.

## 7. States

- **M26.** A node's **state** is its single `state` value (M10), from its trigger (L11) or a field. It is a **built-in state**, `new`, `in-progress`, `waiting` or `done`, or a custom state (M27): one that a custom marker stands for (M29), or else the first in type order (M18) of the node's types that defines it. A value that is none of these is the warning `M26`, and the node keeps it as its state, with no base.
- **M27.** A type's `state` setting defines its **custom states**: `|`-separated parts, each a name (L3), an optional marker in brackets, and an optional `>` and **base**, a built-in state, as in `triage [t] > new | review [r] > in-progress | shipped > done | someday`. Spaces and tabs around the parts don't count. A part in another form, with a base that isn't a built-in state, or with the name of a built-in state or of a custom state the type already has, is the warning `M27.1`, and the part is left out.
- **M28.** A custom state's marker is one character. It can't be a built-in marker (` `, `-`, `?`, `x`), with the warning `M28.1`, and a marker can belong to only one custom state across all types, with the warning `M28.2` on each later one. Such a custom state has no marker.
- **M29.** A custom marker stands for the custom state it belongs to, on any node: `[r] Fix login` has the state `review`. A marker no custom state has is the warning `L11` (Spec section 20), and the node has no state.
- **M30.** A node's **base** is its state's base. A built-in state is its own base. A custom state without a base, and a state that is the warning `M26`, give the node no base.
- **M31.** `done` is true for a node exactly when its base is `done`.

## 8. Expansions

Expansions such as `~today` are a typing aid, written out by an editor as they are typed. To a reader they are plain text, so this section has no rules.

## 9. Field definitions and formulas

This section is a draft outline. Its rules are numbered so they can be referred to, and need tests before 1.0.

- **M36.** A type definition's `fields` setting holds **field definitions** as field lines under its anonymous node (C13). A definition's key is the field it defines; its value is its `value` setting, and its child field lines are its other settings: `value`, `valid`, `options`, `claims` and `target`.
- **M37.** A `value` of `=expr` makes a **computed field**: its value on each node of the type is the formula's result. A `value` of `auto expr` is an **auto value**: evaluated once, when a node has the type and the field is empty, and written as a plain value. Any other `value` is a default shown, never written.
- **M38.** A `label` field definition with `=expr` computes the node's label, which a writer writes into the node line, keeping the line's other forms (L4). Tags written on the line stay, after the computed text, so they are part of the label (L19) without making it differ from the computed one.
- **M39.** A field with a `=expr` value on a node that isn't a type definition is the error `M39`; the formula isn't evaluated.
- **M40.** A `valid` setting holds `=expr`, evaluated per node with `value` bound to the field's value, a list when it has several. A result that isn't true is the error `M40`, and the value stays.
- **M41.** **Formula language.** A formula's expression is JSONata 2.x, written `=(…)` so that `$` is allowed (L12). The current node is `$`, the graph root `$root`, and a node is an object with its fields as members, plus `label`, `id`, `type`, `parent` and `children`. A reference field's value is the node it refers to, and a field with field children (C13) is an object of them.
- **M42.** The function library adds `$node(text)`, `$today()`, `$now()`, `$date(text, days?)`, `$days(a, b?)`, `$format(x, picture)`, `$slug(text)`, `$first(x)` and `$last(x)`. Dates are ISO text. Their exact definitions are to be written here.
- **M43.** Formulas are pure, except `$today()`, `$now()` and `$date()` with a date name such as `"tomorrow"`, whose results follow the clock. Auto expressions may also use `$seq(n)`, `$uuid()`, `$id()` and `$sid()`.
- **M44.** Dependencies are per node and field. Fields in a cycle are empty, with the error `M44.1` naming the whole path; fields that depend on a cycle are empty, with the error `M44.2`. An evaluation error leaves the field empty, with the error `M44.3`.
- **M45.** A **slug** is the text in lowercase, accents removed, each run of characters other than letters and digits turned into one `-`, with leading digits and `-` and trailing `-` removed.
- **M46.** An `options` setting lists the values a field is meant to hold, as its items or as the labels of its node items without child lines (C13), in order. A value written for the field on a node of the type, other than a formula, that equals none of the options of the node's types, ignoring letter case, is the warning `M46`, and the value stays. Options say how a field is meant to be used, not its value type, and tools offer them first when a value is typed.
- **M47.** A `claims` setting lists tags, as `#name` items (`claims:: #bug #feature`) or child lines, the `#` optional on a line of its own. On a node of the type, a tag the field claims, ignoring letter case, is a value of that field instead, spelled as the setting spells it, and the node doesn't have that tag: `Crash +issue #bug` has `issue type:: bug` when the `issue type` field claims `#bug`. On other nodes the tag is a tag. Two words name the field's own lists, without a `#`: `options` claims each of its options that is a tag name as written (M46), so `Tech Debt` is claimed by no tag, while `tech-debt` is claimed by `#tech-debt`, and `target` claims the nodes of its target types (`target` setting), the value then referring to that node: first by its label as written, then by its label without its tags and sigils, each when it is a tag name, and last by its ID. With `target:: project` and `claims:: target`, `#ord-notes` on a task refers to `Ord-notes +project ^abc123`, and `#ord` to `Ord #new +project`, before a node whose ID is `ord`. A tag is matched only against what is written, never against a slug: `Ord Notes +project` is claimed by no tag unless it has an ID. When fields of two of a node's types claim one tag, the first type's field has it. The text stays as written.

## 10. Queries

This section is a draft outline, like section 9.

- **M60.** A node with a `search` field is a **search node**, whether the field comes from a field line, an inline field or an inline search `?? query` ([Ord specification](Ord_Specification.md) L28). Its query is the value of `search`, or, written as `search::` with child lines, each child line's label in parentheses, joined with spaces. When a node has more than one `search` field, such as `??` and a `search::` line under it, the queries are combined with and: each in parentheses, joined with spaces. This is allowed but not recommended. A search with an empty label is named by its query where a name is needed, such as its results document.
- **M61.** A query is terms, joined by spaces for and, `|` for or, and grouped by parentheses; `!` before a term or group negates it. A term is `+type`, `#tag`, a state trigger, a sigil reference, a link, a field `[key:: value]`, a formula `=(…)`, or a word, which matches labels.
- **M62.** A node matches `+type` when it has the type, `#tag` when it has the tag or a tag nested under it, ignoring letter case (`#project` matches `#project/apollo`), `[x]` and the other triggers when its state or its base is that state, a sigil or link when one of its fields refers to the node it resolves to, `[key:: value]` when one of its `key` values equals the text, ignoring letter case, a formula when its result is true, and a word when its label contains it, ignoring letter case.
- **M63.** Nodes tagged `archived` match only queries that name `#archived`.
- **M64.** The view settings (`view`, `columns`, `sort`, `group`, `footer`, `computed`) are implementation, like what they show. What `computed` writes into a file is still to be specified here.

## 11. Diagnostics

Codes, severities and positions work as in the Spec (C35 to C37). Positions are in the file the diagnostic is about.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `M10` | error | A node has a second `label`, `id` or `state` value | the second value |
| `M14.1` | warning | A `parent` or `children` cell names a node owned by indentation | the cell |
| `M14.2` | warning | `parent` and `children` cells give a node different owners | the later cell |
| `M15` | error | A node is in an ownership loop | the node |
| `M17.1` | error | A `defines` value isn't a name | the value |
| `M17.2` | error | A type is defined twice | the second `defines` value |
| `M19.1` | error | A `sigil` value can't be a sigil | the value |
| `M19.2` | error | A sigil is bound twice | the second `sigil` value |
| `M20.1` | error | Two nodes have one ID | each ID after the first, in file order (M5) |
| `M20.3` | error | An ID isn't a name | the ID |
| `M22.1` | error | A reference's ID matches no node | the reference |
| `M22.2` | warning | A reference's label no longer matches the node its ID names | the reference |
| `M22.3` | warning | A reference is one node's ID and another node's label | the reference |
| `M23.1` | warning | A label matches no node | the reference |
| `M23.2` | warning | A label matches several nodes | the reference |
| `M26` | warning | A `state` value is neither a built-in state nor a custom state of the node's types | the value |
| `M27.1` | warning | A custom state isn't `name [m] > base` with a built-in base and a new name | the `state` setting |
| `M28.1` | warning | A custom state uses a built-in marker | the `state` setting |
| `M28.2` | warning | A marker belongs to a custom state already | the `state` setting |
| `M39` | error | A formula on a node that isn't a type definition | the value |
| `M40` | error | A `valid` formula isn't true | the value |
| `M44.1` | error | A computed field is in a cycle | the field's definition |
| `M44.2` | error | A computed field depends on a cycle | the field's definition |
| `M44.3` | error | A formula fails to evaluate | the field's definition |
| `M46` | warning | A value is none of the field's `options` | the value |

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
    ref "link" -> a.ord:2
    ref "link" -> a.ord:1
```

```example graph M10 M26 M27 M29 M30
--- types.ord
Bug [defines:: bug]
	state:: triage [t] > new | review [r] > in-progress | shipped > done | someday
--- work.ord
[r] Fix login +bug
Ship it +bug
	state:: shipped
Later +bug
	state:: someday
[x] Twice +bug
	state:: shipped
[q] Unknown
Odd
	state:: shipped
.
file "types.ord"
  node "Bug" @1
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
! M10 work.ord:7:10
! L11 work.ord:8:1
! M26 work.ord:10:10
```

```example graph M5 M17 M18
--- Ord.workspace
Types
	+task
		fields::
			due::
--- a.ord
Write +task
+task
Task [defines:: task]
.
file "Ord.workspace"
  node "Types" @1
    node "" @2
file "a.ord"
  node "Write" @1
    type "task"
  node "" @2
    type "task"
  node "Task" @3
! M17.2 a.ord:3:6
```

```example graph M13 M14 M15 M20 M21 M24
--- a.ord
Invoice 42 ^i42
Anna +person ^anna
Erik ^anna
Now ^today
Task [defines:: task]
	fields::
		owner::
			target:: person
Fix +task
	owner:: "Anna"
	contacts::
		Erik
--- lines.csv
id,label,parent
l1,Workshop,i42
l2,Review,l3
l3,Loop,l2
.
file "a.ord"
  node "Invoice 42" @1
    id "i42"
    node "Workshop" @lines.csv:2
      id "l1"
      ref "parent" -> a.ord:1
  node "Anna" @2
    id "anna"
    type "person"
  node "Erik" @3
    id "anna"
  node "Now" @4
    id "today"
  node "Task" @5
  node "Fix" @9
    type "task"
    ref "owner" -> a.ord:2
    node "Erik" @12 in "contacts"
file "lines.csv"
  node "Review" @3
    id "l2"
    ref "parent" -> lines.csv:4
  node "Loop" @4
    id "l3"
    ref "parent" -> lines.csv:3
! M20.1 a.ord:3:6
! M15 lines.csv:3:1
! M15 lines.csv:4:1
```

```example graph M23
--- a.ord
Person [defines:: person]
	sigil:: @
Buy milk #urgent
Anna #friend +person
See [[Buy milk]], [[buy milk #urgent]] and @anna
.
file "a.ord"
  node "Person" @1
  node "Buy milk #urgent" @3
  node "Anna #friend" @4
    type "person"
  node "See [[Buy milk]], [[buy milk #urgent]] and @anna" @5
    ref "link" -> a.ord:3
    ref "link" -> a.ord:3
    ref "person" -> a.ord:4
```
