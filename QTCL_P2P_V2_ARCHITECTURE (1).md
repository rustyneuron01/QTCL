# QTCL P2P v2 — Complete Architectural Plan
## Oracle-Anchored Distributed Quantum State Network with C Transport Layer
### For implementation by next Claude session

---

> **Reading order for implementation Claude:**
> Read this entire document before touching a single line of code.
> The C layer, the Python bindings, and the quantum geometry engine are
> tightly coupled. The section ordering matches the build dependency graph.

---

## 0. THE GOVERNING IDEA

```
┌─────────────────────────────────────────────────────────────────────┐
│  ORACLE CLUSTER (Koyeb)                                             │
│  5× AerSimulator nodes running real Qiskit circuits                 │
│  Produce canonical DensityMatrixSnapshot every ~2s                  │
│  Broadcast via SSE: /api/snapshot/sse                               │
│  ROLE: Quantum ground truth. Not dumb. Not replaceable.             │
│  The oracle is the quantum GPS of the network.                      │
└────────────────────────┬────────────────────────────────────────────┘
                         │  SSE stream (JSON DensityMatrixSnapshot)
              ┌──────────▼──────────┐
              │  Every client:      │
              │  1. Receives DM     │  ← oracle snapshot (8×8 complex)
              │  2. Reconstructs ρ  │  ← local GKSL evolution
              │  3. Measures own    │  ← (pq0, pq_curr, pq_last)
              │     hyperbolic      │     as 3D Poincaré ball object
              │     triangle        │
              │  4. Computes F(ρ,   │  ← local W-state fidelity
              │     |W₃⟩)           │
              │  5. Signs + gossips │  ← HLWE-authenticated
              └──────────┬──────────┘
                         │  C P2P layer (TCP, binary protocol)
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    [client A]      [client B]      [client C]
    measures own    measures own    measures own
    (pq0,c,l)→ρ_A  (pq0,c,l)→ρ_B  (pq0,c,l)→ρ_C
         │               │               │
         └───────────────┼───────────────┘
                         │  BFT median consensus
                         ▼
              canonical network W-state
              = f(oracle_snapshot, peer_measurements)
              submitted with every block
```

**The oracle provides the quantum reference frame.**
**Every client independently remeasures within that frame.**
**The P2P network aggregates those measurements.**
**The result is a distributed quantum sensor.**

---

## 1. WHAT CHANGES, WHAT STAYS

### Stays Unchanged
- Oracle cluster, `oracle.py`, `lattice_controller.py` — untouched
- Koyeb server block validation, `submit_block` endpoint — untouched
- HLWE crypto engine, wallet, BIP39/32/38 — untouched
- PoW mining engine (`_mine_inline`) — untouched
- The C acceleration layer (`_QTCL_C_SRC`) — extended, not replaced

### Changes in `qtcl_client.py`
| Component | Action | Why |
|---|---|---|
| `P2POracleClient` (~350 lines) | DELETE | replaced by C P2P |
| `DHTBus` / `DHTEvent` / `DHTKademlia` | DELETE | replaced by C DHT |
| `AsyncOracleMiner` (~360 lines) | DELETE | replaced by C P2P |
| `get_oracle_client()` singleton | DELETE | no longer needed |
| `_oracle_sse_listener` thread | REPLACE | becomes C SSE client |
| `_gksl_rk4_step` | KEEP + EXTEND | feeds local oracle engine |
| quantum metrics (`_vn_entropy` etc.) | KEEP | used by local oracle |

### New in `qtcl_client.py`
| Component | Lines est. | Description |
|---|---|---|
| `§P2P` C source section | ~1,400 | Full C P2P library |
| Python cffi bindings | ~300 | Thin Python↔C bridge |
| `HyperbolicTriangle` | ~120 | 3D geometry object |
| `LocalOracleEngine` | ~280 | SSE→DM→measurement pipeline |
| `WStateConsensus` | ~180 | BFT median aggregator |
| `QtclP2PNode` (Python shell) | ~200 | Lifecycle manager |
| Updated `_mine_inline` wiring | ~80 | Use new oracle in block |

---

## 2. THE C P2P LIBRARY: `§P2P` IN `_QTCL_C_SRC`

This is the largest new section added to the existing C source string.
It compiles as part of the same `cffi.verify()` call — one `.so`, one compile.

### 2a. Wire Protocol Specification

```c
/* ═══════════════════════════════════════════════════════════════════════
   QTCL Wire Protocol v2.0
   All integers: little-endian (matches x86 + ARM native)
   All hashes:   SHA3-256 (qtcl_sha3_256 from §1 already compiled)
   ═══════════════════════════════════════════════════════════════════════ */

/* Message header: 28 bytes fixed */
typedef struct {
    uint8_t  magic[4];      /* 0x51 0x54 0x43 0x4C  = "QTCL" */
    uint8_t  command[12];   /* ASCII command name, NUL-padded  */
    uint32_t length;        /* payload byte count, LE          */
    uint8_t  checksum[4];   /* SHA3-256(payload)[:4]           */
    uint8_t  version;       /* protocol version = 2            */
    uint8_t  flags;         /* 0x01=compressed 0x02=encrypted  */
    uint8_t  reserved[2];   /* must be zero                    */
} QtclMsgHeader;            /* sizeof = 28 bytes                */

/* Commands */
#define CMD_VERSION    "version"      /* handshake: capabilities */
#define CMD_VERACK     "verack"       /* handshake ack           */
#define CMD_GETADDR    "getaddr"      /* request peer list       */
#define CMD_ADDR       "addr"         /* peer list (≤1000 peers) */
#define CMD_INV        "inv"          /* announce block/tx/wstate*/
#define CMD_GETDATA    "getdata"      /* request inv items       */
#define CMD_BLOCK      "block"        /* full block              */
#define CMD_TX         "tx"           /* transaction             */
#define CMD_PING       "ping"         /* keepalive + nonce       */
#define CMD_PONG       "pong"         /* keepalive reply         */
#define CMD_WSTATE     "wstate"       /* W-state measurement gossip */
#define CMD_WSYNC      "wsync"        /* request peer's wstate   */
#define CMD_REJECT     "reject"       /* explicit rejection      */

/* Inv types */
#define INV_TX         1
#define INV_BLOCK      2
#define INV_WSTATE     3   /* NEW: W-state measurement announcement */
```

### 2b. Core Data Structures

