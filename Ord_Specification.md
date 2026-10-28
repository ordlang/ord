# Ord specification

Version 0.5.0-draft, 2026-10-27.

## Introduction

Ord is a plain-text outline format. Each line is a node or a field, indentation makes the tree, and fields make the outline structured data. This spec says how text is read into a tree, how a tree is written back as text, and how the shortform for predefined fields is read out of labels.

The spec has three parts:

- **Part 1, Core Ord.** The syntax and the data model: everything a parser and a writer need.
- **Part 2, Label syntax.** Inline fields and triggers, the shortform for predefined fields such as `+type`, `#tag`, `^id` and `[x]`, read out of labels and items, and value forms such as numbers and dates. Core doesn't depend on it.
- **Part 3, Files.** Which files are Ord: workspaces, `.txt` and `.ord`, and their types.

Companion documents build on it:

- **[Test format](Test_format.md).** The AST dump that tests compare against, and how the examples in the specs are written, so a script can extract and run them.
- **[Semantics](Semantics.md).** What fields mean across a workspace (types, references, states, single-valued fields), formulas and queries.
- **[JSON mapping](JSON_mapping.md)** and **[CSV mapping](CSV_mapping.md).** Mappings to other formats, built on the data model.
- **[Grammar](Grammar.md).** ABNF for the pieces a line is made of, and one way to read a file in linear time (non-normative).
- **[Changelog](CHANGELOG.md).** What changed in each version.

## Design goals

These goals are informative. They explain the rules and decide new ones.

- **Plain text first.** A file is readable and editable in any editor. Tools add power but are never needed to read it.
- **Every file reads.** There are no syntax errors. Every file is read into a tree, and anything doubtful is a diagnostic (C35).
- **Punctuation decides.** Whether a line is a field or a node, and where its key, value and items are, is decided by `::`, `;`, quotes and indentation, never by words or letter case.
- **Lines read on their own.** A line's key, value and inline items depend only on that line, except inside a block. Nothing in Core depends on other files or an index. Part 2 takes the few facts it needs from types, sigil bindings and status markers, as options.
- **Quoting turns syntax off.** Any text can be written literally by quoting it, with `"` or `` ` ``.
- **Writing is stable.** Writing twice changes nothing (C33), and an edit leaves the lines it doesn't touch as they were (C41).
- **Light syntax.** Single symbols and rules that let prose stay prose, instead of heavy delimiters.

## Conventions

- **S1.** Rules are plain statements, and every rule is required unless its section is marked non-normative. "May" says what is allowed: a form a file can have, or a choice an implementation has. Sentences that start with "So" spell out what the rules already imply.
- **S2.** Every rule has an ID: a letter for where it belongs (`C` Core, `L` label syntax, `D` dump, `T` tests, `W` writing conventions, `F` files, `S` this front matter) and a number. Companion specs use their own letters, such as `J`. IDs are given in the order rules are added, so they don't follow the order of the text. An ID never changes and is never reused. A removed rule is deleted, and its ID is listed as retired in the changelog.
- **S3.** A character is a Unicode code point. Whitespace means space or tab, and a no-break space is read as a space (C3). A letter is a Unicode letter (general category L), and a digit is 0 to 9. Positions count characters (D9).
- **S4.** Code spans show text exactly. In examples, `→` stands for a tab and `␣` for a space where one would be invisible (T3).
- **S5.** Grammars use ABNF (RFC 5234), where a string in double quotes matches any letter case and `%s"…"` matches it exactly (RFC 7405). Where a grammar and the prose differ, the prose wins.
- **S6.** Tests ([Test format](Test_format.md), part B) are normative, like rules. A test that disagrees with a rule is a bug in the spec, fixed by changing one of them. Other examples are informative.
- **S10.** The spec's version is `major.minor.patch`. From 1.0.0, a change that can make a file read or write differently raises the major version, a change that only adds, such as a diagnostic or a test for what the rules already say, raises the minor version, and a change to wording raises the patch. Before 1.0.0, any change can make a file read differently and raises the minor version, and wording raises the patch. The specs in this set (this one and its companions) share one version, and a version can carry a pre-release suffix such as `-draft`. Each version has an entry in the [changelog](CHANGELOG.md) that names the rules it changed. Changes since the last version are listed there as unreleased until the next version.

## Conformance

- **S7.** There are three conformance classes:
  - A **reader** reads any file into the data model as Part 1 says, with source ranges ([Test format](Test_format.md) A3), and reports every diagnostic in section 11.
  - A **writer** is a reader that also writes, edits and formats files as section 10 says.
  - A **label reader** is a reader that also applies Part 2 and reports every diagnostic in section 20.
- **S8.** An implementation conforms to a class, in a version of the spec, when it passes that class's tests in that version and follows its rules where no test covers them. Reader tests have neither the `fmt` nor the `labels` option. Writer tests have `fmt`, and every input is also checked for stable writing (T5). Label reader tests have `labels`.
- **S9.** Companion specs define their own classes, such as a JSON mapper and a CSV mapper.

---

# Part 1: Core Ord

## 1. Data model

Reading turns a file into this model, and writing turns the model back into text. The rest of Part 1 says how.

```
Node      label     Text, empty, or absent (anonymous node)
          children  ordered list of Field and Node

Field     key       text
          items     ordered list of Item

Item      Text or Node

Text      value     the text, with any quotes removed (C21)
          quoted    true or false
