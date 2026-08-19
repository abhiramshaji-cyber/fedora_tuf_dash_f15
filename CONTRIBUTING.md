# Contributing

Corrections are the most valuable contribution here. A wrong command in a
partitioning guide can cost somebody their data, so accuracy beats completeness.

## The bar for a change

Every command in this guide is meant to be verified, not guessed. If you change
or add one, say how you checked it:

- the upstream document you read, with a link, or
- the machine, model year, and Fedora version you ran it on.

Other things worth knowing:

- If something is model year dependent, give the check command rather than a
  value. The FX516 and FX517 series differ, and guessing for the reader is how the
  earlier draft of this guide went wrong.
- Keep the ordering intact unless you explain why it is safe to change. Several
  steps only work because of what runs before them.
- Do not reintroduce advice to disable Secure Boot. Phase 5 solves the unsigned
  NVIDIA module properly by enrolling a key.

## What is in scope

This guide targets one laptop family. Notes for closely related ASUS models are
welcome as clearly marked asides. A rewrite covering every laptop would help
nobody, so please open an issue before widening the scope.

Issues labelled `good first issue` are self contained and a good place to start.
