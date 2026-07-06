---
title: "A transparency log for PyPI"
description: "A proposal for auditable package distribution on Python package indexes."
---

# A transparency log for PyPI

_A proposal for auditable package distribution on Python package indexes._

[Read the draft PEP](https://github.com/trailofbits/pypi-transparency/blob/main/peps/pep-xxxx.rst) · [View on GitHub](https://github.com/trailofbits/pypi-transparency)

## Why?

A transparency log can serve as a source of truth for package metadata such as artifact hashes and publishing identity.

For example, a log entry may bind a distribution file to metadata:

```text
pkg-1.0.0-1.whl
sha256: ...
publisher: github.com/.../...
```

Currently this is fetched once from PyPI and pinned in lockfiles. A lockfile protects against the index serving a different file in the future, but it does not protect against the index consistently serving different files for the same artifact to different clients.

```text
Client A requests pkg.whl  ->  Index serves H1
Client B requests pkg.whl  ->  Index serves H2
```

A transparency log provides a way to store claims like `pkg.whl -> hash` such that:

- Every claim stored is discoverable.
- Once stored, claims cannot be modified or deleted without clients noticing.
- The complete state of the log can be attested to by trusted third parties, guaranteeing that they all get the same view of the log.

The combination of these properties means that a client can verify that the package metadata they got from PyPI has not been tampered with, and is the same as what other clients are seeing.

## What this solves

This solves the problem of what to use as a source of truth for package hashes and publishing identities:

- Maintainers can upload an artifact to PyPI and verify that the corresponding claim added to the log is correct.
- Since the log is tamper-evident, clients that verify that the claim in the log matches their downloaded artifact can know that they got the same artifact as the one that was first logged.
- Since the state of the log can be attested to by trusted third parties, clients can know that they got the same metadata and artifact as other clients.
- Since all the claims in the log are discoverable, maintainers can monitor the log for unexpected claims about their packages and/or identity.

## How does it work?

The draft proposal defines:

- a transparency log entry format for distribution metadata;
- discovery mechanisms through package index APIs;
- an API for retrieving inclusion proofs and signed checkpoints; and
- a client verification procedure for confirming that a distribution file was included in the transparency log.
