# Stardome SEAD P2P Discovery

**Decentralized peer discovery and pubsub fabric for SEAD nodes.** Provides mDNS
and Kademlia DHT discovery, gossipsub messaging, and a REST API for publishing
and subscribing to topics. Runs as a sidecar alongside each sead-service stack.

## Architecture

```mermaid
graph LR
    subgraph "SEAD Node A"
        SC_A[sead-core:8080]
        P2P_A[p2pd:8089]
    end

    subgraph "SEAD Node B"
        SC_B[sead-core:8080]
        P2P_B[p2pd:8089]
    end

    P2P_A <-->|mDNS / DHT / gossipsub| P2P_B
    P2P_A -->|"POST /events/{topic}"| SC_A
    P2P_B -->|"POST /events/{topic}"| SC_B
```

Each p2p node:
- Discovers other nodes via **mDNS** (LAN) or **Kademlia DHT** (multi-subnet)
- Provides a **gossipsub** pubsub fabric for opaque byte messages
- Exposes a **REST API** (`/health`, `/peers`, `/events/{topic}`, `/metrics`)
- Streams topic messages via **Server-Sent Events (SSE)**

## Deploy

### Prerequisites

- Docker + Docker Compose plugin
- A running SEAD stack (see [stardome-sead](https://github.com/Stardome-technology/stardome-sead))
- The p2p image is pre-built at `ghcr.io/stardome-technology/stardome-sead-p2p:latest` (multi-arch amd64 + arm64)

### Quick start

```bash
# Pull and run (uses host networking — see note below)
docker compose -f docker-compose.remote.yml pull
docker compose -f docker-compose.remote.yml up -d

# Verify
curl http://localhost:8089/health
```

> **Why host networking?** The container uses `network_mode: host` so that mDNS
> multicast packets reach the physical LAN interface — enabling zero-config peer
> discovery across machines without bootstrap configuration. The p2p sidecar is a
> lightweight daemon with no secrets or TLS, making host networking a safe choice.
> The container binds ports directly on the host's IPs; no `-p` port mapping is
> needed.

### Multi-node setup

On the same LAN, mDNS auto-discovers all peers within seconds — no configuration
needed. For nodes on different subnets (or WSL where mDNS multicast may not
bridge through), configure a DHT bootstrap peer via `.env`:

> **⚠️ IMPORTANT:** `<IP>` and `<PEER_ID>` are **placeholders**. You must replace
> them with real values. Copying them literally will crash the p2p service.
> See [Mesh network](#mesh-network-multi-subnet) below for where to find each value.

```bash
# Create .env with any reachable peer as bootstrap
# Get the peer ID from its /health endpoint first
# ⚠️ Replace <IP> and <PEER_ID> with actual values!
echo 'P2P_BOOTSTRAP_PEERS=/ip4/<IP>/tcp/4001/p2p/<PEER_ID>' >> .env

# Restart to pick up config
docker compose -f docker-compose.remote.yml down
docker compose -f docker-compose.remote.yml up -d
```

### Mesh network (multi-subnet)

When nodes span multiple subnets (e.g. LAN + wireless mesh), provide a bootstrap
peer multiaddr for **each subnet** so DHT can bridge across them. The p2pd listens
on `0.0.0.0` so it binds to all interfaces automatically. mDNS handles same-subnet
discovery; DHT bridges across subnets.

> **⚠️ IMPORTANT:** The examples below use `<LAN_IP>`, `<MESH_IP>`, and `<PEER_ID>` as
> **placeholders**. You must replace them with actual values. Copying them literally
> will cause the p2p service to crash on startup.

#### Where to find the values

| Value | Where to get it |
|-------|----------------|
| **`<LAN_IP>`** | The node's IP on the wired LAN — e.g. `192.168.0.102`. Run `ip addr show` or `hostname -I` on the node. |
| **`<MESH_IP>`** | The node's IP on the wireless mesh — e.g. `192.168.60.1`. Run `ip addr show bat0` on the node. |
| **`<PEER_ID>`** | The p2p node's PeerId — a string like `12D3KooW...`. Get it from the `/health` endpoint of a *running* p2p node. |

#### Chicken-and-egg: Getting the PeerId

Since the p2p service must be running to report its PeerId, but a bad bootstrap
config prevents it from starting, use this bootstrap procedure:

1. **On one node only**, comment out or remove `P2P_BOOTSTRAP_PEERS` from `.env`
   so the service starts without bootstrap peers:
   ```bash
   # .env — no bootstrap peers, just mDNS for now
   # P2P_BOOTSTRAP_PEERS=/ip4/...  ← comment out or delete
   ```
2. Start the p2p service:
   ```bash
   docker compose -f docker-compose.remote.yml down
   docker compose -f docker-compose.remote.yml up -d
   ```
3. Get its PeerId:
   ```bash
   curl http://localhost:8089/health
   # → {"peer_id":"12D3KooW...","listen_addrs":["/ip4/192.168.0.102/tcp/4001",...],...}
   ```
4. Use that PeerId in the `.env` of **all other nodes** (substituting the actual
   IPs of the bootstrap node):
   ```bash
   P2P_BOOTSTRAP_PEERS=/ip4/<LAN_IP>/tcp/4001/p2p/<PEER_ID>,/ip4/<MESH_IP>/tcp/4001/p2p/<PEER_ID>
   ```
5. Restart the bootstrap node with the same config so it connects back.

#### Example: Real `.env` for the bootstrap node

```bash
# P2P_BOOTSTRAP_PEERS is empty on purpose — this node is the bootstrap seed
# Other nodes will connect to it using its PeerId
```

#### Example: Real `.env` for all other nodes

```bash
P2P_BOOTSTRAP_PEERS=/ip4/<LAN_IP>/tcp/4001/p2p/<PEER_ID>,/ip4/<MESH_IP>/tcp/4001/p2p/<PEER_ID>
```

## Configuration

Create a `.env` file in this directory to override defaults:

| Variable | Default | Description |
|----------|---------|-------------|
| `P2P_API_PORT` | `8089` | HTTP API port |
| `P2P_LISTEN_ADDRS` | `/ip4/0.0.0.0/tcp/4001,/ip4/0.0.0.0/udp/4001/quic-v1` | libp2p listen multiaddrs |
| `P2P_ENABLE_MDNS` | `true` | Enable mDNS peer discovery |
| `P2P_MDNS_SERVICE_TAG` | `stardome-p2p-v1` | mDNS service tag |
| `P2P_ENABLE_DHT` | `true` | Enable Kademlia DHT |
| `P2P_DHT_NAMESPACE` | `/stardome-p2p/v1` | DHT namespace for advertising |
| `P2P_BOOTSTRAP_PEERS` | _(empty)_ | Bootstrap peers for DHT (comma-separated multiaddrs) |
| `P2P_MAX_MESSAGE_SIZE` | `262144` | Maximum pubsub message size in bytes (256 KB) |
| `P2P_IDENTITY_KEY_FILE` | `/data/p2p_identity.key` | Path to persistent identity key |

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Node status: peer_id, listen_addrs, peers_connected, dht, topics |
| GET | `/peers` | List connected peers with addresses and protocols |
| GET | `/metrics` | Prometheus metrics |
| POST | `/events/{topic}` | Publish hex-encoded data to a topic |
| GET | `/events/{topic}` | Subscribe to a topic via Server-Sent Events |

### Example: Publish a message

```bash
curl -X POST http://localhost:8089/events/mytopic \
  -H "Content-Type: application/json" \
  -d '{"data_hex": "aabbccdd"}'
```

### Example: Subscribe via SSE

```bash
curl -N http://localhost:8089/events/mytopic
# → event: connected
# → data: {"topic": "mytopic"}
# → data: {"data_hex":"aabbccdd","received_from":"12D3KooW...","topic":"mytopic"}
# → : heartbeat (every 30s)
```

## License

See LICENSE file in the repository root.

## Status

✅ **Deployed and verified** on a 4-node mesh across LAN and wireless subnets.
All nodes connected with DHT active and pubsub propagation confirmed.