```

A label or item records whether it was quoted, since Part 2 reads only unquoted text (L1) and writers treat its trailing whitespace differently (C34).

Every model read from text follows C38 to C40. A writer may reject a model that breaks them, since no text expresses it.

- **C38.** A field's items are its **inline items**, its text items, then its nodes, then at most one anonymous node (C13).
- **C40.** A node is anonymous only as the root (C14) or as the node that holds a field's field children (C13), which holds only fields, at least one. Every other node has a label, which may be empty.

## 2. File

- **C49.** A file is UTF-8. Bytes that aren't valid UTF-8 are read as U+FFFD, the replacement character, as the WHATWG UTF-8 decoder reads them, with a warning. An edit leaves the bytes of lines it doesn't touch as they were (C41).
- **C1.** Lines end at `\n` or `\r\n`. The last line may have no ending.
- **C2.** A BOM at the start of the file is ignored.
- **C3.** Only space and tab are whitespace. A **no-break space** (NBSP, U+00A0) is read as a space wherever it appears, so it works as one in the indent, around `::` and `;`, and between forms, and the model has a space in its place. Phone keyboards type it for a space at the end of a line. Any other whitespace character (U+202F, U+3000, vertical tab, form feed, a lone `\r`) is content. A warning is given when one appears at the start of a line or next to `::`. A writer writes a space for an NBSP that was read, so a literal one is kept only as the escape `\u00a0` in a `"` string (C22).

```example C1 C2 C3 C35 C37
⟨U+FEFF⟩a:: b⟨U+000D⟩
⟨U+3000⟩c
d⟨U+3000⟩:: e
f:: lone⟨U+000D⟩cr
.
root
  field "a"
    "b"
  node "\u3000c"
  field "d\u3000"
    "e"
  field "f"
    "lone\rcr"
! C3.1 2:1
! C3.2 3:2
```

A no-break space is read as a space: in the indent, around `::` and `;`, and in text.

```example C3 C4 C5 C15
Tasks
⟨U+00A0⟩⟨U+00A0⟩Call⟨U+00A0⟩Anna
a⟨U+00A0⟩::⟨U+00A0⟩x⟨U+00A0⟩;⟨U+00A0⟩y
.
root
  node "Tasks"
    node "Call Anna"
  field "a"
    "x"
    "y"
```

```example labels C3 L3 L4
Open slot #hej⟨U+00A0⟩+hej⟨U+00A0⟩^hej
.
root
  node "Open slot #hej +hej ^hej"
    label "Open slot #hej"
    field "tag" embedded
      "hej"
    field "type" inline
      "hej"
    field "id" inline
      "hej"
```

## 3. Reading order

- **C50.** A reader reads a file in this order. Each step uses only what the steps before it found.
  1. Decode the file (C49, C2) and split it into lines (C1).
  2. Find the blocks, from the top (C29). A block's lines, opening and closing lines included, are content and skip steps 3 and 4.
  3. On every other line, take the indent (C4) and find the spans (C25) and brackets (C51). Then find the separator (C5), which splits the line into key and value (C6 to C8).
  4. Unquote the key and a node's label (C21). Split a field's value into items (C15, C16, C19), and unquote each item.
  5. Build the tree from the indents (C9, C10), with each block placed by its opening line (C31), and give each field its items (C13).

  Part 2 reads labels and items after these steps and changes nothing they found.

So a quote can hide `::` and `;`, and a bracket can hide `::` (both come before the separator), but neither hides a block's delimiters (blocks come first).

## 4. Blocks

- **C29.** A line whose text after the indent starts with three or more backticks, its **fence**, and has no backtick after them, opens a **block**. The block ends at the next line whose text after the indent is at least as many backticks as the fence, optionally followed by spaces and tabs. Without such a line, it runs to the end of the file, with a warning. So a block with a longer fence can hold a shorter one, as in Markdown.
- **C30.** Lines inside the block, blank ones included, are content. They don't affect the tree and have no key or value. The opening line has no separator.
- **C31.** The block is a node, placed in the tree by its opening line (C9). Its label is the block's lines, from the opening line through the closing line, joined with `\n`. Up to as many leading spaces and tabs as the opening line's indent are removed from each line. So the label keeps the fences and any info string, e.g. ```` ```csv ````. The content is raw: no escapes apply.

`````example C29 C30 C31 C50
Notes
→```js
→␣␣a:: b
→
→```
"x
````md
```
it's raw \n
```
````
'''
.
root
  node "Notes"
    node "```js\n  a:: b\n\n```"
  node "\"x"
  node "````md\n```\nit's raw \\n\n```\n````"
  node "'''"
`````

## 5. Quoting and spans

- **C21.** The **quote characters** are `"` and `` ` ``. Quotes mark text that is taken literally. A **quoted string** is a quote character, content, and the same quote character. In a `"` string, the content holds `"` only escaped (C22). In a `` ` `` string, the content can't hold a backtick. A key, label, value or item is **quoted** when it is exactly one quoted string. The quotes are removed and the content is taken literally, whitespace included. Anything else is literal: `"a" and "b"` is unchanged, and so is `'a'`, since `'` is not a quote character.
- **C22.** Inside `"` quotes, JSON string escapes apply: `\"`, `\\`, `\/`, `\b`, `\f`, `\n`, `\r`, `\t`, `\uXXXX`. A backslash followed by anything else is not an escape: it is kept, with the next character, and gives a warning. `\u` without four hex digits after it is not an escape either. So `"\q"` reads as `\q`. Inside backticks there are no escapes, and outside quoted strings a backslash is ordinary text in Core.
- **C23.** Inside quotes, `::` and `;` are content.
- **C24.** Spaces and tabs after a closing quote at the end of a line are ignored. This is the only exception to C8.
- **C25.** A **span** is a quoted string that hides `::` and `;`: they are content inside it (C5, C15). A span keeps its quotes in the text.
  - A backtick opens a span anywhere. The span ends at the next backtick; a backtick without one is ordinary text.
  - A `"` opens a span only where a key, label, value or item starts: at the start of the text after the indent, right after the separator (C5), or right after a `;` that separates items and the spaces and tabs touching it (C15). The span ends at the next `"` that isn't escaped (C22). Anywhere else, and without such an end, `"` is ordinary text.

  So `say "hi" now` and `5" screen` are unchanged, `'` never quotes, as in `Anna's`, and `` `a:: b` is a field `` is a node.
