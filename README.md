# Rok docs

The guide, examples and cookbook for [Rok](https://rok-lang.org), in English and Korean. The site at docs.rok-lang.org renders these files; this repository holds only content.

## Branches

- `main`: the next release, served at docs.rok-lang.org/dev/
- `vX.Y.Z`: the docs for that release, served at /X.Y.Z/ (and /X.Y/ for the newest patch of a minor). Fix a mistake in a released version on its branch.

`versions.rokon` on `main` lists the released versions.

## Layout

- `learn/`: the guide, one chapter per file
- `examples/`: complete programs
- `index.md`: the docs home

Every ` ```rok ` block is compiled and run with the matching toolchain before a change goes live.
