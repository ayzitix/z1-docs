# Contributing

Two ways to be useful: report what is wrong, or change it.

## Report

Open an issue with what you did, what you expected, and what happened instead.
For anything visual, a screenshot beats a description. For a build failure, paste
the command and its output.

## Change

1. Work on a branch. `main` is what runs.
2. Keep the change to one area. The editor, the core and the theme are three
   repositories; a change that needs all three is three changes.
3. Run the checks for that area (`docs/commands.md`).
4. For anything visual, restart the editor and look at the element. A passing
   check does not prove the pixels.

## Documentation images

The palette cards in the theme repository are generated:

```bash
python3 script/make-images.py
```

Do not replace them with screenshots. A screenshot records one moment on one
machine; a generated card cannot disagree with the theme file.

## Commits

```
area: summary
```

`area` is `editor`, `core`, `theme` or `docs`. The summary is lowercase and
describes what changed, not which files were touched.

## Licensing of contributions

By contributing you agree your work is released under the MIT license of the
repository you are contributing to. z1 is derived from VS Code, VSCodium and
opencode; their notices stay in place.