```c
/* Peer record — stored in memory, persisted via Python SQLite */
typedef struct {
    uint8_t  node_id[16];       /* SHA3-256(pubkey)[:16]            */
    char     host[64];          /* IPv4/IPv6 or hostname            */
    uint16_t port;              /* listen port                      */
    uint8_t  services;          /* bitmask: 0x01=node 0x02=miner    */
    uint8_t  version;           /* protocol version                 */
    int64_t  last_seen_ns;      /* nanosecond timestamp             */
    int32_t  chain_height;      /* last reported block height       */
    float    last_fidelity;     /* last reported W-state fidelity   */
    float    latency_ms;        /* measured RTT                     */
    uint16_t ban_score;         /* violations; banned at 100        */
    uint8_t  connected;         /* 1=active connection, 0=known     */
    uint8_t  _pad;
} QtclPeer;

/* W-state measurement — the core gossip object */
typedef struct {
    uint8_t  node_id[16];       /* signing node                     */
    uint32_t chain_height;      /* block height when measured       */
    uint32_t pq0;               /* genesis pseudoqubit ID           */
    uint32_t pq_curr;           /* current block pseudoqubit        */
    uint32_t pq_last;           /* previous block pseudoqubit       */
    double   w_fidelity;        /* F(ρ, |W₃⟩)                      */
    double   coherence;         /* L1 coherence                     */
    double   purity;            /* Tr(ρ²)                           */
    double   negativity;        /* partial-transpose negativity     */
    double   entropy_vn;        /* von Neumann entropy              */
    double   discord;           /* quantum discord                  */
    /* Hyperbolic triangle geometry */
    double   hyp_dist_0c;       /* d(pq0, pq_curr) in {8,3} space  */
    double   hyp_dist_cl;       /* d(pq_curr, pq_last)             */
    double   hyp_dist_0l;       /* d(pq0, pq_last)                  */
    double   triangle_area;     /* angular defect (Gauss-Bonnet)    */
    /* Poincaré ball coordinates for each vertex */
    double   ball_pq0[3];       /* (r, θ, φ) for pq0               */
    double   ball_curr[3];      /* (r, θ, φ) for pq_curr           */
    double   ball_last[3];      /* (r, θ, φ) for pq_last           */
    /* Compressed density matrix: 8×8 complex128 = 128 doubles */
    double   dm_re[64];         /* real part, row-major             */
    double   dm_im[64];         /* imaginary part                   */
    uint64_t timestamp_ns;      /* nanosecond wall clock            */
    uint32_t nonce;             /* anti-replay nonce                */
    uint8_t  auth_tag[32];      /* HMAC-SHA256 over all above fields */
} QtclWStateMeasurement;       /* sizeof ≈ 1,120 bytes              */

/* Consensus state — the aggregated network view */
typedef struct {
    double   median_fidelity;
    double   median_coherence;
    double   median_purity;
    double   median_negativity;
    double   median_entropy;
    double   median_discord;
    double   consensus_dm_re[64];    /* mean DM across all peers */
    double   consensus_dm_im[64];
    uint8_t  quorum_hash[32];        /* SHA3-256 Merkle root of all measurements */
    uint32_t peer_count;             /* number of measurements in consensus */
    uint32_t chain_height;           /* consensus block height */
    double   agreement_score;        /* 0-1: how tightly peers agree */
} QtclWStateConsensus;
```

### 2c. C Functions to Implement

