# Ord formulas (JSONata)

Rules: Semantics M36 to M45.

A formula is a JSONata 2.x expression written as `=(…)` (M41, Spec L12). Always wrap it in parentheses: `=` must be followed by a letter, `!` or `(`, so `=$sum(x)` is text and `=($sum(x))` a formula. Inside the parentheses, `;` doesn't split a field's items (C15), so `=($a := 2; $a * 3)` is one value.

## Where formulas go

| Where | Written | Notes |
|---|---|---|
| Computed field, on a type | `total:: =($sum(_children.amount))` under `fields::` | Shown by tools, never written into nodes |
| Computed field, on one node | `total:: =($sum(_children.amount))` on the node's own field line | Replaces the type's formula on that node (M39) |
| Auto value, on a type | `_id:: auto $slug(_label)`, `code:: auto "T-" & $seq(3)` | Written once, when the node gets the type and the field is empty |
| Computed label, on a type | `_label:: =(first & " " & last)` | Written into the node line by editors; tags on the line stay (M38). Don't edit it by hand |
| Check, under a field definition | `valid:: =($value in ["low", "high"])` | A result that isn't true is the error M40; the value stays |
| Inline, in a label | `Total =(a + b)` | Output only |
| Query, template column, footer | `=(due < $today())` | Output only, see queries.md |

What a node writes comes first (M39): a formula on the node's field line replaces its types' formula for that field, and so does a plain value (`total:: 12`). Put a formula on the type when every node of the type needs it, and on a node only for an exception. Not on settings: `footer:: =(…)` or `color:: =(…)` on a type, tag or field definition, or on a search, is the error M39 and isn't evaluated; a field definition's `value` and `valid` are the exceptions.

## The data model a formula sees

- `$` is the current node, so bare names are its fields: `due`, `hours`, `client`.
- `$root` is the graph root; its `_children` are each file's root node.
- A field with one value is that value; several values are an array.
- Numbers are numbers, booleans booleans. Dates, datetimes, weeks and months are ISO text, so `due < $today()` compares correctly.
- System fields have their `_` keys: `_label` (with its tags), `_id`, `_type` (an array of types), `_state`, `_tag` (nested tags as `project/apollo`), `_parent` (the owner) and `_children` (child nodes, not field items). A mirror child stands for its source in `_children` (M49).
- A reference field's value is the node it refers to: `client._label`, `owner.email`.
- A field with field children is an object: with `contact::` holding `primary:: anna` on a child line, `contact.primary` is `anna`. Steps continue through references only when the field is a reference field (`options:: +person`).
- Keys that aren't plain JSONata names go in backticks: `` `Project lead`.email ``. That includes every key with a `-`: `days-left` reads as `days - left`, so write `` `days-left` ``. Prefer keys without `-` for fields other formulas use.
- A field with only a default (a plain `value`) is empty to formulas; defaults are never written.
- Everything else is JSONata: mapping `_children.(price * qty)`, filtering `_children[_state != "done"]`, `&` for text, `+` for numbers only, `and`, `or`, `$not()`, `? :`, `in`, array literals, `$x := …` bindings, and `$`-functions.

## Ord's functions (M42, M43, M74 to M76)

| Function | Gives |
|---|---|
| `$node(text)` | The node with that ID, else that label, like a link. `$node("^i42")` matches the ID only. `$node("i42").gross` |
| `$nodes(under?)` | Every node, at any depth, in file order; with `under` (a node or its ID or label), the nodes under it. Filter on any field: `$nodes()["task" in _type and _state != "done"]` |
| `$query(text, under?)` | The nodes an Ord query matches, by the query rules (queries.md): `$query("+task #project [ ]")`. No formula inside: filter the result instead |
| `$today()` | Today's date, local: `2026-10-22` |
| `$now()` | The date and time, local: `2026-10-22T14:30` |
| `$date(text, days?)` | A date from ISO text or a date name, shifted by whole days: `$date(due, 7)`, `$date("tomorrow")`, `$date("next-monday")` |
| `$date(period)` | A period name gives all its dates: `due in $date("this-week")` |
| `$days(a, b?)` | Whole days from a to b, or from today to a: `$days(due)` |
| `$format(x, picture)` | A number or date with a JSONata picture: `$format(1234.5, "#,##0.00")` gives `1,234.50` |
| `$slug(text)` | A slug (M45): lowercase, accents removed, other runs to `-`, leading digits dropped. `"Åsa Öberg!"` gives `asa-oberg` |
| `$first(x)`, `$last(x)` | The first or last value |
| `$seq(n)` | Auto only: the type's next number, padded to n digits |
| `$uuid()`, `$id()`, `$sid()` | Auto only: a UUID, six random characters, the label's slug plus a short random suffix (`buy-milk-y7k`) |

