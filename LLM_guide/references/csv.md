# CSV files in an Ord workspace

Rules: [CSV mapping](../../CSV_mapping.md) (V1 to V15), Semantics M14 to M16.

Inside a workspace, every `.csv` file is part of the graph: each row after the header is a node, and each column a field. Use CSV for many rows of the same shape (time logs, line items, contacts); use text files for prose and nesting.

```
_id,_types,_label,_parent,date,hours,_tags
l1,timelog,Design work,i42,2026-10-20,3.5,
,timelog,Review,i42,2026-10-21,1,billable; urgent
```

## Reading

- **Format.** UTF-8, `,` between cells, `.` as the decimal point, RFC 4180 quoting (`"…"` with `""` for a quote). The first row is the header.
- **`_label` column.** Its cell is the node's label, taken literally (V8): `Review +task #urgent` in a label cell is that text, with no type and no tag.
- **System columns.** `_id`, `_types`, `_state`, `_tags`, `_parent` and `_children` are read as the system fields (V1, M11). Types come only from the `_types` column, tags only from the `_tags` column. A column `type` or `state` is an ordinary field. All columns are optional.
- **Other cells** are read like a field value in a text file: split into items at `;`, with value forms (numbers, dates, booleans, weeks, months). Links, mirrors, URLs and formulas in cells are text (V10); use a reference column instead.
- **Quotes inside a cell are backticks**, since `"` is CSV's own quote: `` `x; y` `` is one item.
- **References.** `_parent`, `_children` and reference columns (a field with `options:: +type`) hold IDs, without brackets. A label still resolves, but write IDs.
- **Hierarchy (M14 to M16).** A `_parent` cell places the row under that node; a `_children` cell lists the rows a node owns, in order. Only a file's top-level nodes can be placed this way, and the two must not disagree or loop.
- **Missing cells** are empty; extra cells are dropped, with a warning.

## Writing and editing

- **Edit, don't rewrite.** Change only the rows and cells the change needs; keep the column order, unknown columns, line endings and BOM.
- **New rows** go at the end, with a cell for every column.
- **New columns** go at the end of the header, with an empty cell in every existing row.
- **Several values** in one cell are joined with `; `.
- **IDs are lazy.** A row needs an `_id` only when something refers to it.
- **Quoting.** Quote a cell in `"…"` (doubling inner quotes) when it holds `,`, `"` or a line break. Use backticks around an item that holds `;`.
- **Computed fields** are never columns.
- **Delete means archive** when the user means "remove from views": add `archived` to the row's `_tags` cell.

## Converting

- **Text to CSV** works only for nodes without child nodes, or after turning them into `_parent` references. The node line's forms become columns: `+task` fills `_types`, `^t9x1` `_id`, `[x]` `_state` (`done`), `#urgent` `_tags` (and leaves the label, since a label cell is literal), `[due:: 2026-10-10]` `due`. References become IDs.
- **CSV to text**: a row becomes a node line with field lines, or with its forms back on the line: `[x] Review +timelog ^l2` plus `date:: 2026-10-21`.