- **C51.** A **bracket** is a `[` and its matching `]` on the same line, outside spans. Brackets nest, and a backtick span inside one (C25) hides `[` and `]`; `"` is ordinary text there. A `::` inside a bracket is content, so it doesn't separate (C5). A `[` without a matching `]` is ordinary text and hides nothing. A bracket doesn't hide `;`. So `Send draft [due:: 2026-10-10]` and `See [[a::b]]` are nodes, and inline fields (L5) are written in brackets.

```example C5 C51
Send draft [due:: 2026-10-10]
See [[a::b]] now
a [b [c:: d] e:: f] g:: h
a [b:: c
[x] Call:: Anna
.
root
  node "Send draft [due:: 2026-10-10]"
  node "See [[a::b]] now"
  field "a [b [c:: d] e:: f] g"
    "h"
  field "a [b"
    "c"
  field "[x] Call"
    "Anna"
```

```example C21 C22 C25
a:: ""
b:: " hello"
c:: "x; y"
d:: "line 1\nline 2"
e:: "\"Hi\""
f:: 'x; y'
g:: `C:\temp`
h:: say "hi" now
.
root
  field "a"
    quoted ""
  field "b"
    quoted " hello"
  field "c"
    quoted "x; y"
  field "d"
    quoted "line 1\nline 2"
  field "e"
    quoted "\"Hi\""
  field "f"
    "'x"
    "y'"
  field "g"
    quoted "C:\\temp"
  field "h"
    "say \"hi\" now"
```

```example C22 C23 C24 C25
a:: "x"␣␣
b:: "\q"
c:: x "a;b" y; z
d:: "a:: b; c"
e:: x `a;b` y; z
.
root
  field "a"
    quoted "x"
  field "b"
    quoted "\\q"
  field "c"
    "x \"a"
    "b\" y"
    "z"
  field "d"
    quoted "a:: b; c"
  field "e"
    "x `a;b` y"
    "z"
! C22 2:6
```

## 6. Line

- **C4.** The **indent** is the leading spaces and tabs. A line with nothing after its indent is **blank**.
- **C5.** The **separator** is the first `::` after the indent that is not inside a span (C25) or a bracket (C51), together with any spaces and tabs touching it. What follows it doesn't matter, so `due::2026-10-10` is a field line, just like `due:: 2026-10-10`: a line that looks like a field is a field.
  - So a line that is exactly one quoted string is always a node, even when it contains `::`.
- **C6.** With a separator, the **key** is the text before it and the **value** is the text after it.
- **C7.** Without a separator, the key is empty and the value is all text after the indent.
- **C8.** Nothing else is trimmed. Trailing whitespace is part of the value. As a consequence, no value starts with whitespace.

```example C5 C6 C7 C8
a :: b
a:: b::c
a::b
Send draft due::2026-10-10
a::;
hello
:: hello
a::␣␣␣
.
root
  field "a"
    "b"
  field "a"
    "b::c"
  field "a"
    "b"
  field "Send draft due"
    "2026-10-10"
  field "a"
  node "hello"
  node "hello"
  field "a"
```

```example C5
The quick brown fox jumps over the lazy dog and keeps on running:: x
Note:: the key's length doesn't matter
.
root
  field "The quick brown fox jumps over the lazy dog and keeps on running"
    "x"
  field "Note"
    "the key's length doesn't matter"
```

```example C5 C21 C25
"a::b":: c
"a::b"
`a:: b` is a field
Anna's key:: Erik's value
.
root
  field "a::b"
    "c"
  node quoted "a::b"
  node "`a:: b` is a field"
  field "Anna's key"
    "Erik's value"
```

## 7. Tree

- **C9.** The parent of a line is the nearest preceding non-blank line with a smaller indent. If there is none, the line is a root. Indent is counted in characters; a tab and a space each count as 1.
- **C10.** Blank lines don't affect the tree.

```example C4 C9 C10 C12 C14
Project
→Task one
→→due:: 2026-10-20
␣␣
→Task two
␣␣Mixed indent child
Second root
.
root
  node "Project"
    node "Task one"
      field "due"
        "2026-10-20"
    node "Task two"
      node "Mixed indent child"
  node "Second root"
```

## 8. Fields and nodes

- **C11.** A line with a non-empty key is a **field**. A quoted key counts as non-empty even when its content is empty, so `"":: x` is a field. A line with an empty key is a **node**, and its value is the node's **label**.
- **C12.** A node's children are its fields and child nodes, in file order.
- **C13.** A field's **items** are, in order: the items of its own value (section 9), its node children, and, if it has field children, one **anonymous node** that holds them. An anonymous node has no label.
- **C14.** The file is an anonymous root node. Its root lines are the root node's children.

```example C11 C13 C16 C19 C38 C40
"":: x
Note; with semicolon
k:: x
→Anna
→primary:: anna
→Erik
m:: a;;
;
→a
→b:: c
.
root
  field ""
    "x"
  node "Note; with semicolon"
  field "k"
    "x"
    node "Anna"
    node "Erik"
    node
      field "primary"
        "anna"
  field "m"
    "a"
  node ";"
    node "a"
    field "b"
      "c"
```

## 9. Items in a field's value

