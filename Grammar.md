# Ord grammar

Version 0.5.0-draft, 2026-10-27. An informative (non-normative) companion to [Ord specification](Ord_Specification.md), of the same version: an ABNF grammar and one way to read a file. The prose of the Spec is what conforms.

## 1. Grammar

This grammar is a reading aid for the prose, which is what conforms. It works on characters, after decoding (C49), and describes the pieces a line is made of. Some choices depend on what comes before or after a character, which ABNF can't say, so the prose makes them: which `::` is the separator (C5), where spans are (C25), where a block ends (C29), how the tree is built (C9), and where label forms start and end (L3). Names here are separate from the dump's grammar ([Test format](Test_format.md) A5).

```abnf
; ---- Lines (C1 to C4) ----

file          = [BOM] *(line eol) [last-line]
BOM           = %xFEFF
eol           = LF / CR LF          ; a CR right before LF belongs to eol (C1)
line          = *line-char
last-line     = 1*line-char         ; a file that ends with eol has no empty last line
line-char     = %x00-09 / %x0B-10FFFF
indent        = *WSP                ; WSP is space or tab, the only whitespace; an NBSP is read as a space (C3)

; ---- Blocks (C29) ----
; A block runs from an opening line to the next closing line with at least
; as many backticks as its fence, or to the end of the file.

block-open    = indent fence *not-bt
block-close   = indent fence *WSP   ; at least as many backticks as the opening fence
fence         = 3*"`"
not-bt        = %x00-09 / %x0B-5F / %x61-10FFFF

; ---- Quoted strings (C21, C22) ----

quoted-string = dq-string / bt-string
dq-string     = DQUOTE *( dq-char / escape / other-escape ) DQUOTE
dq-char       = %x00-09 / %x0B-21 / %x23-5B / %x5D-10FFFF   ; not LF, " or \
escape        = "\" ( DQUOTE / "\" / "/" / %x62 / %x66 / %x6E / %x72 / %x74
                    / %x75 4HEXDIG )   ; \" \\ \/ \b \f \n \r \t \uXXXX
other-escape  = "\" ( %x00-09 / %x0B-21 / %x23-2E / %x30-5B / %x5D-61
                    / %x63-65 / %x67-6D / %x6F-71 / %x73 / %x76-10FFFF )
              / "\" %x75            ; \u without four hex digits after it
bt-string     = "`" *not-bt "`"

; ---- Separators (C5 to C7, C15) ----

separator     = *WSP "::" *WSP      ; the first "::" outside spans and brackets (C5)
item-sep      = *WSP ";" *WSP       ; outside spans and, in a formula item, groups (C15)
line-text     = key separator value / value
key           = 1*line-char         ; no "::" outside spans and brackets
bracket       = "[" *( bt-string / bracket / bracket-char ) "]"   ; C51, nests
bracket-char  = %x00-09 / %x0B-5A / %x5C / %x5E-5F / %x61-10FFFF
                ; not LF, "[", "]" or a backtick
value         = *line-char

; ---- Label syntax (Part 2) ----
; Unless a rule says otherwise, a form starts at the start of the text or
; after a space or tab, and ends before the next one or at the end (L3).
; No form starts inside a span (L17) or right after a "\" (L18).

name          = letter *( letter / DIGIT / "_" / "-" )      ; L3
letter        = ALPHA / %x80-10FFFF ; a Unicode letter (S3), not every character in this range
non-ws        = %x00-08 / %x0B-1F / %x21-10FFFF             ; not space, tab or LF

list-marker   = ( "-" / "*" / %x2022 / "+" / 1*DIGIT "." ) WSP ; L16, at the start of a label
state         = "[" line-char "]"   ; L11, first after any list marker, then WSP or the end
type          = "+" name            ; L7
id            = "^" name            ; L8
tag           = "#" name *( "/" name ) ; L13, ends after its last name
sigil-ref     = non-ws name         ; L15, where the first character is a sigil a type binds

inline-field  = "[" ( name / quoted-string ) *WSP "::" *WSP inline-value *WSP "]"  ; L5, a bracket
inline-value  = 1*( bt-string / bracket / bracket-char ) ; not empty, split into items
inline-search = "??" 1*WSP non-ws *line-char                 ; L28, first after any list marker
bracket-tag   = "[#" letter *( bt-string / bracket / bracket-char ) "]" ; L29, short for [tag:: …], not before "("

link          = "[[" link-ref "]]"  ; L10, can start anywhere, ends at the first "]]"
link-ref      = link-label [ "^" name ] / "^" name
link-label    = quoted-string / 1*( link-char / "]" link-char )
                ; a label that ends in "^" and a name is split into label and ID
link-char     = %x00-09 / %x0B-5C / %x5E-10FFFF             ; not LF or "]"

formula       = "=" ( paren-group / ( letter / "!" ) *formula-part )   ; L12
formula-part  = paren-group / quoted-string / path-char
paren-group   = "(" *( paren-group / quoted-string / paren-char ) ")"
paren-char    = %x00-09 / %x0B-21 / %x23-26 / %x2A-5F / %x61-10FFFF
                ; not LF, a parenthesis or a quote character
path-char     = %x00-08 / %x0B-1F / %x21 / %x23-26 / %x2A-5F / %x61-10FFFF
                ; not space or tab either

