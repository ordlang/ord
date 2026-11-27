# Ord JSON mapping

Version 0.7.0-draft, 2026-11-22. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D` or `T` refer to the Spec and the [Test format](Test_format.md).

## 1. Scope

- **J17.** This spec maps JSON to the Core data model (Spec, section 1) and back. Its conventions exist so that JSON → Ord → JSON keeps what it can; section 5 lists what it can't.
- **J1a.** The mapping is syntactic. It doesn't read label syntax (Spec Part 2) or resolve references, so a reference is the text written, e.g. `client:: acme` gives `"client": "acme"`. It reads only these value forms: numbers (L22) and booleans (L27), which JSON has types for, and `[]` and `{}`, an empty array and object (J4, J7, J9); and it reads the field `_json` (J19) and index keys (J20).
- **J18.** A **JSON mapper** reads and writes JSON as this spec says, and passes the tests with the `from-json` and `to-json` options (T4).

## 2. System keys

- **J1.** Two JSON keys hold a node's Core label and child nodes, never user fields. They are named like the system fields `_label` and `_children` ([Semantics](Semantics.md) M11), though the mapping, being syntactic (J1a), doesn't read them as such. Every other key is an ordinary field, also one that starts with `_`, which Semantics reserves (M92) but this mapping doesn't check.
  - `_label` holds a node's label.
  - `_children` holds a node's child nodes.
  - A `_label` that isn't a string, or a `_children` that isn't an array, gives a warning and is kept as an ordinary field.
  - A key `_json` becomes a field with its key quoted, `"_json":: x`, an ordinary field; only an unquoted `_json` says what a node is in JSON (J19).
- **J19.** The field `_json` ([Semantics](Semantics.md) M11), with its key unquoted, says what a node is in JSON where the other rules would read it as something else. It is written as a field line, the node's first, and its value is one of:
  - `object`: the node is a JSON object (J3) even when it has only a label, `{"_label": "x"}`, or index fields (J20).
  - `value`, on the root only: the file is the JSON value of its one root line, for a scalar root or an empty one, `[]` (J15).

  Under a field, `_json:: object` is a line of the field's anonymous node (C13). `_json` is never written as a JSON key. One with another value, `array` included, where its kind doesn't allow it, twice, or with lines its kind doesn't have, is the warning `J19.1`, and what doesn't fit is left out. Writers write `_json` only where it is needed: on an object of a label alone, and for a scalar or empty root.
- **J20.** An **index node** is a node without a label or with an empty unquoted one (`::`), the root included, whose children are only field lines, at least one, with unquoted keys `0`, `1`, `2` and so on, in file order: each key is its line's position, in digits without a leading zero. It is a JSON array, and its items are its fields' values (J5), in order. So `0:: a`, then `1::` with `b:: 2` under it, is `["a", {"b": 2}]`, and an index field `1:: x; y` is the item `["x", "y"]`. A node with a label, a child node, a quoted key (`"0":: a`) or `_json:: object` (J19) is an object. Writers write an array as index fields where no other rule maps it (J10, J11, J15), and quote the keys of an object that would read as an index node: `{"0": "a"}` gives `"0":: a`.

## 3. Reading JSON

JSON is read by writing it as Ord, as these rules say, and reading the result (Spec, Part 1).

- **J8.** An object becomes a node. `_label` becomes its label, and an object without one a node with an empty label, `::`. Its other members become field lines, their keys quoted where they would make an index node (J20), and `_children` child node lines. An object without members becomes `{}` (J4), and one with only `_label`, which would be a string, gets `_json:: object` as its first line (J19).
- **J9.** A string becomes the field's value, quoted (C26) when it is a number (L22), a boolean (L27), `[]` or `{}` as written, so that it stays text: `"6"` gives `hours:: "6"`. A number becomes its text as written in the JSON, and a boolean `true` or `false`, so each reads back as its value form: `6`, `9.50` and `true` give `hours:: 6`, `9.50` and `true`. A number written with an exponent or a capital `E`, which L22 doesn't read, becomes its value in plain decimal: `1e21` gives `1000000000000000000000`, and `1e-7` gives `0.0000001`. `null` becomes an empty value, and an object without members `{}`. An object with only ordinary members becomes the field's field children (C13). Any other object becomes one child node following J8.
- **J10.** An array of strings, numbers, booleans and objects without members becomes the field's items, `key:: a; b; {}`, quoted as C26 and J9 say; one with text over several lines becomes the field's node children instead, a line per item following J11, so that the text is a block (C29). An array without `null`, nested arrays or text over several lines becomes field lines with the key. Each holds, in the array's order, such plain values as its items, then objects with a label or `_children` as its node children (J8), then at most one object with only ordinary members as its field children (C13); a new field line starts where that order breaks. Several field lines read back as one array (J7a):

  ```
  rows:: x
  →a:: 1
  rows::
  →a:: 3
  ```

  is `"rows": ["x", {"a": 1}, {"a": 3}]`. Where the key can't repeat, as for an index field (J20), such an array becomes index fields instead. Any other array, one with `null` or a nested array, or with text over several lines among other items, becomes the field's index fields (J20), under the field as its field children: `key::` with `0:: a` and `1::` under it is `"key": ["a", null]`. An array on one field line with one item or none is a list (C16), with a `;` after its items: `key:: a;`, `key:: ;`, and `key:: ;` with the object's fields under it for one object.
- **J11.** Within an array that becomes node lines (J10) or `_children` (J8), `null` becomes `::`, a string, number or boolean a node line, written as J9 writes an item, so a string that is a number, boolean, `[]` or `{}` as written is quoted, `"5"`, an object follows J8, an empty array becomes `[]`, and any other array `::` with its index fields under it (J20).
- **J12.** Text is quoted according to C26.
- **J15.** A root array becomes the root's index fields (J20). A scalar root, and an empty root array, become `_json:: value` as the file's first line, then the value as one root line, following J11: `[]` for the empty array (J19). A `_label` on the root object gives a warning and is kept as an ordinary field.

```example from-json J8 J9 J10
{
  "client": "Acme Corp",
  "contacts": ["Anna", "Erik"],
  "lines": [
    { "_label": "Workshop", "hours": 6 },
    { "_label": "Spec review", "hours": 9.5 }
  ]
}
.
root
  field "client"
    "Acme Corp"
  field "contacts"
    "Anna"
    "Erik"
  field "lines"
    node "Workshop"
      field "hours"
        "6"
    node "Spec review"
      field "hours"
        "9.5"
