# te — design

te is a terminal text editor with three goals: it works well over ssh, it opens files of any size
(many gigabytes) instantly, and it is more pleasant to use than nano.

## The text core

**The file is never loaded.** It is mapped read-only (`mmap`, `PROT_READ`, `MAP_PRIVATE`) and the
descriptor closed; the kernel reads a page only when something touches it. Every inserted byte is
appended to an **add buffer** that only grows. The document is a **piece table**: a sequence of
pieces, each a run of bytes in one source (a mapped file or the add buffer).

**The pieces are held in a balanced persistent tree** — a treap in reading order, heap-ordered by a
random priority. Splitting a tree at a byte offset and joining two trees are the only two
operations; an insert is two splits around the offset and two joins with the new piece between, a
delete two splits and one join. Each is O(log n) in the number of pieces. **Persistent** means an
edit never changes a node: it builds afresh the O(log n) nodes on the path it walked and shares
everything else with the version before.

**Every node caches its subtree's byte count and newline count.** Byte counts give offset → piece in
O(log n); newline counts give line → offset (goto line) and offset → line in O(log n).

**Undo and redo are kept roots.** The history is a list of roots, one per edit, and the current
version is an index into it; undo moves the index back, redo forward, and an edit after an undo drops
the roots ahead of it. Structural sharing makes a version cost O(log n) nodes, never a copy of the
text.

**Opening is instant because nothing is counted on opening.** The mapped file becomes pieces of
1 MiB each (a 4 GB file is 4096 pieces, built directly as a balanced tree in well under a millisecond), every one
with its newline count marked *unknown*. Counts are filled in lazily:

- A line query that needs a piece's count reads that piece's bytes once and stores the count in the
  node. A subtree's sum is stored once all its pieces are known. A count is a cache of bytes that
  never change, so storing it in a node that many versions share changes no version's meaning.
- **goto line `k`** searches rather than counts: an unknown left subtree is searched first, so line 3
  of a 200 GB file reads one piece; a subtree the search passes over is counted exactly once on the
  way, and its count is kept.
- **offset → line** counts everything before the offset (once).
- **Splitting a piece whose count is known** counts the shorter half and subtracts, so edits keep
  counts known; a new piece from the add buffer is counted as it is inserted.
- **The rest is counted between keys** (`Text.count_some`): whenever no key is waiting, the terminal
  loop counts the uncounted pieces in reading order, 4 MiB a step (the step a search takes), on the
  UI thread, so the tree is never shared across threads and no key waits behind more than one step.
  The status bar shows no line total until the count is done, then `of N`; a step that changes
  nothing on screen sends nothing to the terminal. A counted piece of a file is let go of
  (`madvise(MADV_DONTNEED)`): Linux drops the pages from the process; macOS keeps them mapped, so
  `ps` shows the file's size as resident while the footprint stays a few MB (2 GB measured: RSS
  2 GB, footprint 3.7 MB) — clean file pages the kernel reclaims at will.
- **Ctrl+End before the count is done** first counts one more step (which finishes any ordinary
  file), and otherwise goes to the end without knowing the cursor's line: the screen shows the last
  lines, found by reading back from the end to their line breaks (`Text.start_before`), with no
  numbers, and the status bar says `Ln ≥ N` from the newlines counted so far. The cursor learns its
  line when the count finishes. Ctrl+Home, Esc, Ctrl+S, Ctrl+Q and a resize keep that screen; any
  other key needs the line, and counts the rest at once first.

**Lines and columns.** A line ends at `\n`. **CRLF is one line break**: the break is the `\n`, and a
`\r` immediately before it is not part of the line's text, so the line's end, a column past it, and an
offset between the two bytes all mean the end of the line. A lone `\r` is an ordinary character. A
column is a count of code points from the line start; an offset inside a UTF-8 sequence is answered as
that character's start, so no line or column query ever splits one. Display width (wide CJK, zero-width
marks, grapheme clusters) is the view's job, not the core's.

**Save** writes every piece in order to a pending file beside the target, `fsync`s it, gives it the old
file's permissions, and renames it over the target, so the name always holds the whole old text or the
whole new. On failure the pending file is removed and nothing changes. The saved file is then mapped
and becomes the current version's single source; its pieces keep the old pieces' boundaries (folded
together while they fit in 1 MiB) **with their newline counts carried over**, so a save never recounts.
Older versions still point at the files they were made from, which stay mapped — a renamed-over file
lives as long as its mapping, which is what makes undo after save work. (A file changed *in place* by
another program under a live mapping is outside what any mmap editor can protect against.)

**A file changed on disk is noticed on save.** Opening, saving and reloading record the file's size,
modification time and inode; a save that finds them changed (written since, replaced by a rename,
deleted, or created where there was none) asks "Save over it?" — Y saves, N or Esc keeps both. **Ctrl+L
(or F5) reloads**: the file is mapped afresh as one more source and becomes a new version, so the
reload reads nothing, the cursor keeps its line and column where the file still has them, the clipboard
stays valid, and an undo goes back to the text before it.

## Pleasantness

No modes: typing types. The keys are the ones every other program uses, and the bottom bar shows them
the way nano does, but as readable words rather than `^X`:

| key | does |
|---|---|
| Ctrl+S | save |
| Ctrl+Q | quit; asks first when there are unsaved changes |
| Ctrl+Z / Ctrl+Y | undo / redo (typing is one step per word, see below) |
| Ctrl+F; Enter or F3, Shift+F3 | find as you type; next, previous |
| Ctrl+T (in the find or replace prompt) | case matters / any case |
| Ctrl+R | replace: Y this one, N skip, A all |
| Ctrl+G | go to line |
| Ctrl+L or F5 | reload the file from disk (undo goes back) |
| Ctrl+X / C / V | cut / copy / paste |
| Ctrl+A | select all |
| Shift + any move | select |
| Ctrl+arrows | move by word |
| Home / End, PgUp / PgDn | line start / end, page up / down |
| Esc | drop the selection and the highlighted matches; stop a search |

