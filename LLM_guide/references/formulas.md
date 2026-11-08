# Ord formulas (JSONata)

Rules: Semantics M36 to M45, M87.

A formula is a JSONata 2.x expression written as `=(…)` (M41, Spec L12). Without spaces it needs no parentheses: `=$sum(x)`, `=price*qty`, `=_name` and `=2*x` are formulas wherever `=(…)` is (L12). It ends at a space, tab or `;` outside parentheses and strings, and `.`, `,`, `:`, `?` or `!` at its end are prose. Wrap it in parentheses when it has spaces: `=(price * qty)`. Inside the parentheses, `;` doesn't split a field's items (C15), so `=($a := 2; $a * 3)` is one value. In a field's value, a formula can run over several lines (C52): leave `=(` open at the end of the field line, or of a line of only the formula under it, indent the lines after it, and close it with `)`, like a code block:

```
total:: =(
	$sum(
		_children.(price * qty)
	)
)
```

A node line that is only the formula can run over lines the same way, and keeps its child lines after the closing `)`. A formula after other text in a label stays on one line.

## Where formulas go

| Where | Written | Notes |
|---|---|---|
| Computed field, on a node | `total:: =($sum(_children.amount))` on the node's field line, or `=(…)` alone on a child line of `total::` | Written by hand, for that node only. Each formula is an item: its result, one item per list element (M39). Mix with plain items: `tags:: draft; =(project._tags)` |
| Calculated field, on a type | `total:: =($sum(_children.amount))` under `fields::`, the formula on the key's line | Every node of the type has it, never written on them (M87); editors show `total = 1200` first under the node. A node can't write it (M87.1). Changing the formula changes no file |
| One-time formula | `due:: ~$today()`, `code:: ~("T-" & $seq(3))`, `_id:: ~$slug(_label)` on a type | Evaluated once and replaced in the file by its result (M85); on a type, written on each node and replaced there. An error stays, shown after it |
| Check, under a field definition | `valid:: =($value in ["low", "high"])` | A result that isn't true is the error M40; the value stays |
| Inline, in a label | `Total =(a + b)` | Output only |
| Query, template column, footer | `=(due < $today())` | Output only, see queries.md |

A type's formula is calculated on its nodes without being written there (M87); everything else a node has is written on it (M37). On a node, a formula can go wherever a value goes, inline fields included, and a field can hold several; editors show each result after its formula. A node can't override a calculated field: writing `total:: 12` on it is the error M87.1. A type can't calculate `_label` (M58.1). Not on settings: `footer:: =(…)` or `color:: =(…)` on a type, tag or field definition, or on a search, is the error M39 and isn't evaluated; a field definition's `value` and `valid` are the exceptions.

## The data model a formula sees

