# Ord JSON mapping

Version 0.4.0-draft, 2026-10-23. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D` or `T` refer to the Spec and the [Test format](Test_format.md).

## 1. Scope

- **J17.** This spec maps JSON to the Core data model (Spec, section 1) and back. Its conventions exist so that JSON → Ord → JSON keeps what it can; section 5 lists what it can't.
- **J1a.** The mapping is syntactic. It doesn't read label syntax (Spec Part 2) or resolve references, so a reference is the text written, e.g. `client:: ruukki` gives `"client": "ruukki"`.
- **J18.** A **JSON mapper** reads and writes JSON as this spec says, and passes the tests with the `from-json` and `to-json` options (T4).

## 2. Reserved keys

- **J1.** JSON keys starting with `$ord:` are reserved for the mapping. In JSON input they are always read as reserved, never as user keys. An unknown `$ord:` key gives a warning and is kept as an ordinary field.
  - `$ord:label` holds a node's label.
  - `$ord:children` holds a node's child nodes.
  - A `$ord:label` that isn't a string, or a `$ord:children` that isn't an array, gives a warning and is kept as an ordinary field.

## 3. Reading JSON

JSON is read by writing it as Ord, as these rules say, and reading the result (Spec, Part 1).

- **J8.** An object becomes a node. `$ord:label` becomes its label, other members become field lines, and `$ord:children` become child node lines. An object without `$ord:label` becomes a node with an empty label, `""`, with its members as children.
- **J9.** A string becomes the field's value. `null` becomes an empty value. An object with only ordinary members becomes the field's field children (C13). Any other object becomes one child node following J8.
- **J10.** An array becomes the field's node children, one per item.
- **J11.** Within an array, `null` becomes `::`, a string becomes a node line, and an object follows J8.
- **J12.** Text is quoted according to C26.
- **J14.** A nested array becomes a node `""` with one child per item, following J11. An empty nested array becomes `::`, like `null`.
- **J15.** A root array's items become root lines, following J11. A scalar root is wrapped in a one-item root array. A `$ord:label` on the root object gives a warning and is kept as an ordinary field.

```example from-json J8 J9 J10
{
  "client": "Ruukki AB",
  "contacts": ["Anna", "Erik"],
  "lines": [
    { "$ord:label": "Workshop", "hours": "6" },
    { "$ord:label": "Spec review", "hours": "9.5" }
  ]
}
.
root
  field "client"
    "Ruukki AB"
  field "contacts"
    node "Anna"
    node "Erik"
  field "lines"
    node "Workshop"
      field "hours"
        "6"
    node "Spec review"
      field "hours"
        "9.5"
```

```example from-json J8 J9 J10 J11 J12 J14
{
  "a": null,
  "b": "",
  "c": ["x"],
  "d": [],
  "e": [null, "y", { "z": "1" }],
  "f": { "primary": "anna" },
  "g": { "$ord:label": "Ruukki", "owner": "anna" },
  "h": [["a", "b"], []],
  "$ord:children": ["Call Anna", { "$ord:label": "Book room", "due": "2026-10-20" }]
}
.
root
  field "a"
  field "b"
    quoted ""
  field "c"
    node "x"
  field "d"
  field "e"
    node empty
    node "y"
    node quoted ""
      field "z"
        "1"
  field "f"
    node
      field "primary"
        "anna"
  field "g"
    node "Ruukki"
      field "owner"
        "anna"
  field "h"
    node quoted ""
      node "a"
      node "b"
    node empty
  node "Call Anna"
  node "Book room"
    field "due"
      "2026-10-20"
```

```example from-json J15
["a", ["b"]]
.
root
  node "a"
  node quoted ""
    node "b"
```

```example from-json J15
"x"
.
root
  node "x"
```

```example from-json J1 J15
{"$ord:foo": "x", "$ord:label": "y", "$ord:children": "z"}
.
root
  field "$ord:foo"
    "x"
  field "$ord:label"
    "y"
  field "$ord:children"
    "z"
! J1.1 1:2
! J15 1:19
! J1.2 1:38
```

## 4. Writing JSON

- **J2.** The root node becomes a JSON object.
- **J3.** A node with children becomes an object:
  - `$ord:label` is its label. It is omitted when the node is anonymous, or when its label is empty.
  - Each field becomes a key.
  - `$ord:children` is an array of its child nodes. It is omitted when there are none.
- **J4.** A node without children becomes its label as a string, or `null` when the label is empty and unquoted (`::`).
- **J5.** A field's JSON value is:
  - `null`, if it has no items
  - the item itself, if it has one item
  - an array of its items, if it has several
- **J6.** A text item becomes a string. A node item follows J3 and J4.
- **J7.** All values are strings.
- **J7a.** Fields with the same key on one node become one JSON key. Their items are concatenated in file order, and the result is always an array: `tag:: a` and `tag:: b` give `"tag": ["a", "b"]`.
- **J16.** A field whose key starts with `$ord:` is written as that JSON key, with a warning, since it reads back as reserved (J1).

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
client:: [[Ruukki AB]]
Ruukki
→Call Anna
→;
.
{
  "a": null,
  "b": "anna",
  "c": "anna",
  "d": ["anna", "oscar"],
  "e": ["a", "b"],
  "f": null,
  "h": "",
  " hello": "world",
  "i": ["Anna", "Gustav"],
  "j": { "primary": "anna" },
  "k": ["x", "Anna", { "primary": "anna" }],
  "matrix": { "$ord:children": ["a", "b"] },
  "tag": ["a", "b"],
  "client": "[[Ruukki AB]]",
  "$ord:children": [{ "$ord:label": "Ruukki", "$ord:children": ["Call Anna", ";"] }]
}
```

## 5. Known gaps

- Numbers and booleans become strings.
- An empty array comes back as `null`, and a one-item array as its item (J10, J5).
- A nested array, and a root array, come back as objects with `$ord:children` (J14, J15).
- A scalar JSON root comes back as an object (J15).
- A `$ord:label` on the root object has no form, since the root has no label line (J15).
- An object with only `$ord:label`, such as `{"$ord:label": "x"}`, is written as a leaf node and comes back as the string `"x"`.
- An empty object `{}` comes back as `null`.
- An Ord field whose key starts with `$ord:` becomes a reserved key in JSON, so it doesn't round-trip. Writers give a warning (J16).
- Fields with the same key on one node come back from JSON as one field (J7a).
- The order between a node's fields and its child nodes is lost in JSON.
- How a field's items were written (inline, as children, or mixed) is lost in JSON.

## 6. Diagnostics

Codes and severities work as in the Spec (C35 to C37). Positions are in the input: the JSON text when reading JSON, the Ord file when writing JSON. Lines and columns are counted as in the Spec (D9), and a JSON key's position is its opening `"`.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `J1.1` | warning | A JSON key starts with `$ord:` but isn't a reserved key | the key |
| `J1.2` | warning | A `$ord:label` isn't a string, or a `$ord:children` isn't an array | the key |
| `J15` | warning | The root object has a `$ord:label` | the key |
| `J16` | warning | An Ord field's key starts with `$ord:` | the field's key |
