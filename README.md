# Ord - Outline and Relational Data

Ord is a plain-text markup language for note-taking and structured information. One line is one node, indentation makes the tree, and `key:: value` lines make the outline structured data. Files are UTF-8 text, readable in any editor, and the same files work for people and for LLMs.

```
Launch website #project
	[x] Write the intro page [due:: 2026-10-20] #task +docs
	[-] Pick a domain #task +urgent
		Ask Johan about the DNS
	budget:: 1200
```

## Status

Version **0.4.0-draft**. The specs are drafts: before 1.0.0, any version can change how a file is read. All files here share one version; see the [changelog](CHANGELOG.md).

## Files

| File | Status | What it covers |
|---|---|---|
| [Ord_Specification.md](Ord_Specification.md) | normative | The syntax and data model (Part 1), label syntax: inline fields, triggers and value forms (Part 2), and which files are Ord (Part 3). Start here. |
| [Test_format.md](Test_format.md) | normative | The AST dump that tests compare against, and how tests are written and run. |
| [Semantics.md](Semantics.md) | normative | What a workspace means: ownership, kinds, IDs, references, states, formulas and queries. |
| [JSON_mapping.md](JSON_mapping.md) | normative | How JSON maps to and from the data model. |
| [CSV_mapping.md](CSV_mapping.md) | normative | How CSV files map to and from the data model, a row per node. |
| [Grammar.md](Grammar.md) | informative | An ABNF grammar and a parsing strategy. |
| [CHANGELOG.md](CHANGELOG.md) | informative | What changed in each version, and retired rule IDs. |

Every rule has a permanent ID, such as `C5` (Core), `L11` (label syntax), `F3` (files), `D` and `T` (test format), `M` (Semantics), `J` (JSON) and `V` (CSV). IDs never change and are never reused.

## Conformance

An implementation conforms to a class in a version when it passes that class's tests in that version:

- **Reader**: reads any file into the data model (Part 1).
- **Writer**: a reader that also writes and formats files.
- **Label reader**: a reader that also applies Part 2.
- **JSON mapper**, **CSV mapper** and **graph builder**: defined in their companion specs.

## Tests

The examples in the specs are the tests: fenced blocks with the info string `example` (Test format, part B). [ordtest](https://github.com/ordlang/ord-utils) extracts and runs them:

```sh
node ordtest.mjs lint Ord_Specification.md            # check the tests themselves
node ordtest.mjs run Ord_Specification.md --dump "ord dump -" --fmt "ord fmt -"
```

## Changing the spec

1. Change the rules and their tests in a pull request. CI lints every test.
2. Add the change to the changelog, naming the rules it changes.
3. To release, update the version line in every spec file, and tag the commit, such as `v0.4.0-draft`.

## License

This work is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) <img src="https://mirrors.creativecommons.org/presskit/icons/cc.svg" alt="" height="16"><img src="https://mirrors.creativecommons.org/presskit/icons/by.svg" alt="" height="16">

Created by Pontus Sundén. Contact: [hi@ordlang.org](mailto:hi@ordlang.org).
