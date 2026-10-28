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
- The status bar shows `?` for the line total until the counts are known; milestone 4 fills them in
  the background, in idle-time slices on the UI thread, so the tree is never shared across threads.

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
another program under a live mapping is outside what any mmap editor can protect against; detecting
it by mtime/size on focus is a milestone 4 item.)

## Pleasantness

No modes: typing types. The keys are the ones every other program uses, and the bottom bar shows them
the way nano does, but as readable words rather than `^X`:

| key | does |
|---|---|
| Ctrl+S | save |
| Ctrl+Q | quit; asks first when there are unsaved changes |
| Ctrl+Z / Ctrl+Y | undo / redo (typing coalesces into one step per word or pause) |
| Ctrl+F, F3 | find, find next |
| Ctrl+H | replace |
| Ctrl+G | go to line |
| Ctrl+X / C / V | cut / copy / paste |
| Shift+arrows | select |
| Ctrl+arrows | move by word |
| Home / End, PgUp / PgDn | line start / end, page up / down |

The mouse works: click to place, drag to select, wheel to scroll. **The clipboard is the system's,
even over ssh**, by OSC 52 (the terminal sets the local clipboard). Paste arrives as **bracketed
paste**, so a pasted block is one insert and one undo step and is never auto-indented. Line numbers are
shown softly (dim, not part of the text); colours are true-colour with a fallback to 256. Syntax
highlighting comes last, and highlights only the visible rows.

## Rendering

Only the visible rows are ever read from the tree: the view asks for the offset of its top line, and
walks pieces from there for one screen. termbox2 keeps the previous frame and sends only the cells that
changed, which is what keeps it fast over a slow ssh link. Each cell is a grapheme cluster with its
display width (0, 1 or 2 columns); a long line scrolls horizontally rather than wrapping in the first
version.

**The editor never draws to the terminal itself.** `Editor` (`editor.sysl`) holds the cursor (a byte
offset and the line it is on, both kept as it moves, so no offset is ever turned back into a line by
counting), the viewport, and what each key does, and renders into a `Grid` of cells
(`screen.sysl`); `main.sysl` turns termbox's events into `Input` and copies the grid to termbox. So
the tests drive the editor with keys and read the grid, with no terminal. Display widths are
termbox's own `wcwidth`, the table it lays the cells out with, so the cursor and the cells cannot
disagree. Combining marks take no column and move with the character before them, but are not drawn
yet: the termbox2 package is built without `TB_OPT_EGC`, so a cell cannot hold a cluster.

## Milestones

1. **Core** — the piece tree over the mapped file and the add buffer, undo/redo, byte and line
   queries, save. Tested against a string model over thousands of random edits, and on a 200 MB file.
2. **View** — termbox2 screen, cursor movement, typing and deleting, save, quit prompt.
3. **Editing UI** — selection, clipboard (OSC 52), find / replace, undo grouping, goto line, mouse.
4. **Huge files** — line totals counted in the background, progress and cancel on a long search,
   external-change detection.
5. **Highlighting** — per-language, visible rows only.