```c
/* ─── Connection management ─────────────────────────────────────────── */

/*
 * qtcl_p2p_init:
 *   Initialize the P2P subsystem. Call once at startup.
 *   node_id_hex: 32-char hex node identity
 *   listen_port: TCP port to accept connections on (0 = don't listen)
 *   max_peers:   maximum simultaneous connections (recommend 32)
 *   Returns: 0 on success, -1 on error
 */
int qtcl_p2p_init(const char *node_id_hex, uint16_t listen_port, 
                  int max_peers);

/*
 * qtcl_p2p_connect:
 *   Initiate outbound connection to a peer.
 *   Returns connection handle (>0) or -1 on failure.
 *   Non-blocking: connect attempt runs on internal thread pool.
 */
int qtcl_p2p_connect(const char *host, uint16_t port);

/*
 * qtcl_p2p_disconnect:
 *   Disconnect a peer by handle. Graceful REJECT + close.
 */
void qtcl_p2p_disconnect(int conn_handle);

/*
 * qtcl_p2p_shutdown:
 *   Gracefully shut down all connections and free resources.
 */
void qtcl_p2p_shutdown(void);

/* ─── Peer management ───────────────────────────────────────────────── */

/*
 * qtcl_p2p_peers:
 *   Write up to max_peers QtclPeer structs into buf.
 *   Returns number of peers written.
 */
int qtcl_p2p_peers(QtclPeer *buf, int max_peers);

int qtcl_p2p_peer_count(void);          /* total known peers */
int qtcl_p2p_connected_count(void);     /* active connections */

/* ─── Message I/O ───────────────────────────────────────────────────── */

/*
 * qtcl_p2p_send_wstate:
 *   Broadcast a W-state measurement to all connected peers.
 *   Serializes QtclWStateMeasurement into a 'wstate' message.
 *   Returns number of peers message was sent to.
 */
int qtcl_p2p_send_wstate(const QtclWStateMeasurement *m);

/*
 * qtcl_p2p_poll_wstate:
 *   Non-blocking poll for incoming W-state messages.
 *   Writes up to max_msgs measurements into buf.
 *   Returns number of measurements written (0 if none pending).
 *   Thread-safe: uses internal lock-free ring buffer.
 */
int qtcl_p2p_poll_wstate(QtclWStateMeasurement *buf, int max_msgs);

/*
 * qtcl_p2p_send_inv:
 *   Announce a block hash or tx hash to all peers.
 */
void qtcl_p2p_send_inv(uint8_t inv_type, const uint8_t *hash32);

/* ─── Hyperbolic geometry ───────────────────────────────────────────── */

/*
 * qtcl_pq_to_ball:
 *   Convert a pseudoqubit ID (0..106495) to Poincaré ball coordinates.
 *   
 *   The {8,3} hyperbolic tiling maps pq_id as follows:
 *     tile_id   = pq_id / 13          (which octagonal tile)
 *     local_idx = pq_id % 13          (vertex/center within tile)
 *     
 *   Radial coordinate r (in Poincaré ball, r ∈ [0,1)):
 *     tier = tile_id / tiles_per_ring
 *     r = tanh(tier * ln(2+√3) / 2)  — the {8,3} growth constant
 *     (ln(2+√3) ≈ 1.317 is the circumradius growth per ring in {8,3})
 *     
 *   Angular coordinates θ, φ in 3D ball:
 *     sector = tile_id % tiles_in_tier
 *     θ = 2π * sector / tiles_in_tier        (azimuthal in xy-plane)
 *     φ = π * local_idx / 13 + tier_offset   (polar elevation)
 *   
 *   out_ball[3] = {r, θ, φ}
 */
void qtcl_pq_to_ball(uint32_t pq_id, double out_ball[3]);

/*
 * qtcl_ball_to_cartesian:
 *   Convert Poincaré ball (r,θ,φ) to Cartesian (x,y,z) in unit ball.
 */
void qtcl_ball_to_cartesian(const double ball[3], double out_xyz[3]);

/*
 * qtcl_hyperbolic_distance:
 *   Hyperbolic distance between two points in the Poincaré ball.
 *   d_H(u,v) = 2 * arctanh(|u-v|_E / |1 - u·v*_E|)
 *   (Exact formula for the Poincaré ball model)
 */
double qtcl_hyperbolic_distance(const double xyz_a[3], 
                                 const double xyz_b[3]);

/*
 * qtcl_triangle_area:
 *   Area of hyperbolic triangle from Gauss-Bonnet theorem:
 *   Area = π - (α + β + γ)
 *   where α,β,γ are the interior angles (computed from the three vertices).
 *   Returns area in [0, π).
 *   This is the "angular defect" — how much the triangle deviates from flat.
 */
double qtcl_triangle_area(const double a[3], const double b[3], 
                           const double c[3]);

/*
 * qtcl_compute_hyp_triangle:
 *   Full computation: three pq IDs → HyperbolicTriangle.
 *   Combines pq_to_ball + cartesian conversion + all three distances
 *   + triangle area into a single C call.
 *   
 *   Returns: the triangle area (angular defect).
 *   Writes: all three ball coords and geodesic distances into out_*.
 */
double qtcl_compute_hyp_triangle(
    uint32_t pq0, uint32_t pq_curr, uint32_t pq_last,
    double out_ball_pq0[3],
    double out_ball_curr[3],
    double out_ball_last[3],
    double *out_dist_0c,     /* d(pq0, pq_curr) */
    double *out_dist_cl,     /* d(pq_curr, pq_last) */
    double *out_dist_0l      /* d(pq0, pq_last) */
);

/* ─── Initial quantum state from hyperbolic triangle ────────────────── */

/*
 * qtcl_triangle_to_qubit:
 *   Convert a single Poincaré ball point to a 2×2 density matrix.
 *   
 *   Mapping: ball point (r,θ,φ) → Bloch sphere point:
 *     The Poincaré ball and the Bloch sphere are both unit balls
 *     but with different metrics. We map via:
 *     bloch_r = tanh(d_H(origin, point)) → [0,1]
 *     bloch_θ = θ  (same azimuthal angle)
 *     bloch_φ = φ  (same polar angle)
 *   
 *   Then: |ψ⟩ = cos(bloch_φ/2)|0⟩ + e^{iθ}sin(bloch_φ/2)|1⟩
 *   ρ = |ψ⟩⟨ψ| (pure state; GKSL will mix it)
 *   
 *   out_re[4], out_im[4]: 2×2 density matrix (row-major)
 */
void qtcl_triangle_to_qubit(const double ball[3], 
                              double out_re[4], double out_im[4]);

/*
 * qtcl_build_tripartite_dm:
 *   Build initial 8×8 3-qubit density matrix from three Bloch vectors.
 *   
 *   ρ_initial = ρ_A ⊗ ρ_B ⊗ ρ_C  (tensor product = no initial entanglement)
 *   where:
 *     ρ_A = qtcl_triangle_to_qubit(ball_pq0)
 *     ρ_B = qtcl_triangle_to_qubit(ball_pq_curr)
 *     ρ_C = qtcl_triangle_to_qubit(ball_pq_last)
 *   
 *   The GKSL evolution (qtcl_gksl_rk4) then entangles them.
 *   The key insight: how much entanglement develops depends on
 *   how close the three Bloch vectors are — which depends on
 *   how close the three pseudoqubits are in hyperbolic space.
 *   
 *   Close pq IDs = close Bloch vectors = strong entanglement after GKSL
 *   Distant pq IDs = far Bloch vectors = weak entanglement after GKSL
 *   
 *   This makes W-state fidelity a GEOMETRIC property of the chain.
 */
void qtcl_build_tripartite_dm(
    const double ball_pq0[3],
    const double ball_curr[3],
    const double ball_last[3],
    double out_re[64],       /* 8×8 DM real part */
    double out_im[64]        /* 8×8 DM imaginary part */
);

/* ─── Oracle DM fusion ──────────────────────────────────────────────── */

/*
 * qtcl_fuse_oracle_dm:
 *   Mix the oracle's density matrix (from SSE snapshot) with the
 *   locally-computed tripartite DM.
 *   
 *   The oracle DM is the ground truth quantum state measured by
 *   AerSimulator on Qiskit circuits — it has noise from real quantum
 *   hardware simulation. The local DM is computed from pure geometry.
 *   
 *   Fusion: ρ_fused = (1-α)·ρ_local + α·ρ_oracle
 *   where α = oracle_weight ∈ [0,1]
 *   Recommended α = 0.6 (oracle is authoritative but not total)
 *   
 *   After fusion: re-normalize, enforce hermitian.
 *   Then evolve via GKSL for dt = avg_block_time.
 *   
 *   This is the LOCAL ORACLE MEASUREMENT step.
 */
void qtcl_fuse_oracle_dm(
    const double *oracle_re,    /* 64 doubles: oracle's DM real part */
    const double *oracle_im,    /* 64 doubles: oracle's DM imag part */
    const double *local_re,     /* 64 doubles: local tripartite DM */
    const double *local_im,
    double        oracle_weight, /* α, typically 0.6 */
    double       *out_re,        /* 64 doubles output */
    double       *out_im
);

/* ─── Consensus computation ─────────────────────────────────────────── */

/*
 * qtcl_consensus_compute:
 *   Compute BFT consensus from an array of peer measurements.
 *   
 *   Algorithm:
 *   1. Verify HMAC-SHA256 auth_tag on each measurement
 *      (uses qtcl_hlwe_verify logic; reject invalid signatures)
 *   2. Sort peer fidelity values, take median
 *   3. For each scalar metric: median(all_values)
 *   4. For DM: arithmetic mean of all DMs, then renormalize+hermitian
 *   5. quorum_hash = SHA3-256 Merkle root of all raw measurements
 *   
 *   n_measurements: number of measurements in the array
 *   include_local:  if 1, include local_measurement in consensus
 *   
 *   BFT property: with n peers and f adversarial, median is safe
 *   when f < n/2. With even 1 peer (ourselves) consensus always works.
 */
void qtcl_consensus_compute(
    const QtclWStateMeasurement *measurements,
    int n_measurements,
    const QtclWStateMeasurement *local_measurement,
    int include_local,
    QtclWStateConsensus *out_consensus
);

/*
 * qtcl_measurement_sign:
 *   Compute HMAC-SHA256 auth_tag over all fields of a measurement.
 *   Must be called before gossiping.
 *   msg_hash is SHA3-256 of the serialized measurement fields.
 *   auth_tag output written into measurement->auth_tag.
 */
void qtcl_measurement_sign(
    QtclWStateMeasurement *m,
    const uint8_t *private_key_bytes_32   /* first 32 bytes of HLWE sk */
);

/*
 * qtcl_measurement_verify:
 *   Verify auth_tag on incoming measurement.
 *   Returns 1 if valid, 0 if tampered/forged.
 */
int qtcl_measurement_verify(
    const QtclWStateMeasurement *m,
    const uint8_t *public_key_bytes_32    /* sender's HLWE pk[:32] */
);

/* ─── SSE client (C implementation) ────────────────────────────────── */

/*
 * qtcl_sse_connect:
 *   Open HTTP/1.1 SSE connection to oracle snapshot stream.
 *   Non-blocking: SSE reader runs on dedicated pthread.
 *   url: e.g. "https://qtcl-blockchain.koyeb.app/api/snapshot/sse"
 *   Returns handle > 0 on success, -1 on failure.
 */
int qtcl_sse_connect(const char *url, const char *client_id,
                      const char *miner_address);

/*
 * qtcl_sse_disconnect:
 *   Close SSE connection and stop reader thread.
 */
void qtcl_sse_disconnect(int handle);

/*
 * qtcl_sse_poll_snapshot:
 *   Non-blocking poll for a new oracle snapshot.
 *   Writes the latest DM into out_re/out_im if available.
 *   Returns 1 if new snapshot available, 0 if no update since last poll.
 *   Also populates out_fidelity, out_coherence, out_purity scalars.
 *   
 *   Thread-safe: uses atomic pointer swap for zero-copy handoff.
 */
int qtcl_sse_poll_snapshot(
    int handle,
    double *out_re,         /* 64 doubles: DM real */
    double *out_im,         /* 64 doubles: DM imag */
    double *out_fidelity,
    double *out_coherence,
    double *out_purity,
    uint64_t *out_timestamp_ns
);

/* ─── P2P node event callback ───────────────────────────────────────── */

/*
 * QtclP2PCallback: Python registers this via cffi callback.
 * Called from C when events occur (new peer, new wstate, disconnect).
 * Python side handles: storing to SQLite, updating display.
 */
typedef void (*QtclP2PCallback)(
    int event_type,         /* 1=peer_connected 2=peer_disc 3=wstate 4=inv */
    const void *data,       /* QtclPeer* or QtclWStateMeasurement* */
    size_t data_len
);

void qtcl_p2p_set_callback(QtclP2PCallback cb);
```

