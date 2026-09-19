# Running SAHC

The single guide to getting the prototype up. Linear, copy-paste.

- To run **locally, without SGX hardware** → follow §1 through §5.
- To run on **real Intel hardware** → go straight to [`HW.md`](HW.md).

---

## 1. Prerequisites

Debian 13 / Ubuntu 22+:

```bash
# Toolchain and libraries
sudo apt install build-essential git python3 python3-cryptography libssl-dev curl

# Intel SGX SDK (simulation mode) — installs into /opt/intel/sgxsdk
# https://github.com/intel/linux-sgx ("for Linux" .bin installer)

# Gramine 1.9 — only needed for the gramine_server path
sudo curl -fsSLo /etc/apt/keyrings/gramine.asc \
    https://packages.gramineproject.io/gramine-keyring.gpg
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/gramine.asc] \
    https://packages.gramineproject.io/ $(lsb_release -sc) main" | \
    sudo tee /etc/apt/sources.list.d/gramine.list
sudo apt update && sudo apt install gramine
```

Check:
```bash
ls /opt/intel/sgxsdk/environment      # must exist
gramine-direct --version              # any 1.9.x
```

## 2. Clone and first-time setup

```bash
git clone https://github.com/afarturc/sgx-confidential-data-lake.git sahc && cd sahc
source /opt/intel/sgxsdk/environment

# DuckDB (~57 MB, lands in Common/third_party/duckdb/, gitignored)
./scripts/fetch_duckdb.sh

# Identities + authorized_parties.json (once per checkout)
python3 scripts/gen_identity.py hosp-santa-maria
python3 scripts/gen_identity.py hosp-sao-joao
python3 scripts/gen_identity.py hosp-santo-antonio
python3 scripts/gen_identity.py fcup-research

# three founder hospitals, one researcher admitted on a 2-of-3 quorum;
# the script prints the document, so redirect it to the repository root
python3 scripts/build_authorized_parties.py --quorum 2 \
    --hospital hosp-santa-maria \
    --hospital hosp-sao-joao \
    --hospital hosp-santo-antonio \
    --researcher fcup-research \
    --signed-by hosp-santa-maria \
    --signed-by hosp-sao-joao \
    > authorized_parties.json
```

After this step `parties/*.{key,pub}` exist and `authorized_parties.json`
sits at the repository root. The `.key` files are the long-term private
keys — they are gitignored and must stay that way. The version of
`authorized_parties.json` tracked in the repository is only an example of
the format; regenerating it as above overwrites it with keys you hold.

## 3. Build (simulation mode — no hardware)

There are **two server paths** sharing one client. Pick one, or build
both:

```bash
# Path A — SGX SDK (classic enclave, hand-rolled query engine)
make sgx_server sgx_client

# Path B — Gramine + DuckDB (recommended: real SQL)
make gramine_server gramine_manifest
```

To wipe everything: `make clean`.

## 4. Running

### 4.A — SGX SDK path

Terminal 1:
```bash
./sgx_server                              # 127.0.0.1:7878 by default
```

Terminal 2:
```bash
# Upload (3 hospitals, 14 records total)
./sgx_client 127.0.0.1 7878 hosp-santa-maria   data/hospital_0.csv
./sgx_client 127.0.0.1 7878 hosp-sao-joao      data/hospital_1.csv
./sgx_client 127.0.0.1 7878 hosp-santo-antonio data/hospital_2.csv

# Aggregate query (researcher)
./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```

### 4.B — Gramine path

Terminal 1:
```bash
gramine-direct gramine_server             # 0.0.0.0:7878 by default
```

Terminal 2 — under `gramine-direct` the MRENCLAVE is a placeholder, so
the pin has to be disabled with `SAHC_EXPECTED_MRENCLAVE=''`:
```bash
SAHC_EXPECTED_MRENCLAVE='' ./sgx_client 127.0.0.1 7878 hosp-santa-maria   data/hospital_0.csv
SAHC_EXPECTED_MRENCLAVE='' ./sgx_client 127.0.0.1 7878 hosp-sao-joao      data/hospital_1.csv
SAHC_EXPECTED_MRENCLAVE='' ./sgx_client 127.0.0.1 7878 hosp-santo-antonio data/hospital_2.csv
SAHC_EXPECTED_MRENCLAVE='' ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```