Date names: `today`, `tomorrow`, `yesterday`, `monday` to `sunday` (the next one, today included), `this-monday`, `next-monday`, `last-monday` and so on, and the periods `this-week`, `next-week`, `last-week`, and the same for `month` and `year`. Weeks start on Monday (ISO 8601).

JSONata's built-ins work as usual: text (`$uppercase`, `$lowercase`, `$substring`, `$contains`, `$split`, `$join`, `$replace`, `$trim`, `$pad`, `$match`, `$length`, `$string`), numbers (`$sum`, `$count`, `$average`, `$min`, `$max`, `$round`, `$floor`, `$ceil`, `$abs`, `$number`, `$formatNumber`) and lists (`$exists`, `$distinct`, `$sort`, `$reverse`, `$map`, `$filter`, `$reduce`).

## Rules

- **Date arithmetic.** Dates are text, so `due + 7` doesn't work. Use `$date(due, 7)` and `$days(start, end)`.
- **Purity.** Only `$today()`, `$now()` and `$date()` with a date name follow the clock.
- **Generators only in `auto`.** `$seq`, `$uuid`, `$id` and `$sid` would change on every evaluation, so they are errors in live formulas and queries.
- **No `~` in formulas.** Use `$date("tomorrow")`, not `~tomorrow`.
- **Errors and cycles (M44).** A failing formula (M44.3), a cycle (M44.1) or a dependency on a cycle (M44.2) leaves the field empty with a diagnostic. Cycles are per node, so `hours:: =($sum(_children.hours))` on a tree is fine.
- **`valid` checks (M40).** `$value` is the field's value, a list when it has several.

## Examples

```
Invoice +type-def
	name:: invoice
	fields::
		client::
			options:: +company
		net:: =($sum(_children.amount))
		gross:: =(net * 1.25)
		code:: auto "INV-" & $seq(3)
Task +type-def
	name:: task
	fields::
		due::
		late:: =($exists(due) and due < $today() and _state != "done")
		daysleft:: =($days(due))
		thisweek:: =(due in $date("this-week"))
		priority::
			valid:: =($value in ["low", "medium", "high"])
Plant +type-def
	name:: plant
	fields::
		watered::
		every::
		next:: =($date(watered, every))
		thirsty:: =(next <= $today())
Person +type-def
	name:: person
	fields::
		_id:: auto $slug(_label)
		_label:: =(first & " " & last)
		initials:: =($uppercase($substring(first, 0, 1) & $substring(last, 0, 1)))
```

More one-liners:

- `=($count(_children[_state = "done"]))`: done children
- `=($node("i42").gross)`: a field on another node
- `=("task" in _type)`: has the type (`_type` is a list, so use `in`, not `=`)
- ``=(`days-left` < 3)``: a hyphenated field, in backticks
- `=($count(_children[("task" in _type) and _state != "done"]))`: open tasks under this node
- `=($count($nodes($)[("task" in _type) and _state != "done"]))`: open tasks at any depth under this node
- `=($count($query("+task [ ]")))`: open tasks in the workspace, by the query rules (`#project` also matches `#project/apollo`)
- `=($sum($query("+task", "Apollo").hours))`: hours of the tasks under the node `Apollo`
- `=(_parent._label)`: the owner's label
- `=($node("journal-" & $today()))`: today's journal node
- `=($sum(_children.(price * qty)))`: a sum of products
- `=($format(net, "#,##0.00") & " SEK")`: formatted text