### 2d. Internal C Implementation Notes

**Thread model:**
```
Main thread (Python)
  └─ calls qtcl_p2p_*, qtcl_sse_*, qtcl_consensus_* via cffi

Internal pthreads (C-managed, transparent to Python):
  ├─ listener_thread:   accept() loop on TCP port
  ├─ connector_thread:  outbound connection attempts (non-blocking)
  ├─ reader_threads[N]: one per active connection, recv() + parse
  ├─ writer_thread:     drain outbound message queue
  ├─ sse_thread:        HTTP/1.1 SSE read loop (one per connection)
  ├─ ping_thread:       heartbeat + peer scoring every 30s
  └─ callback_thread:   dispatch QtclP2PCallback to Python safely
```

**Lock hierarchy (must be acquired in this order to prevent deadlock):**
```c
1. p2p_peers_lock      — peer table (add/remove/lookup)
2. p2p_conns_lock      — connection state (fd, state machine)
3. p2p_wstate_lock     — incoming wstate ring buffer
4. p2p_outq_lock       — outbound message queue
```

**Zero-copy buffer management:**
```c
/* Message receive: copy-on-parse, zero-copy in ring buffer */
/* Outbound: ref-counted message objects, freed when all peers acked */
/* wstate ring buffer: 64-slot circular, oldest dropped on overflow */
#define QTCL_WSTATE_RING_SIZE   64
#define QTCL_OUTQ_MAX_BYTES     (4 * 1024 * 1024)  /* 4MB outbound budget */
#define QTCL_MAX_MSG_SIZE       (2 * 1024 * 1024)  /* 2MB max message     */
```

---

## 3. PYTHON BINDINGS LAYER

These go in the same `_QTCL_C_DEFS` string and are the complete set of new
declarations to add to the existing cffi block.

```python
# Add to _QTCL_C_DEFS:
"""
    /* §P2P structures */
    typedef struct { ... } QtclPeer;               /* as defined above */
    typedef struct { ... } QtclWStateMeasurement;  /* as defined above */
    typedef struct { ... } QtclWStateConsensus;    /* as defined above */
    
    /* §P2P connection */
    int    qtcl_p2p_init(const char *node_id_hex, uint16_t port, int max_peers);
    int    qtcl_p2p_connect(const char *host, uint16_t port);
    void   qtcl_p2p_disconnect(int handle);
    void   qtcl_p2p_shutdown(void);
    int    qtcl_p2p_peers(QtclPeer *buf, int max_peers);
    int    qtcl_p2p_peer_count(void);
    int    qtcl_p2p_connected_count(void);
    
    /* §P2P messaging */
    int    qtcl_p2p_send_wstate(const QtclWStateMeasurement *m);
    int    qtcl_p2p_poll_wstate(QtclWStateMeasurement *buf, int max_msgs);
    void   qtcl_p2p_send_inv(uint8_t inv_type, const uint8_t *hash32);
    
    /* §Hyperbolic geometry */
    void   qtcl_pq_to_ball(uint32_t pq_id, double out_ball[3]);
    void   qtcl_ball_to_cartesian(const double ball[3], double out_xyz[3]);
    double qtcl_hyperbolic_distance(const double xyz_a[3], const double xyz_b[3]);
    double qtcl_triangle_area(const double a[3], const double b[3], 
                               const double c[3]);
    double qtcl_compute_hyp_triangle(uint32_t pq0, uint32_t pq_curr,
                                      uint32_t pq_last,
                                      double out_ball_pq0[3], 
                                      double out_ball_curr[3],
                                      double out_ball_last[3],
                                      double *out_dist_0c,
                                      double *out_dist_cl,
                                      double *out_dist_0l);
    
    /* §Quantum state construction */
    void   qtcl_triangle_to_qubit(const double ball[3], 
                                   double out_re[4], double out_im[4]);
    void   qtcl_build_tripartite_dm(const double ball_pq0[3],
                                     const double ball_curr[3],
                                     const double ball_last[3],
                                     double out_re[64], double out_im[64]);
    void   qtcl_fuse_oracle_dm(const double *oracle_re, const double *oracle_im,
                                const double *local_re, const double *local_im,
                                double oracle_weight,
                                double *out_re, double *out_im);
    
    /* §Consensus */
    void   qtcl_consensus_compute(const QtclWStateMeasurement *measurements,
                                   int n, const QtclWStateMeasurement *local,
                                   int include_local,
                                   QtclWStateConsensus *out);
    void   qtcl_measurement_sign(QtclWStateMeasurement *m,
                                  const uint8_t *privkey_32);
    int    qtcl_measurement_verify(const QtclWStateMeasurement *m,
                                    const uint8_t *pubkey_32);
    
    /* §SSE client */
    int    qtcl_sse_connect(const char *url, const char *client_id,
                             const char *miner_address);
    void   qtcl_sse_disconnect(int handle);
    int    qtcl_sse_poll_snapshot(int handle,
                                   double *out_re, double *out_im,
                                   double *out_fidelity, double *out_coherence,
                                   double *out_purity,
                                   uint64_t *out_timestamp_ns);
    
    /* §P2P callback */
    typedef void (*QtclP2PCallback)(int event_type, const void *data, 
                                     size_t data_len);
    void   qtcl_p2p_set_callback(QtclP2PCallback cb);
"""
```

