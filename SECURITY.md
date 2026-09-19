# Security policy

## What this is

SAHC is an academic prototype, built for a Master's-level course on
trusted hardware. It implements real cryptography and real Intel DCAP
remote attestation, and it was validated on Intel SGX2 hardware — but it
has **not** been audited, and it is not meant to protect real patient
data.

The gaps are documented rather than hidden. See "Known limitations" in
the [README](README.md); the ones that matter most for anyone tempted to
reuse this code:

- **No application-level side-channel mitigation.** The query loop and
  the DuckDB parser have data-dependent access patterns, so cache,
  page-fault and branch-timing attacks can partially infer record
  content.
- **No rollback protection.** The enclave accepts any syntactically valid
  sealed blob produced by its own MRENCLAVE, so an operator with
  filesystem access can restore an older state. The fix is a monotonic
  counter in the sealing AAD.
- **No dynamic revocation**, and **no handshake rate limiting** — the
  attestation channel is trivially floodable.
- **Real DCAP only on the Gramine path.** `sgx_server` emits the SAHC
  stub quote even in a hardware build.

The trust boundary itself is only as strong as the enclave measurement:
a client that does not pin MRENCLAVE gets confidentiality against a
passive host and nothing against an active one.

## Reporting an issue

If you find a flaw in the protocol or the implementation, open a GitHub
issue describing it. Since no production deployment exists, there is
nothing to embargo — a public issue is the fastest route to a fix, and
the finding is useful to anyone reading the code as a reference.
