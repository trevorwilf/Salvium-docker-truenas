# Salvium on TrueNAS (salviumd + P2Pool + optional Stats UI + optional XMRig)

This repo contains Docker Compose YAMLs intended for **TrueNAS SCALE** (or any Docker host) to run a Salvium node and mine via P2Pool:

- **`salviumd`** (full node, pruned)
- **`p2pool-salvium`** (P2Pool + Stratum)
- **P2Pool Stats UI** (optional web UI)
- **XMRig** (optional CPU miner pointed at your local P2Pool)

> ⚠️ **Important: salviumd v1.0.7 bootstrap bug**
>
> There is a known bug in **salviumd v1.0.7** that affects initial chain sync.
> **You must fully sync the chain using v1.0.6 first**, then upgrade to **v1.0.7** using the *same* persisted data volume.

---

## Repo contents

- `salviumd-1.0.6-working.yaml`  
  Builds a Debian-based image, downloads/verifies the official **v1.0.6** zip, and runs `salviumd` as a non-root user with a persistent volume.

- `salviumd-1.0.7-working.yaml`  
  Builds an Ubuntu 22.04-based image, downloads/verifies the official **v1.0.7** zip, runs `salviumd`, and connects it to the external network used by P2Pool.

- `salvium_p2pool.yaml`  
  Builds **p2pool-salvium** from source (clones `mxhess/p2pool-salvium`), runs it against `salviumd`, exposes Stratum and the P2Pool p2p port, and writes stats for the UI.

- `salvium-p2pool-ui.yaml` (optional)  
  Builds a small Flask UI by cloning `trevorwilf/p2pool-salvium` (branch `updatestatistics`) and serving `p2pool_statistics.py` on port 3000.

- `salvium-xmrig.yaml` (optional)  
  Builds XMRig from source and runs it **privileged** (for MSR + HugePages) pointed at your local P2Pool Stratum.

---

## Prerequisites (TrueNAS host setup)

### 1) Create the external Docker network

All services expect an external network named `salvium_p2pool_net`:

```bash
docker network create \
  --driver bridge \
  --subnet=172.17.0.0/24 \
  --ipv6=false \
  --opt "com.docker.network.bridge.enable_icc=true" \
  --opt "com.docker.network.bridge.enable_ip_masquerade=true" \
  salvium_p2pool_net
```

### 2) Reserve 1GB HugePages (recommended for XMRig RandomX)

This adds HugePage reservation to the GRUB boot options:

```bash
midclt call system.advanced.update '{"kernel_extra_options": "hugepagesz=1G default_hugepagesz=1G hugepages=3"}'
```

Reboot after setting this.

### 3) Load the MSR kernel module at boot (recommended for XMRig)

Add this to your TrueNAS **post-init** script:

```bash
modprobe msr
```

---

## Quick start (recommended deployment order)

### Step 1 — Run salviumd **v1.0.6** and fully sync

```bash
docker compose -f salviumd-1.0.6-working.yaml up -d
```

This uses a persistent Docker volume named `ix_volume` mounted at:

- `/home/salvium/.salvium`

Published ports (v1.0.6):
- `19080/tcp` (P2P)
- `19081/tcp` (RPC)
- `19083/tcp` (ZMQ)
- `19089/tcp` (Restricted RPC)

Healthcheck hits:
- `http://127.0.0.1:19081/get_info`

> Wait until the node is fully synced before moving to v1.0.7.

---

### Step 2 — Upgrade to salviumd **v1.0.7** (after sync completes)

Stop v1.0.6:

```bash
docker compose -f salviumd-1.0.6-working.yaml down
```

Start v1.0.7 using the same `ix_volume` data:

```bash
docker compose -f salviumd-1.0.7-working.yaml up -d
```

Notes (v1.0.7):
- Joins external network `salvium_p2pool_net` with alias `salviumd`
- Runs pruned sync with `--prune-blockchain` and `--sync-pruned-blocks`
- Uses additional IPv6-related flags (`--p2p-use-ipv6`, `--rpc-use-ipv6`, etc.)
- Uses a more locked down container config (`cap_drop: ALL`, `read_only: true`, `no-new-privileges`, `tmpfs: /tmp /run`)
- RPC is published **only to localhost** on the host:
  - `127.0.0.1:19081:19081/tcp`

Published ports (v1.0.7):
- `19080/tcp` (P2P)
- `127.0.0.1:19081/tcp` (RPC, localhost only)
- `19089/tcp` (Restricted RPC)

---

### Step 3 — Start P2Pool (Stratum + P2Pool p2p)

**Edit your wallet address first** in `salvium_p2pool.yaml`:

Find:

```yaml
- '--wallet'
- <YOUR_ADDRESS_HERE>
```

Replace the existing address with your Salvium carrot address. Or keep mine ;) 

Then start P2Pool:

```bash
docker compose -f salvium_p2pool.yaml up -d
```

How it connects to `salviumd`:
- Uses the external network alias `salviumd`
- Connects to:
  - Restricted RPC: `salviumd:19089` (`--rpc-port 19089`)
  - ZMQ: `salviumd:19083` (`--zmq-port 19083`)