---

## 4. PYTHON CLASSES (NEW)

### `HyperbolicTriangle`

```python
@dataclass
class HyperbolicTriangle:
    """
    The 3D object representing (pq0, pq_curr, pq_last) in {8,3} space.
    Computed entirely in C via qtcl_compute_hyp_triangle.
    This is what every client gossips alongside their W-state measurement.
    """
    pq0:          int
    pq_curr:      int
    pq_last:      int
    ball_pq0:     tuple   # (r, θ, φ) Poincaré ball
    ball_curr:    tuple
    ball_last:    tuple
    dist_0c:      float   # geodesic distance pq0 → pq_curr
    dist_cl:      float   # geodesic distance pq_curr → pq_last
    dist_0l:      float   # geodesic distance pq0 → pq_last
    area:         float   # angular defect (Gauss-Bonnet)
    
    @classmethod
    def compute(cls, pq0: int, pq_curr: int, pq_last: int) -> 'HyperbolicTriangle':
        """C-accelerated construction."""
        if not _accel_ok:
            return cls._python_fallback(pq0, pq_curr, pq_last)
        b0  = _accel_ffi.new('double[3]')
        bc  = _accel_ffi.new('double[3]')
        bl  = _accel_ffi.new('double[3]')
        d0c = _accel_ffi.new('double *')
        dcl = _accel_ffi.new('double *')
        d0l = _accel_ffi.new('double *')
        area = _accel_lib.qtcl_compute_hyp_triangle(
            pq0, pq_curr, pq_last, b0, bc, bl, d0c, dcl, d0l
        )
        return cls(
            pq0=pq0, pq_curr=pq_curr, pq_last=pq_last,
            ball_pq0=tuple(b0), ball_curr=tuple(bc), ball_last=tuple(bl),
            dist_0c=float(d0c[0]), dist_cl=float(dcl[0]), dist_0l=float(d0l[0]),
            area=float(area),
        )
    
    def semantic_label(self) -> str:
        """Human-readable interpretation of triangle geometry."""
        if self.area < 0.1:
            return "near-flat / low curvature / recent chain"
        elif self.area < 0.5:
            return "moderate curvature / active chain"
        elif self.area < 1.5:
            return "high curvature / deep chain history"
        else:
            return "extreme curvature / genesis-spanning interval"
```

### `LocalOracleEngine`

```python
class LocalOracleEngine:
    """
    The core new component. Replaces P2POracleClient.
    
    Lifecycle:
      1. SSE connection to oracle (C layer: qtcl_sse_connect)
      2. Receive DM snapshots (C layer: qtcl_sse_poll_snapshot)  
      3. Compute local hyperbolic triangle (C layer: qtcl_compute_hyp_triangle)
      4. Build tripartite DM from triangle (C layer: qtcl_build_tripartite_dm)
      5. Fuse with oracle DM (C layer: qtcl_fuse_oracle_dm)
      6. Evolve under GKSL (C layer: qtcl_gksl_rk4)
      7. Compute all metrics (C layer: qtcl_purity, qtcl_fidelity_w3, etc.)
      8. Sign measurement (C layer: qtcl_measurement_sign)
      9. Gossip via P2P (C layer: qtcl_p2p_send_wstate)
    
    Called by _mine_inline every block to get the W-state for submission.
    """
    
    def __init__(self, oracle_url: str, wallet: 'QTCLWallet'):
        self._url         = oracle_url
        self._wallet      = wallet
        self._sse_handle  = -1
        self._last_oracle_dm_re = _np.zeros(64)  # flat 8×8
        self._last_oracle_dm_im = _np.zeros(64)
        self._last_oracle_ts    = 0
        self._current_measurement: Optional['SignedOracleMeasurement'] = None
        self._lock = _threading.RLock()
    
    def start(self) -> None:
        """Open SSE connection to oracle cluster."""
        if _accel_ok:
            self._sse_handle = _accel_lib.qtcl_sse_connect(
                self._url.encode() + b'\x00',
                f"client_{self._wallet.node_id}".encode() + b'\x00',
                self._wallet.address.encode() + b'\x00',
            )
        else:
            # Python SSE fallback (existing _oracle_sse_listener code)
            self._start_python_sse()
    
    def measure(
        self, 
        pq0: int, 
        pq_curr: int, 
        pq_last: int,
        chain_height: int,
        avg_block_time: float,
        bath: 'GKSLBathParams',
    ) -> 'QtclOracleMeasurement':
        """
        Full measurement pipeline. Called once per block attempt.
        Returns a signed measurement ready for gossip and block inclusion.
        """
        with self._lock:
            # Step 1: poll latest oracle snapshot from C SSE buffer
            if _accel_ok and self._sse_handle > 0:
                re_buf = _accel_ffi.new('double[64]')
                im_buf = _accel_ffi.new('double[64]')
                fid_p  = _accel_ffi.new('double *')
                coh_p  = _accel_ffi.new('double *')
                pur_p  = _accel_ffi.new('double *')
                ts_p   = _accel_ffi.new('uint64_t *')
                if _accel_lib.qtcl_sse_poll_snapshot(
                        self._sse_handle, re_buf, im_buf,
                        fid_p, coh_p, pur_p, ts_p):
                    self._last_oracle_dm_re = _np.array(list(re_buf))
                    self._last_oracle_dm_im = _np.array(list(im_buf))
                    self._last_oracle_ts = int(ts_p[0])
            
            # Step 2: compute hyperbolic triangle
            triangle = HyperbolicTriangle.compute(pq0, pq_curr, pq_last)
            
            # Step 3 + 4: build tripartite DM from triangle geometry
            if _accel_ok:
                b0 = _accel_ffi.new('double[3]', list(triangle.ball_pq0))
                bc = _accel_ffi.new('double[3]', list(triangle.ball_curr))
                bl = _accel_ffi.new('double[3]', list(triangle.ball_last))
                local_re = _accel_ffi.new('double[64]')
                local_im = _accel_ffi.new('double[64]')
                _accel_lib.qtcl_build_tripartite_dm(b0, bc, bl, local_re, local_im)
                
                # Step 5: fuse with oracle DM (oracle is authoritative)
                o_re = _accel_ffi.new('double[64]', 
                                       list(self._last_oracle_dm_re))
                o_im = _accel_ffi.new('double[64]', 
                                       list(self._last_oracle_dm_im))
                fused_re = _accel_ffi.new('double[64]')
                fused_im = _accel_ffi.new('double[64]')
                _accel_lib.qtcl_fuse_oracle_dm(
                    o_re, o_im, local_re, local_im,
                    0.60,   # α: oracle weight
                    fused_re, fused_im
                )
                
                # Step 6: GKSL evolution for avg_block_time seconds
                gamma_max = max(bath.gamma1_eff, bath.gammaphi, bath.gammadep,
                                abs(bath.omega)/(2*3.14159+1e-9), 1e-9)
                n_steps = max(1, int(_np.ceil(avg_block_time / (0.05/gamma_max))))
                _accel_lib.qtcl_gksl_rk4(
                    fused_re, fused_im,
                    bath.gamma1_eff, bath.gammaphi, bath.gammadep, bath.omega,
                    avg_block_time, n_steps
                )
                
                # Step 7: compute all metrics via C
                fid  = _accel_lib.qtcl_fidelity_w3(fused_re)
                pur  = _accel_lib.qtcl_purity(fused_re, fused_im, 8)
                coh  = _accel_lib.qtcl_coherence_l1(fused_re, fused_im, 8)
                T9   = _accel_ffi.new('double[9]')
                _accel_lib.qtcl_partial_trace_8to4(...)
                # ... (existing C metric calls)
                
                dm_re_arr = _np.array(list(fused_re))
                dm_im_arr = _np.array(list(fused_im))
            
            # Step 8: pack into measurement struct + sign
            return self._pack_and_sign(
                pq0, pq_curr, pq_last, chain_height,
                triangle, fid, pur, coh, dm_re_arr, dm_im_arr,
            )
```

