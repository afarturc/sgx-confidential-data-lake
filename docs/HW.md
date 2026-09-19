# Running and validating on Intel SGX hardware

Setup, build, expected test outcomes, benchmarks and troubleshooting for
running the prototype on a machine with Intel SGX enabled.

This is the test plan that was actually executed against real hardware:
the DCAP path was validated end to end on an Azure `Standard_DC2s_v3`
(Ice Lake-SP, SGX2 with Flexible Launch Control) — see
[`AZURE_SETUP.md`](AZURE_SETUP.md) for provisioning that VM from scratch.
It is deliberately deterministic, with the expected output stated for
every step, so that any failure is easy to isolate as either a code bug
or a setup problem.

---

## 1. Hardware / OS prerequisites

- Intel CPU with SGX enabled in the BIOS (Coffee Lake or newer; ideally
  Ice Lake-SP / DCsv3 with FLC).
- Linux with the in-kernel drivers `/dev/sgx_enclave` and
  `/dev/sgx_provision` (kernel ≥ 5.11).
- User in the `sgx_prv` group.
- DCAP infrastructure: `libsgx-dcap-quote-verify-dev`,
  `libsgx-dcap-default-qpl`, `libsgx-urts`, `libsgx-dcap-ql`.
- A PCCS configured in `/etc/sgx_default_qcnl.conf` (Azure exposes one
  per region; on-prem you run a local one).
- Gramine 1.9 (for the `gramine_server` path).

Verify before going further:
```bash
ls -l /dev/sgx_enclave /dev/sgx_provision   # both must exist
groups | grep -E 'sgx_prv'                  # user in the group
dpkg -l | grep -E 'libsgx-dcap|libsgx-urts'
cat /etc/sgx_default_qcnl.conf | head -5    # PCCS endpoint
gramine-sgx --version                       # 1.9.x
```

If any of these fail, stop here — the tests below will produce
downstream errors and make diagnosis harder.

## 2. Build

```bash
git clone https://github.com/afarturc/sgx-confidential-data-lake.git sahc && cd sahc
source /opt/intel/sgxsdk/environment
./scripts/fetch_duckdb.sh
for p in hosp-santa-maria hosp-sao-joao hosp-santo-antonio fcup-research; do
    python3 scripts/gen_identity.py "$p"   # if parties/ is not populated yet
done
python3 scripts/build_authorized_parties.py --quorum 2 \
    --hospital hosp-santa-maria \
    --hospital hosp-sao-joao \
    --hospital hosp-santo-antonio \
    --researcher fcup-research \
    --signed-by hosp-santa-maria \
    --signed-by hosp-sao-joao \
    > authorized_parties.json
gramine-sgx-gen-private-key           # if you don't have one yet

make clean
make hw                                # all-in-one: SDK + Gramine + signing + Gramine pin
```

`make hw` is equivalent to:
```bash
make SGX_MODE=HW SAHC_HW=1 gramine_server gramine_manifest_hw
make SGX_MODE=HW SAHC_HW=1 sgx_server sgx_client
```
and regenerates `Include/expected_mrenclave_gramine.h` (extracted from
`gramine_server.sig`) — that is the pin the client compares against on
hardware.

**Expected:** every command exits 0. For incremental builds during
development, the individual targets still work on their own.

`SAHC_HW=1` flips the DCAP switch:
- The Gramine server reads `/dev/attestation/{user_report_data,quote}`
  and sends a real `sgx_quote3_t` (`PROTO_QUOTE_FORMAT_DCAP=0x01`).
- The client parses the `sgx_quote3_t`, calls the QvL's
  `sgx_qv_verify_quote()`, and validates the chain, the `report_data`
  binding and the MRENCLAVE pin.

## 3. Path A — SGX SDK (`sgx_server`)

### A.1 MRENCLAVE sanity check
```bash
./sgx_server --print-mrenclave
sha256sum Include/expected_mrenclave.h
```
**Expected:** the printed hex matches the array in
`expected_mrenclave.h`. If it doesn't, the header generation is broken.

### A.2 Run
```bash
rm -f data/sealed/state.bin              # if inherited from a SIM run
./sgx_server 127.0.0.1 7878 &
./sgx_client 127.0.0.1 7878 hosp-santa-maria data/hospital_0.csv
```
**Expected:** success (`Server: state persisted`, `UPLOAD_ACK`).

### A.3 Known limitation
```bash
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-santa-maria data/hospital_0.csv
```
**Expected: failure.** Reason: `sgx_server` (the SDK path) emits the
hand-rolled SAHC quote even in a HW build; moving it to real DCAP would
require `sgx_qe_get_quote()` inside the enclave, which was out of scope
for this deliverable. **This is not a regression.** The path that does
real DCAP is **Gramine** (§4).

## 4. Path B — Gramine-SGX (`gramine_server`) *(the main test)*

### B.1 Bring-up

Delete any sealed blob inherited from the SIM/SDK path (incompatible
format):
```bash
rm -f data/sealed/state.bin
gramine-sgx gramine_server 127.0.0.1 7878
```

**Expected:** `parties loaded — 3 hospitals, 1 researchers`, with no
`/dev/attestation absent` warnings (those only appear under
`gramine-direct`).

### B.2 Upload and query with DCAP forced