Published ports:
- `0.0.0.0:3333 -> 3333/tcp` (Stratum)
- `0.0.0.0:38888 -> 38888/tcp` (P2Pool p2p)

It also enables:
- `Required for salvium-stats server
  - `--data-api /home/p2pool/stats` (writes stats)
  - `--stratum-api`
- `--no-igd`

Volumes created:
- `p2pool_volume` → `/home/p2pool/.p2pool`
- `salvium-p2pool_p2pool_stats` → `/home/p2pool/stats`

---

### Step 4 — Start the Stats UI (optional)

```bash
docker compose -f salvium-p2pool-ui.yaml up -d
```

Access it at:

- `http://<truenas-ip>:3000`

Notes:
- Spent way to much time on this, but the new stats dashboard has a similar theme to https://salvium.io/
- It mounts the external stats volume read-only:
  - `salvium-p2pool_p2pool_stats` → `/data:ro`
- On container start it does a `git pull` and recopies the app into `/app` before running.

---

### Step 5 — Start XMRig (optional)

```bash
docker compose -f salvium-xmrig.yaml up -d
```

Defaults from the YAML:
- Mines against your local P2Pool Stratum:
  - `--url=salvium-p2pool:3333`
- Enables RandomX 1GB pages:
  - `--randomx-1gb-pages`
- Threads:
  - `--threads=4`
- Exposes XMRig HTTP API **inside the container**:
  - `--http-host=0.0.0.0 --http-port=14141`
  - (not published to the LAN by default; add `ports:` if you want that)

> Security warning: `salvium-xmrig.yaml` runs **privileged** and mounts `/dev/mem`, `/dev/cpu`, and HugePages paths. Treat it as full host access.

---

## Configuration knobs you probably want to change

### P2Pool wallet (required)
Edit `salvium_p2pool.yaml` and replace the `--wallet` value with your address (or keep it ;) )

### XMRig tuning (optional)
Edit `salvium-xmrig.yaml`:
- `--threads=<n>` for your CPU/cache
- (optional) `--cpu-affinity=...` if you know what you’re doing, tells which cores to use in binary
- If you want the HTTP API reachable from your LAN, add:
  ```yaml
  ports:
    - "14141:14141"
  ```

### Port exposure / security
- v1.0.6 publishes RPC (`19081`) to the network by default.
- v1.0.7 maps RPC to `127.0.0.1` on the host, but still binds `--rpc-bind-ip=0.0.0.0` inside the container.
- Consider firewall rules if exposing:
  - `19080` (P2P) This must be forwarded on your router for communication
  - `3333` (Stratum) - Only forward on your router if you intend for external systems to use your p2pool instance
  - `38888 or 38889` (P2Pool p2p) This must be forwarded on your router for communication

---

## Volumes / persistence

- `ix_volume`  
  Salvium node data dir (`/home/salvium/.salvium`).  
  **Do not delete** unless you want to resync.

- `p2pool_volume`  
  P2Pool working data (`/home/p2pool/.p2pool`).

- `salvium-p2pool_p2pool_stats`  
  P2Pool stats output (`/home/p2pool/stats`), consumed by the UI.

---

## Port reference

| Component | Container Port | Host Publish | Purpose |
|---|---:|---:|---|
| salviumd (v1.0.6) | 19080 | 19080 | P2P |
| salviumd (v1.0.6) | 19081 | 19081 | RPC |
| salviumd (v1.0.6) | 19083 | 19083 | ZMQ |
| salviumd (v1.0.6) | 19089 | 19089 | Restricted RPC |
| salviumd (v1.0.7) | 19080 | 19080 | P2P |
| salviumd (v1.0.7) | 19081 | 127.0.0.1:19081 | RPC (localhost only) |
| salviumd (v1.0.7) | 19089 | 19089 | Restricted RPC |
| P2Pool | 3333 | 3333 | Stratum |
| P2Pool | 38888 | 38888 | P2Pool p2p |
| Stats UI | 80 | 3000 | Web UI |
| XMRig | 14141 | *(not published)* | HTTP API |

---

## Troubleshooting

### P2Pool can’t reach salviumd
- Ensure `salviumd` is running the v1.0.7 compose (it joins `salvium_p2pool_net` and has alias `salviumd`).
- Confirm `19089` and `19083` are reachable inside the network:
  - P2Pool expects `salviumd:19089` and `salviumd:19083`.

### HugePages not working in XMRig
- Confirm you set the kernel args via `midclt` and rebooted.
- Confirm these mounts exist and are mapped in the container:
  - `/sys/kernel/mm/hugepages`
  - `/dev/hugepages`

### MSR errors / low hashrate
- Ensure `modprobe msr` runs at boot (post-init script).
- Confirm XMRig is running privileged.

---

## Next steps

- The containers need to be hardened.
- Add logic to ensure proper startup sequence

---

## Credits / upstream projects

- Salvium: `https://github.com/salvium/salvium`
- P2Pool Salvium fork used for the miner build: `https://github.com/mxhess/p2pool-salvium`
- Stats UI source: `https://github.com/trevorwilf/p2pool-salvium` (branch `updatestatistics`)
- XMRig: `https://github.com/xmrig/xmrig`
- Salvium project: `https://salvium.io/`