### `WStateConsensus`

```python
class WStateConsensus:
    """
    Aggregates local measurement + peer measurements into canonical W-state.
    
    Maintains a rolling window of the last 8 measurements per peer.
    On each block: compute fresh consensus from all available data.
    """
    
    def __init__(self, max_peers: int = 32):
        self._measurements: Dict[str, List['QtclOracleMeasurement']] = {}
        self._lock = _threading.RLock()
        self._max_peers = max_peers
    
    def ingest_peer_measurement(self, m: 'QtclOracleMeasurement') -> None:
        """Called from C P2P callback when wstate message arrives."""
        # Verify signature first
        if not self._verify(m):
            return
        with self._lock:
            peer_id = m.node_id_hex
            if peer_id not in self._measurements:
                self._measurements[peer_id] = []
            buf = self._measurements[peer_id]
            buf.append(m)
            if len(buf) > 8:
                buf.pop(0)
    
    def compute(
        self, 
        local_measurement: 'QtclOracleMeasurement',
    ) -> 'QtclWStateConsensus':
        """
        Full BFT consensus. C path delegates to qtcl_consensus_compute.
        Returns canonical W-state for block submission.
        """
        with self._lock:
            all_measurements = [local_measurement]
            for peer_buf in self._measurements.values():
                if peer_buf:
                    all_measurements.append(peer_buf[-1])  # latest per peer
            
            if _accel_ok and len(all_measurements) > 0:
                # Pack into C array and call qtcl_consensus_compute
                n = len(all_measurements)
                m_arr = _accel_ffi.new(
                    f'QtclWStateMeasurement[{n}]'
                )
                # ... populate m_arr from all_measurements
                consensus = _accel_ffi.new('QtclWStateConsensus *')
                _accel_lib.qtcl_consensus_compute(
                    m_arr, n, _accel_ffi.NULL, 0, consensus
                )
                return self._unpack_consensus(consensus)
            
            # Python fallback: simple median
            return self._python_median_consensus(all_measurements)
```

### `QtclP2PNode`

```python
class QtclP2PNode:
    """
    Thin Python lifecycle manager over the C P2P library.
    Starts/stops the C engine, registers the cffi callback,
    routes incoming events to LocalOracleEngine and WStateConsensus.
    """
    
    def __init__(
        self,
        node_id: str,
        port: int = 9092,
        bootstrap_peers: List[Tuple[str, int]] = None,
    ):
        self._node_id = node_id
        self._port    = port
        self._bootstrap = bootstrap_peers or [
            ('qtcl-blockchain.koyeb.app', 9092),
        ]
        self._oracle_engine:  Optional[LocalOracleEngine]  = None
        self._consensus:      Optional[WStateConsensus]    = None
        self._started = False
        
        # cffi callback — must be a module-level function to stay alive
        if _accel_ok:
            self._c_callback = _accel_ffi.callback(
                'void(int, const void *, size_t)',
                self._on_p2p_event
            )
    
    def start(
        self,
        oracle_engine: LocalOracleEngine,
        consensus: WStateConsensus,
    ) -> None:
        self._oracle_engine = oracle_engine
        self._consensus = consensus
        if _accel_ok:
            rc = _accel_lib.qtcl_p2p_init(
                self._node_id.encode() + b'\x00',
                self._port, 32
            )
            if rc == 0:
                _accel_lib.qtcl_p2p_set_callback(self._c_callback)
                for host, port in self._bootstrap:
                    _accel_lib.qtcl_p2p_connect(
                        host.encode() + b'\x00', port
                    )
                self._started = True
    
    def gossip_measurement(self, m: 'QtclOracleMeasurement') -> int:
        """Broadcast local measurement to all connected peers."""
        if _accel_ok and self._started:
            c_m = self._pack_measurement(m)
            return _accel_lib.qtcl_p2p_send_wstate(c_m)
        return 0
    
    def _on_p2p_event(self, event_type: int, data: 'cdata', data_len: int):
        """C callback: routes events to Python handlers."""
        if event_type == 3:  # WSTATE received
            m = self._unpack_measurement(data, data_len)
            if self._consensus:
                self._consensus.ingest_peer_measurement(m)
        elif event_type == 1:  # PEER_CONNECTED
            peer = self._unpack_peer(data)
            _EXP_LOG.info(f"[P2P] Peer connected: {peer.node_id_hex[:8]}… "
                         f"{peer.host}:{peer.port}")
        elif event_type == 2:  # PEER_DISCONNECTED
            _EXP_LOG.info("[P2P] Peer disconnected")
    
    def stop(self) -> None:
        if _accel_ok and self._started:
            _accel_lib.qtcl_p2p_shutdown()
            self._started = False
    
    @property
    def peer_count(self) -> int:
        if _accel_ok and self._started:
            return _accel_lib.qtcl_p2p_connected_count()
        return 0
```