- **C15.** In a field's own value, a `;` that is not inside a span (C25) separates items. Spaces and tabs touching a `;` belong to it. In a **formula item**, one that starts with `=`, a `;` inside a **group** is content too: a `(` outside spans opens a group that ends at its matching `)`, counting nested parentheses and skipping spans and **strings**: a `"` or `'` inside a group starts a string that ends at the next same character not escaped by `\`, as in JSONata. A `(` without a match is ordinary text. So `=$f(a; b); c` has two items, `=($a & "x; y")` one, and `x(a; b)` two, since it is not a formula.
- **C16.** Each part between separators, and before and after them, is an item unless it is empty. So an empty value gives no item, `a;;b` has two items, and `a;`, `;a` and `;` have one, one and none.
- **C19.** `;` splits only a field's own value. Node labels are never split.

```example C15 C16
a::
b:: anna
c:: anna;
d:: anna; oscar
e:: a;;b
f::;
g:: ;;a
h:: a\;b; c
i:: `a;b`; c
.
root
  field "a"
  field "b"
    "anna"
  field "c"
    "anna"
  field "d"
    "anna"
    "oscar"
  field "e"
    "a"
    "b"
  field "f"
  field "g"
    "a"
  field "h"
    "a\\"
    "b"
    "c"
  field "i"
    quoted "a;b"
    "c"
```

```example C15
a:: =$f(a; b); c
b:: =($a & ')'; $b)
c:: x(a; b)
d:: =f(a; b
e:: =(a\;b)
f:: =($a & "x; y"); z
.
root
  field "a"
    "=$f(a; b)"
    "c"
  field "b"
    "=($a & ')'; $b)"
  field "c"
    "x(a"
    "b)"
  field "d"
    "=f(a"
    "b"
  field "e"
    "=(a\\;b)"
  field "f"
    "=($a & \"x; y\")"
    "z"
```

## 10. Writing

- **C41.** A writer either edits a file or writes a model as new text. An **edit** changes only the lines it touches: every other line, blank lines included, stays exactly as written. Every line a writer produces is in **canonical form**, as this section describes. `fmt` rewrites every line of a file in canonical form.
- **C26.** A writer quotes a key, label, value or item with `"` quotes (C22), which can hold any text, when it:
  - is empty, where an empty text must be kept
  - is a label or item that is quoted in the model (section 1), so it stays quoted and Part 2 still skips it (L1)
  - starts or ends with whitespace (any Unicode whitespace), except trailing spaces and tabs that C34 drops
  - starts with `"`, or starts and ends with `` ` ``, so it would read as quoted (C21) or open a span (C25)
  - contains a backtick (keys, field values and items), which could otherwise open a span that hides a `::` or `;` written after it (C25); in a formula item (C15), only a span that doesn't close within the item counts
  - starts with three backticks (keys and labels), which would otherwise open a block (C29), unless C32 applies
  - contains a newline or another control character, except tab
  - contains `::` (keys)
  - contains `[` (keys), which could otherwise open a bracket that hides the `::` written after it (C51)
  - is a label with a `::` outside spans and brackets (C5, C51)
  - contains `;` (field values and items), except inside a group in a formula item (C15)
  - is a key that ends with `:`, which would otherwise merge into the `::` after it
- **C27.** Everything else is written bare.
- **C47.** Inside `"` quotes, a writer escapes in one canonical form, so equal text is always written the same:
  - `"` and `\` are written `\"` and `\\`.
  - Backspace, form feed, newline, carriage return and tab are written `\b`, `\f`, `\n`, `\r` and `\t`.
  - Other control and format characters, and whitespace other than space, are written `\uXXXX` in lowercase hex, as a surrogate pair above U+FFFF. So NBSP is `\u00a0`.
  - Everything else is written as is, `/` included.
- **C32.** A label that is a block is written as its lines, each with the node's indent, not quoted. A label is a block when its first line is a fence with no backtick after it, its last line would end the block (C29), it has at least two lines, no line in between would end the block, and no line ends with `\r` (C1).
- **C42.** A field line is its key, `::`, one space and its inline items (C43): `key:: value`. Without inline items it is `key::`. A field's nodes, and the fields its anonymous node holds, are its child lines (C13). A node with an empty label is written `""` (C26). Only a label that is empty and not quoted in the model is written `::`; a tool that creates a node with an empty label makes the label quoted, so `::` is left for the JSON mapping's `null` (J4, J11).
- **C43.** Inline items are written in order, separated by `;` and a space: `a; b`.
- **C44.** Root lines have no indent. In an edit, a new line copies the indent of a sibling next to it. Without one, it gets its parent's indent plus one step. A step is a tab, unless the file is indented only with spaces; then it is the most common difference in indent between a line and its parent. Writing a model and `fmt` indent with one tab per level.
- **C45.** A new file uses `\n` line endings, ends with a line ending and has no BOM. An edit keeps the file's line ending, taken from its first line (`\n` if there is none), whether its last line has an ending, and any BOM. `fmt` edits every line (C41), so it keeps them too, and every line gets the file's line ending.
- **C46.** Writing a model gives no blank lines. Edits and `fmt` don't add, remove or merge blank lines, but `fmt` removes the spaces and tabs from blank lines outside blocks.
- **C48.** In an edit, a new field goes after the node's last field and that field's child lines, or right after the node line when it has no fields. A new child node goes after the node's last child and its child lines.
- **C33.** Writing is stable: reading what a writer wrote and writing it again gives the same text. The first write may change how a file is written; a second one changes nothing.
- **C34.** A writer drops trailing spaces and tabs from an unquoted label or text item instead of quoting it for them (C26). Otherwise a stray space at the end of a line would quote the text when written, and Part 2 would stop reading it (L1). Quoted text keeps its whitespace.

```example fmt C26 C34 C42 C43 C44 C46 C47
a  ::   x ;y;;  z
b::;
c :: "plain"
Send→draft→+task␣␣
␣␣
f::
  g:: "tab→here"
  h:: "nb\u00A0sp"
.
a:: x; y; z
b::
c:: "plain"
Send→draft→+task

f::
→g:: "tab\there"
→h:: "nb\u00a0sp"
```

````example fmt C31 C32 C33 C41 C44
Notes
␣␣```
␣␣x
␣␣␣␣y
␣␣```
.
Notes
→```
→x
→␣␣y
→```
````

```example fmt C26 C27 C47
k:: x;
"a:b:":: v
Key:: ends with \
::
"Label: with:: sep"
'single'
a:: say `hi`; "x
.
k:: x
"a:b:":: v
Key:: ends with \
::
"Label: with:: sep"
'single'
a:: "say `hi`"; "\"x"
```

```example fmt C41 C45
⟨U+FEFF⟩a :: b⟨U+000D⟩
c::  d
.
⟨U+FEFF⟩a:: b⟨U+000D⟩
c:: d⟨U+000D⟩
```

## 10a. Writing conventions (non-normative)

These are recommendations for writers and editors. A file that doesn't follow them is still valid.

- **W1.** Write a node's fields before its child nodes. Readers accept them in any order (C12).

## 11. Diagnostics

- **C35.** A **diagnostic** reports a problem at a position in the file, with a code and a severity. Diagnostics never change how text is read: every file is read into a tree.
- **C36.** A **warning** marks text that is read as this spec says but is likely a mistake. An **error** marks text whose meaning the spec can't fully settle, such as two IDs on one node (L8). Core gives only warnings.
- **C37.** A **code** is the ID of the rule that gives the diagnostic, followed by `.` and a number when the rule gives more than one. A code is never reused for a different diagnostic. The position is where the diagnostic applies, as listed.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `C3.1` | warning | A whitespace character other than space or tab starts a line's text, after the indent | that character |
| `C3.2` | warning | A whitespace character other than space or tab is next to a `::` outside spans, or separated from it only by spaces and tabs | that character |
| `C22` | warning | A backslash in `"` quotes is not followed by an escape character | the backslash |
| `C29` | warning | A block has no closing line and runs to the end of the file | the opening line's first character after the indent |
| `C49` | warning | A line holds bytes that aren't valid UTF-8, once per line | the first character read from them |

---

# Part 2: Label syntax

A field can be written in three ways: as a field line (`key:: value`, Part 1), as an inline field in a label (`[key:: value]`), or as a **trigger**, a shortform for an inline field. Part 2 reads inline fields and triggers out of labels and items, and value forms, such as numbers and dates, out of items. It doesn't change Core: labels and items stay strings there, and Part 2 can be applied to them at any time. What the fields mean (single-valued fields, merging, types, references) is in [Semantics](Semantics.md).

## 12. Scope

- **L1.** Part 2 reads unquoted node labels and unquoted text items of fields. Quoted labels and items (C21), blocks (C29) and keys are never read.
- **L2.** In a label, every form in this part applies. In a field item, only the value triggers apply: `link`, `url`, `formula` and sigils, and value forms (L21). Inline fields and the `type`, `id`, `state` and `tag` triggers belong to nodes.
- **L3.** Unless a rule says otherwise, a form starts at the beginning of the text or right after a space or tab, and ends before the next space or tab or at the end of the text. A **name** is a letter followed by letters, digits, `_` and `-`. Letters are any Unicode letters.
- **L4.** Inline fields and triggers are read wherever they are written on a node line. A writer keeps the line in a fixed order: the list marker (L16), the `state` trigger, the label (L19), then inline fields, `type` triggers and the `id` trigger. It moves forms that are out of place and keeps the written order within each group. To move a form, it cuts the form with the whitespace L19 removes with it, and writes it into its group with one space between it and its neighbours. It edits the line this way rather than building it from the label, so escapes (L18) and other spacing stay as written, and writing stays stable (C33). An inline field for `type`, `id` or `state` whose value can be written as the trigger is written as the trigger, in its group. Links, URLs, formulas, tags and sigils stay where they are written, as they are part of the label. An inline `tag` field stays an inline field, since writing it as a tag would put it in the label. The `state` trigger is only read at the start (L11), so `[x]` later in a line is text. A line with an inline search (L28) is written as it is: nothing on it is read as a form, so nothing is moved.

```example fmt labels L4
+task Send draft [due:: 2026-10-20] #urgent [x] ^s1
- [x] Done +b #a
Plan [type:: task] [tag:: x]
Ship [state:: done]
.
Send draft #urgent [x] [due:: 2026-10-20] +task ^s1
- [x] Done #a +b
Plan [tag:: x] +task
[x] Ship
```

## 13. Inline fields

- **L5.** `[key:: value]` is an **inline field**: a bracket (C51) that holds a key, `::` and a value. The key is a name, or a quoted string (C21) for any other key, as in `["field name":: value]`. Spaces and tabs around the `::` are allowed. The value is the text after them up to the bracket's closing `]`, without trailing spaces and tabs, so it can hold spaces. It is split into items like a field value (C15 to C20). An empty value is not an inline field. A quoted value is text and never a reference, even in a reference field; a reference is written as a link. An inline field can start anywhere in a label and ends at its `]`. A `]` in the value needs a matching `[` before it, or goes in backticks. Examples: `[due:: 2026-10-10]`, `[note:: two words]`, `["Project lead":: anna]`, `[parts:: a; b]`, `[client:: [[Ruukki AB]]]`, `[net:: =(gross * 0.8)]`.

- **L29.** `[#text]` is short for the inline field `[tag:: text]`: a bracket (C51) that holds `#`, a letter, and any text up to the bracket's closing `]`, read as the value of L5, so it can hold spaces and `;` splits it into items. It is a way to write a tag that is more than a name (L13), such as `[#Ord notes]`. Like any inline field it can start anywhere and is not part of the label (L19), and a writer keeps it as written. `[#` followed by anything other than a letter is not one, so `[#1]` and `[#]` are text, and `[#text](target)` is a Markdown link (L14).

```example labels L29
Plan [#Ord notes] now +task
Also [#a; b c]
[#x](https://example.com) and [#1]
.
root
  node "Plan [#Ord notes] now +task"
    label "Plan now"
    field "tag" inline
      "Ord notes"
    field "type" inline
      "task"
  node "Also [#a; b c]"
    label "Also"
    field "tag" inline
      "a"
      "b c"
  node "[#x](https://example.com) and [#1]"
    field "url" embedded
      "https://example.com"
```

- **L28.** A label that starts with `??` followed by a space or tab, after any list marker (L16), is an **inline search**: short for the inline field `[search:: …]` holding the rest of the label, with the spaces and tabs after `??` removed. Nothing in the rest is read as a form, so `+task` in it doesn't give the node a type, and the label (L19) is empty. `??` without a space or tab after it, as in `??x`, is text, as is `??` followed only by spaces and tabs, and `??` later in a line. `\??` is text (L18). What a search means is in [Semantics](Semantics.md) M60.

```example labels L28
?? [ ] +task @anna
- ?? #urgent
??x and ?? alone
\?? not a search
.
root
  node "?? [ ] +task @anna"
    label ""
    field "search" inline
      "[ ] +task @anna"
  node "- ?? #urgent"
    label ""
    field "search" inline
      "#urgent"
  node "??x and ?? alone"
  node "\\?? not a search"
    label "?? not a search"
```

## 14. Triggers

- **L6.** A trigger is shortform for an inline field. Triggers exist only for the predefined fields below and for sigils that types bind (L15). Each predefined field holds one form of value, except `tag`, whose value is ordinary text.

| Field | Trigger | Read as | Value |
|---|---|---|---|
| `type` | `+task` | `[type:: task]` | a type name |
| `id` | `^t9x1` | `[id:: t9x1]` | an ID |
| `link` | `[[Node label]]`, `[[anna-47xu]]`, `[[Node label^anna-47xu]]` | `[link:: [[Node label]]]`, `[link:: [[anna-47xu]]]` | a node's label or ID, or both |
| `state` | `[ ]`, `[x]` | `[state:: new]`, `[state:: done]` | a state name |
| `formula` | `=sum(x)`, `=(a + b)` | `[formula:: =sum(x)]`, `[formula:: =(a + b)]` | an expression |
| `tag` | `#blue`, `#project/apollo`, `[#Ord notes]` (L29) | `[tag:: blue]`, `[tag:: project/apollo]`, `[tag:: Ord notes]` | ordinary text |
| `url` | `https://example.com`, `./spec.pdf` | `[url:: https://example.com]`, `[url:: ./spec.pdf]` | a URL or a path |

- **L7.** `type`: `+` followed by a name. `+46`, `C++` and a list marker `+ ` (L16) are not types.
- **L8.** `id`: `^` followed by a name. IDs match case-insensitively. More than one on a label is an error.
- **L10.** `link`: `[[`, a reference, and a closing `]]`, which is required: without it, the `[[` is text, with a warning. It can start anywhere and ends at the first `]]`. The reference is one of:
  - `[[Node label]]` or `[[anna-47xu]]`: a label or an ID. It refers to the node with that ID, or else to the node with that label.
  - `[[Node label^anna-47xu]]`: a label and an ID. The text ends with `^` directly followed by a name, with no space before the ID, the form Tana Paste uses.
  - `[[^anna-47xu]]`, `[["Node label"]]` and `[["Node label"^anna-47xu]]` are allowed but usually redundant: `^` says the rest is an ID, and quotes (C21) say the text is a label. They settle the case where an ID and a label match different nodes.

  In a reference field, a plain value is a reference too, without the brackets, and so is a quoted one in a field line, where the quotes mark a label. In an inline field, a quoted value stays text (L5). Brackets and `^` are only needed where plain text would otherwise be read as text. Reference fields and how references resolve are in [Semantics](Semantics.md), section 6.
- **L11.** `state`: `[`, one character and a closing `]` at the start of a label, after any list marker (L16), followed by a space, a tab or the end. `[ ]` reads as `new`, `[-]` as `in-progress`, `[?]` as `waiting` and `[x]` as `done`, the built-in states. Any other character is a custom marker, defined by a type's custom state ([Semantics](Semantics.md) M27), and reads as itself, for the index to map to that state. A marker no type defines is a diagnostic.
- **L12.** `formula`: `=` followed by a letter, `!` or `(`. The value keeps the `=`. `=(…)` ends at the matching `)`, which is required. A path or function call, such as `=node("i42").total`, ends before the next space or tab outside parentheses and quoted strings; its parentheses must balance. A formula with an `(` that is never closed is text, with a warning. `2 + 2 = 4` stays text. What formulas mean is in [Semantics](Semantics.md), section 8.
- **L13.** `tag`: `#` followed by a name, then any number of `/` and a name, for a nested tag such as `#project/apollo`. It ends after its last name, unlike other forms (L3), so punctuation after it is text: `#projectX.` at the end of a sentence is the tag `projectX` and a `.`. `#` followed by a space (a list marker at the start of a label, L16), a digit (`#123`, an issue number) or another `#` is not a tag, and neither is a `#` that doesn't start a form, as in `C#` or a URL. A tag stays in the label (L19).
- **L14.** `url`: a bare URL (a scheme of a letter, then letters, digits, `+`, `-` and `.`, then `://`), a bare path (`./`, `../`, `~/` or `/`, then a character that is neither whitespace nor `/`), or a Markdown link `[text](target)`, which can start anywhere and reads as the target. A target with spaces goes in `<…>`. A bare URL or path doesn't include a trailing `.`, `,`, `;`, `:`, `!`, `?` or `)`. A URL is a value, never a reference to a node.

## 15. Sigils

- **L15.** A type can bind a **sigil**, a single character, with the `sigil` setting on its definition ([Semantics](Semantics.md) M19). The sigil followed by a name is shortform for a field named after the type, holding a reference to a node of that type: the node whose ID is the name, or else the node whose label is that one word, ignoring case. With `%` bound to the type `project`, `Plan the launch %apollo` reads as:

```
Plan the launch
	project:: apollo
```

  A node with a longer label is referenced through its ID, or with a link (L10). A character that no type binds is plain text. No sigil is bound unless a type binds it. A reader without type definitions takes the bindings as an option, by default none, and leaves resolving the name to the index. `team%apollo` is not a reference, since the sigil doesn't follow whitespace.

## 16. Start of a label

- **L16.** A list marker at the start of a label is decoration: `-`, `*`, `+` or `#`, or digits followed by `.`, then a space or tab. It is set aside before the rest of this part applies. Writers keep it. Only one is read, so in `- # x` the `#` is part of the label. Editors may draw a `#` line as a heading; it is a list marker like the others, and the node's place in the tree comes from its indent alone.

```example labels L16
# Heading +task
- # Not a second marker
#tag starts the label
.
root
  node "# Heading +task"
    label "Heading"
    field "type" inline
      "task"
  node "- # Not a second marker"
    label "# Not a second marker"
  node "#tag starts the label"
    field "tag" embedded
      "tag"
```

## 17. Literal text

- **L17.** Spans are literal (C25): no form starts inside one. An unmatched quote is plain text.
- **L18.** A backslash before the first character of a form makes it plain text: `\+task`, `\^id`, `\#tag`, `\[[x]]`, `\=x`, `\[due:: x]`, `\[#x]`. Part 2 reads the text without the backslash; Core keeps it.

## 18. Label and items

- **L19.** A node's **label** is its line without the list marker, the `state` trigger, inline fields, and the `type` and `id` triggers. A removed form takes the spaces and tabs before it with it, or those after it when only the list marker and removed forms come before it. So `Send +task draft` and `+task Send draft` both have the label `Send draft`. The system field `label` holds it. Links, URLs, formulas, tags and sigils are part of it, where they were written (L4): `Call #anna to decide on budget #projectX +task` has the label `Call #anna to decide on budget #projectX`, and references to it by label (L10) name it with its tags. Editors may render them differently.
- **L20.** A field item that is exactly one value trigger (L2) is that value. Any other item is text, and the triggers in it are read as in a label.

```example labels C36 L3 L6 L7 L8 L11 L13 L16 L19
- [-] Buy milk +errand #urgent ^m1lk
+task Call Anna
1. Price +46, C++, C#, #5 and # heading
Two ids ^a1 ^b2
[ ] Call back
[?] Ask Erik
+ [x] Read the spec
.
root
  node "- [-] Buy milk +errand #urgent ^m1lk"
    label "Buy milk #urgent"
    field "state" inline
      "in-progress"
    field "type" inline
      "errand"
    field "tag" embedded
      "urgent"
    field "id" inline
      "m1lk"
  node "+task Call Anna"
    label "Call Anna"
    field "type" inline
      "task"
  node "1. Price +46, C++, C#, #5 and # heading"
    label "Price +46, C++, C#, #5 and # heading"
  node "Two ids ^a1 ^b2"
    label "Two ids"
    field "id" inline
      "a1"
    field "id" inline
      "b2"
  node "[ ] Call back"
    label "Call back"
    field "state" inline
      "new"
  node "[?] Ask Erik"
    label "Ask Erik"
    field "state" inline
      "waiting"
  node "+ [x] Read the spec"
    label "Read the spec"
    field "state" inline
      "done"
! L8 4:13
```

```example labels L7 L13 L18 L19
Call #anna to decide on budget #projectX. +task
Nested #project/apollo, #a/ and #b-2/c
Not tags: #123 # C# ##x \#y and #
.
root
  node "Call #anna to decide on budget #projectX. +task"
    label "Call #anna to decide on budget #projectX."
    field "tag" embedded
      "anna"
    field "tag" embedded
      "projectX"
    field "type" inline
      "task"
  node "Nested #project/apollo, #a/ and #b-2/c"
    field "tag" embedded
      "project/apollo"
    field "tag" embedded
      "a"
    field "tag" embedded
      "b-2/c"
  node "Not tags: #123 # C# ##x \\#y and #"
    label "Not tags: #123 # C# ##x #y and #"
```

```example labels L1 L2 L5 L10 L12 L14 L17 L18 L20
Plan [due:: 2026-10-20] ["Project lead":: anna] [parts:: a;b] [note:: "two words"] [memo:: two words]
\#literal and \[due:: x] and `#code` here
"Quoted +task label"
#key:: +task #x
who:: [[Anna]]
See https://example.com/a. and ./spec.pdf, not //x
Total =(a + b) then =sum(x
Broken [[link
.
root
  node "Plan [due:: 2026-10-20] [\"Project lead\":: anna] [parts:: a;b] [note:: \"two words\"] [memo:: two words]"
    label "Plan"
    field "due" inline
      "2026-10-20" date
    field "Project lead" inline
      "anna"
    field "parts" inline
      "a"
      "b"
    field "note" inline
      quoted "two words"
    field "memo" inline
      "two words"
  node "\\#literal and \\[due:: x] and `#code` here"
    label "#literal and [due:: x] and `#code` here"
  node quoted "Quoted +task label"
  field "#key"
    "+task #x"
  field "who"
    "[[Anna]]" link
  node "See https://example.com/a. and ./spec.pdf, not //x"
    field "url" embedded
      "https://example.com/a"
    field "url" embedded
      "./spec.pdf"
  node "Total =(a + b) then =sum(x"
    field "formula" embedded
      "=(a + b)"
  node "Broken [[link"
! L12 7:21
! L10 8:8
```

```example labels L6 L10 L20 D12 D13 D14
Call Anna about [[Ruukki AB]] +task [client:: [[Ruukki AB]]] [due:: 2026-10-20]
See [[anna-47xu]], [[Anna Smith^anna-47xu]] and [[^x1]]
client:: [[Ruukki AB]]; see [[Acme]] now
.
root
  node "Call Anna about [[Ruukki AB]] +task [client:: [[Ruukki AB]]] [due:: 2026-10-20]"
    label "Call Anna about [[Ruukki AB]]"
    field "link" embedded
      "[[Ruukki AB]]"
    field "type" inline
      "task"
    field "client" inline
      "[[Ruukki AB]]" link
    field "due" inline
      "2026-10-20" date
  node "See [[anna-47xu]], [[Anna Smith^anna-47xu]] and [[^x1]]"
    field "link" embedded
      "[[anna-47xu]]"
    field "link" embedded
      "[[Anna Smith^anna-47xu]]"
    field "link" embedded
      "[[^x1]]"
  field "client"
    "[[Ruukki AB]]" link
    "see [[Acme]] now"
      field "link" embedded
        "[[Acme]]"
```

## 19. Value forms

- **L21.** A text item, of a field or an inline field (L5), that is exactly one of the forms below is a value of that form. Value forms are read only there: never in labels, keys or quoted text (L1). The text stays as written; a form is only how it reads. There are no other forms.
- **L22.** `number`: an optional `-`, then `0` or a digit from 1 to 9 followed by any digits, then optionally `.` and one or more digits. So `-12`, `0` and `9.5` are numbers, and `007`, `+46`, `1e3`, `1.`, `.5` and `1 000` are text. A year such as `2026` is a number.
- **L27.** `boolean`: `true` or `false`, in any letter case, so `True` and `FALSE` are booleans too. `yes` and `no` are text, and `1` and `0` are numbers.
- **L23.** `date`: `YYYY-MM-DD`, a day that exists in the Gregorian calendar, e.g. `2026-10-15`.
- **L24.** `datetime`: a date, then `T` or a space, then `HH:MM`, optionally followed by `:SS` and a fraction of a second after `.`, and optionally an offset: `Z`, or `+` or `-` followed by `HH:MM`. Hours run from 00 to 23, minutes and seconds from 00 to 59. Without an offset, the time is local. E.g. `2026-10-15T14:30`, `2026-10-15 14:30:05` and `2026-10-15T12:30Z`.
- **L25.** `week`: `YYYY-Www`, with a week from 01 to 53, e.g. `2026-W42`. `month`: `YYYY-MM`, with a month from 01 to 12, e.g. `2026-10`. Weeks are ISO 8601 weeks, which start on Monday, so a year has a week 53 when it starts on a Thursday, or is a leap year that starts on a Wednesday.
- **L26.** Text with the digits and separators of a date, datetime, week or month, but a part out of range, is text, with a warning: `2026-02-30`, `2026-13-01`, `2026-W54`, `2026-10-15T24:00`.

```example labels L21 L22 L23 L24 L25 L26 L27
a:: -12; 0; 9.5; 2026
b:: 007; +46; 1e3; 1.; .5
c:: 2026-10-15; 2026-W42; 2026-10
d:: 2026-10-15T14:30; 2026-10-15 14:30:05.5+02:00; 2026-10-15T12:30Z
e:: "2026-10-15"; 2026-02-30
f:: true; false; True; FALSE; yes; "true"
Call Anna [due:: 2026-10-15 14:30]
.
root
  field "a"
    "-12" number
    "0" number
    "9.5" number
    "2026" number
  field "b"
    "007"
    "+46"
    "1e3"
    "1."
    ".5"
  field "c"
    "2026-10-15" date
    "2026-W42" week
    "2026-10" month
  field "d"
    "2026-10-15T14:30" datetime
    "2026-10-15 14:30:05.5+02:00" datetime
    "2026-10-15T12:30Z" datetime
  field "e"
    quoted "2026-10-15"
    "2026-02-30"
  field "f"
    "true" boolean
    "false" boolean
    "True" boolean
    "FALSE" boolean
    "yes"
    quoted "true"
  node "Call Anna [due:: 2026-10-15 14:30]"
    label "Call Anna"
    field "due" inline
      "2026-10-15 14:30" datetime
! L26 5:19
```

## 20. Diagnostics

Codes, severities and positions work as in section 11.

| Code | Severity | Given when | Position |
|---|---|---|---|
| `L8` | error | A label has more than one `id` trigger | each `id` trigger after the first |
| `L10` | warning | A `[[` has no closing `]]` | the `[[` |
| `L11` | warning | A `state` marker that no type defines | the `[` |
| `L12` | warning | A formula has an `(` that is never closed | the `=` |
| `L26` | warning | Text has the digits and separators of a date, datetime, week or month, but a part out of range | the text's first character |

`L11` needs the types' custom markers ([Semantics](Semantics.md) M27). A reader without type definitions takes them as an option, like sigil bindings (L15).

---

# Part 3: Files

## 21. Files

These rules are for tools that find files, such as editors, the index and the language server. A reader is given a file and reads it as Ord.

- **F1.** An Ord file has no header line and states no version: the workspace (F3), or the extension `.ord`, is what makes a file Ord. Tools don't guess from the content. `.txt` is the usual extension; `.ord` tells an editor to treat a file as Ord anywhere, and is redundant inside a workspace.
- **F2.** Its media type is `text/plain; charset=utf-8`, since no media type is registered for Ord. On Apple platforms, `.ord` is the uniform type `com.pensiveworks.ord`, which conforms to `public.plain-text`, so apps that don't know Ord open it as text.
- **F3.** In a workspace, `.txt` files, files without an extension and `.ord` files are read as Ord, so `.ord.txt` is too. Outside a workspace, only `.ord` files are, so a tool never claims `.txt` or extensionless files in general. Editors switch a file to Ord only when they would otherwise open it as plain text, so a `Makefile` keeps its own language.
- **F4.** A **workspace** is a folder that holds a workspace file, with everything under it. A file belongs to the nearest workspace above it, its own folder included.
- **F5.** A **workspace file** is named `Ord.workspace`, in any letter case, such as `ord.workspace`. It is an Ord file, though its name doesn't end in `.ord`; its first line can name the workspace, and what else it holds is in [Semantics](Semantics.md). Other names, such as `workspace.ord`, are ordinary files.
