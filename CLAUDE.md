# pintc-docs

Design docs and specs for the Pint language and pintc compiler.

## Branch policy

Always on `main`. No feature branches. `merge.ff = only` is set.

## Changelog structure

Entries go under `## Language Spec` (for spec versions) or `## Compiler` (for
compiler releases). Each spec version heading links to the archived spec file.
Subsections use `####`. New versions start with `In progress.` until finalized.

## Archive

Superseded spec versions live in `archive/`. When a new spec version is finalized,
move the previous version to `archive/` and update `archive/README.md`.
