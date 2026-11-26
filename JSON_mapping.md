# Ord JSON mapping

Version 0.7.0-draft, 2026-11-22. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here, and rule IDs that start with `C`, `L`, `D` or `T` refer to the Spec and the [Test format](Test_format.md).

## 1. Scope

- **J17.** This spec maps JSON to the Core data model (Spec, section 1) and back. Its conventions exist so that JSON → Ord → JSON keeps what it can; section 5 lists what it can't.
- **J1a.** The mapping is syntactic. It doesn't read label syntax (Spec Part 2) or resolve references, so a reference is the text written, e.g. `client:: acme` gives `"client": "acme"`. It reads only two value forms, numbers (L22) and booleans (L27), which JSON has types for (J7, J9), and the field `_json` (J19).
- **J18.** A **JSON mapper** reads and writes JSON as this spec says, and passes the tests with the `from-json` and `to-json` options (T4).

## 2. System keys

- **J1.** Two JSON keys hold a node's Core label and child nodes, never user fields. They are named like the system fields `_label` and `_children` ([Semantics](Semantics.md) M11), though the mapping, being syntactic (J1a), doesn't read them as such. Every other key is an ordinary field, also one that starts with `_`, which Semantics reserves (M92) but this mapping doesn't check.
  - `_label` holds a node's label.
  - `_children` holds a node's child nodes.
  - A `_label` that isn't a string, or a `_children` that isn't an array, gives a warning and is kept as an ordinary field.
  - A key `_json` gives the warning `J1.3` and is left out, since in Ord that field says what a node is in JSON (J19).
- **J19.** The field `_json` ([Semantics](Semantics.md) M11) says what a node is in JSON where the other rules would read it as something else. It is written as a field line, the node's first, and its value is one of:
  - `array`: the node is a JSON array of its child nodes, each as J4 and J6 say. It has no label and no other fields; a label or another field is the warning `J19.1` and is left out.
  - `object`: the node is a JSON object (J3) even when it has no fields or child nodes, `{}`, or only a label, `{"_label": "x"}`.

  On the root, `_json:: array` makes the file a JSON array of its root lines (J15). Under a field, `_json:: object` is a line of the field's anonymous node (C13): `x::` with `_json:: object` under it is `"x": {}`. `_json` is never written as a JSON key, and one with any other value, or more than one, is the warning `J19.1` and is left out. Writers write `_json` only where it is needed: a file that has no arrays in arrays, empty objects or objects of a label alone maps without it, and nothing else in this spec reads label syntax or tricks of layout to tell JSON types apart.

## 3. Reading JSON

JSON is read by writing it as Ord, as these rules say, and reading the result (Spec, Part 1).

- **J8.** An object becomes a node. `_label` becomes its label, and an object without one a node with an empty label, `::`. Its other members become field lines, and `_children` child node lines. An object that would otherwise read as something else gets `_json:: object` as its first line (J19): one without members, `{}`, which would be `null`, and one with only `_label`, which would be a string (J4).
- **J9.** A string becomes the field's value, quoted (C26) when it is a number (L22) or a boolean (L27) as written, so that it stays text: `"6"` gives `hours:: "6"`. A number becomes its shortest decimal form, as JSON writes it, and a boolean `true` or `false`, so each reads back as its value form: `6`, `9.5` and `true` give `hours:: 6`, `9.5` and `true`. A number whose shortest form has an exponent, as `1e21`, becomes that text. `null` becomes an empty value. An object with only ordinary members becomes the field's field children (C13), and one with no members `_json:: object` under the field (J19). Any other object becomes one child node following J8.
- **J10.** An array of strings, numbers and booleans becomes the field's items, `key:: a; b`, quoted as C26 says. So does an array whose such items all come before its objects; the objects then follow J11, except that one object with only ordinary members becomes the field's field children (C13), as in J9. Any other array, one with `null`, a nested array, an item after an object, or text with a line break, becomes the field's node children, one per item. An array of one item or none is a list (C16), with a `;` after its items: `key:: a;`, `key:: ;`, and `key:: ;` with the object's fields under it for one object, or `_json:: object` when it has none.
- **J11.** Within an array that becomes node children (J10), `null` becomes `::`, a string, number or boolean a node line, an object follows J8, and an array J14.
- **J12.** Text is quoted according to C26.
- **J14.** A nested array becomes `::` with `_json:: array` as its first line and one child node per item, following J11 (J19). An empty one is `::` with `_json:: array` alone.
- **J15.** A root array becomes `_json:: array` as the file's first line, then its items as root lines, following J11 (J19). A scalar root is wrapped in a one-item root array. A `_label` on the root object gives a warning and is kept as an ordinary field.

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

