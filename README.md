# SAHC: a confidential data lake for multi-hospital analytics on Intel SGX

A client/server system where several hospitals upload encrypted patient records to a
server they do not trust, and researchers run aggregate queries (AVG, MIN, MAX, COUNT)
over the combined dataset. Records are decrypted only inside an Intel SGX enclave. The
host OS, the hypervisor and the cloud operator never see a plaintext record.

Built in C/C++ on the Intel SGX SDK and Gramine 1.9, with an embedded DuckDB query
engine and real Intel DCAP remote attestation. Validated end to end on Intel SGX2
hardware (Azure `Standard_DC2s_v3`, Xeon Platinum 8370C).

Course project for *Segurança e Aplicações de Hardware Confiável*, MSc in Information
Security, Faculdade de Ciências da Universidade do Porto, 2025/26.
By **Artur Correia**, **Gonçalo Sousa** and **Tiago Pinheiro**.

## The problem

A consortium of hospitals wants to compute statistics over their pooled patient records:
disease prevalence, average lab values, correlations across institutions. Each hospital
is legally and ethically barred from handing raw records to the others, and none of them
wants to trust the cloud provider hosting the pool.

The question this project answers: **can you place the aggregation inside a hardware
enclave, prove to each participant that the code holding their records is the code they
audited, and still get usable performance?**

The adversary is the cloud operator. It has root on the host, controls the hypervisor and
the filesystem, can read and rewrite process memory and network traffic, can restart the
server at will, and can present itself as a legitimate hospital or researcher. Everything
outside the enclave boundary is hostile.

## Results

Measured on Azure `Standard_DC2s_v3` (2 vCPU Xeon Platinum 8370C, Ice Lake-SP, SGX2 with
Flexible Launch Control), client and server on loopback to isolate the enclave overhead
from the network. SIM is the SGX SDK simulation path with the hand-rolled query engine;
HW is Gramine on real SGX hardware with full DCAP attestation and DuckDB.

**Session handshake** (socket open to `KEY_ACK` accepted, 50 iterations):

| Metric | SIM (ms) | HW (ms) | Ratio |
|---|---:|---:|---:|
| mean | 3.20 | 32.15 | 10.0x |
| p50 | 3.10 | 31.56 | 10.2x |
| p95 | 3.66 | 33.55 | 9.2x |
| p99 | 8.19 | 62.24 | 7.6x |

**Upload throughput** by batch size (5 uploads per size, one open session):

| Batch | SIM mean (ms) | SIM rec/s | HW mean (ms) | HW rec/s |
|---:|---:|---:|---:|---:|
| 1 | 3.29 | 304 | 5.41 | 185 |
| 5 | 3.47 | 1440 | 5.67 | 882 |
| 25 | 2.89 | 8650 | 6.02 | 4152 |
| 100 | 2.72 | 36738 | 6.02 | 16615 |

**Query latency** by aggregation (20 iterations):

| Query | SIM p50 (ms) | HW p50 (ms) | matched |
|---|---:|---:|---:|
| AVG age, no filter | 0.22 | 5.77 | 655 |
| MIN temperature | 0.09 | 5.73 | 655 |
| MAX blood sugar, diabetes | 0.08 | 5.85 | 170 |
| COUNT age, hypertension | 0.09 | 5.81 | 148 |
| AVG blood sugar | 0.09 | 5.76 | 655 |

## Findings

- **The 26x query slowdown is not SGX, it is the engine.** The SIM path walks the record
  array once with no dynamic allocation; the HW path pays DuckDB table creation, SQL
  parsing and planning on every query. The absolute cost stays around 6 ms, which is fine
  for interactive use, and real SQL expressiveness is worth it. Attributing this gap to
  enclave overhead would have been the easy wrong conclusion.

- **Attestation cost is a one-off, not a tax on throughput.** The roughly 29 ms of extra
  handshake latency is dominated by the round trip to Intel's Quoting Enclave through
  AESM and by collateral validation in `sgx_qv_verify_quote`. It is paid once per session
  and amortizes to nothing over a working session.

- **Intel's Quote Verification Library does not check what you probably care about most.**
  The QvL validates the signature chain, the PCK certificates, the QE identity and the TCB
  level, but the binding between the client's nonce and the enclave's ephemeral ECDH key
  is an application-level property. It has to be recomputed and compared explicitly in the
  client, or you have a verified quote with no freshness guarantee. This is the kind of
  gap that makes an attestation implementation look correct while being replayable.

- **Both adversarial tests failed exactly where they should, and no earlier.** Forcing a
  hardware-to-simulation downgrade aborts in the signature-chain stage before any session
  key exists; forcing an MRENCLAVE mismatch passes the DCAP chain and the report binding,
  then aborts in the identity-pinning stage. The three verifier stages are genuinely
  independent, each able to kill the session on its own.

