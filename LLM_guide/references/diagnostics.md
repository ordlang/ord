# Ord diagnostics

Every file reads; problems are diagnostics with a code, a severity and a position (C35 to C37). A code is the rule that gives it, so look it up in the Spec (C, L, F), Semantics (M) or CSV mapping (V). Codes with a `/`, such as `ord/outlier`, belong to an implementation.

Fix every error you caused. Fix warnings you caused unless the user wants the text as written.

## Syntax (Spec)

| Code | Means | Usual fix |
|---|---|---|
| `C3.1`, `C3.2` | An odd whitespace character starts a line or touches `::` | Replace it with a space or tab |
| `C22` | `\` in `"…"` not followed by an escape | Write `\\`, or use backticks |
| `C29` | A block is never closed | Add the closing fence, as long as the opening one |
| `C49` | Bytes that aren't UTF-8 | Re-save the file as UTF-8 |
| `L8` | Two `^id` on one node line | Keep one |
| `L10` | `[[` without `]]` | Close the link, or write `\[[` |
| `L11` | A state marker like `[r]` that no type defines | Use a built-in marker, or define the custom state on the type |
| `L12` | A formula's `(` is never closed | Balance the parentheses |
| `L26` | A date, week or month out of range: `2026-02-30` | Fix the date, or quote it to keep it text |

## Workspace (Semantics)

| Code | Means | Usual fix |
|---|---|---|
| `M10` | A second `_label`, `_id`, `_state` or `_mirror` on one node | Remove one, e.g. a `_state::` line under a `[x]` node |
| `M11.1` | A `_name` value written on a node | Remove it: `_name` is read from the label |
| `M14.1`, `M14.2` | A CSV `_parent`/`_children` cell names a node owned by indentation, or the two disagree | Point at a top-level node, or make the cells agree |
| `M15` | An ownership loop through `_parent`/`_children` cells | Break the loop |
| `M17.1` | A type or tag definition's `name` isn't a name | Rename it |
| `M17.2` | A type or tag defined twice, or a system type defined in a file | Merge the definitions, or remove the one for a system type |
| `M17.3` | A `+type-def` without a `name` | Add `name::` |
| `M20.1` | Two nodes share an ID | Rename one ID and its references |
| `M20.3` | An ID isn't a name (`^12`) | Start it with a letter |
| `M22.1` | A reference's ID matches no node | Fix the ID, or add it to the intended node |
| `M22.2` | `[[Label^id]]` whose label no longer matches | Update the label in the link |
| `M22.3` | Text that is one node's ID and another's label; the ID wins | Use `[[^id]]` or `[["Label"]]` to say which |
| `M23.1` | A label matches no node (of the field's types) | Fix the spelling, create the node, or reference by ID |
| `M23.2` | A label matches several nodes | Reference by ID |
| `M26` | A state that is neither built in nor defined by the node's types | Use a defined state or define it |
| `M27.1`, `M28.1`, `M28.2` | A malformed custom state, or a marker built in or already used | Fix the type's `state` setting |
| `M39` | `key:: =(…)` on a setting of a type, tag or field definition, or of a search other than `footer` | Remove it; formulas go under a type's `fields::`, in a field definition's `value` or `valid`, or on an ordinary node's field |
| `M40` | Error: a value fails its field's `valid` check | Fix the value (it is kept as written) |
| `M44.1`, `M44.2` | A computed field is in, or depends on, a cycle | Break the cycle in the formulas, on the nodes and their types |
| `M44.3` | A formula fails, e.g. a generator outside `auto` | Fix the formula |
| `M46` | A value is none of the field's `options` | Use an option, or add it to `options` |
| `M50.1` | A mirror under its own source | Move or remove the mirror |
| `M50.2` | Lines under a mirror | Move them into the source |
| `M55.1` | Two `+field-def` nodes for one key | Merge them |
| `M55.2` | A `+field-def` for a system field | Remove it |
| `M55.3` | A `+field-def` without a `key` | Add `key::` |
| `M58.1` | A type lists a `+field-def` key with a value or child lines | Move the settings to the field definition node |

## CSV (CSV mapping)

| Code | Means |
|---|---|
| `V5.1` to `V5.3` | A stray `"`, text after a quoted cell, or an unclosed quoted cell |
| `V11.1`, `V11.2` | A row with fewer or more cells than the header |
| `V11.3`, `V11.4` | A second `_label` column, or a column without a name |

## Mistakes that give no diagnostic

These read fine but mean something else. Check for them yourself; the Gotchas in the guide have more.

- `due: 2026-10-10` (one colon) is a node, not a field.
- `Call note:: two words` makes the whole line a field with the key `Call note`. Write `Call [note:: two words]`.
- `Use std::vector` in prose is a field line; write ``Use `std::vector` ``.
- `=$sum(x)` is text; write `=($sum(x))`.
- `@anna` in a label is text.
- `+urgent` makes a type; a tag is `#urgent`.
- `state:: done` is an ordinary field, not the node's state; write `[x]` or `_state:: done`.
- `#client.` at the end of a sentence is the tag `client` and a full stop; `#123` and `C#` are not tags.
- `• item` is a node labelled `• item`.
- `~today` in a file is text, not a date.
- `days-left` in a formula is `days - left`; write `` `days-left` ``.
