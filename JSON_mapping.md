# Ord JSON mapping

Version 0.6.0-draft, 2026-11-03. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D` or `T` refer to the Spec and the [Test format](Test_format.md).

## 1. Scope

- **J17.** This spec maps JSON to the Core data model (Spec, section 1) and back. Its conventions exist so that JSON → Ord → JSON keeps what it can; section 5 lists what it can't.
- **J1a.** The mapping is syntactic. It doesn't read label syntax (Spec Part 2) or resolve references, so a reference is the text written, e.g. `client:: acme` gives `"client": "acme"`.
- **J18.** A **JSON mapper** reads and writes JSON as this spec says, and passes the tests with the `from-json` and `to-json` options (T4).

## 2. System keys

- **J1.** Two JSON keys hold a node's Core label and child nodes, never user fields. They are named like the system fields `_label` and `_children` ([Semantics](Semantics.md) M11), though the mapping, being syntactic (J1a), doesn't read them as such. Every other key is an ordinary field, also one that starts with `_`.
  - `_label` holds a node's label.
  - `_children` holds a node's child nodes.
  - A `_label` that isn't a string, or a `_children` that isn't an array, gives a warning and is kept as an ordinary field.

## 3. Reading JSON

JSON is read by writing it as Ord, as these rules say, and reading the result (Spec, Part 1).

- **J8.** An object becomes a node. `_label` becomes its label, other members become field lines, and `_children` become child node lines. An object without `_label` becomes a node line that holds its first member that can be written as an inline field (L5) as one, `[key:: items]`, with its other members as children: a member whose key is names separated by single spaces and whose value is a string, number or boolean, or an array of them, with each item written bare in the inline field, without `;`, `[`, `]`, backticks, quotes or line breaks (C26). Without such a member it becomes a node with an empty label, `""`, with its members as children. An array of one item is written with its `;`, `[key:: a;]` (C16). Writing JSON reads such a line back as the object (J3a).
- **J9.** A string becomes the field's value. `null` becomes an empty value. An object with only ordinary members becomes the field's field children (C13). Any other object becomes one child node following J8.
- **J10.** An array of strings, numbers and booleans becomes the field's items, `key:: a; b`, quoted as C26 says. So does an array whose such items all come before its objects; the objects then follow J11, except that one object with only ordinary members becomes the field's field children (C13), as in J9. Any other array, one with `null`, a nested array, an item after an object, or text with a line break, becomes the field's node children, one per item. An array of one item or none is a list (C16), with a `;` after its items: `key:: a;`, `key:: ;`, and `key:: ;` with the object's fields under it for one object.
- **J11.** Within an array that becomes node children (J10), `null` becomes `::`, a string, number or boolean a node line, and an object follows J8.
- **J12.** Text is quoted according to C26.
- **J14.** A nested array becomes a node `""` with one child per item, following J11. An empty nested array becomes `::`, like `null`.
- **J15.** A root array's items become root lines, following J11. A scalar root is wrapped in a one-item root array. A `_label` on the root object gives a warning and is kept as an ordinary field.

```example from-json J8 J9 J10
{
  "client": "Acme Corp",
  "contacts": ["Anna", "Erik"],
  "lines": [
    { "_label": "Workshop", "hours": "6" },
    { "_label": "Spec review", "hours": "9.5" }
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

```example from-json J8 J9 J10 J11 J12 J14
{
  "a": null,
  "b": "",
  "c": ["x"],
  "d": [],
  "e": [null, "y", { "z": "1" }],
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
    node empty
    node "y"
    node "[z:: 1]"
  field "f"
    node
      field "primary"
        "anna"
  field "g"
    node "Acme"
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

```example from-json J8 J10
{
  "tags": ["a", "b; c"],
  "one": ["x", { "p": "1" }],
  "rows": ["x", { "a": "1", "n": { "b": "2" } }, { "a": "3" }],
  "deep": [{ "n": { "b": "2" } }, { "a": "3" }]
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
    node "[a:: 1]"
      field "n"
        node
          field "b"
            "2"
    node "[a:: 3]"
  field "deep"
    node quoted ""
      field "n"
        node
          field "b"
            "2"
    node "[a:: 3]"
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

## 4. Writing JSON

- **J2.** The root node becomes a JSON object.
- **J3.** A node with children becomes an object:
  - `_label` is its label. It is omitted when the node is anonymous, or when its label is empty.
  - Each field becomes a key.
  - `_children` is an array of its child nodes. It is omitted when there are none.
- **J3a.** A node whose unquoted label holds only inline fields (Spec L5), as `[a:: 1] [b:: x;]`, and nothing else, becomes an object of them, as J8 writes an object without `_label`: each inline field is a key, read as a field is (J5), and the node's fields and child nodes follow J3, without `_label`. This is the one place writing JSON reads label syntax (Spec Part 2).
- **J4.** A node without children becomes its label as a string, or `null` when the label is empty and unquoted (`::`), unless J3a applies.
- **J5.** A field's JSON value is:
  - an array of its items, if it is a list of one item or none (C16): `a;` gives `["a"]` and `;` gives `[]`
  - `null`, if it has no items
  - the item itself, if it has one item
  - an array of its items, if it has several
- **J6.** A text item becomes a string. A node item follows J3 and J4.
- **J7.** All values are strings.
- **J7a.** Fields with the same key on one node become one JSON key. Their items are concatenated in file order, and the result is always an array: `tag:: a` and `tag:: b` give `"tag": ["a", "b"]`.
- **J16.** A field whose key is `_label` or `_children` is written as that JSON key, with a warning, since it reads back as the label or the child nodes (J1).

```example to-json J3a J5
rows::
→[a:: 1] [b:: x;]
→→n:: 2
→[a:: 3]
→plain
.
{
  "rows": [
    { "a": "1", "b": ["x"], "n": "2" },
    { "a": "3" },
    "plain"
  ]
}
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

## 5. Known gaps

- Numbers and booleans become strings.
- A nested array, and a root array, come back as objects with `_children` (J14, J15).
- A scalar JSON root comes back as an object (J15).
- A `_label` on the root object has no form, since the root has no label line (J15).
- An object with only `_label`, such as `{"_label": "x"}`, is written as a leaf node and comes back as the string `"x"`.
- An empty object `{}` comes back as `null`.
- An Ord field whose key is `_label` or `_children` becomes the label or the child nodes in JSON, so it doesn't round-trip. Writers give a warning (J16).
- Fields with the same key on one node come back from JSON as one field (J7a).
- The order between a node's fields and its child nodes is lost in JSON.
- How a field's items were written (inline, as children, or mixed) is lost in JSON.

## 6. Diagnostics

Codes and severities work as in the Spec (C35 to C37). Positions are in the input: the JSON text when reading JSON, the Ord file when writing JSON. Lines and columns are counted as in the Spec (D9), and a JSON key's position is its opening `"`.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `J1.2` | warning | A `_label` isn't a string, or a `_children` isn't an array | the key |
| `J15` | warning | The root object has a `_label` | the key |
| `J16` | warning | An Ord field's key is `_label` or `_children` | the field's key |