- **The LibOS is what keeps the trusted core portable.** On Gramine, quote production is
  entirely pseudo-file I/O (`/dev/attestation/user_report_data`, `/dev/attestation/quote`),
  so `EnclaveLogic/` needs no attestation code at all and compiles unchanged against both
  the SGX SDK and Gramine backends.

Honest divergence from the design: the plan specified AES-256-GCM and the implementation
uses AES-128-GCM, matching the single HKDF expansion and the `sgx_rijndael128GCM`
parameter on the SDK path. 128-bit symmetric security is sufficient under the stated
adversary model, and the report says so rather than quietly shipping the change.

## How it works

![Architecture](report/pictures/architecture.png)

Split-trust, with two server paths sharing one client and one protocol. The trusted core
in `EnclaveLogic/` is backend-neutral and compiles into both:

- **`sgx_server`**: classic SGX SDK path, `enclave.signed.so` loaded over ECALLs, with
  the hand-rolled query engine. Development path.
- **`gramine_server`**: Gramine LibOS path, the same `EnclaveLogic/` linked against
  OpenSSL and Gramine pseudo-files, with DuckDB for real SQL, and real DCAP attestation
  under `SAHC_HW=1`. Production path.

The client-to-enclave channel passes physically through the untrusted server but is
encrypted end to end under a key neither the server nor the operator can derive.

**A session, step by step:**

1. Server boots, tries to unseal `data/sealed/state.bin`. On a miss it loads
   `authorized_parties.json`, validates the researcher quorum, and seals the initial state.
2. `ATTEST_REQ` (client to server): `party_id || nonce(16) || client_ecdh_pub(64) ||
   ECDSA_sig(64)`, the signature covering `"SAHC-attest-v1" || nonce || client_ecdh_pub`
   under the client's long-term key. This authenticates the client.
3. `ATTEST_RESP` (server to client): a format byte, then the quote. In HW the server
   writes `SHA-256(nonce || enclave_ecdh_pub)` into
   `/dev/attestation/user_report_data` and returns the real `sgx_quote3_t` from
   `/dev/attestation/quote`. This authenticates the enclave *and* binds it to this
   session's ephemeral key.
4. Client verification, four independent stages: structural parse, signature chain
   (`sgx_qv_verify_quote`, accepting `OK`, `CONFIG_NEEDED` and `SW_HARDENING_NEEDED`,
   rejecting `REVOKED` and `OUT_OF_DATE`), report binding, MRENCLAVE pin.
5. `HKDF`: `PRK = HMAC-SHA256("SAHC-v1", ECDH_shared)`, expanded to a 16-byte AES-128
   session key and a 4-byte IV prefix.
6. `KEY_CONFIRM` / `KEY_ACK`: the enclave verifies the confirmation MAC and assigns the
   role registered for that `party_id`.
7. Every frame from here is AEAD: `[type | len | seq(8) | iv(12) | ciphertext | tag(16)]`,
   `iv = iv_prefix || seq`, header as AAD, a monotonic sequence number per direction.
   Any replay, sequence mismatch or bad tag closes the session.
8. `UPLOAD` (hospitals only), then `QUERY_REQ` (any role). If a query matches fewer than
   `K_ANON_THRESHOLD` (5) records the enclave returns `E_INSUFFICIENT_RECORDS` with no
   aggregate and no count, so adaptive queries cannot narrow in on an individual.

**Identity and admission.** Every participant holds an ECDSA P-256 long-term keypair.
Hospitals are founders, registered directly in `authorized_parties.json`. Researchers are
admitted only on quorum: at least `M=2` valid hospital signatures over
`SHA256("SAHC-approve-v1" || researcher_id || researcher_pubkey)`, re-validated by the
enclave on every load. This moves admission from a single administrator to the consortium.

| Role | Upload | Query | k-anonymity |
|---|---|---|---|
| `HOSPITAL` | yes | yes | 5 |
| `RESEARCHER` | no | yes | 5 |

**Persistence.** State is sealed with the key policy pinned to MRENCLAVE, so an operator
who swaps in a modified enclave binary cannot unseal blobs written by the original, even
with full filesystem access. Records and identities survive a server restart; clients
reconnect and re-attest transparently.

## Repository layout