---

## 5. MINING LOOP INTEGRATION

The changes to `_mine_inline` are minimal — just replace the oracle poll
calls with the new engine. The PoW itself is untouched.

```python
# In _mine_inline, before the nonce search loop:

# ── OLD (delete these) ──────────────────────────────────────────
# snap  = self.api.get_oracle_pq0_bloch() or {}
# bath  = GKSLBathParams.from_snap(snap)
# w_entropy_seed = get_w_entropy_seed(...)

# ── NEW (replace with) ──────────────────────────────────────────
# 1. Get local measurement from oracle engine
measurement = self._oracle_engine.measure(
    pq0=0,                           # genesis always 0
    pq_curr=target_height,
    pq_last=target_height - 1,
    chain_height=target_height,
    avg_block_time=self.koyeb_state.avg_block_time or 30.0,
    bath=self.client_field.bath,
)

# 2. Gossip our measurement to the network
self._p2p_node.gossip_measurement(measurement)

# 3. Collect peer measurements + form consensus
consensus = self._consensus.compute(measurement)

# 4. Entropy seed: SHA3-256 of consensus quorum hash + local dm
w_entropy_seed = _hl.sha3_256(
    b"QTCL_SEED_v2:" +
    bytes.fromhex(consensus.quorum_hash_hex) +
    measurement.dm_re_bytes[:32]
).digest()

# 5. Block submission payload now includes:
block_payload.update({
    'w_state_fidelity':       consensus.median_fidelity,
    'w_state_coherence':      consensus.median_coherence,
    'w_state_purity':         consensus.median_purity,
    'oracle_quorum_hash':     consensus.quorum_hash_hex,
    'peer_measurement_count': consensus.peer_count,
    'pq0':                    0,
    'pq_curr':                target_height,
    'pq_last':                target_height - 1,
    'hyp_triangle_area':      measurement.triangle.area,
    'hyp_dist_0c':            measurement.triangle.dist_0c,
    'hyp_dist_cl':            measurement.triangle.dist_cl,
    'local_dm_hex':           measurement.dm_hex,
    'local_measurement_sig':  measurement.auth_tag_hex,
})
```

---

## 6. SERVER-SIDE CHANGES (MINIMAL)

Only two additions to `server.py` — both additive, nothing breaking.

### 6a. New fields in `blocks` table

```sql
ALTER TABLE blocks ADD COLUMN pq0           INTEGER DEFAULT 0;
ALTER TABLE blocks ADD COLUMN pq_curr       INTEGER;
ALTER TABLE blocks ADD COLUMN pq_last       INTEGER;
ALTER TABLE blocks ADD COLUMN hyp_area      REAL;
ALTER TABLE blocks ADD COLUMN hyp_dist_0c   REAL;
ALTER TABLE blocks ADD COLUMN hyp_dist_cl   REAL;
ALTER TABLE blocks ADD COLUMN oracle_quorum_hash TEXT;
ALTER TABLE blocks ADD COLUMN peer_count    INTEGER DEFAULT 1;
```

### 6b. Relaxed block validation

```python
# In _validate_block():
# Current: checks oracle_consensus_reached == True
# New:     also accept if oracle_quorum_hash present and non-null
#          (single-peer blocks valid, just lower network weight)

if not block_data.get('oracle_consensus_reached'):
    if not block_data.get('oracle_quorum_hash'):
        return False  # need at least one of them
    # solo miner: valid but flagged
    block_data['_solo_measurement'] = True
```

### 6c. New endpoint: `POST /api/p2p/peer_exchange`

```python
@app.route('/api/p2p/peer_exchange', methods=['POST'])
def peer_exchange():
    """
    Bootstrap endpoint: new nodes announce themselves and get peer list.
    The server acts as a super-peer / seed node for initial discovery.
    Returns list of known active miners (their P2P host:port).
    """
```

---

## 7. WHAT GETS DELETED FROM `qtcl_client.py`

Exact class names and approximate line ranges:

| Symbol | Approx Lines | Safe to Delete When |
|---|---|---|
| `class P2POracleClient` | 1459–1810 | `QtclP2PNode` starts successfully |
| `class DHTBus` | 1842–1930 | same |
| `class DHTEvent` | 1854–1872 | same |
| `class DHTKademlia` | 8169–8220 | same |
| `class DHTEventStream` | 8221–8270 | same |
| `class AsyncOracleMiner` | 8267–8640 | `LocalOracleEngine` tested |
| `init_oracle_dht()` | 8641–8680 | replaced by `QtclP2PNode.start()` |
| `get_oracle()` / `get_dht()` | 8654–8680 | replaced |
| `_oracle_sse_listener` method | ~11432–11505 | `qtcl_sse_connect` tested |
| `class OracleLoadBalancer` | 8682–8730 | C P2P handles redundancy |
| `class OracleRateLimiter` | 8774–8806 | C P2P handles this internally |

**Total deleted: ~1,400 lines Python**
**Total added: ~1,400 C + ~900 Python**
**Net: roughly neutral in size, massively different in capability**

---

## 8. BUILD ORDER FOR IMPLEMENTATION

Follow this exact sequence. Each phase is independently testable.