```bash
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-santa-maria   data/hospital_0.csv
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-sao-joao      data/hospital_1.csv
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-santo-antonio data/hospital_2.csv
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```

**Expected from EACH client:**
```
Client: ATTEST_RESP received (...) bytes
quote_verify: DCAP chain OK + binding OK + MRENCLAVE pin OK
Server: state persisted
```

**The server prints:**
```
DCAP: quote read OK (~4096 bytes)         # exact size varies
```

**Final query:** `result=49.143 matched=14 applied_k=5` — the same as in
SIM, but with the real DCAP path underneath.

### B.3 Deliberate failures (enforcement checks)

A wrong MRENCLAVE must be refused:
```bash
SAHC_EXPECTED_MRENCLAVE=$(printf 'aa%.0s' {1..32}) \
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```
**Expected:** `quote_verify: MRENCLAVE mismatch vs env override`.

A client built without `SAHC_HW=1` must refuse a HW server
(anti-downgrade):
```bash
# rebuild the client without SAHC_HW=1 first
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```
**Expected:** `quote_verify: DCAP format received (qlen=...) but this
build does not include the DCAP verifier — Refusing.`

### B.4 k-anonymity
Every diagnosis in the sample set has fewer than 5 records, so any
filtered query is refused:
```bash
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - blood_sugar avg diabetes
```
**Expected:** `query: refused — below k-anonymity threshold`
(`E_INSUFFICIENT_RECORDS`), with no aggregate and no count returned.

### B.5 Persistence

Kill the server (Ctrl-C) and restart it **without** deleting anything:
```bash
gramine-sgx gramine_server 127.0.0.1 7878
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```
**Expected:** the server prints `Server: state loaded from sealed blob`
and the query returns the same `49.143` with no re-upload. A blob that
fails to unseal is a regression — the sealing key derived from
`/dev/attestation/keys/_sgx_mrenclave` is the critical piece for
surviving a restart.

## 5. What actually changes from direct to sgx

| Aspect                    | gramine-direct (dev)            | gramine-sgx (HW)                                        |
|---------------------------|---------------------------------|---------------------------------------------------------|
| MRENCLAVE / MRSIGNER      | placeholder `0xDE`*32           | real, via `/dev/attestation/{mrenclave,mrsigner}`       |
| Sealing key               | fixed DEV-ONLY (loud warning)   | PSW-derived via `/dev/attestation/keys/_sgx_mrenclave`  |
| Quote                     | stub (not verifiable)           | real DCAP (`sgx.remote_attestation = "dcap"`)           |
| `SAHC_REQUIRE_DCAP`       | must be 0 / unset               | **1** (refuses to run without DCAP)                     |
| `SAHC_EXPECTED_MRENCLAVE` | must be `''` (the pin can't match) | unset (client pins against the HW build's `.sig`)    |
| Sealed blob compatible?   | direct ↔ direct only            | sgx ↔ sgx only (delete `data/sealed/state.bin`)         |

## 6. Benchmarks
```bash
make sgx_bench
rm -f data/sealed/state.bin
gramine-sgx gramine_server 127.0.0.1 7878 &
SAHC_REQUIRE_DCAP=1 ./sgx_bench > bench-hw.md
```
Output: handshake p50/p95/p99, upload throughput per batch size, query
latency. Compare against `bench-sim.md` to quantify the DCAP + EPC
overhead.

## 7. Diagnosing a failure

When a test fails, collect:
1. The exact command.
2. Full client and server output (stdout + stderr).
3. `dmesg | tail -50` if the SGX driver is suspect.
4. `cat /etc/sgx_default_qcnl.conf` (no secrets in it).
5. Versions: `gramine-sgx --version`, `dpkg -l | grep sgx`.

## 8. Fault tolerance

See the "Tolerância a falhas" section of the final report
([`../report/main.pdf`](../report/main.pdf)): crash recovery (sealed
state + restart), rotation of compromised keys, and behaviour under TCP
timeout, decrypt failure and insufficient k-anonymity.

## 9. Troubleshooting

| Symptom                                                | Cause / fix                                                     |
|--------------------------------------------------------|-----------------------------------------------------------------|
| `aesm_service` / `SGX_ERROR_NO_DEVICE`                 | in-kernel drivers missing, or user not in the `sgx_prv` group   |
| `gramine-sgx: enclave-key.pem not found`               | run `gramine-sgx-gen-private-key` once                          |
| `quote_verify: MRENCLAVE mismatch`                     | binary rebuilt without `make clean`, or SIM/HW mixed            |
| `unseal failed` on start                               | the backend was switched → `rm data/sealed/state.bin`           |
| `sgx_qv_verify_quote 0x...A001` (NO_QPL)               | `libsgx-dcap-default-qpl` missing                               |
| `sgx_qv_verify_quote 0x...A002` (CRL_UNAVAILABLE)      | PCCS unreachable / wrong `qcnl.conf`                            |
| `qv_result rejected 0xA006` (OUT_OF_DATE)              | CPU TCB out of date — apply a microcode update                  |
| `qv_result rejected 0xA00C` (REVOKED)                  | PCK revoked — the machine is banned from the TCB                |
| `DCAP report_data binding mismatch`                    | a code bug — capture a hexdump of the quote for diagnosis       |