- `$` is the current node, so bare names are its fields: `due`, `hours`, `client`.
- `$root` is the graph root; its `_children` are each file's root node.
- A field with one value is that value; several values are an array.
- Numbers are numbers, booleans booleans. Dates, datetimes, weeks and months are ISO text, so `due < $today()` compares correctly.
- System fields have their `_` keys: `_label` (with its tags), `_name` (the label without the tags at its end, each formula replaced by its result, as links match it: `Send proof #mail` is `Send proof`, `Parent =("bad")` is `Parent bad`; alias `N`), `_result` (the results of the label's formulas, alias `R`; an object result's members are read through it, `R.a`, not as the node's fields), `_id`, `_types` (an array of types), `_state`, `_tags` (nested tags as `project/apollo`), `_parent` (the owner) and `_children` (child nodes, not field items). A mirror child stands for its source in `_children` (M49).
- A reference field's value is the node it refers to: `client._label`, `owner.email`.
- A field with field children is an object: with `contact::` holding `primary:: anna` on a child line, `contact.primary` is `anna`. A formula in such a nested field line is evaluated on the node, not on the field: `$`, `_label`, `_parent` and the rest are the node's, at any depth (M41). Steps continue through references only when the field is a reference field (`options:: +person`).
- A result that is an object is Ord fields (M39): `=({"a": 1, "b": {"c": 2}})` shows `a = 1`, then `b::` with `c = 2` under it, under its line, as calculated fields show (`a:: 1, b.c:: 2` where it must fit one line, as in a name), and `x:: ~({"a": 1, "b": {"c": 2}})` is written as `x::` with `a:: 1` and `b::` under it, `c:: 2` under `b::`. A list is items joined by `; `. Several objects are node lines under the field, each with its first plain field inline: `x:: ~([{"a": 1}, {"a": 2}])` becomes `x::` with `[a:: 1]` and `[a:: 2]` under it, as JSON arrays are read (J10).
- Keys that aren't plain JSONata names go in backticks: `` `Project lead`.email ``. That includes every key with a `-`: `days-left` reads as `days - left`, so write `` `days-left` ``. Prefer keys without `-` for fields other formulas use.
- A node has only the fields written on it; a type adds nothing at read time.
- A result can be a function, and other formulas call it: `double:: =(function($x){ $x * 2 })` on `Lib`, then `$node("Lib").double(21)` is 42. Its `$` and bare names are `Lib`'s, so pass the caller in: `$node("Lib").label($$)`, since `$` inside the path is `Lib`. Editors show it as `ƒ($x)`. A one-time formula can't give one.
- Workspace formulas: fields under a `formulas::` line in `Ord.workspace`, at any depth, are `$` names in every formula: `formulas::` with `double:: =(function($x){ $x * 2 })`, `events:: =($nodes()["event" in _types])` and `rate:: 0.25` under it, then `$double(21)`, `$count($events)`, `100 * $rate` anywhere. In them, `$` is `$root`, wherever they are written. Keys must be `$` names and not a function's (`sum`, `node`): M41.1; given twice: M41.2.
- Everything else is JSONata: mapping `_children.(price * qty)`, filtering `_children[_state != "done"]`, `&` for text, `+` for numbers only, `and`, `or`, `$not()`, `? :`, `in`, array literals, `$x := …` bindings, and `$`-functions.

## Ord's functions (M42, M43, M74 to M76)

| Function | Gives |
|---|---|
| `$node(text)` | The node with that ID, else that label, like a link. `$node("^i42")` matches the ID only. `$node("i42").gross` |
| `$nodes(under?)` | Every node, at any depth, in file order; with `under` (a node or its ID or label), the nodes under it. Filter on any field: `$nodes()["task" in _types and _state != "done"]` |
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
- **Purity.** Only `$today()`, `$now()`, `$date()` with a date name and `$days()` with one argument follow the clock.
- **Generators only in `~`.** `$seq`, `$uuid`, `$id` and `$sid` would change on every evaluation, so they are errors in `=` formulas and queries (M43).
- **`~` writes once.** `~$today()` becomes the date once the cursor leaves it; `=$today()` stays a formula and follows the clock. A result is written as any field is: `~(["a"])` gives `a;`, text with `;` or a line break is quoted. `auto expr` is gone: write `~(expr)` (M85.1).
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
		code:: ~("INV-" & $seq(3))
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
		_id:: ~$slug(_label)
		full name:: =(first & " " & last)
		initials:: =($uppercase($substring(first, 0, 1) & $substring(last, 0, 1)))
```

More one-liners:

- `=($count(_children[_state = "done"]))`: done children
- `=($node("i42").gross)`: a field on another node
- `=("task" in _types)`: has the type (`_types` is a list, so use `in`, not `=`)
- ``=(`days-left` < 3)``: a hyphenated field, in backticks
- `=($count(_children[("task" in _types) and _state != "done"]))`: open tasks under this node
- `=($count($nodes($)[("task" in _types) and _state != "done"]))`: open tasks at any depth under this node
- `=($count($query("+task [ ]")))`: open tasks in the workspace, by the query rules (`#project` also matches `#project/apollo`)
- `=($sum($query("+task", "Apollo").hours))`: hours of the tasks under the node `Apollo`
- `=(_parent._label)`: the owner's label
- `=($node("journal-" & $today()))`: today's journal node
- `=($sum(_children.(price * qty)))`: a sum of products
- `=($format(net, "#,##0.00") & " SEK")`: formatted text