url           = bare-url / bare-path / md-link              ; L14
bare-url      = ALPHA *( ALPHA / DIGIT / "+" / "-" / "." ) "://" *non-ws
bare-path     = ( "./" / "../" / "~/" / "/" ) path-start *non-ws
path-start    = %x00-08 / %x0B-1F / %x21-2E / %x30-10FFFF   ; not whitespace or "/"
                ; a bare URL or path doesn't take a trailing . , ; : ! ? or )
md-link       = "[" *md-text "]" "(" ( "<" *md-angle ">" / 1*md-target ) ")"
                ; can start anywhere, reads as the target
md-text       = %x00-09 / %x0B-5C / %x5E-10FFFF             ; not LF or "]"
md-angle      = %x00-09 / %x0B-3D / %x3F-10FFFF             ; not LF or ">"
md-target     = %x00-08 / %x0B-1F / %x21-28 / %x2A-10FFFF   ; not whitespace or ")"

; ---- Value forms (L21 to L27) ----
; A value form is a whole text item (L21). Dates must exist (L23).

number        = [ "-" ] ( "0" / %x31-39 *DIGIT ) [ "." 1*DIGIT ]
boolean       = "true" / "false"                             ; any letter case (L27)
date          = year "-" month-num "-" day-num
datetime      = date ( %s"T" / SP ) time [ offset ]
time          = hour ":" minute [ ":" second [ "." 1*DIGIT ] ]
offset        = %s"Z" / ( "+" / "-" ) hour ":" minute
week          = year "-" %s"W" week-num
month         = year "-" month-num
year          = 4DIGIT
month-num     = "0" %x31-39 / "1" %x30-32                   ; 01 to 12
week-num      = "0" %x31-39 / %x31-34 DIGIT / "5" %x30-33   ; 01 to 53
day-num       = "0" %x31-39 / %x31-32 DIGIT / "3" %x30-31   ; 01 to 31
hour          = %x30-31 DIGIT / "2" %x30-33                 ; 00 to 23
minute        = %x30-35 DIGIT                               ; 00 to 59
second        = %x30-35 DIGIT                               ; 00 to 59
```

## 2. Parsing strategy

One way to read a file that follows the reading order (C50) and takes time linear in the size of the file. Any other way that gives the same results is fine.

1. **Decode.** Decode the bytes with a UTF-8 decoder that gives U+FFFD for bad bytes instead of failing (C49), such as `TextDecoder` in JavaScript. For the `C49` warning, the decoder or a check of the raw bytes has to tell replaced bytes apart from a U+FFFD written in the file. Drop a BOM at the start (C2).
2. **Count characters.** Columns and ranges count code points, not bytes or UTF-16 units. In JavaScript, an emoji is one character but two string units.
3. **Split lines.** Split at `\n`, then remove one `\r` from the end of each line (C1). Any other `\r` stays as content (C3). Replace each no-break space (U+00A0) with a space (C3); positions don't change. A file that ends with `\n` has no empty last line.
4. **Blocks.** Go through the lines from the top. At a line that opens a block (C29), note its fence's length and its indent, and take lines up to the next closing line with at least that many backticks, or to the end of the file with a `C29` warning. The label is these lines, each with up to the opening indent's length of leading spaces and tabs removed, joined with `\n` (C31). The later steps skip these lines, except to place the block in the tree.
5. **Spans.** On each other line that isn't blank, find the spans in the text after the indent, left to right (C25). A backtick opens a span that ends at the next backtick. A `"` opens one only at the start of the text, after the separator, or after an item's `;` and its spaces and tabs, and it ends at the next `"` that isn't escaped: one that follows an odd run of backslashes. To stay linear when quotes don't close, note where the next backtick and the next unescaped `"` are as you go, instead of searching again from each quote.
6. **Separator.** Take the first `::` outside spans and brackets (C5, C51). To skip a bracket, count nested `[` and `]`, skipping backtick spans, until the count is back to zero; a `[` that never gets there is ordinary text. To stay linear, find each `[`'s matching `]` once per line, with a stack. The separator takes the spaces and tabs on both sides of it, and the key and value are the text before and after it (C6).
7. **Key and label.** Unquote the key (C21, C22). A quoted key is a key even when its content is empty (C11). Without a key, the value is the node's label: unquote it, ignoring spaces and tabs after a closing quote at the end of the line (C24).
8. **Items.** Split a field's value at each `;` outside spans, together with the spaces and tabs touching it (C15). In an item that starts with `=`, also skip groups: at a `(` outside spans, find its matching `)`, counting nested parentheses and skipping spans and `"` and `'` strings, and continue after it; a `(` without a match is ordinary text. Drop empty parts (C16) and unquote each item.
9. **Tree.** Keep a stack of open lines and their indents, a tab and a space each counting as 1 (C9). For each line that isn't blank, and each block at its opening line, pop the lines whose indent isn't smaller. The top of the stack is the parent, or the root if the stack is empty. Under a node, fields and nodes are children (C12). Under a field, nodes are items, and fields go into one anonymous node, made at the first such field and kept after the other items (C13).
10. **Ranges and diagnostics.** Record each element's range as you go (A3); the writer needs them (C41). Collect diagnostics with their positions, and sort them by line, column and code at the end (D7).
11. **Label syntax.** Part 2 runs afterwards, on unquoted labels and text items (L1), and never changes the tree. Value forms are read from whole text items (L21).
