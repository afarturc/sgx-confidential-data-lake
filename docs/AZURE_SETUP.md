# Provisioning an Azure DCsv3 VM (Intel SGX) from scratch

A sequential runbook for preparing a `Standard_DC*s_v3` VM (Ice Lake-SP
with SGX). One command per block, copy-paste directly. For the end-to-end
validation plan see [`HW.md`](HW.md).

---

## 0. Check that SGX is alive

```
lscpu | grep 'Model name'
```
```
grep -o sgx /proc/cpuinfo | head -1
```
```
ls -l /dev/sgx_enclave /dev/sgx_provision
```
```
sudo dmesg | grep -i sgx | head -5
```

Expected: a Xeon Platinum 83xx, the `sgx` flag, both device nodes
present, EPC ≥ 168 MiB. If the vendor is `AuthenticAMD` or cpuinfo has no
`sgx`, the VM size is wrong — recreate it as `DC*s_v3` (Intel).

## 1. Intel SGX repository (GPG signed)

```
sudo rm -f /usr/share/keyrings/intel-sgx.gpg /etc/apt/sources.list.d/intel-sgx.list
```
```
curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | sudo gpg --dearmor -o /usr/share/keyrings/intel-sgx.gpg
```
```
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-sgx.gpg] https://download.01.org/intel-sgx/sgx_repo/ubuntu noble main" | sudo tee /etc/apt/sources.list.d/intel-sgx.list
```
```
sudo apt update
```

`apt update` must show `Get: ... intel-sgx ...` **without** a GPG error.
On `NO_PUBKEY`, import the key from a keyserver:
```
sudo gpg --no-default-keyring --keyring /usr/share/keyrings/intel-sgx.gpg --keyserver keyserver.ubuntu.com --recv-keys E5C7F0FA1C6C6C3C
```

## 2. SGX userspace stack + DCAP

```
sudo apt install -y libsgx-urts libsgx-dcap-ql libsgx-dcap-default-qpl libsgx-dcap-quote-verify-dev libsgx-quote-ex sgx-aesm-service libsgx-aesm-quote-ex-plugin libsgx-aesm-ecdsa-plugin libsgx-ae-qe3 libsgx-ae-qve
```

Validate:
```
sudo systemctl status aesmd --no-pager | head -5
```
```
cat /etc/sgx_default_qcnl.conf
```
Expected: `aesmd` active (running), and the Azure PCCS URL
(`acccache.azure.net`) already configured.

## 3. SGX SDK

```
cd /tmp && wget https://download.01.org/intel-sgx/sgx-linux/2.23/distro/ubuntu22.04-server/sgx_linux_x64_sdk_2.23.100.2.bin
```
```
chmod +x sgx_linux_x64_sdk_2.23.100.2.bin
```
```
sudo ./sgx_linux_x64_sdk_2.23.100.2.bin --prefix=/opt/intel
```
```
echo 'source /opt/intel/sgxsdk/environment' >> ~/.bashrc
```
```
source /opt/intel/sgxsdk/environment
```

## 4. Gramine 1.9

```
sudo mkdir -p /etc/apt/keyrings
```
```
sudo curl -fsSLo /etc/apt/keyrings/gramine-keyring.gpg https://packages.gramineproject.io/gramine-keyring.gpg
```
```
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/gramine-keyring.gpg] https://packages.gramineproject.io/ noble main" | sudo tee /etc/apt/sources.list.d/gramine.list
```
```
sudo apt update && sudo apt install -y gramine
```
```
gramine-sgx --version
```
```
gramine-sgx-gen-private-key
```

## 5. SGX permissions

```
sudo usermod -aG sgx_prv $USER
```
```
newgrp sgx_prv
```
```
groups | grep sgx_prv
```

## 6. Get the repository onto the VM

```
git clone https://github.com/afarturc/sahc-project.git ~/sahc
```

To push local work in progress instead of cloning, rsync from your
machine:
```
rsync -av --exclude='.git' --exclude='*.o' <local-repo-path>/ azureuser@<VM-IP>:~/sahc/
```

## 7. Build (on the VM)

```
cd ~/sahc && source /opt/intel/sgxsdk/environment
```
```
./scripts/fetch_duckdb.sh
```
```
for p in hosp-santa-maria hosp-sao-joao hosp-santo-antonio fcup-research; do python3 scripts/gen_identity.py "$p"; done
```
```
python3 scripts/build_authorized_parties.py --quorum 2 --hospital hosp-santa-maria --hospital hosp-sao-joao --hospital hosp-santo-antonio --researcher fcup-research --signed-by hosp-santa-maria --signed-by hosp-sao-joao > authorized_parties.json
```
```
make clean
```
```
make hw
```
`make hw` does everything in the right order: SDK enclave, signed Gramine
manifest, automatic extraction of the Gramine MRENCLAVE, and a client
linked against that pin.

## 8. Smoke test

Server, in a `tmux` session:
```
tmux new -s sahc
```
```
rm -f data/sealed/state.bin
```
```
gramine-sgx gramine_server 127.0.0.1 7878
```
Detach with `Ctrl-B` then `D`.

Azure DCsv3 note (once per SSH session):
```
export AZDCAP_COLLATERAL_VERSION=v3
```
Intel's QvL does not yet accept the `v4` collateral Azure returns by
default; `v3` resolves the `sgx_qv_verify_quote 0xe03a` failure.

Client, in another SSH session:
```
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-santa-maria data/hospital_0.csv
```
```
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-sao-joao data/hospital_1.csv
```
```
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 hosp-santo-antonio data/hospital_2.csv
```
```
SAHC_REQUIRE_DCAP=1 ./sgx_client 127.0.0.1 7878 fcup-research - age avg any
```

Expected from every client: `quote_verify: DCAP chain OK + binding OK +
MRENCLAVE pin OK`. Final query: `result=49.143 matched=14 applied_k=5`.

From here follow [`HW.md`](HW.md) §B.3 (negative tests), §B.5
(persistence) and §6 (benchmarks).

## 9. When you stop working

Stop-deallocate (from your local machine) to stop billing:
```
az vm deallocate -g <resource-group> -n <vm-name>
```
Resume:
```
az vm start -g <resource-group> -n <vm-name>
```
