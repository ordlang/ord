# Ord in brief

Version 0.6.0-draft, 2026-11-03.

> Ord is a plain-text outline format. One line is one node, indentation makes the tree, and `key:: value` lines make the outline structured data. Files are UTF-8 text, usually `.txt` in a workspace (a folder with an `Ord.workspace` file) or `.ord` anywhere. This page is enough to read and write Ord correctly.

## Lines and the tree

- One line is one node. A child is indented one tab deeper than its parent.
- Blank lines don't affect the tree.
- List markers at the start of a line (`-`, `*`, `+`, `#`, `1.` followed by a space) are decoration. The node is the same with or without them.
- There are no headings. A node with children is a section. `# Heading` is a node with the label `Heading`; the `#` is a list marker, which editors may draw as a heading.

```
Project Apollo
	Plan the launch
		Book the venue
		Send invitations
	Budget
```

## Fields

- A **field line** is a key, `::`, a space and a value: `rate:: 1200`. It belongs to the node above it, one level up.
- Any key works, spaces included: `Project lead:: Anna`.
- `key::` with nothing after it is an empty field. Its child lines are its values.
- A line with `::` is a field line: the first `::` splits it into key and value, with or without a space after it, so `due::2026-10-15` is a field too. A `::` inside `[…]`, quotes or backticks doesn't count. A single colon is text, so `Note: remember this` is a node.
- `;` separates values: `tags:: red; blue`. It only splits field values, never node labels.
- An **inline field** sits on the node line in brackets: `Call Anna [due:: 2026-10-15]`. It means the same as a field line under the node. The value runs to the `]`, so it can have spaces: `[note:: two words]`. The key can be several words with single spaces between them, `[due date:: 2026-10-15]`; any other key is quoted, `["Project lead (2026)":: Anna]`.

```
Invoice Ruukki +invoice
	period:: 2026-Q3
	rate:: 1200
	Project lead:: Anna
	contact::
		email:: anna@example.com
```

## Marks on a node line

Marks are shorthand for system fields, whose keys start with `_`. They can be written anywhere on the line; a formatter moves them into this order: list marker, state, label, inline fields, types, ID. Tags, links, URLs and formulas are part of the label and stay where they are written.

| Mark | Meaning | Same as |
|---|---|---|
| `[ ]` `[-]` `[?]` `[x]` | state: new, in progress, waiting, done. Only at the start of the line | `[_state:: done]` |
| `+task` | type | `[_type:: task]` |
| `#urgent`, `#project/apollo` | tag, nested with `/` | `[_tag:: urgent]` |
| `[#Ord notes]` | a tag that is more than a name | `[_tag:: Ord notes]` |
| `^t9x1` | ID, one per node | `[_id:: t9x1]` |
| `[[Label]]`, `[[t9x1]]`, `[[Label^t9x1]]` | link to another node by label, ID or both | `[_link:: [[Label]]]` |
| `https://…`, `./file.pdf`, `[text](url)` | link to a URL or file | `[_url:: …]` |
| `=(net * 1.25)` | a computed value | `[_formula:: =(net * 1.25)]` |
| `@ Label`, `@ Label^t9x1` | a mirror: shows that node here. Only at the start of the line | `[_mirror:: Label]` |
| `? +task [ ]` | a search. Only at the start of the line | `+search [query:: +task [ ]]` |

- The **label** is what is left after removing the list marker, state, inline fields, types and ID. Tags, links, URLs and formulas stay in the label.
- `@anna` inside a line and `%apollo` are plain text. To refer to a node, link to it, `[[Anna]]`, or use a field whose definition has `options:: +person` (see below).
- `C++`, `C#`, `+46`, `#5` and `2 + 2 = 4` are not marks.

```
[x] Send mapping draft to [[Anna]] #urgent [due:: 2026-10-10] +task ^t9x1
```

State `done`, label `Send mapping draft to [[Anna]] #urgent`, field `due`, type `task`, tag `urgent`, ID `t9x1`.

## Types and references

A type is defined by a `+type-def` node with a `name` field, in any file, usually the workspace file. Its `fields::` list the type's fields, and a field whose `options` name a type, as `+person`, holds references to nodes of that type: a plain value is enough.