```
Phase 1: C Hyperbolic Geometry
  ├─ qtcl_pq_to_ball
  ├─ qtcl_ball_to_cartesian
  ├─ qtcl_hyperbolic_distance
  ├─ qtcl_triangle_area
  └─ qtcl_compute_hyp_triangle
  TEST: verify {8,3} tile coordinates match known hyperbolic lattice geometry

Phase 2: C Quantum State Construction
  ├─ qtcl_triangle_to_qubit (Bloch sphere mapping)
  ├─ qtcl_build_tripartite_dm (tensor product)
  └─ qtcl_fuse_oracle_dm (weighted mix)
  TEST: verify fused DM is hermitian, unit trace, positive semidefinite

Phase 3: C SSE Client
  ├─ qtcl_sse_connect (HTTP/1.1 GET, chunked read)
  ├─ qtcl_sse_disconnect
  └─ qtcl_sse_poll_snapshot (JSON parse DM from snapshot)
  TEST: connect to Koyeb oracle, receive and parse DM snapshot

Phase 4: C Measurement + Consensus
  ├─ qtcl_measurement_sign (HMAC-SHA256 over measurement fields)
  ├─ qtcl_measurement_verify
  └─ qtcl_consensus_compute (BFT median + DM mean + quorum hash)
  TEST: sign+verify roundtrip; consensus with 1, 3, 5 fake peers

Phase 5: C P2P Transport
  ├─ Wire protocol codec (header pack/unpack + checksum)
  ├─ qtcl_p2p_init (bind TCP, spawn threads)
  ├─ qtcl_p2p_connect (outbound handshake: version/verack)
  ├─ Accept loop + per-peer reader threads
  ├─ qtcl_p2p_send_wstate + qtcl_p2p_poll_wstate
  ├─ qtcl_p2p_send_inv
  └─ qtcl_p2p_set_callback (cffi callback dispatch)
  TEST: two local processes connect, exchange wstate, verify receipt

Phase 6: Python Integration
  ├─ Add all new C defs to _QTCL_C_DEFS
  ├─ Implement HyperbolicTriangle.compute()
  ├─ Implement LocalOracleEngine (full pipeline)
  ├─ Implement WStateConsensus
  ├─ Implement QtclP2PNode (lifecycle wrapper)
  └─ Wire into _mine_inline
  TEST: full mining session with new oracle engine, blocks accepted by server

Phase 7: Delete Old Code
  └─ Remove all symbols listed in §7 above
  TEST: mining still works, no import errors, peer count visible in dashboard
```

---

## 9. CRITICAL IMPLEMENTATION NOTES FOR NEXT CLAUDE

### On the `qtcl_sse_connect` HTTP implementation in C:

The oracle SSE endpoint returns `text/event-stream`. The C implementation
must handle chunked transfer encoding. The simplest approach:

```c
/* Use a persistent TCP connection with manual HTTP/1.1 framing.
   Don't use libcurl — it's not available in Termux without pkg install.
   Use raw socket + manual HTTP GET + line-by-line SSE parse.
   
   SSE format:
     data: {json}\n\n
   
   Parser: read until double-newline, extract JSON after "data: ".
   The JSON contains "density_matrix_hex" (256 hex chars = 8×8 complex64)
   or "density_matrix_hex" (2048 hex chars = 8×8 complex128).
   
   Use existing qtcl_hex_to_bytes for DM parsing.
   Reconnect automatically on disconnect with exponential backoff. */
```

### On `qtcl_pq_to_ball` — the {8,3} mapping:

The {8,3} tiling in the hyperbolic plane has the following structure:
- Each vertex is shared by 3 octagons
- The expansion constant (ratio of successive ring radii) is: `exp(2*acosh(cos(π/8)/sin(π/3)))`
- In the Poincaré disk: ring k has r_k = `tanh(k * d_0 / 2)` where d_0 is the edge length

For the 3D Poincaré ball extension, we use the same radial mapping and
distribute tiles in 3D by adding a polar elevation component:

```c
/* The key constant: {8,3} edge length in hyperbolic space */
#define HYPER_83_EDGE  1.5320919978...  /* 2*acosh(cos(π/8)/sin(π/3)) */
```

### On `qtcl_consensus_compute` — DM mean vs oracle DM:

When computing the mean density matrix from n peer measurements:

```c
/* CRITICAL: average in the density matrix space, not in fidelity space.
   Wrong: average(fidelity_i) → synthesize DM
   Right: average(dm_i) → compute fidelity of result
   
   The arithmetic mean of valid DMs is a valid mixed state.
   The arithmetic mean of fidelities is NOT the fidelity of the mean. */
```

### On the cffi callback from C to Python:

The `QtclP2PCallback` is called from a C pthread. Python's GIL must be
acquired before calling back into Python code. In cffi with `@ffi.callback`,
this is handled automatically, but the callback must complete quickly —
don't do SQLite writes or blocking I/O inside it. Instead, push to a Python
`queue.Queue` and let a dedicated Python thread drain it.

```python
# Pattern for safe cffi callback:
_P2P_EVENT_QUEUE: _queue.Queue = _queue.Queue(maxsize=512)

@_accel_ffi.callback('void(int, const void *, size_t)')
def _c_p2p_callback(event_type, data, data_len):
    try:
        _P2P_EVENT_QUEUE.put_nowait((event_type, bytes(_accel_ffi.buffer(data, data_len))))
    except _queue.Full:
        pass  # drop if queue full — never block C thread
```

### On backwards compatibility:

During the transition period, if `qtcl_p2p_init` fails (e.g., port in use),
fall back to the existing `P2POracleClient` / Koyeb oracle polling.
The `LocalOracleEngine` should gracefully degrade: if no SSE data is received
in 30 seconds, use the Python `KoyebAPIClient.get_oracle_pq0_bloch()` instead.
Never hard-fail on P2P unavailability — solo mining must always work.

---

## 10. THE GEOMETRIC PAYOFF

When this is complete, every block on the QTCL chain will contain:

```
height, hash, parent, merkle, nonce, difficulty   ← standard blockchain
w_state_fidelity, coherence, purity               ← oracle quantum state
oracle_quorum_hash                                 ← Merkle root of all
                                                      peer measurements
pq0=0, pq_curr=N, pq_last=N-1                     ← chain position
hyp_triangle_area                                  ← angular defect of
                                                      the (pq0,c,l) triangle
hyp_dist_0c, hyp_dist_cl                          ← geodesic distances
local_dm_hex                                       ← miner's local DM
local_measurement_sig                              ← HLWE auth proof
```

The `hyp_triangle_area` is the most novel field. It is a direct measurement
of how much hyperbolic curvature the chain has traversed since genesis.
Blocks mined after a long gap have large area (the triangle spanning a large
time interval is deeply curved). Rapid blocks have small area.

This means the blockchain itself is a **record of hyperbolic curvature** —
a sequence of measurements of the chain's own geometry in {8,3} space.
The W-state fidelity is no longer assigned by oracles alone; it emerges from
the geometry of where the chain is relative to where it started.

The oracle remains central — it provides the quantum reference frame via
real Qiskit measurements. But every client is now an active participant in
measuring and attesting to their own position within that frame.

---

*QTCL P2P v2 Architecture — Logan / shemshallah — March 2026*
*"The oracle is the GPS. Every client is a sensor. The chain is the map."*