```

```example from-json J8 J9 J10 J11 J12 J20
{
  "a": null,
  "b": "",
  "c": ["x"],
  "d": [],
  "e": [null, "y", { "z": 1 }],
  "f": { "primary": "anna" },
  "g": { "_label": "Acme", "owner": "anna" },
  "h": [["a", "b"], []],
  "_children": ["Call Anna", { "_label": "Book room", "due": "2026-10-20" }]
}
.
root
  field "a"
  field "b"
    quoted ""
  field "c" list
    "x"
  field "d" list
  field "e"
    node
      field "0"
      field "1"
        "y"
      field "2"
        node
          field "z"
            "1"
  field "f"
    node
      field "primary"
        "anna"
  field "g"
    node "Acme"
      field "owner"
        "anna"
  field "h"
    node
      field "0"
        "a"
        "b"
      field "1" list
  node "Call Anna"
  node "Book room"
    field "due"
      "2026-10-20"
```

```example from-json J8 J10
{
  "tags": ["a", "b; c"],
  "one": ["x", { "p": 1 }],
  "rows": ["x", { "a": 1, "n": { "b": 2 } }, { "a": 3 }],
  "deep": [{ "n": { "b": 2 } }, { "a": 3 }],
  "mixed": [{ "a": 1 }, {}, "y", "z"],
  "named": [{ "a": 1 }, { "_label": "N" }]
}
.
root
  field "tags"
    "a"
    quoted "b; c"
  field "one"
    "x"
    node
      field "p"
        "1"
  field "rows"
    "x"
    node
      field "a"
        "1"
      field "n"
        node
          field "b"
            "2"
  field "rows"
    node
      field "a"
        "3"
  field "deep"
    node
      field "n"
        node
          field "b"
            "2"
  field "deep"
    node
      field "a"
        "3"
  field "mixed"
    node
      field "a"
        "1"
  field "mixed"
    "{}"
    "y"
    "z"
  field "named"
    node
      field "a"
        "1"
  field "named"
    node "N"
      field "_json"
        "object"
