PEP: XXXX
Title: Index support for auditable package distribution
Author: TODO <TODO@TODO.com>
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 01-Jan-2026


Abstract
========

This PEP proposes a series of changes to strengthen trust and transparency
in artifacts distributed by a Python package index, such as PyPI.

More specifically, this PEP proposes:

* A format for transparency log entries that record distribution metadata.
* Backwards-compatible updates to the HTML and JSON "Simple" APIs,
  allowing clients to discover transparency endpoints for individual
  distribution files.
* A new transparency API for retrieving inclusion proofs and signed
  checkpoints.
* A standardized verification procedure that clients should follow to confirm
  a distribution file was included in the transparency log.


Concepts and Terminology
========================

This section defines the core concepts used throughout this PEP. Readers
familiar with transparency log systems (such as Certificate Transparency or
Go's checksum database) may skip this section.

Transparency Log
----------------

A transparency log is an append-only data structure that records entries in a
way that makes tampering detectable. It provides three key properties:

- **Append-only:** New entries can be added, but existing entries cannot be
  modified or deleted without detection.
- **Auditable:** Clients can verify the log's contents and consistency.
- **Efficient verification:** Clients can verify that a specific entry exists
  without downloading the entire log.

Log Entry
---------

A log entry is a single record in the transparency log. For the transparency log
described in this PEP, each entry contains metadata about a distribution file:
the checksum, filename, and a :pep:`740` attestation identity (when available).

Checkpoint
----------

A checkpoint is a signed snapshot of the log's current state. It contains the
log's identity, the number of entries (tree size), and a root hash that
cryptographically commits to the entire log contents. 

Inclusion Proof
---------------

An inclusion proof is a compact cryptographic proof that a specific entry exists
in the log at a given checkpoint.  For a log with millions of entries, the
amount of data necessary to prove a given entry exists in the log only requires
a few hundred bytes.

Monitor
-------

A monitor is a service that watches the transparency log and cross-references
it against other sources to detect anomalies. Monitors verify log contents—for
example, detecting packages served without corresponding log entries or
notifying maintainers of new releases of their software.


Rationale and Motivation
========================

The Python packaging ecosystem has made significant security improvements in
recent years. :pep:`458` protects against compromised mirrors and CDNs,
:pep:`740` provides verifiable build provenance. However, these
mechanisms share a common assumption: that PyPI itself is honest and not
compromised. :pep:`480`, which extends :pep:`458` provides protection against
a compromised index, but only for packages that opt-in to sign their artifacts.
Furthermore, both :pep:`458` and :pep:`480` are not currently implemented.

Today, PyPI operates as a trusted but unauditable intermediary. There is no
public record of what packages were available at any given time, no mechanism
to detect if different users were served different content, and no way to
verify after the fact what PyPI actually served. A sophisticated attacker who
compromises PyPI's infrastructure could serve malicious packages to targeted
users while showing legitimate packages to everyone else, then remove all
evidence of the attack.

A concrete example of this problem is when pinning dependencies by hash,
which is done by hashing the artifacts once and storing the digests into a
file. This is a trust-on-first-use scheme, where we trust that the artifacts
downloaded and hashed during pinning are the correct ones, and future installs
can verify the artifacts downloaded match the original ones. There is no way
of verifying, during that first download, if the index has at any point in time
served different artifacts for the same distribution filename. What is missing
is an auditable, canonical source of truth for these hashes (and other package
metadata).

This PEP addresses this gap by specifying how an index can maintain a
transparency log — a public, append-only, cryptographically verifiable ledger
of every distribution file it serves. Transparency logs are already widely and
successfully used in web browsers through `Certificate Transparency
<https://en.wikipedia.org/wiki/Certificate_Transparency>`_ (`see list of CT logs
<https://certificate.transparency.dev/logs/>`_), where they solve a roughly
analogous problem: ensuring that certificate authorities cannot issue
fraudulent certificates without detection. Similarly, `Key Transparency
<https://en.wikipedia.org/wiki/Key_Transparency>`_ uses the same approach in
end-to-end encrypted messaging systems like Signal (`Signal's Key Transparency
server <https://github.com/signalapp/key-transparency-server>`_), ensuring that
public keys served to users are publicly auditable and cannot be silently
replaced to intercept communications.

A more relevant example of successful at-scale use of transparency logs is `Go's
checksum database
<https://go.dev/blog/module-mirror-launch#checksum-database>`_, which has secured module downloads since 2019 by serving as a
source of truth for the checksums of all the Go modules that are fetched through
the Go proxy. The go client downloads modules and verifies them against the checksum
logged in the transparency log, with the guarantee that any tampering with the
checksum database is cryptographically detectable by third parties monitoring
the log (`Supply chain security for Go
<https://security.googleblog.com/2023/06/supply-chain-security-for-go-part-2.html>`_).

With a transparency log in place:

- Every distribution served has a corresponding public log entry, eliminating
  undetectable serving of packages. Indices, even if compromised, can't serve
  a distribution to a client without publicly logging the distribution file in the
  transparency log.
- Clients can cryptographically verify that an artifact is in the log, ensuring
  they have the same artifact that was logged during upload, and that the
  artifact is auditable by monitors.
- The append-only property makes evidence destruction detectable: entries cannot
  be modified or deleted without detection.
- Third parties can implement monitors that cross-reference log contents
  against packages actually served, detecting discrepancies and potential
  compromises.
- Maintainers can monitor the log for new entries that refer to the packages
  they maintain, allowing them to detect unexpected artifacts being uploaded
  and/or added to the log. This reduces the time to notice a compromise.



Design Considerations
=====================

- This PEP focuses on public indices. While transparency logs can be added to
  private indices, the security properties are different: for example, this
  PEP assumes that anyone can run a log monitor (since the index and log are
  public), which is not the case for private deployments.
- This PEP considers the proposed Upload 2.0 API (:pep:`694`) since it also
  changes the upload behavior for indices and clients. Under the new upload API,
  indices must log the distribution files only during a Publishing Session (and
  not during an Upload Session). This is because files uploaded during an Upload
  Session can be replaced and/or deleted. Since the log must maintain only one
  entry per distribution file, entries must be logged only for files that will
  not be changed anymore.
- Indices implementing this PEP must store the inclusion proof of each
  distribution file they host. This means the storage requirements depend on
  the number of distribution files hosted by the index, and the cryptographic
  algorithm used to sign the inclusion proofs. Ed25519 signatures are ~X bytes,
  whereas ML-KEM signatures are ~Y bytes. For reference, as of April 2026, PyPI
  hosts around 21 million distribution files. This means PyPI would need an
  approximately A-B GB of extra storage, depending on the chosen cryptographic
  algorithm. Assuming ~N new packages per year, this would mean an ~Z GB increase
  per year.


Specification
=============

Log-Side Specification
----------------------

Log public key
~~~~~~~~~~~~~~

The Ed25519 public key of the transparency log MUST be available at the
following well-known URL: ``$LOG_URL/.well-known/bt/tlog-key``.

Index-Side Specification
------------------------

Log discovery
~~~~~~~~~~~~~

The information about the log for this index MUST be available at the following
well-known URL: ``$INDEX_URL/.well-known/bt/tlog-info`` in the following schema:
following scheme:

.. code-block:: json

    {
      "log_url": "<log_url>",
      "log_key": "<log_public_key>"
    }

Fields:

- **log_url**: The URL of the transparency log for the index.
- **log_key**: The publick key of the transparency log at ``log_url``.


Package Upload Behavior
~~~~~~~~~~~~~~~~~~~~~~~

When using the legacy upload API, the index MUST insert an entry for a
distribution file into the transparency log upon upload, ensuring that every
distribution served by the index is logged. The distribution should not be made
available for download until this log entry is created.

Furthermore, when a client uploads a distribution file, the index MUST wait
until the entry is successfully recorded in the transparency log before
confirming a successful upload back to the client.

When using the Upload 2.0 API, the above applies only during the Publishing
Session. During the Upload Session, the index MUST NOT insert an entry for a
distribution file in the log, since these files are subject to change until the
publishing step is finished.


Log Entry Format
~~~~~~~~~~~~~~~~

Each log entry represents metadata about a distribution, such as the filename,
the checksum, optionally the publisher, etc. The ``checksum`` and ``filename``
properties are mandatory, while ``publisher`` is optional. The schema is as
follows:

.. code-block:: json

    {
      "checksum": "sha256:<lowercase hex digest>",
      "filename": "<distribution filename>",
      "publisher": {
          "kind": "<publisher_kind>",
          "subject": "<publisher_SAN>"
      }
    }

Fields:

- **checksum**: The SHA-256 hash of the distribution file, prefixed with
  ``sha256:``.
- **filename**: The distribution filename (e.g.,
  ``urllib3-2.6.3-py3-none-any.whl``).
- **publisher**: (Optional) The publisher information, if the distribution was
  uploaded alongside a :pep:`740` publish attestation. If the upload did not
  include a publish attestation, this field is omitted. See Appendix A for
  examples.

  - **kind:** The type of publisher (e.g, ``google``, ``gitlab``, ``github``,
    etc). Defined in Appendix A.
  - **subject:** the Subject Alternative Name (SAN) of the certificate used to
    sign the artifact's publish attestation

Serialization format
^^^^^^^^^^^^^^^^^^^^

When inserting data in the transparency log, it is serialized to a text format
that is line based, where each line is terminated by ASCII newline, and the
items on each line are separated by a single ASCII space. The format is inspired
by the C2SP tlog formats, and is defined as follows:

- The entry is a sequence of lines separated by a single newline (U+000A).
- The first line MUST be the string ``pypi-transparency/v1``.
- The second line MUST start with the string ``sha256:`` followed by the
  SHA256 digest of the distribution in hexadecimal encoding, all lowercase.
- The third line MUST be the filename of the distribution.
- The fourth line MAY start with the string ``publisher``, followed by a
  space (U+0020), followed by the publisher kind, followed by a space (U+0020)
  and then followed by the Subject Alternative Name (SAN) of the certificate
  used to sign the artifact's publish attestation.

For example::

    pypi-transparency/v1
    sha256:<lowercase hex digest>
    <distribution filename>
    publisher <publisher-kind> <publisher-SAN>

Clients verifying inclusion proofs must be aware of this serialization scheme,
as they need to canonicalize the entry data themselves to recompute the leaf
hash during verification.


Example Log Entry
^^^^^^^^^^^^^^^^^

Example metadata with its text-based representation as included in the
transparency log:

.. code-block:: json

    {
      "checksum": "sha256:bf272323e553dfb2e87d9bfd225ca7b0f467b919d7bbd355436d3fd37cb0acd4",
      "filename": "urllib3-2.6.3-py3-none-any.whl",
      "publisher": {
        "kind": "GitHub",
        "subject": "https://github.com/urllib3/urllib3/.github/workflows/publish.yml@refs/tags/2.6.3"
      }
    }

::

    pypi-transparency/v1
    sha256:bf272323e553dfb2e87d9bfd225ca7b0f467b919d7bbd355436d3fd37cb0acd4
    urllib3-2.6.3-py3-none-any.whl
    publisher github https://github.com/urllib3/urllib3/.github/workflows/publish.yml@refs/tags/2.6.3

Checkpoint Format
~~~~~~~~~~~~~~~~~

The log state is captured in a checkpoint following the C2SP `tlog-checkpoint
<https://c2sp.org/tlog-checkpoint>`_ specification:

::

    bt-log.pypi.org
    1234567
    SssviGxgLpKjoCiBhzOvGwrP+okDxwCv/pF8MnHxKa8=

    — bt-log.pypi.org Az3grlgtzOB6MwPjQh...
    — witness.example.org/1 Abcd1234...
    — witness.example.org/2 Efgh5678...

The checkpoint consists of:

- **Line 1:** Log origin identifier (e.g., ``bt-log.pypi.org``)
- **Line 2:** Tree size (number of entries in the log)
- **Line 3:** Root hash (Base64-encoded SHA-256 Merkle root)
- **Blank line:** Separator between body and signatures
- **Remaining lines:** Ed25519 signatures from the log and witnesses, each
  prefixed with ``—``. For this PEP, only the first signature (from the log)
  will be present. Witnesses (described in Appendix D) are not a part of this
  PEP, but might be added in the future.

The signed portion is lines 1–3.
Signature lines follow the same C2SP checkpoint format: each line
begins with ``-`` followed by a key name (which MUST NOT contain spaces), a
space, and the Base64-encoded signature.


Inclusion proof format
~~~~~~~~~~~~~~~~~~~~~~

The inclusion proof served by the index follows the C2SP tlog-proof specification:

::

    c2sp.org/tlog-proof@v1
    extra YWdlLXYxLjIuMS1kYXJ3aW4tYXJtNjQudGFyLmd6kYXJ3aW4tYXJ
    index 73894
    gSKyXoYZUgZ6jduWYrkDOARinOMGJveXjgMkBTcdPlQ=
    B95lDa8R83lS8n0eG+o0buTxRKQTYFi//1U8anccXmA=
    EKNzoDWG8LGC0Yp9o+sv3qllpMP9uHQ9B20KNL+Q1zs=
    RoopEkOdqkYqMB4MJXrbt/hMjOxsVn0IrWjpz1ZMMes=
    AHCioX9nLjsrse6YhjRRmk1WUEirVOLLRoOQ6vfO5vk=

    example.com/fancylog
    109482
    sFodV/vSp5O8n9a8QpW6PRY97tfOSW5bsc2Xl/EQi08=

    — example.com/fancylog hI2DJw[...]1roloI=
    — witness1.example mJirIklj[...]qY9v2B/5bg==
    — witness2.example TnKKVHLX[...]xwYwrSjgow==
    — witness3.example S4X82uH5[...]3oEcROGLFQ==

See `C2SP tlog-proof <https://c2sp.org/tlog-proof>`_
for the full specification.

Simple API Extension
~~~~~~~~~~~~~~~~~~~~

The Simple API is extended to include a reference to the transparency endpoint
for each distribution.

HTML Format (:pep:`503`)
^^^^^^^^^^^^^^^^^^^^^^^^

A ``data-inclusion-proof`` attribute is added to distribution links:

.. code-block:: html

    <a href="https://files.pythonhosted.org/.../urllib3-2.6.3-py3-none-any.whl#sha256=bf2723..."
       data-dist-info-metadata="sha256=abc123..."
       data-provenance="https://...provenance"
       data-inclusion-proof="https://pypi.org/transparency/urllib3/2.6.3/urllib3-2.6.3-py3-none-any.whl/proof">
       urllib3-2.6.3-py3-none-any.whl
    </a>

JSON Format (:pep:`691`)
^^^^^^^^^^^^^^^^^^^^^^^^

An ``inclusion-proof`` field is added to file objects:

.. code-block:: json

    {
      "filename": "urllib3-2.6.3-py3-none-any.whl",
      "url": "https://files.pythonhosted.org/.../urllib3-2.6.3-py3-none-any.whl",
      "hashes": {"sha256": "bf2723..."},
      "provenance": "https://...provenance",
      "inclusion-proof": "https://pypi.org/transparency/urllib3/2.6.3/urllib3-2.6.3-py3-none-any.whl/proof"
    }

The ``data-inclusion-proof`` attribute (HTML) and ``inclusion-proof`` field
(JSON) MUST be present if the index supports this PEP. If present, the value
MUST be an absolute URL to the transparency endpoint for that distribution.

Indices that do not support this PEP MUST NOT include these fields.

Transparency Endpoint
~~~~~~~~~~~~~~~~~~~~~

The transparency endpoint provides inclusion proofs for distributions.

URL Structure
^^^^^^^^^^^^^

Following the pattern established by :pep:`740`'s Integrity API:

::

    GET /transparency/{project}/{version}/{filename}/proof

Where:

- **project**: Normalized project name (e.g., ``urllib3``)
- **version**: Normalized version (e.g., ``2.6.3``)
- **filename**: Distribution filename (e.g., ``urllib3-2.6.3-py3-none-any.whl``)

Response Format
^^^^^^^^^^^^^^^

The endpoint returns a JSON object:

.. code-block:: json

    {
      "version": 1,
      "log_origin": "bt-log.pypi.org",
      "entry_index": 12345,
      "entry": {
        "checksum": "sha256:bf272323e553dfb2e87d9bfd225ca7b0f467b919d7bbd355436d3fd37cb0acd4",
        "filename": "urllib3-2.6.3-py3-none-any.whl",
        "publisher": {
          "kind": "github",
          "subject": "https://github.com/urllib3/urllib3/.github/workflows/publish.yml@refs/tags/2.6.3"
        }
      },
      "checkpoint": "<base64-encoded signed checkpoint>",
      "inclusion_proof": "<base64-encoded proof object>"
    }

Fields:

- **version**: Schema version (currently ``1``)
- **log_origin**: Identifier for the transparency log
- **entry_index**: Position of this entry in the log (zero-indexed)
- **entry**: The log entry object (same format as stored in the log)
- **checkpoint**: Base64-encoded signed checkpoint
- **inclusion_proof**: The inclusion proof object, encoded in base64

Response Codes
^^^^^^^^^^^^^^

- **200 OK**: Success, returns inclusion proof JSON
- **404 Not Found**: Distribution not found, or index does not support this
  PEP.

Client-Side Specification
-------------------------

Client Configuration
~~~~~~~~~~~~~~~~~~~~

Clients implementing inclusion proof verification MUST be able to retrieve:

* ``log_public_key``: The Ed25519 public key of the transparency log, MUST be
  retrieved from the well-known URL specified in the Index-side specification
  (e.g. ``pypi.org/.well-known/bt/tlog-info``).

Clients SHOULD cache the public key for future downloads of that index. Since
keys are not expected to change unless the log is compromised or there is a
catastrophic failure that requires re-building the log, clients SHOULD NOT
try to refresh the key unless there is a verification failure. More
specifically:

- Clients downloading from an index for the first time retrieve the log's key
  from the well-know endpoint and cache it for future downloads from that index.
- Clients always try first to verify using the cached key, without refreshing.
- In case of verification failure, clients can try to refresh the key using
  the well-known endpoint.

  - If there is no new key, installation should be blocked, since the index
    served an inclusion proof not signed by the current log's key.
  - If there is a new key, clients should retry verification but should
    warn the user that the public key has changed and direct the user to check
    index announcements (e.g. the PyPI blog) to verify that the key change is
    legitimate.


Client Behavior
~~~~~~~~~~~~~~~

When downloading a distribution from an index supporting this PEP, clients MUST
verify that the distribution was included in the index's transparency log by
following the algorithm specified in the next section. If verification fails,
clients MUST NOT install the distribution, and MUST return an error warning
the user about the failure.

Similarly, when uploading a distribution to an index supporting this PEP,
using the legacy upload API, clients SHOULD verify the distribution against the
inclusion proof returned by the index after a succesful upload. If verification
fails, clients MUST return an error warning the user about the failure.

When using the proposed Upload 2.0 API, since inclusion proofs will only be
returned during the publishing step, clients publishing distributions SHOULD
verify them against the inclusion proof returned by the index if they have
access to the original distributions' metadata. This is due to the fact that
the client publishing an artifact might be different from the client publishing
it, so the latter might not have access to the ground truth information about
the original uploaded artifact.


Verification Algorithm
~~~~~~~~~~~~~~~~~~~~~~

Clients MUST perform the following steps to verify a distribution:

Step 1: Compute Distribution Checksum
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    checksum = sha256(distribution_file).hexdigest()

Step 2: Verify Entry Matches Distribution
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    entry = proof["entry"]
    assert entry["filename"] == distribution_file
    assert entry["checksum"] == f"sha256:{checksum}"

Note: This step is important since, when verifying an entry is in the log,
the client must make sure the entry matches the ground truth they're trying
to verify (in this case, the downloaded distribution). This is done here by
comparing the entry returned in the response of the transparency endpoint
against the downloaded artifact's filename and checksum.

The extra metadata in the entry (i.e: the publisher identity) is not checked
against a ground truth because it's unnecessary: since each distribution has a
unique entry in the log, if the entry `(filename_A, checksum_A, publisher_A)`
is successfully verified then there is no other `(filename_A, checksum_A,
publisher_B)` entry in the log.

Step 3: Verify Checkpoint Signatures
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from pypi_transparency import verify_checkpoint, InclusionProof

    log_public_key = # retrieve from well-known URL

    checkpoint = base64_decode(proof["checkpoint"])
    verify_checkpoint(checkpoint, log_public_key)

Step 4: Verify Inclusion Proof
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from pypi_transparency import verify_inclusion, Entry

    log_public_key = # retrieve from well-known URL

    entry = Entry(data=proof["entry"])
    index = proof["entry_index"]
    checkpoint = proof["checkpoint"]

    verify_inclusion(e, index, proof["inclusion_proof"], checkpoint)

If all steps pass, the distribution and its hash are present in the
transparency log.


Backwards Compatibility
=======================

This PEP is fully backwards compatible with existing clients and indices.

API Compatibility
-----------------

The changes to the Simple API are additive and optional:

- The ``data-inclusion-proof`` attribute (HTML) and ``inclusion-proof`` field
  (JSON) are optional for indices. Indices that do not support binary
  transparency simply omit these fields. Indices that support binary
  transparency MUST include these fields.
- Existing clients that do not implement this PEP will ignore the new
  attribute and field, continuing to function as before.

Clients configured to require inclusion proofs from an index will not be able
to download packages if the index does not provide their inclusion proofs.
This is intentional—such clients are explicitly opting into stricter security
guarantees.

No Transition Period
--------------------

Clients implementing this PEP MUST require inclusion proofs for all packages.
A missing proof indicates one of:

- A misconfigured index that claims to support binary transparency but failed
  to log an entry
- A potential attack where a package is being served without being logged

In either case, the client MUST refuse to install the package. Indices that
implement this PEP are expected to do a one-time population of the log to
backfill existing distributions (see Appendix B), which means there is no valid
situation where an artifact being served by an index supporting this PEP doesn’t
have an inclusion proof.


Index Adoption
--------------

Public indices other than PyPI may adopt this specification at their discretion.

Clients MUST NOT rely on the presence or absence of ``data-inclusion-proof``
or ``inclusion-proof`` fields to determine whether an index supports binary
transparency, since a malicious index could omit these fields to bypass
verification.

Instead, clients MUST maintain an out-of-band list of indices that are
expected to provide inclusion proofs. This list may be hardcoded (e.g., PyPI
is included by default) or user-configured. For any index on this list, the
client MUST require a valid inclusion proof and refuse to install packages
without one.

Security Implications
=====================

Security Model
--------------

Trust Assumptions
~~~~~~~~~~~~~~~~~

For the log specified in this PEP to provide its security guarantees:

- **There are third party monitors continuously monitoring the log:**
  monitors should check the log with regards to, at least, a unique mapping
  between distribution filenames and hashes, to ensure the same distribution
  file is never duplicated in the log.
- **Cryptographic primitives:** SHA-256 collision resistance and Ed25519
  signature security.
- **The log does not perform a split-view attack:** the log stores a single
  log state, which is served to all clients.

The last item can be addressed by adding support for `witnesses
<https://c2sp.org/tlog-witness>`, which are third parties that cosign the log's
checkpoint, attesting that they all see the same log state. See Appendix D for a
detailed description. Witness support should be considered as a future improvement
to the log specified in this document.

This PEP guarantees:

* Detection of artifact tampering by the index or mirrors: If client
  verification succeeds, then the downloaded artifact is the same as the one the
  index added to the transparency log. If a log was compromised and added duplicated
  entries for existing artifacts, this can be detected by monitors.
* Packages upload events are auditable and tamper-evident, so compromised
  indices cannot deny having provided distributions that were logged in the
  binary transparency log.

Role of Monitors
----------------

Monitors are services that watch the transparency log and detect anomalies.

Monitors can check distribution uniqueness in the log, ensuring that there are not
multiple entries in the log for the same distribution.

In addition, maintainers can use monitors to get notifications when new versions
of their packages are logged, or when their identities are used to upload a
package, helping them detect unauthorized uploads.

The first function, verifying the uniqueness of the artifacts in the log
entries, is important because the utility of the log as an immutable source of
truth is only valid if there are no duplicate entries.

Monitors provide an additional layer of detection that benefits the ecosystem as
a whole. Monitors can be operated by security organizations, package maintainers
watching their own packages, or community volunteers.

Attack Scenarios
----------------

This PEP addresses the following classes of attacks:

**Serving artifacts different from the ones logged:** If the index serves a
package that doesn't have a corresponding entry in the log, client
verification fails immediately (no valid inclusion proof).

**Evidence destruction:** If an attacker compromises the index and later
attempts to delete evidence, the append-only property makes this detectable.
Any modification to the log changes the root hash, which is immediately visible
to anyone comparing checkpoints. Furthermore monitors can maintain independent
copies of the latest seen checkpoint.

Limitations
-----------

The transparency log specified in this PEP does not protect against all threats:

- **Malicious packages:** A package can be both present in the transparency
  log and contain malicious code. Binary transparency records what was
  uploaded, not whether its contents are safe.
- **Immediate prevention:** Upload events are auditable, but not blocked in
  real-time. A malicious package may be installed before monitors detect
  anomalies. Compromised indices might manipulate the log (e.g. add duplicated
  entries) and that will only be detected by monitors, not by clients.
- **Freeze/replay attacks:** A malicious index can serve outdated metadata to
  clients requesting the latest version of a package, which means clients will
  downloaded outdated (and potentially vulnerable) versions of the package.
- **Privacy:** Log entries are public. While this is necessary for
  auditability, it means package upload patterns are visible to anyone.
- **Split-view attacks:** A malicious log can show different log
  states to different users. This can be addressed by adding `witnesses
  <https://c2sp.org/tlog-witness>`_ to the log (see also Appendix D).


In short, transparency is about auditability: it allows clients to detect and
respond to compromise of the package index. It does not prevent the compromise
itself.

Complementary Defenses
----------------------


How to Teach This
=================


Reference Implementation
========================

The library ``pypi-transparency`` has the reference implementation for log
verification. 


Appendices
==========

Appendix A
----------

Publisher Schemas
~~~~~~~~~~~~~~~~~

The ``publisher-kind`` field varies by provider. This table defines the field
for the currently supported providers by PyPI:

.. list-table::
   :header-rows: 1

   * - Provider
     - Integrity API field
     - ``publisher-kind``
   * - GitHub Actions
     - GitHub
     - github
   * - GitLab CI/CD
     - GitLab
     - gitlab
   * - Google Cloud
     - Google
     - google

Appendix B
----------

Bulk Ingestion
~~~~~~~~~~~~~~

Before enabling binary transparency verification in clients, the index (e.g.,
PyPI) will perform a **bulk ingestion** of all existing distribution files
into the transparency log. This ensures that every package—past and
present—has an inclusion proof available.

The bulk ingestion process will:

- Create log entries for all existing distributions using their current
  metadata (checksum, filename).
- For distributions uploaded with a :pep:`740` attestation, include the
  publisher identity in the log entry.
- For distributions uploaded without an attestation, omit the ``publisher``
  field.

The bulk ingestion must complete before clients begin requiring inclusion
proofs.


Appendix C
----------

Merkle Tree transparency logs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This section contains implementation details for transparency logs implemented
using merkle trees.

Merkle Tree
^^^^^^^^^^^

A Merkle tree is the cryptographic data structure underlying a transparency log.
It is a binary tree where:

- Each leaf node contains the hash of a log entry.
- Each internal node contains the hash of its two children.
- The root hash is a single value that cryptographically commits to the entire
  tree contents.

Changing any entry changes its leaf hash, which propagates up to change the
root hash. This makes tampering detectable to anyone who has seen a previous
root hash.

Checkpoint (Merkle-tree logs)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

A checkpoint is a signed statement of the log's current state, containing an
identifier for the log (its origin), the number of entries (tree size), and the
Merkle tree root hash. Checkpoints are signed by the log.

Inclusion Proof (Merkle-tree logs)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

An inclusion proof is a compact cryptographic proof that a specific entry exists
in the log at a given checkpoint. It consists of a sequence of hashes (the
Merkle path) that allows a verifier to recompute the root hash from the entry.
For a log with millions of entries, the amount of data necessary to prove a
given entry exists in the log only requires a few hundred bytes.

Consistency Proof
^^^^^^^^^^^^^^^^^

A consistency proof demonstrates that one checkpoint is an extension of
another—that the log has only appended entries, not modified or deleted any. It
consists of a sequence of hashes that prove the older tree is a prefix of the
newer tree.

Appendix D
----------

Role of witnesses
~~~~~~~~~~~~~~~~~

Witnesses are independent third parties that verify the log's consistency and
cosign checkpoints. They prevent the index from showing different log states
to different parties (split-view attacks).

The witness `protocol <https://c2sp.org/tlog-witness>`_
works as follows:

1. The log produces a new checkpoint and submits it to witnesses requesting
   cosignatures.
2. Each witness verifies that the new checkpoint is consistent with its
   previously-seen checkpoint (i.e., the log is append-only).
3. If verification succeeds, the witness cosigns the checkpoint and returns
   its signature.
4. If verification fails, the witness refuses to cosign and may publish an
   alert.
5. The log collects cosignatures and publishes the fully-signed checkpoint.

Clients verify that checkpoints carry signatures from a threshold of trusted
witnesses before accepting them. This ensures that even if the index attempts
to provide divergent log states, it cannot obtain valid witness signatures for
inconsistent checkpoints.

For the witness ecosystem to be effective:

- Witnesses must be operated by organizations independent of the index.
- The witness threshold should balance security (more witnesses required)
  against availability (some witnesses may be temporarily unreachable).


Appendix E
----------

Disaster recovery playbook
~~~~~~~~~~~~~~~~~~~~~~~~~~

TODO


Copyright
=========

This document is placed in the public domain or under the CC0-1.0-Universal
license, whichever is more permissive.