Replace is **Ctrl+R, not Ctrl+H**: a terminal sends Ctrl+H as the same byte as Backspace (0x08), and
many send it for the Backspace key itself. The case toggle is **Ctrl+T, not Alt+C**: with the Escape
key reported at once (termbox's `ESC` input mode), Alt+C arrives as Escape and then `c`, so it would
close the prompt.

The mouse works: click to place, drag to select (dragging past the top or bottom scrolls a line at a
time), double-click to select a word, wheel to scroll. **The clipboard is the system's, even over
ssh**, by OSC 52 (`ESC ] 52 ; c ; <base64> BEL`; the terminal sets the local clipboard). A copy over
256 KiB is not sent — terminals refuse or cut one much longer — and the status bar says it stays in
te's own clipboard. Paste is meant to arrive as **bracketed paste**, so a pasted block is one insert
and one undo step and is never auto-indented; te turns it on at start-up and termbox2's `Paste` event
becomes `Input.Paste`. Line numbers are shown softly (dim, not part of the text);
colours are true-colour with a fallback to 256. Syntax highlighting comes last, and highlights only
the visible rows.

**One undo step is one thing the reader did.** A run of typed characters is one step until the
cursor is moved or anything else is done, more than a second passes between keys, or a letter or
digit is typed after a space or punctuation — so each word, with the spaces after it, is one step,
as in most editors. A run of Backspace and Delete is one step by the same rules, less the word rule.
A save ends a run, so undo always reaches the saved text. A replaced selection (a delete and an
insert), a paste and a replace-all are one step each. The core keeps one root per version; grouping
folds the versions of a step into one (`Text.squash`), so a replace-all of a million matches holds
one extra version, not a million.

**The clipboard is a slice of the tree**, not a copy of the bytes: the pieces that held the selection,
shared with the version they came from. The sources only grow, so a slice stays valid across every
later edit and save, and a copy, cut or paste of any size costs O(log n) and reads nothing. Ctrl+A
on a file whose line total is not known yet puts the cursor at the start and the other end of the
selection at the end, so selecting, copying or deleting all of a 4 GB file reads none of it.

**Search is a job, not a call.** It reads the text a window (4 MiB) at a time from the cursor,
wrapping at the end, and the terminal loop runs a window whenever no key is waiting — so the screen
stays live, the status bar shows how far it has got, Esc stops it, and typing into the find prompt
starts it again from where the prompt opened. A key's own search gets one window before the screen is
drawn, so a search of an ordinary file never shows as running. Matching ignores the case of the ASCII
letters by default; every other byte matches only itself. Matches are highlighted on the visible rows
only, each row searched as it is drawn.

## Rendering

Only the visible rows are ever read from the tree: the view asks for the offset of its top line, and
walks pieces from there for one screen. termbox2 keeps the previous frame and sends only the cells that
changed, which is what keeps it fast over a slow ssh link. Each cell is a grapheme cluster with its
display width (0, 1 or 2 columns); a long line scrolls horizontally rather than wrapping in the first
version.

**A line over 32 KiB is laid out a byte to a column.** Laying a line out cell by cell means walking
it from its start to find any screen column, which on a 500 MB minified file or a log with no line
breaks is the whole file per key. A line longer than `long_line` (32 KiB) instead puts each byte in
a column: a character's glyph in its first, blanks after (a tab is one column), its combining marks
on the glyph. The cursor's column is then a subtraction, drawing starts at the first byte in view,
a match is looked for only around the view, and every move and frame costs the same anywhere along
the line; the status bar says `Byte` where it says `Col`, and Ctrl+G's `line:column` takes a byte
there. A word move, a double click and Home's indentation look no further than 32 KiB. What is
still paid once: the first screen of a file whose first line is gigabytes long finds that line's end
by counting to it (2 GB of zeros: 0.8 s).

**The editor never draws to the terminal itself.** `Editor` (`editor.sysl`) holds the cursor (a byte
offset and the line it is on, both kept as it moves, so no offset is ever turned back into a line by
counting), the viewport, and what each key does, and renders into a `Grid` of cells
(`screen.sysl`); `main.sysl` turns termbox's events into `Input` and copies the grid to termbox. So
the tests drive the editor with keys and read the grid, with no terminal. Display widths are
termbox's own `wcwidth`, the table it lays the cells out with, so the cursor and the cells cannot
disagree. Combining marks take no column and move with the character before them; a grid cell holds its
base character and the marks, and they are drawn together with `set_cluster`. A sequence joined by
a zero-width joiner is still laid out character by character, as `wcwidth` counts it.

## Milestones

1. **Core** — the piece tree over the mapped file and the add buffer, undo/redo, byte and line
   queries, save. Tested against a string model over thousands of random edits, and on a 200 MB file.
2. **View** — termbox2 screen, cursor movement, typing and deleting, save, quit prompt.
3. **Editing UI** — selection, clipboard (OSC 52), find / replace with progress and cancel, undo
   grouping, goto line, mouse. Bracketed paste and combining marks are done.
4. **Huge files** — line totals counted in the background, Ctrl+End before the count, byte columns
   on long lines, external-change detection and reload.
5. **Highlighting** — per-language, visible rows only.
