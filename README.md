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

**Status:** the text core is done and tested (milestone 1): the mapped file, the add buffer, the
piece tree, undo and redo, line and column queries, and saving. There is no editing view yet, so `te`
only reports a file's size and line count; the view is next.
