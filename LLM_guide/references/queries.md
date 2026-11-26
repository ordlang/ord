# Ord queries and search nodes

Rules: Semantics M60 to M65.

## Search nodes

A search node is a saved search (M60), written one way: `? query` (L28), on its own line, after any list marker. The query is everything after `? `, on that one line. Its view settings go on field lines under it. Results are never written into the file.

```
? +task ![x]
? +task [[Anna]] =(late)
	view:: table
	sort:: due asc
```

- Nothing after `? ` is read as a form, so the line gets no type, tags or ID, and no label: a search is named by its query.
- `?x`, `??` and `\?` are text.
- Never write `+search` or a `query::` line: both are the error M60.1, and make no search. Older files used them; rewrite them as `? query`.

## Query terms (M61, M62, M65)

A query looks like the node lines it matches.

| Term | Matches nodes |
|---|---|
| `+task` | of that type |
| `#urgent` | with that tag, or one nested under it: `#project` matches `#project/apollo` |
| `#apollo` | also with a field whose `tags:: true` would take the tag: `#apollo` matches `[project:: Apollo]` when `project` has `options:: +project` and `tags:: true` |
| `[ ]`, `[-]`, `[?]`, `[x]` | in that built-in state, or with that base: `[x]` also matches custom states based on done |
| `[r]` | in the custom state whose marker is `r` |
| `[[Acme Corp]]`, `[[acme]]` | with a field referring to that node, by label or ID |
| `@Acme Corp` | the node itself; in a scoped query, when it or one of its mirrors is in scope |
| `[key:: value]` | with a `key` value equal to the text, ignoring case: `[client:: acme]`, `[due:: 2026-10-10]`. A calculated or computed field's value counts: `[total:: 2050]`. Compare with a formula: `=(total > 1000)` |
| `=(expr)` | where the JSONata formula is true for the node |
| `word`, `"two words"` | whose label contains it, ignoring case |

- **Combining.** A space means and, `|` or, parentheses group, and `!` negates a term or group: `+task (#red | #blue) ![x] ![[Erik]]`.
- **Mirrors.** A query matches a node in its own place, never a mirror, so a node mirrored in three places is one result.
- **Archived nodes (M63).** Nodes tagged `archived` are left out unless the query names `#archived`.
- **System fields (M11)** keep their `_` keys in terms and filters: `[_state:: done]`, `[_id:: t9x1]`, `[_tags:: urgent]`, `=(_parent._label = "Inbox")`.
- **Filters are formulas.** There are no comparison operators in the query itself: write `=(due < $today() and hours > 2)`. Formulas in queries may use `$today()` and `$date("this-week")`, not generators. See formulas.md.
- **Hyphenated keys in filters** need backticks: ``=(`days-left` < 3)``.

## View settings (M64)

View settings (Semantics M64); how they look is up to the editor:

| Setting | Value |
|---|---|
| `view` | `table`, `list`, `board` or `calendar` |
| `fields` | one child field line per field shown (below) |
| `sort` | conditions separated by `;`, each `field <asc|desc> <number|num|text|date>`, asc and text by default: `sort:: flow; due desc date` |
| `group` | a field to group by: `group:: project` |
| `footer` | `=expr` over the results, with their nodes as `$`: `=($sum(hours))`. Shown with each group, or board column |

### Fields

```
? +timelog =(date in $date("this-month"))
	view:: table
	sort:: date desc
	fields::
		_label::
		date::
		client::
		hours::
		daysago:: =($days(date, $today()))
```

- `key::` shows a field. System fields keep their `_` keys: `_label::`, `_id::`, `_state::`, `_tags::`. `key:: =expr` is a template field, computed per row, output only.
- Nothing else goes under `fields::`: no `header`, `width` or `format`. Name a template column by its key.
- `columns::` is the old name of `fields::`.

## Writing searches well

- Put the search where the user will look for it: a project node, the workspace file, or today's journal day for a quick `?`.
- Prefer types, tags and states over words; words only search labels.
- Use `![x]` for "not done": it also covers custom states based on done.
- When a filter repeats, make it a calculated field on the type (`late:: =(…)`) and query `=(late)`.
- Don't write results, counts or dates computed from the query into the file.
