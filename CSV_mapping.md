# Ord CSV mapping

Version 0.4.0-draft, 2026-10-23. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D` or `T` refer to the Spec and the [Test format](Test_format.md).

## 1. Scope

- **V1.** This spec maps CSV files to the Core data model (Spec, section 1) and back. A row is a node and a column is a field. The mapping is syntactic: columns such as `type`, `id`, `parent` and `children` are ordinary fields, which the index reads as it reads them in text files ([Semantics](Semantics.md)).
- **V2.** A **CSV mapper** reads and writes CSV as this spec says, and passes the tests with the `from-csv` and `to-csv` options (T4).

## 2. Reserved columns

- **V3.** A column named `label` holds the nodes' labels. Every other column is a field.

## 3. Reading CSV

- **V4.** A CSV file is UTF-8 (C49), with an optional BOM (C2). It is a list of records. Outside quoted cells, a record ends at `\n` or `\r\n`, and the last one may have no ending. A blank line is skipped.
- **V5.** Cells are separated by `,`. A cell that starts with `"` is quoted: it runs to the next `"` that isn't doubled, `""` in it stands for one `"`, and it can hold `,` and line endings. Mistakes are kept as text, with a warning:
  - a `"` in a cell that isn't quoted is kept
  - text after a quoted cell's closing `"`, up to the next `,`, is added to the cell
  - a quoted cell that is never closed runs to the end of the file
- **V6.** The first record is the **header**. Its cells, taken as written, are the column names.
- **V7.** The file is the root node (C14). Each record after the header is a node, a child of the root, in file order.
- **V8.** The first `label` column gives each node its label, as written. The label is quoted (Spec, section 1) only where it has to be, so that a cell's text never gains a meaning:
  - when Part 2, reading it as a node's label, would find a form or an escape in it (L1 to L20), such as `+task`, `#tag`, `[x] `, `- `, `[[x]]`, a URL or `\#`
  - when a writer would quote it (C26), or drop spaces or tabs at its end (C34)

  So `Send draft +task` is quoted and `+task` stays text, while `Short row` isn't quoted. An empty cell, or no `label` column, gives an empty label that is quoted, so a writer writes it `""` (Spec C42).
- **V9.** Every other column gives each node a field, in column order, whose key is the column name. Its value is the cell, read as a field's value in a text file (Spec, section 9): split into items at `;`, with spans and quotes as there. So an empty cell gives a field without items (C16).
- **V10.** In a node read from CSV, Part 2 reads only value forms (L21 to L27). Label syntax and value triggers are never read in cells, so `@anna`, `[[Ruukki AB]]`, `https://example.com` and `=sum(x)` are text.
- **V11.** A record with fewer cells than the header gets empty cells for the rest. Cells after the last column are left out. A second `label` column is an ordinary field, and a column whose name is empty is left out. Each of these gives a warning.

```example from-csv labels V3 V4 V6 V7 V8 V9 V10
id,type,label,state,due,attendees
t9x1,task,Send draft +task,new,2026-10-10,@anna; erik
,task,"Book room, big",new,,
.
root
  node quoted "Send draft +task"
    field "id"
      "t9x1"
    field "type"
      "task"
    field "state"
      "new"
    field "due"
      "2026-10-10" date
    field "attendees"
      "@anna"
      "erik"
  node "Book room, big"
    field "id"
    field "type"
      "task"
    field "state"
      "new"
    field "due"
    field "attendees"
```

```example from-csv V8
label
Plain
Anna's notes
Send draft +task
- [x] Done
Meet ~today
See https://example.com
\#hashtag
"Trailing␣"
;
Ends with colons::
.
root
  node "Plain"
  node "Anna's notes"
  node quoted "Send draft +task"
  node quoted "- [x] Done"
  node "Meet ~today"
  node quoted "See https://example.com"
  node quoted "\\#hashtag"
  node quoted "Trailing "
  node ";"
  node quoted "Ends with colons::"
```

```example from-csv V5 V9 V11
label,tags
"Say ""hi""","a; b;"
Short row
x,y,extra
.
root
  node "Say \"hi\""
    field "tags"
      "a"
      "b"
  node "Short row"
    field "tags"
  node "x"
    field "tags"
      "y"
! V11.1 3:1
! V11.2 4:4
```

## 4. Writing CSV

- **V12.** Only a flat model can be written as CSV: the root has no fields, its children are nodes, and their children are fields with inline items only (C38). A writer rejects any other model.
- **V13.** The header has a `label` column first, when any node has a label that isn't empty, and then each key in the order it first appears, going through the nodes and their fields in order.
- **V14.** A node's label is written as it is, without Ord quotes. A field's cell is its inline items, written as in a field line (C43) and quoted when C26 says so, with `` ` `` quotes when the text holds no `` ` `` and nothing C47 escapes, and with `"` quotes otherwise. Fields with the same key on one node share a cell: their items in order. A key the node doesn't have gives an empty cell.
- **V15.** A cell that holds `,`, `"`, `\r` or `\n` is written in `"` quotes, with each `"` doubled. Every record ends with `\n`, and a new file has no BOM. An edit keeps the file's line ending and BOM, as C45 says for text.

```example to-csv V3 V12 V13 V14 V15
Send draft +task
→type:: task
→due:: 2026-10-10
→attendees:: anna; erik
"Book room, big"
→due::
→tags:: a;
Plain
→note:: "x; y"
.
label,type,due,attendees,tags,note
Send draft +task,task,2026-10-10,anna; erik,,
"Book room, big",,,,a,
Plain,,,,,`x; y`
```

## 5. Known gaps

- CSV is flat: there are no child nodes, node items, list nodes or field children. Hierarchy is written as fields, such as `parent` and `children` ([Semantics](Semantics.md), section 4).
- Every row has every column, so a field that a node doesn't have comes back as a field without items.
- Fields with the same key on one node come back as one field (V14).
- A node's fields come back in column order.
- A label that Part 2 reads in a text file, such as `Send draft +task`, keeps its text but is no longer read. Converting moves such forms into columns first.
- Value triggers in cells are text (V10).
- A label keeps its quotes only where they're needed (V8): `"Plain"` in a text file comes back as `Plain`, and an empty label comes back quoted, so `::` in a text file comes back as `""`.
- A field named `label` doesn't round-trip.

## 6. Diagnostics

Codes and severities work as in the Spec (C35 to C37). Positions are in the CSV text, with lines and columns counted as in the Spec (D9).

| Code | Severity | Given when | Position |
|---|---|---|---|
| `V5.1` | warning | A cell that isn't quoted holds a `"` | the `"` |
| `V5.2` | warning | Text follows a quoted cell's closing `"` | the first character after it |
| `V5.3` | warning | A quoted cell is never closed | its opening `"` |
| `V11.1` | warning | A record has fewer cells than the header | the record's first character |
| `V11.2` | warning | A record has more cells than the header | the `,` before the first extra cell |
| `V11.3` | warning | The header has a second `label` column | that column's name |
| `V11.4` | warning | A column's name is empty | the `,` before it, or 1:1 for the first column |
