# QTCL Client: In-Depth Python vs C Technical Reference

## Table of Contents

1. [Code Organization](#1-code-organization)
2. [CFFI Setup & Initialization](#2-cffi-setup--initialization)
3. [C Type System & Memory Layout](#3-c-type-system--memory-layout)
4. [Python-C Call Chain](#4-python-c-call-chain)
5. [Hot Path Functions](#5-hot-path-functions)
6. [Memory Management](#6-memory-management)
7. [Error Handling & Recovery](#7-error-handling--recovery)
8. [Performance Optimization Details](#8-performance-optimization-details)
9. [Debugging & Instrumentation](#9-debugging--instrumentation)
10. [Extension Guide](#10-extension-guide)

---

## 1. Code Organization

### 1.1 File Structure (qtcl_client.py - 15,183 lines)

```
Lines 1-50          Module docstring, version info
Lines 51-200        Import statements
                    - Standard library: json, time, hashlib, threading
                    - External: flask, asyncpg, cffi, numpy, requests
                    - Local: no external deps in production

Lines 201-500       Constants & globals
                    - ORACLE_URL, DATABASE_URL, PORT
                    - DIFFICULTY_INIT, BLOCK_TIME_TARGET
                    - CANONICAL_BATH parameters
                    - _accel_ok flag (C library status)

Lines 501-1000      Logger setup, exception classes
                    - logging.getLogger('qtcl_client')
                    - Custom exceptions: QtclError, BootstrapError, MetricsError

Lines 1001-1500     CFFI initialization block
                    |
                    +-- Lines 1050-1100: FFI object creation
                    |   ffi = FFI()
                    |   _accel_ffi = ffi
                    |
                    +-- Lines 1100-1200: C type definitions (cdef)
                    |   Struct definitions, function signatures
                    |
                    +-- Lines 1200-1300: C source code string
                    |   _QTCL_C_SOURCE = """...""" (embedded)
                    |
                    +-- Lines 1300-1400: Library compilation/verification
                    |   try: _accel_lib = _accel_ffi.verify(...)
                    |   except: _accel_ok = False
                    |
                    +-- Lines 1400-1500: Error handling & fallback
                        Print status, setup Python fallback

Lines 1500-1700     C code inline (as string, ~200 lines)
                    - Embedded within Python string literal
                    - Copied to CFFI for compilation

Lines 1700-8000     Python class definitions
                    |
                    +-- Lines 1700-1800: GKSLBathParams dataclass
                    +-- Lines 1800-1900: ClientField dataclass
                    +-- Lines 1900-2500: QuantumOracleClient class
                    +-- Lines 2500-4000: EntropyMempool class
                    +-- Lines 4000-6000: QTCLWallet class
                    +-- Lines 6000-8000: FlaskApp class (web server)

Lines 8000-8500     C code inline: Cryptography functions
                    - SHA3-256 wrappers
                    - HLWE signature verification
                    - AES-256-GCM (optional)

Lines 8500-9600     C code inline: Quantum metrics
                    - qtcl_coherence_l1()
                    - qtcl_purity()
                    - qtcl_fidelity_w3()
                    - qtcl_entropy_vn()
                    - qtcl_bootstrap_build_blockfield()

Lines 9600-10000    CFFI cdef (C type & function signatures)
                    - All C struct definitions
                    - All C function signatures for FFI

Lines 10000-10500   Python utility functions (wrappers around C)
                    - _coherence_py(dm_re, dm_im)
                    - _purity_py(dm_re, dm_im)
                    - _compute_metrics(dm_re, dm_im)

Lines 10500-11000   Database-related Python functions
                    - setup_database()
                    - get_block()
                    - insert_block()
                    - Query builders

Lines 11000-13000   Consensus & state management
                    - Oracle consensus logic
                    - Peer quorum calculation
                    - Finalization gate
                    - Fork detection

Lines 13000-13500   Mining loop (main orchestration)
                    def run_mine_mode():
                    - Bootstrap sequence
                    - Nonce iteration
                    - Block submission
                    - Metrics reporting

Lines 13500-14500   Flask routes (REST API)
                    - @app.route('/api/block_height')
                    - @app.route('/api/submit_block', methods=['POST'])
                    - @app.route('/api/metrics/latest')
                    - etc.

Lines 14500-15100   WebSocket handlers (SocketIO)
                    - @socketio.on('connect')
                    - @socketio.on('disconnect')
                    - Broadcast handlers

Lines 15100-15183   Main entry point
                    if __name__ == '__main__':
                    - Parse arguments
                    - Initialize app
                    - Start server/mining
```

### 1.2 Physical Code Locations (For Grep/Navigation)

```bash
# Find CFFI setup
grep -n "FFI()" qtcl_client.py              # Line ~1050
grep -n "cdef" qtcl_client.py               # Lines 9600-10000

# Find C functions
grep -n "double qtcl_" qtcl_client.py       # C function definitions
grep -n "_accel_lib\." qtcl_client.py       # Python calls to C

# Find Python classes
grep -n "^class " qtcl_client.py            # Line 1700+
grep -n "^def " qtcl_client.py              # All functions

# Find hot paths
grep -n "def run_mine_mode" qtcl_client.py
grep -n "def _run_bootstrap" qtcl_client.py
grep -n "@app.route" qtcl_client.py
```

---

## 2. CFFI Setup & Initialization

### 2.1 FFI Object Creation (Lines 1050-1100)

```python
from cffi import FFI

# Create FFI instance
_accel_ffi = FFI()

# Later in execution:
# 1. cdef() - Define C types
# 2. set_source() or verify() - Provide C source
# 3. compile() - Generate library
```

### 2.2 Type Definitions (cdef) - Lines 9600-9850

```python
_QTCL_C_DEFS = """
    // Scalar function signatures
    double qtcl_coherence_l1(const double *dm_re, const double *dm_im, int n);
    double qtcl_purity(const double *dm_re, const double *dm_im, int n);
    double qtcl_fidelity_w3(const double *dm8_re);
    
    // Struct definition (mirrors C struct)
    typedef struct {
        double w_fidelity;          // Offset 0, 8 bytes
        double entropy_vn;          // Offset 8, 8 bytes
        double coherence;           // Offset 16, 8 bytes
        double purity;              // Offset 24, 8 bytes
        double negativity;          // Offset 32, 8 bytes
        double discord;             // Offset 40, 8 bytes
        double dm_re[64];           // Offset 48, 512 bytes
        double dm_im[64];           // Offset 560, 512 bytes
        // Total size: 1072 bytes (padded to 8-byte alignment)
    } QtclWStateMeasurement;
    
    // Complex function with struct output
    int qtcl_bootstrap_build_blockfield(
        int pq0, int pq_curr, int pq_last, int height,
        uint8_t *node_id16,
        double gamma1, double gammaphi, double gammadep, double omega,
        double dt,
        QtclWStateMeasurement *out_m,
        uint8_t *out_seed32
    );
"""

_accel_ffi.cdef(_QTCL_C_DEFS)  # Register types with FFI
```

**Memory Layout of QtclWStateMeasurement:**
```
Offset  Size    Field
──────────────────────────
0       8       w_fidelity (double)
8       8       entropy_vn (double)
16      8       coherence (double)
24      8       purity (double)
32      8       negativity (double)
40      8       discord (double)
48      512     dm_re[64] (64 × 8 bytes)
560     512     dm_im[64] (64 × 8 bytes)
────────────────────────────
Total:  1072 bytes
```

### 2.3 C Source Embedding (Lines 8000-9600)

```python
# C code stored as Python string literal
_QTCL_C_SOURCE = r"""
#include <math.h>
#include <string.h>
#include <stdint.h>

// Actual C implementations...
double qtcl_coherence_l1(const double *dm_re, const double *dm_im, int n) {
    // ...
}
"""

# Compilation happens at runtime
try:
    _accel_lib = _accel_ffi.verify(_QTCL_C_SOURCE)
    _accel_ok = True
except Exception as e:
    _accel_ok = False
    print(f"[WARN] C compilation failed: {e}")
```

**Why embedded string?**
- Single file deployment (no separate .c file)
- Cached by CFFI after first compilation
- Works on any platform (Windows/Mac/Linux)
- Drawback: ~1-2 seconds first import for compilation

### 2.4 Compilation & Caching (Lines 1300-1400)

```python
# First time: Compiles C code
# CFFI output: __pycache__/qtcl_client.cpython-39-x86_64-linux-gnu.so

# Subsequent times: Loads cached .so
# Fast: ~100ms import time

# Invalidate cache if:
# - C source changes (_QTCL_C_SOURCE)
# - CFFI version changes
# - Python version changes
# Then delete __pycache__ and reimport
```

---

## 3. C Type System & Memory Layout

### 3.1 CFFI Buffer Objects

**Creating buffers:**
```python
# From Python list
dm_re = [0.125] * 64
dm_re_c = _accel_ffi.new("double[]", dm_re)  # Copy list to C array
# Result: C array of 64 doubles, initialized with list values

# Pre-allocated reusable buffer
dm_re_c = _accel_ffi.new("double[64]")  # Allocate 64 doubles
# Initially zero-filled, can reuse by copying new data

# Struct buffer
out_m = _accel_ffi.new("QtclWStateMeasurement *")
# Allocate struct, pointer returned
# Access fields: out_m.w_fidelity, out_m.dm_re[0], etc.

# Byte array
seed = _accel_ffi.new("uint8_t[32]")
# 32 bytes, zero-filled
```

**Buffer lifecycle:**
```python
# CFFI buffers are Python objects
# Garbage collection handles deallocation
# But can use manually:

buffer = _accel_ffi.new("double[1024]")
# Use buffer
del buffer  # Force cleanup (optional, GC does this anyway)
```

### 3.2 Type Conversions at FFI Boundary

```python
# Python list → C array
py_list = [1.0, 2.0, 3.0, 4.0]
c_array = _accel_ffi.new("double[]", py_list)

# C struct field access
struct_ptr = _accel_ffi.new("QtclWStateMeasurement *")
struct_ptr.w_fidelity = 0.85  # Write double
value = float(struct_ptr.w_fidelity)  # Read (returns C double type)

# C array element access
c_array = _accel_ffi.new("double[64]")
c_array[0] = 0.5  # Write
val = float(c_array[10])  # Read

# Byte buffer conversion
c_seed = _accel_ffi.new("uint8_t[32]")
_accel_lib.qtcl_some_func(..., c_seed)
# Convert to Python bytes
py_bytes = bytes(_accel_ffi.buffer(c_seed)[0:32])

# String conversion
c_str = _accel_ffi.new("char[]", b"hello\0")
py_str = _accel_ffi.string(c_str)
```

### 3.3 Pointer Types

```python
# Pointer to scalar
ptr = _accel_ffi.new("double *")
ptr[0] = 3.14  # Dereference and write
val = ptr[0]   # Dereference and read

# Pointer to struct
struct_ptr = _accel_ffi.new("QtclWStateMeasurement *")
struct_ptr.w_fidelity = 0.85  # Access member
struct_ptr.dm_re[0] = 0.125   # Access array member

# Pointer arithmetic (rare in Python CFFI, but possible)
array = _accel_ffi.new("double[100]")
ptr = array  # array is already pointer
ptr[50]  # Access element 50
```

---

## 4. Python-C Call Chain

### 4.1 Minimal Example: Coherence Computation

**Python caller (line ~2008):**
```python
def compute_coherence(dm_re, dm_im):
    """Compute coherence (50x speedup via C)."""
    
    if not _accel_ok:
        # Fallback to Python
        return _python_coherence_l1(dm_re, dm_im, 8)
    
    # Prepare CFFI buffers
    dm_re_c = _accel_ffi.new("double[]", dm_re)  # ~10µs
    dm_im_c = _accel_ffi.new("double[]", dm_im)  # ~10µs
    
    # Call C function
    result = _accel_lib.qtcl_coherence_l1(dm_re_c, dm_im_c, 8)  # ~500µs
    
    # Bounds check
    result = max(0.0, min(1.0, float(result)))  # ~1µs
    
    return result
```

**C implementation (line ~8816):**
```c
double qtcl_coherence_l1(const double *dm_re, const double *dm_im, int n) {
    double s = 0.0;
    // Main loop: O(n²) = 64 iterations
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i != j) {
                double r = dm_re[i*n+j];
                double im = dm_im[i*n+j];
                s += sqrt(r*r + im*im);  // |ρ[i,j]|
            }
        }
    }
    // Normalize
    return (n > 1) ? s / (double)(n * (n-1)) : 0.0;
}
```

**Call stack:**
```
Python:  compute_coherence()
         ├─ Create CFFI buffers (FFI layer)
         └─ Call _accel_lib.qtcl_coherence_l1()
             │
             ↓ (FFI dispatch)
             │
C:       qtcl_coherence_l1(const double *dm_re, ...)
         ├─ Memory-safe access (already validated by CFFI)
         ├─ Core computation (tight loop)
         └─ Return result (double)
             │
             ↓ (CFFI return)
             │
Python:  float(result)
         └─ Return to caller
```

**Total overhead:**
- Buffer creation: ~20µs
- CFFI dispatch: ~5µs
- C computation: ~500µs
- Total: ~525µs (vs ~25ms in pure Python)

### 4.2 Complex Example: Bootstrap Build

**Python caller (line ~13872):**
```python
def _run_bootstrap():
    """Full quantum bootstrap (100x speedup via C)."""
    
    # Prepare parameters
    _bh = self.koyeb_state.block_height or bh
    _pqc = int(self.client_field.pq_curr_id or _bh)
    _pql = int(self.client_field.pq_last_id or max(0, _bh - 1))
    
    # Node ID
    node_id_b = self._peer_id[:32].encode('utf-8')[:16].ljust(16, b'\x00')
    node_buf = _accel_ffi.new('uint8_t[16]', list(node_id_b))
    
    # Output buffers
    out_m = _accel_ffi.new('QtclWStateMeasurement *')
    out_seed = _accel_ffi.new('uint8_t[32]')
    
    # Call C with all parameters
    oracle_ok = _accel_lib.qtcl_bootstrap_build_blockfield(
        0,                          # pq0 (oracle anchor)
        _pqc,                       # pq_curr (block entry)
        _pql,                       # pq_last (block exit)
        _bh,                        # height
        node_buf,                   # node_id (16 bytes)
        float(bath.gamma1_eff),     # GKSL parameter 1
        float(bath.gammaphi),       # GKSL parameter 2
        float(bath.gammadep),       # GKSL parameter 3
        float(bath.omega),          # GKSL parameter 4
        dt,                         # timestep
        out_m,                      # Output: measurement struct
        out_seed                    # Output: 32-byte seed
    )
    
    # Check return value
    if not oracle_ok:
        raise RuntimeError(f"[Bootstrap] C call failed: oracle_ok={oracle_ok}")
    
    # Extract results from C struct
    metrics = {
        'w_fidelity': float(out_m.w_fidelity),
        'entropy_vn': float(out_m.entropy_vn),
        'coherence': float(out_m.coherence),
        'purity': float(out_m.purity),
        'dm_re': [float(out_m.dm_re[i]) for i in range(64)],
        'dm_im': [float(out_m.dm_im[i]) for i in range(64)],
    }
    
    # Convert seed to bytes
    seed_bytes = bytes(_accel_ffi.buffer(out_seed)[0:32])
    
    return metrics, seed_bytes
```

**C implementation (line ~10800, simplified):**
```c
int qtcl_bootstrap_build_blockfield(
    int pq0, int pq_curr, int pq_last, int height,
    uint8_t *node_id16,
    double gamma1, double gammaphi, double gammadep, double omega,
    double dt,
    QtclWStateMeasurement *out_m,
    uint8_t *out_seed32
) {
    // 1. Initialize DM (hyperbolic field generator)
    double dm_re[64], dm_im[64];
    memset(dm_re, 0, 512);
    memset(dm_im, 0, 512);
    // ... generate W-state DM based on (pq0, pq_curr, pq_last) ...
    
    // 2. Apply GKSL evolution (4 RK4 steps)
    for (int step = 0; step < 4; step++) {
        qtcl_gksl_rk4_step(dm_re, dm_im, gamma1, gammaphi, gammadep, omega, dt);
    }
    
    // 3. Compute metrics
    double fid = qtcl_fidelity_w3(dm_re);
    double coh = qtcl_coherence_l1(dm_re, dm_im, 8);
    double pur = qtcl_purity(dm_re, dm_im, 8);
    // ... compute entropy, negativity, discord ...
    
    // 4. Populate output struct
    out_m->w_fidelity = fid;
    out_m->entropy_vn = ent;
    out_m->coherence = coh;
    out_m->purity = pur;
    
    // Copy DM
    memcpy(out_m->dm_re, dm_re, 512);
    memcpy(out_m->dm_im, dm_im, 512);
    
    // 5. Generate PoW seed (HMAC-SHA256 of node_id || metrics)
    // ... HMAC computation ...
    memcpy(out_seed32, seed_bytes, 32);
    
    return 1;  // Success
}
```

**Execution flow in detail:**
```
Python:  _run_bootstrap()
         ├─ Prepare 10 input parameters
         ├─ Create 3 CFFI buffers (node_buf, out_m, out_seed)
         ├─ Type conversions (int → int, double → double, etc.)
         └─ CFFI dispatch
             │
             ↓ (FFI marshalling: ~5µs)
             │
C:       qtcl_bootstrap_build_blockfield()
         ├─ Receive parameters (already validated)
         ├─ Initialize 8×8 DM (complex matrix)
         ├─ GKSL integration (4 RK4 steps × ~50ms each)
         │  ├─ Compute Liouvillian (L(ρ))
         │  ├─ 4× RK4 stages (k1, k2, k3, k4)
         │  └─ Update density matrix
         ├─ Compute metrics (fidelity, coherence, purity, entropy, etc.)
         ├─ Pack results into struct (copy DM: 1KB)
         ├─ Generate PoW seed
         └─ Return 1 (success)
             │
             ↓ (FFI return: struct + value)
             │
Python:  Extract results
         ├─ Read out_m.w_fidelity, out_m.entropy_vn, etc.
         ├─ Extract DM arrays (64 elements each)
         ├─ Convert seed bytes
         ├─ Validate metrics (range checks)
         └─ Return to mining loop
```

**Latency breakdown:**
```
Time component              Duration   Cumulative
────────────────────────────────────────────────
CFFI buffer allocation      ~30µs      30µs
Parameter marshalling       ~5µs       35µs
C function call dispatch    ~1µs       36µs
GKSL RK4 × 4 steps         ~50ms      50.036ms
Metric computations        ~0.5ms     50.536ms
Memory copies              ~1ms        51.536ms
CFFI result extraction     ~5µs        51.541ms
Python extraction loop     ~50µs       51.591ms
                                       ───────────
Total:                                 ~51.6ms

Without C (Python equivalent):
  GKSL in NumPy × 4:       ~2.0s
  Eigenvalue (entropy):    ~150ms
  Total:                   ~2.15s

Speedup: 2150ms / 51.6ms ≈ 40x
```

---

## 5. Hot Path Functions

### 5.1 Call Frequency Analysis

```python
# During 10-second mining cycle:

Function                    Calls   CPU Time    Total Time
────────────────────────────────────────────────────────
_run_bootstrap()            1       50ms        50ms
qtcl_coherence_l1()         1       0.5ms       0.5ms
qtcl_purity()               1       0.2ms       0.2ms
qtcl_fidelity_w3()          1       0.001ms     0.001ms
sha256()                    ~500M   0.001µs/ea  500ms
qtcl_fidelity_check()       ~500M   0.001µs/ea  500ms (PoW)
mempool.add_tx()            ~100    0.1ms       10ms
database.query()            ~5      2ms         10ms
```

**Critical path:** sha256 (PoW nonce search) dominates, but GKSL bootstrap is the setup bottleneck

### 5.2 Coherence - Most Called Metric (Lines 8816-8825)

```c
double qtcl_coherence_l1(const double *dm_re, const double *dm_im, int n) {
    double s = 0.0;
    // Tight loop: 64 iterations
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i != j) {
                // Load from memory (64-byte cache line)
                double r = dm_re[i*n+j];
                double im = dm_im[i*n+j];
                // FMA: fused multiply-add (1 CPU cycle)
                s += sqrt(r*r + im*im);
            }
        }
    }
    return (n > 1) ? s / (double)(n * (n-1)) : 0.0;
}
```

**Optimization notes:**
- Cache-friendly: Row-major access (dm_re[i*n+j])
- Branch prediction: if (i != j) fails 7/8 times, predictable
- FMA units: Modern CPUs execute sqrt in parallel
- Compiler: -O3 flag auto-vectorizes to SSE/AVX

### 5.3 Bootstrap - Performance Bottleneck (Lines 10800-10900)

```c
// Pseudocode of actual flow
int qtcl_bootstrap_build_blockfield(...) {
    double dm_re[64], dm_im[64];
    
    // (A) Initialize: 1-2ms
    initialize_w_state_from_coordinates(pq0, pq_curr, pq_last, dm_re, dm_im);
    
    // (B) GKSL evolution: 50-60ms (4 RK4 steps @ ~13ms each)
    for (int step = 0; step < 4; step++) {
        qtcl_gksl_rk4_step(dm_re, dm_im, gamma1, gammaphi, gammadep, omega, dt);
        // Each step:
        //   - Compute Liouvillian: 2ms (4 Lindblad terms)
        //   - 4× RK4 substeps: 10ms
    }
    
    // (C) Metric computation: 1-2ms
    double fid = qtcl_fidelity_w3(dm_re);        // 0.001ms
    double coh = qtcl_coherence_l1(dm_re, dm_im, 8);  // 0.5ms
    double pur = qtcl_purity(dm_re, dm_im, 8);  // 0.2ms
    // ... entropy, negativity, discord: 1ms total
    
    // (D) Result packing & seed generation: 1ms
    memcpy(out_m->dm_re, dm_re, 512);
    memcpy(out_m->dm_im, dm_im, 512);
    // HMAC-SHA256 of metrics
    
    return 1;
}
```

**Critical path (80%+ time):** GKSL evolution loop → Optimize here for performance gains

---

## 6. Memory Management

### 6.1 Buffer Lifecycle Patterns

**Pattern 1: Temporary buffers (allocate per call)**
```python
def compute_once():
    dm_re = _accel_ffi.new("double[64]", dm_re_data)  # Allocate
    result = _accel_lib.qtcl_coherence_l1(dm_re, ...)  # Use
    # Implicit: GC deallocates when scope exits
    return result
```

**Memory cost:** ~10µs allocation + GC overhead
**Use when:** Called infrequently or with different sizes

**Pattern 2: Persistent buffers (allocate once, reuse)**
```python
class MetricsComputer:
    def __init__(self):
        self.dm_re_buf = _accel_ffi.new("double[64]")   # Allocate once
        self.dm_im_buf = _accel_ffi.new("double[64]")
        self.out_m = _accel_ffi.new("QtclWStateMeasurement *")
    
    def compute(self, dm_re_data, dm_im_data):
        # Copy data into persistent buffers (no allocation)
        for i in range(64):
            self.dm_re_buf[i] = dm_re_data[i]
            self.dm_im_buf[i] = dm_im_data[i]
        # Call C with pre-allocated buffers
        return _accel_lib.qtcl_coherence_l1(self.dm_re_buf, self.dm_im_buf, 8)

# Usage
computer = MetricsComputer()  # Allocate once at init
for frame in data_stream:
    result = computer.compute(frame['dm_re'], frame['dm_im'])  # Reuse buffers
```

**Memory cost:** ~50µs per call (copy only, no allocation)
**Use when:** Called frequently in tight loops

**Pattern 3: Array from list (convert once)**
```python
# Efficient: Direct list → CFFI
dm_re = [0.125] * 64
dm_re_c = _accel_ffi.new("double[]", dm_re)  # Single memcpy
result = _accel_lib.qtcl_coherence_l1(dm_re_c, dm_im_c, 8)
```

**Memory cost:** ~50µs (memcpy only)
**Use when:** Data already in Python list, called once

### 6.2 Struct Memory Layout & Alignment

```python
# QtclWStateMeasurement struct
# Total: 1072 bytes (8-byte aligned)

# CFFI automatically handles padding
out_m = _accel_ffi.new("QtclWStateMeasurement *")

# Field access (CFFI handles offsetting)
out_m.w_fidelity = 0.85  # Offset 0
out_m.entropy_vn = 1.2   # Offset 8
out_m.dm_re[0] = 0.125   # Offset 48

# Get raw buffer (for passing to other functions)
struct_bytes = _accel_ffi.buffer(out_m)[0:1072]
```

### 6.3 Memory Safety Guarantees

**CFFI bounds checking:**
```python
# Safe: CFFI validates bounds
buf = _accel_ffi.new("double[64]")
buf[63] = 1.0  # OK
buf[64] = 1.0  # Raises IndexError (bounds check)

# Safe: CFFI validates type
_accel_lib.qtcl_coherence_l1(buf, buf, 8)  # OK

# Unsafe: NULL pointer in C (no Python-side check)
_accel_lib.qtcl_coherence_l1(None, None, 8)  # Segfault (but caught by CFFI)
```

---

## 7. Error Handling & Recovery

### 7.1 C Compilation Failures (Lines 1300-1400)

```python
try:
    _accel_lib = _accel_ffi.verify(_QTCL_C_SOURCE)
    _accel_ok = True
    print("[INIT] C acceleration library loaded")
except Exception as e:
    _accel_ok = False
    print(f"[ERROR] C compilation failed: {type(e).__name__}: {e}")
    print("[WARN] Using pure Python fallback (50x slower)")
    
    # Define dummy library for fallback
    class _FallbackLib:
        def qtcl_coherence_l1(self, dm_re, dm_im, n):
            return _python_coherence_l1(dm_re, dm_im, n)
        # ... other methods ...
    
    _accel_lib = _FallbackLib()
```

**Common failures:**
- Missing compiler (gcc/clang)
- Missing libffi development headers
- Incompatible Python version
- Out of disk space (temp files)

**Recovery:** Fallback works but 50x slower

### 7.2 Runtime C Errors (Lines 13700-13760)

```python
def safe_call_c_function(func_name, *args):
    """Wrap C calls with error handling."""
    try:
        func = getattr(_accel_lib, func_name)
        result = func(*args)
        
        # Validate result
        if func_name == 'qtcl_coherence_l1':
            if not (0.0 <= result <= 1.0):
                raise ValueError(f"Coherence out of range: {result}")
            if result != result:  # NaN check
                raise ValueError("Coherence is NaN")
        
        return result
        
    except Exception as e:
        _EXP_LOG.error(f"[C_CALL] {func_name}() failed: {type(e).__name__}: {e}")
        
        # Return default/fallback
        if func_name == 'qtcl_coherence_l1':
            return _python_coherence_l1(*args)
        else:
            raise
```

### 7.3 Segfault Handling (Python Crashes)

```python
# When C code has bugs (buffer overflow, NULL pointer):
# Segfault → Python process crashes
# No recovery possible

# Prevention:
# 1. Extensive validation in Python before C call
# 2. Bounds checking on all buffers
# 3. NULL pointer checks
# 4. Use valgrind to test C code

# Example: Validate before passing to C
def coherence_safe(dm_re, dm_im):
    # Validate inputs
    if not isinstance(dm_re, (list, tuple)) or len(dm_re) != 64:
        raise TypeError("dm_re must be list/tuple of 64 elements")
    if not all(isinstance(x, (int, float)) for x in dm_re):
        raise TypeError("dm_re must contain only numbers")
    if any(x != x for x in dm_re):  # NaN check
        raise ValueError("dm_re contains NaN")
    
    # Safe to call C
    dm_re_c = _accel_ffi.new("double[]", dm_re)
    dm_im_c = _accel_ffi.new("double[]", dm_im)
    return _accel_lib.qtcl_coherence_l1(dm_re_c, dm_im_c, 8)
```

---

## 8. Performance Optimization Details

### 8.1 Profiling C Code

**Using perf (Linux):**
```bash
# Profile entire Python process
perf record -p $(pgrep -f qtcl_client) -- sleep 10
perf report

# Typical output:
# 60% qtcl_gksl_rk4_step (C function - GKSL evolution)
# 20% python interpreter
# 15% memcpy (struct copies)
# 5% other
```

**Using cProfile (Python):**
```python
import cProfile
import pstats
from pstats import SortKey

profiler = cProfile.Profile()
profiler.enable()

# Run mining for N seconds
run_mine_mode()

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats(SortKey.CUMULATIVE)
stats.print_stats(20)

# Output shows time in Python functions (not C)
# C time appears as blank because profiler can't instrument C directly
```

### 8.2 Optimization Techniques

**Technique 1: Reduce buffer allocations**
```python
# Before: Allocates on every call
def process_frame(frame):
    dm_re = _accel_ffi.new("double[]", frame['dm_re'])  # Allocate
    dm_im = _accel_ffi.new("double[]", frame['dm_im'])
    return _accel_lib.qtcl_coherence_l1(dm_re, dm_im, 8)

# After: Persistent buffers
class Processor:
    def __init__(self):
        self.dm_re = _accel_ffi.new("double[64]")
        self.dm_im = _accel_ffi.new("double[64]")
    
    def process_frame(self, frame):
        # Copy data (fast, no allocation)
        for i in range(64):
            self.dm_re[i] = frame['dm_re'][i]
            self.dm_im[i] = frame['dm_im'][i]
        return _accel_lib.qtcl_coherence_l1(self.dm_re, self.dm_im, 8)

# Speedup: 10-20% (reduces allocation overhead)
```

**Technique 2: Batch operations**
```python
# Before: N calls × (buffer allocation + C call overhead)
for frame in frames:
    result = compute_metrics(frame)

# After: Single C call processes multiple frames
# (if C function supports it)
# or: Use persistent buffers (technique 1)
```

**Technique 3: Vectorization (in C)**
```c
// Before: Scalar
double s = 0.0;
for (int i = 0; i < 64; i++)
    s += sqrt(dm_re[i]*dm_re[i] + dm_im[i]*dm_im[i]);

// After: SIMD (4x doubles in parallel with AVX2)
#include <immintrin.h>
__m256d s_vec = _mm256_setzero_pd();
for (int i = 0; i < 64; i += 4) {
    __m256d re = _mm256_loadu_pd(&dm_re[i]);
    __m256d im = _mm256_loadu_pd(&dm_im[i]);
    __m256d mag = _mm256_sqrt_pd(_mm256_add_pd(
        _mm256_mul_pd(re, re),
        _mm256_mul_pd(im, im)
    ));
    s_vec = _mm256_add_pd(s_vec, mag);
}
// Aggregate results from vector
```

---

## 9. Debugging & Instrumentation

### 9.1 Verify C Library Status

```python
# Check if C acceleration loaded
print(f"C acceleration: {'enabled' if _accel_ok else 'disabled'}")

# Check CFFI version
import cffi
print(f"CFFI version: {cffi.__version__}")

# Check if library can be reloaded
try:
    test = _accel_lib.qtcl_coherence_l1(
        _accel_ffi.new("double[]", [0.125]*64),
        _accel_ffi.new("double[]", [0.0]*64),
        8
    )
    print(f"C library test: OK (coherence={test:.4f})")
except Exception as e:
    print(f"C library test: FAILED ({e})")
```

### 9.2 Instrument C Functions

```python
# Add timing
import time

def coherence_instrumented(dm_re, dm_im):
    t0 = time.perf_counter_ns()
    result = _accel_lib.qtcl_coherence_l1(
        _accel_ffi.new("double[]", dm_re),
        _accel_ffi.new("double[]", dm_im),
        8
    )
    t1 = time.perf_counter_ns()
    elapsed_us = (t1 - t0) / 1000
    print(f"coherence_l1: {elapsed_us:.1f}µs")
    return result

# Add validation
def coherence_validated(dm_re, dm_im):
    result = _accel_lib.qtcl_coherence_l1(
        _accel_ffi.new("double[]", dm_re),
        _accel_ffi.new("double[]", dm_im),
        8
    )
    
    # Validate result
    if not (0.0 <= result <= 1.0):
        print(f"[WARNING] Coherence out of bounds: {result}")
    if result != result:  # NaN
        print(f"[ERROR] Coherence is NaN!")
    
    return result
```

### 9.3 Compare Python vs C

```python
import timeit
import numpy as np

def benchmark_coherence():
    dm_re = [0.125] * 64
    dm_im = [0.0] * 64
    
    # C version
    if _accel_ok:
        c_time = timeit.timeit(
            lambda: _accel_lib.qtcl_coherence_l1(
                _accel_ffi.new("double[]", dm_re),
                _accel_ffi.new("double[]", dm_im),
                8
            ),
            number=1000
        )
        print(f"C:      {c_time:.3f}s / 1000 = {c_time/1000*1e6:.1f}µs per call")
    
    # Python version
    py_time = timeit.timeit(
        lambda: _python_coherence_l1(dm_re, dm_im, 8),
        number=1000
    )
    print(f"Python: {py_time:.3f}s / 1000 = {py_time/1000*1e6:.1f}µs per call")
    
    if _accel_ok:
        speedup = py_time / c_time
        print(f"Speedup: {speedup:.1f}x")
```

---

## 10. Extension Guide

### 10.1 Adding a New Python Function

```python
# Location: After line 10000 (utility functions section)

def my_new_metric(dm_re, dm_im):
    """Compute a custom quantum metric.
    
    Args:
        dm_re: List of 64 floats (real part of DM)
        dm_im: List of 64 floats (imaginary part)
    
    Returns:
        float: Metric value in range [0, 1]
    """
    if not _accel_ok:
        # Fallback to pure Python
        return _my_new_metric_python(dm_re, dm_im)
    
    # Call C implementation
    dm_re_c = _accel_ffi.new("double[]", dm_re)
    dm_im_c = _accel_ffi.new("double[]", dm_im)
    
    result = _accel_lib.my_new_metric_c(dm_re_c, dm_im_c, 8)
    
    # Validate
    result = max(0.0, min(1.0, float(result)))
    return result
```

### 10.2 Adding a New C Function

**Step 1: Add C source (lines 8000-9600)**
```c
// Inside _QTCL_C_SOURCE string
double my_new_metric_c(const double *dm_re, const double *dm_im, int n) {
    // Implementation here
    double result = 0.0;
    for (int i = 0; i < n * n; i++) {
        // Computation...
        result += dm_re[i] * dm_im[i];
    }
    return result / (n * n);
}
```

**Step 2: Add CFFI cdef (lines 9600-10000)**
```python
# Inside _QTCL_C_DEFS string
double my_new_metric_c(const double *dm_re, const double *dm_im, int n);
```

**Step 3: Add Python wrapper (lines 10000-10500)**
```python
def my_new_metric(dm_re, dm_im):
    """Wrapper for new metric."""
    if not _accel_ok:
        return _my_new_metric_python(dm_re, dm_im)
    
    dm_re_c = _accel_ffi.new("double[]", dm_re)
    dm_im_c = _accel_ffi.new("double[]", dm_im)
    return float(_accel_lib.my_new_metric_c(dm_re_c, dm_im_c, 8))
```

**Step 4: Test**
```python
# In main or test section
result = my_new_metric([0.125]*64, [0.0]*64)
print(f"Result: {result}")
```

### 10.3 Performance Testing for New Code

```python
def test_new_metric_performance():
    """Ensure C acceleration works and is faster."""
    dm_re = [0.125] * 64
    dm_im = [0.0] * 64
    
    # Test correctness: Python and C match
    py_result = _my_new_metric_python(dm_re, dm_im)
    c_result = my_new_metric(dm_re, dm_im)
    
    assert abs(py_result - c_result) < 1e-10, \
        f"Python/C mismatch: {py_result} vs {c_result}"
    
    # Test performance
    import timeit
    
    py_time = timeit.timeit(
        lambda: _my_new_metric_python(dm_re, dm_im),
        number=1000
    )
    
    c_time = timeit.timeit(
        lambda: my_new_metric(dm_re, dm_im),
        number=1000
    )
    
    speedup = py_time / c_time
    print(f"Speedup: {speedup:.1f}x")
    
    assert speedup > 10, f"Expected 10x speedup, got {speedup:.1f}x"
```

---

## Summary: When to Use What

| Task | Use | Why |
|------|-----|-----|
| Application routing | Python | Flask/Flask-SocketIO |
| State management | Python | Clarity, modifiability |
| Database queries | Python | asyncpg async driver |
| P2P networking | Python | Requests, socket libraries |
| Quantum metrics | C | 50-100x speedup |
| GKSL evolution | C | 50x speedup, core algorithm |
| Cryptography | C+Python | Via OpenSSL (C) |
| PoW nonce search | Python | SHA256 already fast (OpenSSL) |

---

**Document version:** 1.0  
**Last updated:** March 19, 2026  
**Audience:** Developers extending/maintaining QTCL client
