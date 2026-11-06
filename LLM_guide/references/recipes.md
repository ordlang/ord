# Ord recipes

Worked examples for common requests. Indentation is one tab per level.

## Start a workspace

Create `Ord.workspace` at the folder's root. By convention it holds the definitions.

```
Task +type-def
	name:: task
	color:: project
	checkbox:: true
	state:: someday [s] > waiting
	fields::
		due::
		owner::
			options:: +person
		project::
			options:: +project
			tags:: true
		late:: =($exists(due) and due < $today() and _state != "done")
Person +type-def
	name:: person
	fields::
		_id:: ~$slug(_label)
		email::
		phone::
Project +type-def
	name:: project
	color:: purple
	fields::
		status::
			options:: active; paused; closed
		open:: =($count(_children[("task" in _types) and _state != "done"]))
Issue +type-def
	name:: issue
	fields::
		issue type::
			options:: bug; feature
			tags:: true
Urgent +tag-def
	name:: urgent
	color:: red
```

- `issue type` takes its options as tags: `Login fails #bug +issue` has `issue type:: bug`, not the tag `bug`.
- `project` takes projects as tags: `[ ] Call Anna #acme +task` refers to `ACME +project`.
- `owner:: Anna Smith` is a reference to the `+person` node `Anna Smith`.
- Add a `.txt` file per topic.

## Journal days

Day nodes are found by ID: `^journal-2026-10-22` for a day, `^journal-2026-W43` for a week, `^journal-2026-10` for a month. A day's label is `Thursday 22, 2026`. Which file and parent hold them is a tool setting; a common layout is one file per week, with the week at the top and its days in date order:

```
Week 43, 2026 ^journal-2026-W43
	Wednesday 21, 2026 ^journal-2026-10-21
		Called the plumber
	Thursday 22, 2026 ^journal-2026-10-22
		[ ] Buy tiles [due:: 2026-10-24] +task
		Meeting with Anna +meeting
			Agreed on the kitchen layout
			[ ] Send drawings to Anna +task
		? [ ] +task [[Anna]]
```

- To add to a day: find `^journal-YYYY-MM-DD`. If it is missing, add it under its week or month in date order, labelled like its neighbours; if that is missing too, create it, in a file named like the existing journal files.
- A week belongs to the year and month of its Thursday (ISO 8601).
- Write the actual date; never `~today`.

## Tasks

```
[ ] Book electrician #urgent [due:: 2026-10-30] +task
[-] Paint the hallway +task
	owner:: Anna Smith
	Second coat on Saturday
[?] Quote from Ruukki #waiting-on +task
[x] Order tiles for #home/kitchen +task
[s] Build a sauna +task
```

- Complete: `[ ]` to `[x]`. Start: `[-]`. Waiting: `[?]`. Custom: the type's marker (`[s]`).
- Tags go in the text where they read naturally and stay part of the label. `#home/kitchen` is a nested tag: `? #home` finds it too.
- A due date goes inline when short, as a field line when the node already has field lines.
- Notes about the task are child lines, not fields.
- Open tasks anywhere: `? +task ![x]`.

## Meeting notes

```
Kitchen planning #home +meeting
	date:: 2026-10-22
	attendees::
		@ Anna Smith
		@ Erik Berg
	Budget is 120 000 SEK
	Tiles: Anna prefers the grey ones
	Decisions
		Keep the old cabinets #decision
	Actions
		[ ] Erik gets two quotes [due:: 2026-10-29] +task
			owner:: Erik Berg
		[ ] Anna sends the drawings +task
			owner:: Anna Smith
```

- `attendees` holds mirrors of the people, so each is a reference to the person (M49); `attendees:: @Anna Smith; @Erik Berg` on one line means the same.
- `Tiles: Anna prefers…` is prose: one colon never makes a field.

## Show a node in a second place

To list a task under today without moving it from its project:

```
Thursday 22, 2026 ^journal-2026-10-22
	@ Send mapping draft to Anna
```

The task stays where it is. Write nothing under the mirror.

## A small database

```
Ruukki AB +company
	org-nr:: "556123-4567"
Invoice 42 +invoice
	client:: Ruukki AB
	date:: 2026-10-15
	Design work
		amount:: 1000
	Review
		amount:: 200
```

With `client` defined with `options:: +company` and `net:: =($sum(_children.amount))` on the invoice type, `Invoice 42` shows `net` = 1200. The quotes keep the org number as text. For hundreds of line items, move them to a CSV file with a `_parent` column holding `i42`, and only then give the invoice that ID, `Invoice 42 +invoice ^i42` (see csv.md).

## Converting Markdown to Ord

| Markdown | Ord |
|---|---|
| `# Heading` with paragraphs under it | `Heading` with the paragraphs indented one tab under it; `# ` may stay as a list marker |
| `- item`, `1. item` | Keep the markers, or drop them; nesting is by tab |
| `- [ ] todo`, `- [x] done` | Same |
| `**Owner:** Anna` | `owner:: Anna`, when it is an attribute; else `Owner: Anna` as prose |
| `[text](url)` | Same; reads as a `_url` |
| Tables, code | A fenced block with an info string: ```` ```csv ```` |
| Front matter `key: value` | Field lines under the document's root node |

Bold, italics and headings don't structure Ord: nesting does.

## Promote a local field to a type

```
Monstera
	watered:: 2026-10-01
	every:: 7
```

When the same fields repeat, define the type and give it to the nodes, `Monstera +plant`:

```
Plant +type-def
	name:: plant
	fields::
		watered::
		every::
		next:: =($date(watered, every))
		thirsty:: =(next <= $today())
```

Then `? +plant =(thirsty)` lists the plants to water.
