---
title: "A transparency log for PyPI"
description: "A proposal for auditable package distribution on Python package indexes."
---

![PyPI Transparency logo](/logo.png)

# A transparency log for PyPI

_A proposal for auditable package distribution on Python package indexes._

[Try the test log](https://log.pytransparency.dev) · [Read the draft PEP](https://github.com/trailofbits/pypi-transparency/blob/main/peps/pep-xxxx.rst) · [View on GitHub](https://github.com/trailofbits/pypi-transparency)

## Why?

A transparency log can serve as a source of truth for package metadata such as artifact hashes and publishing identities:

```text
filename: urllib3-2.7.0-py3-none-any.whl
sha256: 9fb4c81ebbb1ce9531cce37674bbc6f1360472bc18ca9a553ede278ef7276897
publisher: https://github.com/urllib3/urllib3
```

Currently, clients that want this metadata get it from PyPI, either by using the API or by directly downloading the artifact. Optionally, they pin it in lockfiles to ensure that future downloads match. However, this is a trust-on-first-use scheme, which leaves open the question: why should the client trust that first artifact?

A compromised index could serve different files for the same artifact to different clients:

```text
Client A requests pkg.whl  ->  Index serves pkg.whl with hash X
Client B requests pkg.whl  ->  Index serves pkg.whl with hash Y
```

In this scenario, lockfiles don't help: Clients A and B will continue to see the files they originally pinned, while the index can continue serving different files to different clients without being detected.

A transparency log provides a way to store claims like "The artifact with filename `pkg.whl` has a hash equal to `X`" such that:

- Every stored claim is discoverable.
- Once stored, claims cannot be modified or deleted without clients noticing (the log is tamper-evident).
- The complete state of the log can be independently attested by trusted third parties.

The combination of these properties means that a client can verify that the package they got from PyPI has not been tampered with by the index, and is consistent with what other clients are seeing.

This solves the problem of what to use as a source of truth for package hashes and publishing identities:

- Maintainers can upload an artifact to PyPI and verify that the corresponding claim added to the log matches their artifact.
  - Since the log is tamper-evident, the log cannot change or delete the claim without detection.
  - Since all the claims in the log are discoverable, maintainers can monitor the log for unexpected claims about their packages and/or identity.
- Clients can download a package and verify that the claim in the log matches the file they downloaded.
  - Since the state of the log can be attested to by trusted third parties, clients can know that they got the same metadata and artifact as other clients.
  - Since the log is tamper-evident, clients can know that they got the same artifact that was first logged.


## FAQ

### Who should operate the log?

The index itself (e.g. PyPI) can host and operate the log. There are a few advantages to this:

- Only the index adds entries to the log. For everyone else, the log is read-only.
- When a file is uploaded to the index, the index can add the corresponding claim to the log and return the inclusion proof to the uploader. The uploader can then verify the inclusion of the claim in the log.
- The index can host the inclusion proofs, which means clients can get everything they need from the index.

### Doesn't this just move trust from PyPI to the log operator?

The key idea of the log is that it offers transparency: any tampering from the log operator can be trivially detected. This means the clients don't trust the log itself, but rather the fact that they can detect if the log is misbehaving.

Since the log operator should be the index itself, having a transparency log just means that a misbehaving index can be detected, not that the index should be trusted.

### How does this protect against a compromised PyPI account/token uploading a malicious package?

It doesn't, a compromised account will still be able to upload malicious artifacts to PyPI, and these artifacts will be added to the log. Clients that download these artifacts won't see any problems, since they can only check if an artifact is in the log or not.

The transparency log does not protect against this type of supply chain attack, where the compromise happens before the artifact reaches the index.

However, it does make it easy for maintainers of the package to monitor the log for new entries with claims that mention the package and/or their identities. Since this monitoring can be done automatically with very little latency, maintainers can be notified of unexpected entries in the log shortly after they happen, without relying on 3rd parties noticing the compromise and notifying them.

### What prevents someone from adding "fake" entries to the log?

Specifically, we say fake to mean any of:
- The entry refers to a file that does not currently exist in the index.
- The entry refers to a file that exists in the index (the index serves a file with the same filename), but the hash (or other associated metadata) in the entry does not match the actual artifact.
- The entry refers to a file with filename `A`, but there already is an existing entry with filename `A`.

From the configuration side, the log should be set up so that only the index can add entries. However, assuming a compromised/malicious index, there is nothing
stopping the index from adding "fake" entries, the log just makes these entries discoverable.

In practical terms, this type of misbehavior should be detected by monitors of the log. A monitor can, for every entry, check that the corresponding artifact served by PyPI exists and matches the entry's contents.

As a concrete example, let's assume a compromised index logs an entry for `pkg 2.0.0`, where `pkg` has not released a version `2.0.0` yet. This could be in preparation to serve a malicious artifact once the `pkg` maintainers release version `2.0.0`.

The fake entry could be detected in the following ways:
- Monitors that check new entries in the log match existing artifacts served by the index.
- The maintainer of `pkg` monitoring new entries for claims about `pkg`, receiving a notification when the fake entry is added.
- The maintainer of `pkg` when uploading version `2.0.0` and verifying if the inclusion proof returned by the index is valid.
  - The index could still return a valid inclusion proof by logging the new `pkg` artifact in the log, but this results in duplicate entries (their claims refer to the same filename), which are trivially detectable by monitors.

### Who will monitor the log?

Maintainers are incentivized to monitor the log for claims related to the packages they maintain and their identities. However, not all maintainers will do this, and many of the security properties the log offers depend on having monitors that regularly check the log for tampering and inconsistencies like:

- Entries being added without a corresponding artifact being available on the index
- Duplicate entries (with claims referring to the same filename)

Luckily, there are already third parties with incentives to perform this kind of monitoring: security companies, in particular those who offer products and services related to supply chain security. These companies already monitor the index for suspicious packages, and monitoring the transparency log would be an easy and cheap addition to that.

In addition, running a monitor requires almost no mainteinance and requires very little in terms of compute, so volunteers and maintainers can easily run their own monitors for the ecosystem.


### How much does it cost to run a log?

A log with 22 million entries (following the format from the draft PEP) using Ed25519 as the signing algorithm requires around:
- 4 GB of space for the log
- 40 GB of space for the inclusion proofs of the 22 million entries
- The bandwidth required to serve one inclusion proof (~1.5 KB) everytime someone downloads an artifact from the index.

See the [Design considerations](https://github.com/trailofbits/pypi-transparency/blob/main/peps/pep-xxxx.rst#design-considerations) section of the PEP for more estimates for different log sizes.

