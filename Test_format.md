# Ord test format

Version 0.4.0-draft, 2026-10-23. A companion spec to [Ord specification](Ord_Specification.md), of the same version. The Spec's conventions (S1 to S6) and versioning (S10) apply here. Its rules start with `D` (the dump) and `T` (tests), and other rule IDs refer to the Spec.

Part A defines the **dump**, the text form of the data model that tests compare against. Part B says how tests are written in the specs and how a runner runs them.

## A. AST dump

The **dump** is a text form of the data model (Spec, section 1). Read tests compare it as a string, so implementations only need to print it. Two test options add to it: ranges (A3) and label syntax (A4). With both off, the dump is Core only.

### A1. Lines

- **D1.** One element per line. Every line ends with `\n`, the last one too. There are no blank lines and no trailing whitespace.
- **D2.** The first line is `root`. Every other line is indented two spaces more than its parent.
- **D3.** A node's children are written in file order (C12): `field` and `node` lines.
- **D4.** The children of a field are its items, in order (C13): a string for a text item and a `node` line for a node. A quoted text item is written `quoted` and the string.
- **D5.** A node is `node` followed by its label: a string, `quoted` and a string for a quoted label, `empty` for an unquoted empty label, or nothing for an anonymous node.
- **D6.** A field is `field` and its key as a string.
- **D7.** Diagnostics follow the tree, one per line, sorted by line, column and code: `!`, the code and the position (C37). The severity isn't written, since the code sets it.

### A2. Strings

- **D8.** Text is written as a `"` quoted string, escaped as in C47, so equal text always prints the same.

### A3. Ranges

- **D9.** With the ranges option, each element that comes from the source ends with `@start-end`. A position is `line:column`. Lines and columns start at 1, a column counts characters, and a tab counts as 1 (C9). The end is the position just after the element's last character.
- **D10.** The range of a node or field is its line after the indent, up to the line ending. A block's range runs from its opening line through its closing line, or through the last line of the file. A text item's range is its text, quotes included, without the `;` and whitespace touching it. A field read from a label (D12) covers the form as written, and its items cover their value text, or the marker for `state`. The root, anonymous nodes and `label` lines have no range.

### A4. Label syntax

- **D11.** With the labels option, Part 2 is applied and adds lines. Core lines are unchanged. Sigil bindings and custom markers are part of the option, by default none (L11, L15).
- **D12.** Under a node that Part 2 reads (L1), the first children are:
  - `label` and the node's label (L19) as a string, when it differs from the Core label
  - the fields read from the label, in the order written, as `field` lines marked `inline` or `embedded` (D14), with their items as in L5 and L6

  The node's other children follow (D3).
- **D13.** A text item of a field line or an `inline` field that is exactly one value trigger (L20) is followed by the name of the field it stands for, e.g. `"https://example.com" url`. One that is exactly a value form (L21) is followed by the form's name, e.g. `"2026-10-15" date`. Items of `embedded` fields have no name, since the field already says what they are. Any other text item gets the fields read from it as `embedded` field children, as in D12.
- **D14.** A field read from a form that L19 removes from the label, an inline field or a `state`, `kind`, `tag` or `id` trigger, is marked `inline`: a writer moves it (L4), and it means the same as a field line. A field read from a form that stays in the label, a link, URL, formula or sigil, is marked `embedded`, since where it is written is part of what the text says. A trigger's item is the value of the inline field it reads as (L6), so a link keeps its brackets, as in `"[[Ruukki AB]]"`, and a sigil gives the name, as in `"anna"`.

### A5. Grammar

Which elements may appear where is set by D3, D4 and D12 to D14, not by the grammar.

```abnf
dump      = %s"root" LF *elem-line *diag-line
elem-line = indent elem [SP range] LF
indent    = 1*(2SP)                  ; depth, see D2
elem      = node / field / item / label
node      = %s"node" [SP (text / %s"empty")]
field     = %s"field" SP string [SP (%s"inline" / %s"embedded")]
item      = text [SP name]
text      = [%s"quoted" SP] string
label     = %s"label" SP string
diag-line = "!" SP code SP position LF
code      = 1*(ALPHA / DIGIT / ".")
range     = "@" position "-" position
position  = 1*DIGIT ":" 1*DIGIT
name      = 1*(ALPHA / DIGIT / "_" / "-" / %x80-10FFFF)
string    = DQUOTE *(unescaped / escape) DQUOTE
escape    = "\" (DQUOTE / "\" / %x62 / %x66 / %x6E / %x72 / %x74
            / %x75 4(DIGIT / %x61-66))
unescaped = %x20-21 / %x23-5B / %x5D-10FFFF  ; minus what C47 escapes
```

### A6. Examples
These examples are tests (part B).

```example C13 C16 C29
client:: Ruukki; Acme
tags:: a;;b;
contact::
→primary:: anna
matrix::
→;
→→x
::
```csv
a,b
.
root
  field "client"
    "Ruukki"
    "Acme"
  field "tags"
    "a"
    "b"
  field "contact"
    node
      field "primary"
        "anna"
  field "matrix"
    node ";"
      node "x"
  node empty
  node "```csv\na,b"
! C29 9:1
```

```example ranges labels L10 L11 L19 L20
- [x] Send draft to [[Anna]] #task [due:: 2026-10-10]
site:: https://example.com
.
root
  node "- [x] Send draft to [[Anna]] #task [due:: 2026-10-10]" @1:1-1:54
    label "Send draft to [[Anna]]"
    field "state" inline @1:3-1:6
      "done" @1:4-1:5
    field "link" embedded @1:21-1:29
      "[[Anna]]" @1:21-1:29
    field "kind" inline @1:30-1:35
      "task" @1:31-1:35
    field "due" inline @1:36-1:54
      "2026-10-10" date @1:43-1:53
  field "site" @2:1-2:27
    "https://example.com" url @2:8-2:27
```

## B. Tests

The examples in the specs are tests. A script extracts them and runs them against an implementation.

- **T1.** A **test** is a block (C29) whose info string is `example`, followed by any options and the IDs of the rules it shows, separated by spaces. A test whose input holds a closed ```` ``` ```` block uses a longer fence, such as ````` ```` `````.
- **T2.** The block's lines before the first line that is exactly `.` are the **input**, and the lines after it are the **expected output**. Every line of both ends with `\n`, the last one too.
- **T3.** In the input and in expected output other than a dump, `→` stands for a tab and `␣` for a space, and `⟨U+XXXX⟩` for the character with that code point, such as `⟨U+00A0⟩` for NBSP or `⟨U+000D⟩` for a carriage return. `␣` and `⟨U+XXXX⟩` are used where a character would otherwise be invisible, such as a space at the end of a line. Dumps use neither.
- **T4.** Options:
  - With no option, the test reads the input and expects its dump (part A).
  - `ranges` and `labels` turn on those dump options (A3, A4).
  - `fmt` expects the input after `fmt` (C41), as Ord. With `labels`, the writer also follows Part 2 (L4).
  - `from-json` and `from-csv` read the input as JSON or CSV, as the companion specs say, and expect its dump.
  - `to-json` and `to-csv` expect the input written as JSON or CSV.
- **T5.** Every Ord input is also checked for stable writing (C33): `fmt` applied to the result of `fmt` changes nothing.
- **T6.** Tests are numbered in the order they appear. The numbers change as tests are added; the rule IDs don't.
- **T7.** A runner gives each input to the implementation on standard input and compares standard output with the expected output, byte for byte. JSON output is compared as JSON values instead, so the order of an object's members doesn't matter. The exit code isn't tested.