### 4.C — REPL mode (either path)

With no extra arguments the client drops into a REPL:
```bash
./sgx_client 127.0.0.1 7878 fcup-research
sahc> upload data/hospital_0.csv          # HOSPITAL roles only
sahc> query age avg diabetes
sahc> query temperature max any
sahc> help
sahc> quit
```

Commands:
- `upload <csv_path>`
- `query <field> <op> [diag]`
  - `field`: `age` | `temperature` | `blood_sugar`
  - `op`: `avg` | `min` | `max` | `count`
  - `diag`: `any` | `healthy` | `diabetes` | `hypertension` | `infection`

## 5. What to expect

A healthy smoke run (with all three hospitals loaded, 14 records total):

| Command                            | Expected result                          |
|------------------------------------|------------------------------------------|
| `query age avg any`                | `result=49.143 matched=14 applied_k=5`   |
| `query temperature max any`        | `result=39.000 matched=14 applied_k=5`   |
| `query age count any`              | `result=14.000 matched=14 applied_k=5`   |
| `query blood_sugar avg diabetes`   | `refused — below k-anonymity threshold`  |

That last row is the k-anonymity guard doing its job, not a failure: the
tracked sample set has at most 4 records per diagnosis, so **every**
filtered query over it falls below `K_ANON_THRESHOLD` (5). Load a larger
dataset to see filtered aggregates come back with a result.

The server logs `Enclave: state sealed` (and `Server: state persisted
(N bytes)`) after each upload — that is the sealing step. Restarting the
server *without* deleting `data/sealed/state.bin` logs
`Enclave: state unsealed` and keeps the records: querying straight away
returns the same numbers with no re-upload.

## 6. Real Intel hardware

On a machine with SGX enabled, to run DCAP for real rather than
simulated, do **not** follow sections 3-5 above. Go straight to
[`HW.md`](HW.md), which covers setup, build, expected test outcomes,
benchmarks and troubleshooting in one document.

## 7. Quick troubleshooting

| Symptom                                                  | Fix                                                                   |
|----------------------------------------------------------|-----------------------------------------------------------------------|
| `bash: ./sgx_client: No such file or directory`          | `make sgx_client` was not run                                         |
| `tcp_listen: invalid host 7878`                          | only the port was passed — arguments are `[host] [port]`              |
| `quote_verify: MRENCLAVE mismatch`                       | under `gramine-direct` you need `SAHC_EXPECTED_MRENCLAVE=''`          |
| `unseal failed` on server start                          | backend was switched (SDK↔Gramine) — `rm data/sealed/state.bin`       |
| `fetch_duckdb.sh: ... not found`                         | run `chmod +x scripts/*.sh` if the scripts lost their exec bit        |
| `fatal error: sgx_dcap_quoteverify.h`                    | only relevant to HW builds; harmless in SIM (that path is never hit)  |
| `gramine-direct: command not found`                      | the Gramine step in §1 was skipped                                    |
| Build fails on `_GLIBCXX_USE_CXX11_ABI`                  | libstdc++ too old; use Debian 13 / Ubuntu 22+                         |

## 8. Datasets

`data/hospital_{0,1,2}.csv` — 5, 5 and 4 synthetic records, 14 in total.
Diagnosis counts across the set: 3 healthy, 4 diabetes, 3 hypertension,
4 infection — all below the k-anonymity threshold by design, so filtered
queries are refused unless you supply more data.

Format:
```csv
patient_id,age,temperature,blood_sugar,diagnosis
1001,45,36.5,95.0,1
```

`diagnosis` codes: `0` healthy, `1` diabetes, `2` hypertension,
`3` infection.