```
Task +type-def
	name:: task
	fields::
		owner::
			options:: +person
Anna +person
Call the venue +task
	owner:: Anna
```

`owner:: Anna` refers to the node `Anna +person`. Without `options:: +person`, it would be text.

- `options:: a; b` lists the values a field is meant to hold. With `tags:: true` as well, an option can be written as a tag on the node: with `options:: bug; feature` and `tags:: true` on an `issue type` field, `Crash +issue #bug` has `issue type:: bug`.
- `checkbox:: true` on a type definition gives its nodes a checkbox: a tool that adds the type to a node without a state adds `[ ]`.
- `nodes::` on a type definition is a template: a tool that adds the type to a node copies the nodes under `nodes::` under it.
- A field that several types share is defined once, as a `+field-def` node with a `key` field and the field's settings, and listed by its key under each type's `fields::`. It applies on every node that has the field.
- A tag's color is set by a `+tag-def` node: `Urgent +tag-def` with `name:: urgent` and `color:: red` under it.
- Keys that start with `_` are system fields, which the marks are short for, such as `_type`, `_id` and `_state`. `type::`, `state::` and `id::` are ordinary fields.

## Mirrors and searches

- A line `@ Label` is a mirror: it shows the node `Label` in this place without moving it. Add `^id` to name it by ID: `@ Label^t9x1`. Write nothing under a mirror; its children are its source's.
- A line `? query` is a search: `? +task [ ]` finds open tasks. Terms are joined by spaces for and, `|` for or, and `!` negates one.

```
Monday
	@ Send mapping draft^t9x1
	? +task [ ] #urgent
Apollo +project
	[ ] Send mapping draft #urgent +task ^t9x1
```

## Value forms

A field value that is exactly one of these forms reads as that type. Labels are never read this way.

| Form | Examples | Stays text |
|---|---|---|
| number | `1200`, `-12`, `9.5` | `007`, `+46`, `1e3`, `.5` |
| boolean | `true`, `False` | `yes`, `no` |
| date | `2026-10-15` | `2026-02-30` (with a warning) |
| datetime | `2026-10-15T14:30`, `2026-10-15 14:30`, `2026-10-15T12:30Z` | `2026-10-15T24:00` |
| week | `2026-W42` | `2026-W54` |
| month | `2026-10` | `2026-13` |

Quote a value to keep it text: `"2026-10-15"`.

## Turning syntax off

- `"…"` around a whole key, label or value makes it literal. JSON escapes apply inside: `\"`, `\\`, `\n`.
- Backticks around a word or phrase make it literal anywhere: `` `a::b; c` ``.
- A backslash before a mark makes it text: `\#tag`, `\+type`, `\^id`, `\[[x]]`, `\=x`, `\[due:: x]`.
- Three or more backticks on their own line open a raw block, closed by as many backticks. Nothing inside is read.
- `;` inside a value must be quoted: `title:: "Fish; chips"`.

## Rules for writing Ord

1. Indent with tabs, one per level.
2. Notes, comments and prose about a node are added as child lines under it.
3. Use fields for data points, attributes you would filter or compare on, references, and other structured information that is repeated on several nodes.
4. Write a node's fields before its child nodes.
5. Keep marks in order on the node line: state first, then the label with its `#tags`, then inline fields, `+types`, `^id`.
6. Refer to another node with `[[Label]]`, or with a plain value in a field whose definition has `options:: +type`. To list a node in a second place, use a mirror line `@ Label`, not a line that is only `[[Label]]`.
7. Quote values that contain `;` or start or end with spaces.
8. Don't use Markdown headings or bold for structure; use nesting.

## Right and wrong

Prose as a field:

```
Wrong:
Meeting with Anna
	note:: Ask about the budget
Right:
Meeting with Anna
	Ask about the budget
```

A colon is not a field:

```
Wrong (a node, not a field):
	Due: 2026-10-15
Right:
	due:: 2026-10-15
```

`::` in prose:

```
Wrong (a field line with the key "Use std"):
	Use std::vector here
Right:
	Use `std::vector` here
```