```example from-json J8 J9 J10 J11 J12 J14
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
    node empty
    node "y"
    node empty
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
    node empty
      field "_json"
        "array"
      node "a"
      node "b"
    node empty
      field "_json"
        "array"
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
  "deep": [{ "n": { "b": 2 } }, { "a": 3 }]
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
    node empty
      field "a"
        "1"
      field "n"
        node
          field "b"
            "2"
    node empty
      field "a"
        "3"
  field "deep"
    node empty
      field "n"
        node
          field "b"
            "2"
    node empty
      field "a"
        "3"
```

```example from-json J15
["a", ["b"]]
.
root
  field "_json"
    "array"
  node "a"
  node empty
    field "_json"
      "array"
    node "b"
```

```example from-json J15
"x"
.
root
  field "_json"
    "array"
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
    node empty
      field "k"
        quoted "1"
    node empty
      field "k"
        "1"
```

## 4. Writing JSON

- **J2.** The root node becomes a JSON object, or, with `_json:: array`, an array (J19).
- **J3.** A node with children becomes an object:
  - `_label` is its label. It is omitted when the node is anonymous, or when its label is empty.
  - `_json` is not a key (J19).
  - Each field becomes a key.
  - `_children` is an array of its child nodes. It is omitted when there are none.
- **J4.** A node without children becomes its label as a string, or `null` when the label is empty and unquoted (`::`). A node with `_json` follows J19.
- **J5.** A field's JSON value is:
  - an array of its items, if it is a list of one item or none (C16): `a;` gives `["a"]` and `;` gives `[]`
  - `null`, if it has no items
  - the item itself, if it has one item
  - an array of its items, if it has several
- **J6.** A text item becomes a string. A node item follows J3 and J4.
- **J7.** An unquoted text item that is a number (L22) becomes a JSON number, and one that is a boolean (L27) `true` or `false`: `n:: 1.50; TRUE` gives `"n": [1.5, true]`. Every other value is a string, quoted text (`"3"`), other value forms such as dates, and labels included.
- **J7a.** Fields with the same key on one node become one JSON key. Their items are concatenated in file order, and the result is always an array: `tag:: a` and `tag:: b` give `"tag": ["a", "b"]`.
- **J16.** A field whose key is `_label` or `_children` is written as that JSON key, with a warning, since it reads back as the label or the child nodes (J1).

```example to-json J19 J5
rows::
→::
→→a:: 1
→→n:: 2
→::
→→_json:: array
→→x
→→y
→N
→→_json:: object
empty::
→_json:: object
.
{
  "rows": [
    { "a": 1, "n": 2 },
    ["x", "y"],
    { "_label": "N" }
  ],
  "empty": {}
}
```

```example to-json J2 J19
_json:: array
a
::
→_json:: array
→b
→::
→→_json:: array
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

```example to-json J7
n:: 1.50; -2; TRUE
s:: "3"
d:: 2026-10-15
v:: 007; 1e3
.
{
  "n": [1.5, -2, true],
  "s": "3",
  "d": "2026-10-15",
  "v": ["007", "1e3"]
}
```

## 5. Known gaps

- A number or boolean that becomes a node line (J11), as in `[null, 5]`, comes back as a string, since labels have no value forms. So does a number written with an exponent (J9).
- A number comes back in its shortest form (`1.50` as `1.5`), and with no more digits than a 64-bit floating-point number holds.
- A scalar JSON root comes back as a one-item array (J15).
- A `_label` on the root object has no form, since the root has no label line (J15).
- An Ord field whose key is `_label` or `_children` becomes the label or the child nodes in JSON, so it doesn't round-trip. Writers give a warning (J16).
- A JSON key `_json` is left out, with a warning (J1.3).
- Fields with the same key on one node come back from JSON as one field (J7a).
- The order between a node's fields and its child nodes is lost in JSON.
- How a field's items were written (inline, as children, or mixed) is lost in JSON.

## 6. Diagnostics

Codes and severities work as in the Spec (C35 to C37). Positions are in the input: the JSON text when reading JSON, the Ord file when writing JSON. Lines and columns are counted as in the Spec (D9), and a JSON key's position is its opening `"`.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `J1.2` | warning | A `_label` isn't a string, or a `_children` isn't an array | the key |
| `J15` | warning | The root object has a `_label` | the key |
| `J1.3` | warning | A JSON object has a key `_json` | the key |
| `J16` | warning | An Ord field's key is `_label` or `_children` | the field's key |
| `J19.1` | warning | A `_json` isn't `array` or `object` on a node's first field line, comes twice, or is on a node `_json:: array` with a label or other fields | the field's key, or the node's line |
