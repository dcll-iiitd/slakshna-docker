# SLAKSHNA Node (Docker)

Run one SLAKSHNA federated-learning node in Docker. Several nodes (on one machine or on different networks) train together: each node fine-tunes `allenai/OLMo-2-1124-7B` with LoRA on its local data and shares compressed updates with its peers.

## Repository contents

| Path | Purpose |
|---|---|
| `docker-compose.yml` | Defines the node container, GPU access, volumes and ports |
| `node.toml` | Node settings: federation ID, node ID, ports, peers, discovery |
| `node_template.yaml` | Training settings: model, LoRA, batch size, number of epochs, dataset path |
| `data/South_Asia.jsonl` | Training dataset, mounted read-only at `/data` in the container |
| `logs/` | Node logs are written here |

## Requirements

- Linux with an NVIDIA GPU and driver 525.60.13 or newer.
- Docker with Compose v2 and the NVIDIA Container Toolkit. Check with `docker run --rm --gpus all ubuntu nvidia-smi`.
- A 7B model needs a large GPU (we used 141 GB H200s, and a 48 GB A6000 may not be enough).
- At least 40 GB free on the disk that holds `/var/lib/docker`. A full disk is the most common cause of crashes.
- Internet access to `ghcr.io` and `huggingface.co`.

## 1. Setup

```bash
git clone https://github.com/AnantJainn/slakshna-dockerize.git
cd slakshna-dockerize

docker pull ghcr.io/distributed-computing-and-learning-lab/slakshna:latest   # download the image (~12 GB); public, no login needed
docker volume create slakshna-hf                       # cache for the model download (reused across restarts)
chmod 777 logs                                         # the container writes logs as a different user
df -h / /var/lib/docker                                # check there is enough free space
```

## 2. Configure

Edit `node.toml`:
- `[node] id`: must be unique for every node (`client-node-1`, `client-node-2`, ...).
- `[node] gpu_id`: the GPU to use. Keep `docker-compose.yml` on `count: 1`, or set `device_ids: ["1"]` there to pick another GPU.
- `[network] peers`: leave `[]` on the first node (see section 4 for the other nodes).
- `[federation] id`, `[training]` and `[compression]` must be identical on all nodes, or they won't sync.

To use your own data, replace `data/South_Asia.jsonl`. The file name must match `train_path` in `node_template.yaml`, which is `/data/South_Asia.jsonl`, the path inside the container. Do not put a host path there.

## 3. Run the first node

```bash
docker compose up -d                                   # start in the background
docker logs -f slakshna-node 2>&1 | tee logs/console.log   # follow the log and save it to logs/
```

Starting takes a few minutes: the model downloads on the first run, and then the node waits for a shared clock boundary (`Syncing clock... waiting Ns`) before training starts.

When it is up, the log prints the join string for other nodes:
```
Any peer can join this federation with: peers = ["<endpoint_id>"]
```

## 4. Connect another node

**Same network:** set `expected_peers = 1` and leave `peers = []`. Nodes find each other with mDNS.

**Different networks (another machine, another site):**
1. On the first machine, open port **9000 TCP and UDP** (firewall and router/port-forward) to the public IP.
2. On the second machine, clone the repo, do section 1, and edit `node.toml`:
   ```toml
   [node]
   id = "client-node-2"

   [network]
   peers = ["<endpoint_id_of_node_1>@<public_ip_of_node_1>:9000"]
   ```
3. Start it with the same commands as section 3.
4. The first node's log should show `Peer joined gossip mesh`. If the nodes cannot connect directly, set `relay = true` in `[discovery]` on both nodes, or connect them over a VPN such as Tailscale or WireGuard.

Keep the clocks of both machines synced (`timedatectl`), because epochs start at a shared time.

## 5. Monitor

```bash
docker ps --filter name=slakshna-node                  # status; should say "healthy"
docker logs --since 10m slakshna-node                  # recent log lines only
watch -n 30 df -h /                                    # disk space (see Troubleshooting)
nvidia-smi                                             # GPU usage
```
Training metrics are in `logs/` (for example `epoch_loss_tracking_muon.csv` and `trust_scores_new.csv`).

## 6. Stop and clean up

```bash
docker compose stop                                    # stop, keep container and data
docker compose down                                    # remove the container, keep saved data
docker compose down -v                                 # also delete the node's saved data and identity (keeps slakshna-hf)
```
Before a new run, `docker compose down` gives a clean log, since `docker logs` replays the old container's history.

## Troubleshooting

- **`Node config not found: /app/config.toml`**: `node.toml` is missing, or Docker created a directory with that name. Run `rm -rf node.toml`, restore the file and start again.
- **`Dataset not found: /data/South_Asia.jsonl`**: `data/South_Asia.jsonl` is missing, or `train_path` in `node_template.yaml` is not `/data/South_Asia.jsonl`.
- **`OSError: [Errno 28] No space left on device`** (Ray spilling to `/tmp`): the disk is full. Free space, or move Docker's data to a bigger disk, then restart.
- **`waiting for peers` never ends**: check `peers` and the node IDs, port 9000 (TCP and UDP), and the clocks.
- **Pull denied**: `docker login ghcr.io` with a token that has `read:packages`.