```

```example from-json J15
["a", ["b"]]
.
root
  field "0"
    "a"
  field "1" list
    "b"
```

```example from-json J15 J19
"x"
.
root
  field "_json"
    "value"
  node "x"
```

```example from-json J1 J15
{"_foo": "x", "_label": "y", "_children": "z"}
.
root
  field "_foo"
    "x"
  field "_label"
    "y"
  field "_children"
    "z"
! J15 1:15
! J1.2 1:30
```

```example from-json J1a J9
{
  "n": 1.5,
  "b": true,
  "l": [2, false],
  "s": "6",
  "t": "true",
  "v": "007",
  "rows": [{ "k": "1" }, { "k": 1 }]
}
.
root
  field "n"
    "1.5"
  field "b"
    "true"
  field "l"
    "2"
    "false"
  field "s"
    quoted "6"
  field "t"
    quoted "true"
  field "v"
    "007"
  field "rows"
    node
      field "k"
        quoted "1"
  field "rows"
    node
      field "k"
        "1"
```

```example from-json J9 J11 J19
{
  "big": 12345678901234567890,
  "e": 1e21,
  "mixed": [null, 5, true, "5"]
}
.
root
  field "big"
    "12345678901234567890"
  field "e"
    "1000000000000000000000"
  field "mixed"
    node
      field "0"
      field "1"
        "5"
      field "2"
        "true"
      field "3"
        quoted "5"
```

```example from-json J15 J19
5
.
root
  field "_json"
    "value"
  node "5"
