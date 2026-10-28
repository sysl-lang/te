# te

A terminal text editor in [sysl](https://github.com/sysl-lang/sysl), meant to be more pleasant than
nano: no modes, the keys every other program uses (Ctrl+S, Ctrl+Z, Ctrl+F, Ctrl+C/V, Shift+arrows),
the mouse, and the system clipboard even over ssh. It opens files of any size instantly — the file is
mapped, never loaded, and the text is a piece table held as a balanced persistent tree, so every edit
and every line query is O(log n) and undo is going back to an earlier version.
[docs/design.md](docs/design.md) has the design.

```
sysl build .
./te notes.txt
sysl test .
```

`te` with no file is an unnamed buffer that asks for a name on Ctrl+S; a file that does not exist yet
is created on the first save.

**Status:** milestones 1 and 2 are done. The view shows line numbers, scrolls sideways for long
lines, draws tabs to a width of 4 and East Asian wide characters in two cells. Typing, Enter (keeping
the line's indentation, and a CRLF file's line breaks), Backspace and Delete, the arrows, Home/End
(Home goes to the indentation first), PgUp/PgDn, Ctrl+Home/End and Ctrl+arrows by word all work, as
do Ctrl+Z / Ctrl+Y (the cursor goes back with the text), Ctrl+S, Ctrl+Q (asks when there are unsaved
changes), Ctrl+G (`line` or `line:column`), clicking and the wheel. Combining marks are not drawn on
their base character yet (the termbox2 package is built without grapheme-cluster cells); the cursor
already moves over a character and its marks as one. Selection, the clipboard, find and undo grouping
come next.
