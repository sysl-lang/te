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

**Status:** milestones 1 to 4 are done. The view shows line numbers, scrolls sideways for long
lines, draws tabs to a width of 4 and East Asian wide characters in two cells. Typing, Enter (keeping
the line's indentation, and a CRLF file's line breaks), Backspace and Delete, the arrows, Home/End
(Home goes to the indentation first), PgUp/PgDn, Ctrl+Home/End and Ctrl+arrows by word all work, as
do Ctrl+Z / Ctrl+Y (the cursor goes back with the text), Ctrl+S, Ctrl+Q (asks when there are unsaved
changes), Ctrl+G (`line` or `line:column`), clicking and the wheel. Combining marks are drawn on
their base character, and the cursor moves over a character and its marks as one.

Shift with any move selects, as do Ctrl+A, a drag and a double click; typing, Backspace, Delete and
Enter replace the selection. Ctrl+C / Ctrl+X / Ctrl+V copy, cut and paste — the copy also reaches the
system clipboard by OSC 52, over ssh too, up to 256 KiB. Ctrl+F finds as you type (any case; Ctrl+T
makes case matter), Enter and F3 go to the next match, Shift+F3 to the one before, and a long search
over a huge file shows its progress and stops on Esc. Ctrl+R replaces one at a time (Y / N) or all
(A). Typing undoes a word at a time; a paste, a replaced selection and a replace-all undo in one step.

Bracketed paste is on: a terminal paste arrives as one insert, is never auto-indented, and undoes in one step.

Huge files: the first screen shows before anything is counted, and the line total is counted between
keys, so "of N" appears without holding up typing (2 GB in about three seconds here). Ctrl+End before
the count is done goes to the end at once, showing `Ln ≥ N` until the count catches up. A line over
32 KiB (a minified file, a log with no line breaks) is laid out a byte to a column, so moving and
drawing on it cost the same anywhere along it; the status bar says `Byte` in place of `Col` there. A
save over a file that changed on disk since it was opened asks first, and Ctrl+L (or F5) reloads the
file, keeping the cursor's line; an undo goes back to the text before the reload.