```

## 4. Writing JSON

- **J2.** The root node becomes a JSON object, or, as an index node, an array (J20), or, with `_json:: value`, its one root line's value (J19).
- **J3.** A node with children becomes an object, unless it is an index node (J20):
  - `_label` is its label. It is omitted when the node is anonymous, or when its label is empty.
  - `_json` is not a key (J19).
  - Each field becomes a key.
  - `_children` is an array of its child nodes. It is omitted when there are none.
- **J4.** A node without children becomes its label, read as a field's text item is (J6, J7): an unquoted number or boolean is a JSON number or boolean, an unquoted `[]` or `{}` an empty array or object, any other label a string, and `null` when the label is empty and unquoted (`::`). So in JSON, a line alone and an item are read alike, though elsewhere labels have no value forms (L21). A node with `_json` follows J19.
- **J5.** A field's JSON value is:
  - an array of its items, if it is a list of one item or none (C16): `a;` gives `["a"]` and `;` gives `[]`
  - `null`, if it has no items
  - the item itself, if it has one item
  - an array of its items, if it has several
- **J6.** A text item becomes a string. A node item follows J3, J4 and J20.
- **J7.** An unquoted text item that is a number (L22) becomes a JSON number, written as it is written in Ord, and one that is a boolean (L27) `true` or `false`; an unquoted `[]` becomes an empty array and `{}` an empty object: `n:: 1.50; TRUE; []` gives `"n": [1.50, true, []]`. A number keeps all its digits, whatever a JSON reader can hold. Every other value is a string, quoted text (`"3"`), other value forms such as dates, and labels included.
- **J7a.** Fields with the same key on one node become one JSON key. Their items are concatenated in file order, and the result is always an array: `tag:: a` and `tag:: b` give `"tag": ["a", "b"]`. So a field line per record is an array of objects: `rows::` with `a:: 1` under it, then `rows::` with `a:: 3` under it, gives `"rows": [{"a": 1}, {"a": 3}]` (J10).
- **J16.** A field whose key is `_label` or `_children` is written as that JSON key, with a warning, since it reads back as the label or the child nodes (J1).

```example to-json J19 J20 J5 J7
rows::
→0::
→→a:: 1
→→n:: 2
→1:: x; y
→2::
→→N
→→→_json:: object
empty:: {}
none:: []
text:: "{}"
same::
→"0":: a
→1:: b
.
{
  "rows": [
    { "a": 1, "n": 2 },
    ["x", "y"],
    { "_label": "N" }
  ],
  "empty": {},
  "none": [],
  "text": "{}",
  "same": { "0": "a", "1": "b" }
}
```

```example to-json J2 J20
0:: a
1::
→0:: b
→1:: ;
.
["a", ["b", []]]
```


```example to-json J1a J2 J3 J4 J5 J6 J7 J7a
a::
b:: anna
c:: anna;
d:: anna; oscar
e:: a;;b
f::;
h:: ""
" hello":: world
i::
→Anna
→Gustav
j::
→primary:: anna
k:: x
→Anna
→primary:: anna
matrix::
→::
→→a
→→b
tag:: a
tag:: b
client:: [[Acme Corp]]
Acme
→Call Anna
→;
.
{
  "a": null,
  "b": "anna",
  "c": ["anna"],
  "d": ["anna", "oscar"],
  "e": ["a", "b"],
  "f": [],
  "h": "",
  " hello": "world",
  "i": ["Anna", "Gustav"],
  "j": { "primary": "anna" },
  "k": ["x", "Anna", { "primary": "anna" }],
  "matrix": { "_children": ["a", "b"] },
  "tag": ["a", "b"],
  "client": "[[Acme Corp]]",
  "_children": [{ "_label": "Acme", "_children": ["Call Anna", ";"] }]
}
```

```example to-json J7a J10
rows:: x
→a:: 1
→n:: 2
rows::
→a:: 3
rows:: y
.
{
  "rows": ["x", { "a": 1, "n": 2 }, { "a": 3 }, "y"]
}
```

```example to-json J7
n:: 1.50; -2; TRUE
s:: "3"
d:: 2026-10-15
v:: 007; 1e3
.
{
  "n": [1.50, -2, true],
  "s": "3",
  "d": "2026-10-15",
  "v": ["007", "1e3"]
}
```

```example to-json J2 J4 J19 J20
_json:: value
::
→0:: 1.50
→1:: TRUE
→2:: "7"
→3:: yes
.
[1.50, true, "7", "yes"]
```

```example to-json J4 J11 J20
::
→0:: a
[]
{}
::
.
{
  "_children": [["a"], [], {}, null]
}
```


## 5. Known gaps

JSON read as Ord and written back is the same JSON value, with every number as written except those J9 writes in plain decimal. What is lost is Ord's layout, which JSON has no place for:

- An Ord field whose key is `_label` or `_children` becomes the label or the child nodes in JSON, since JSON has one set of keys. Writers give a warning (J16).
- Fields with the same key on one node come back from JSON as one field, or as one field line per object for an array of objects (J7a, J10).
- The order between a node's fields and its child nodes is lost in JSON.
- How a field's items were written (inline, as children, or mixed) is lost in JSON.
- A node without a label whose only lines are fields `0::`, `1::` and so on, in order, is an array in JSON (J20), and an unquoted `[]` or `{}` is an empty array or object (J4, J7), even where they were written as text. They come back as index fields, `[]` and `{}`.

## 6. Diagnostics

Codes and severities work as in the Spec (C35 to C37). Positions are in the input: the JSON text when reading JSON, the Ord file when writing JSON. Lines and columns are counted as in the Spec (D9), and a JSON key's position is its opening `"`.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `J1.2` | warning | A `_label` isn't a string, or a `_children` isn't an array | the key |
| `J15` | warning | The root object has a `_label` | the key |
| `J16` | warning | An Ord field's key is `_label` or `_children` | the field's key |
| `J19.1` | warning | A `_json` isn't a kind J19 names (`object` or `value`), isn't its node's first field line, comes twice, or is on a node that has what its kind doesn't allow | the field's key, or the line |