```
EnclaveLogic/           trusted core, backend-neutral (SDK or Gramine)
  enclave_logic.cpp     attest_begin, key_confirm, upload, query, seal
  crypto_backend_*.cpp  sgx_tcrypto (SDK) or OpenSSL (Gramine)
  identity_backend_*.cpp  sgx self-report or /dev/attestation pseudo-files
  seal_backend_*.cpp    sgx_seal_data_ex or Gramine MRENCLAVE-bound key
  query_engine_*.cpp    hand-rolled single pass (SDK) or DuckDB (Gramine)
Client/                 one client, shared by both server paths
  session.cpp           ClientSession API, reused by the benchmark harness
  quote_verify.cpp      the four-stage verifier, SAHC stub vs real DCAP
  identity.cpp          ECDSA P-256 load/sign/verify
  secure_frame.cpp      AES-128-GCM AEAD with sequence numbers
Server/                 SGX SDK path: accept loop, dispatcher, enclave host
Gramine/                Gramine path: same dispatcher, no ECALL boundary
  server.manifest.template  Jinja, both gramine-direct and gramine-sgx
Enclave/                SDK enclave: Enclave.{cpp,edl,config.xml}
Bench/                  handshake latency, upload throughput, query latency
Include/                patient.h, protocol.h, party.h
report/                 final report, LaTeX sources and PDF (Portuguese)
docs/                   RUNNING.md, HW.md, AZURE_SETUP.md, diagrams, M1 artifacts
scripts/                identity generation, MRENCLAVE extraction, DuckDB fetch
```

The full write-up, including the adversary model, the security-property analysis and the
requirement-to-test mapping, is **`report/main.pdf`** (in Portuguese).

## Running it

Full operational guide in **[`docs/RUNNING.md`](docs/RUNNING.md)** for simulation mode,
**[`docs/HW.md`](docs/HW.md)** for real Intel hardware, and
**[`docs/AZURE_SETUP.md`](docs/AZURE_SETUP.md)** to provision an SGX VM from scratch.

Simulation mode needs no SGX hardware:

```bash
source /opt/intel/sgxsdk/environment

./scripts/fetch_duckdb.sh                      # not tracked, ~57 MB
for p in hosp-santa-maria hosp-sao-joao hosp-santo-antonio fcup-research; do
    python3 scripts/gen_identity.py "$p"       # writes parties/<id>.{key,pub}
done

# three founder hospitals, one researcher admitted on a 2-of-3 quorum
python3 scripts/build_authorized_parties.py --quorum 2 \
    --hospital hosp-santa-maria \
    --hospital hosp-sao-joao \
    --hospital hosp-santo-antonio \
    --researcher fcup-research \
    --signed-by hosp-santa-maria \
    --signed-by hosp-sao-joao \
    > authorized_parties.json

make gramine_server gramine_manifest           # or: make sgx_server sgx_client
```

Not tracked, generated on first build or by the scripts above: the enclave signing key
(`Enclave/Enclave_private.pem`, created by the Makefile), the DuckDB amalgamation, the
per-party long-term keys, the sealed state under `data/sealed/`, and the benchmark output.
The tracked CSVs in `data/` are small synthetic samples; the benchmark harness generates
its own record sets.

## Known limitations

- **Real DCAP only on the Gramine path.** `sgx_server` emits the SAHC stub quote even in a
  HW build. Moving it to real DCAP would need `sgx_qe_get_quote()` inside the enclave. The
  production path is `gramine_server` with `SAHC_HW=1`.
- **No application-level side-channel mitigation.** The query loop and the DuckDB parser
  have data-dependent access patterns, so cache, page-fault and branch-timing attacks can
  partially infer record content. Oblivious primitives and ORAM were out of scope.
- **No rollback protection.** The enclave accepts any syntactically valid blob produced by
  its own MRENCLAVE, so an operator can restore an old sealed state. The fix is a
  monotonic counter in the sealing AAD.
- **No dynamic revocation.** Removing a party means editing the JSON and deleting
  `data/sealed/state.bin` to force a reload.
- **Connection cap is unhandled.** The server holds 8 concurrent sessions; the 9th gets a
  generic `E_INTERNAL` instead of a meaningful error.
- **No handshake rate limiting**, which makes flooding the attestation channel a trivial
  denial of service.
- **Sealed blobs do not migrate.** Switching SGX SDK to Gramine, or SIM to HW, invalidates
  `data/sealed/state.bin`, by design rather than by accident.

## Stack

| Component | Technology |
|---|---|
| Language | C/C++ |
| Trusted crypto | `sgx_tcrypto` (SDK) or OpenSSL (Gramine): AES-128-GCM, ECDSA P-256, ECDH, HMAC-SHA256, sealing |
| Query engine | Hand-rolled single pass (SDK) or DuckDB v1.1.3 with a SQL allowlist (Gramine) |
| LibOS | Gramine 1.9 |
| Attestation | Intel DCAP, real on the Gramine path under `SAHC_HW=1` |
| Transport | Raw TCP with a 5-byte header, AEAD frames after key exchange |
| Build | GNU Make, `sgx_edger8r`, `sgx_sign` |

## Security

This is an academic prototype: real cryptography and real attestation,
but unaudited, and not built to hold real patient data. The gaps that
matter to anyone reusing the code are spelled out in
[`SECURITY.md`](SECURITY.md).

## License

MIT, see [`LICENSE`](LICENSE).
