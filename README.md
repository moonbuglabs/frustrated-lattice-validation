# Frustrated Lattice Validation

Open validation notebooks and API-client utilities for black-box frustrated-lattice experiments.

This repository is intended for independent inspection of observable results returned by a hosted API. It does not contain the proprietary execution engine, source implementation, or internal update rules.

## Purpose

The goal is to make the validation workflow easy to inspect and hard to hand-wave:

- call a public HTTPS API
- verify the runtime reports CPU-only execution
- request frustrated-lattice experiments with explicit parameters
- load returned JSON and CSV data
- recompute reported observables independently in Python or pandas
- compare primary windowed readouts against diagnostic snapshots

The initial focus is FCC antiferromagnetic validation, with room for additional frustrated and control lattices as the protocol matures.

## Validation Frame

This is not a Max-Cut service or graph optimizer. The API tests a local lattice medium whose sites carry state and material conditions. Bonds are stateless nearest-neighbor relations: they identify which sites interact, but carry no state, strength, or configurable weight. The claim is not that an arbitrary hard combinatorial problem is solved, but that this medium can reproduce known lattice-ordering observables without representing the task as a graph search problem.

If arbitrary edge weights are introduced, the experiment has been converted back into the graph abstraction this API is not designed to test.

Valid test variations: topology, boundary condition (open vs. periodic), seed or seed batch, lattice size, site material rate, FCC uniform vs. gradient, local contrast below threshold.

Out of scope for this validation: arbitrary edge weights, arbitrary graph instances, treating bonds as objective terms.

## Why This Matters

Frustrated lattice ordering problems are often represented as combinatorial optimization problems. In that representation, the search space can become hard very quickly.

This validation surface tests a different possibility: that the same observable outcome can emerge from a local conservative medium without constructing or searching that combinatorial space.

This is relevant to researchers working on physical computing, frustrated magnets, or the relationship between physical dynamics and computational complexity.

The important claim is not that this API is a general NP-hard problem solver. The claim is narrower:

> For certain physical lattice-ordering problems, the hard search space may be introduced by the abstraction, while the corresponding spatial medium can reach the ground-state-related observable directly through local mediation.

If true, this matters because it points to a different computational path: not faster search, but avoiding the search representation for problems whose natural form is spatial, local, and physical.

## Non-Expert Starting Point

You do not need spin-glass expertise to run the first checks. The notebook first verifies CPU-only runtime, then runs simple controls where the reference observable is known: square and cubic lattices should approach 1.0; triangular and FCC lattices should approach the frustrated reference near 2/3.

The system does not freeze into an answer; it runs continuously, and the observable is averaged over the stable running state.

## Core Readout

The medium does not converge to a static assignment. It enters a stable limit-cycle regime. The primary AFM observable is windowed unsigned bond displacement: for each bond, unsigned displacement is accumulated over the sample window and averaged, then classified once against the quarter-circle threshold.

Instantaneous per-tick and final site snapshot readouts are returned as diagnostics. They are not the primary measure. Bipartite controls show why this matters: square and cubic torus runs return a primary windowed fraction of 1.0 while the instantaneous per-tick mean can read near 0.85 on the same run, because the bipartite attractor is a sign-flipping limit cycle. Windowed unsigned displacement recovers the correct baseline; instantaneous snapshots undercount it.

Bond displacement columns:

- `signed_bond_displacement`: oriented circular displacement from `site_a` to `site_b`.
- `unsigned_bond_displacement`: magnitude-only readout used for antiphase classification.
- `is_antiphase`: derived from unsigned displacement.
- `window_antiphase_fraction`: windowed bond-level readout; mean across bonds should reproduce the JSON summary observable.

## Integer-Exclusive Engine Execution

The execution engine is integer-exclusive. No floating-point arithmetic is used in site state updates or bond displacement readouts. Observable classification is computed from integer state and displacement values.

API timing fields, JSON serialization, and summary statistics may use standard host-language numeric types, but the lattice evolution and primary observable classification are integer-valued from initialization through measurement.

Given the same parameters and seed, the API returns identical results across runs. This is a direct consequence of integer-exclusive execution: there is no floating-point nondeterminism or rounding accumulation.

Supported public bit depths are 9 through 18. The quick-start examples use 10-bit resolution by default; higher bit depths are available for resolution-sensitivity checks and larger material-rate ranges.

## Current API

Base URL:

```text
https://api.semulo.com
```

Topology discovery:

```text
GET  /health
GET  /system
GET  /observe/topologies
```

Topology matrix endpoints (JSON summary, primary windowed readout; FCC torus also supports gradient material conditions):

```text
POST /observe/square/2d/open
POST /observe/square/2d/torus
POST /observe/triangular/2d/open
POST /observe/triangular/2d/torus
POST /observe/triangular/2d/hex
POST /observe/cubic/3d/open
POST /observe/cubic/3d/torus
POST /observe/fcc/3d/open
POST /observe/fcc/3d/torus
```

Expected observable by lattice family:

```text
square / cubic (bipartite, periodic)       -> antiphase fraction near 1.0
triangular periodic                        -> near 2/3
FCC periodic with gradient condition       -> near 2/3
FCC periodic uniform                       -> stable sub-reference attractor
open boundary                              -> finite material patch, boundary effects visible
FCC open                                   -> finite frustrated boundary behavior; boundary effects expected
```

FCC gradient and CSV endpoints (existing routes, unchanged):

```text
POST /observe/fcc/torus
POST /observe/fcc/torus/sites.csv
POST /observe/fcc/torus/bonds.csv
POST /observe/fcc/torus/bonds_window.csv
```

The existing FCC torus route supports gradient material conditions and exposes full site and bond CSV data for independent recomputation.

## Validation Posture

This repository does not ask readers to trust a claimed internal mechanism. It exposes the experiment inputs and returned observables so that researchers can test whether the reported behavior survives independent recomputation and control comparisons.

The execution engine and internal update rules are patent-pending. The black-box posture is not an attempt to obscure a weak result; it is a legal requirement during the patent process. The observable outputs are fully open precisely so that independent validation can proceed without access to the protected mechanism.

Important distinctions:

- The implementation is black-box (patent-pending).
- The validation harness is open.
- Site initialization and site material conditions are controlled through endpoint parameters.
- Bonds are stateless nearest-neighbor relations: they identify which sites interact, but carry no state, strength, or configurable weight.
- Bond observables are derived from endpoint site states.
- Signed bond displacement preserves orientation; unsigned bond displacement is used for the antiphase fraction.
- Raw per-tick state is diagnostic; the primary reported observable is time-window based.

## Quick Check

First smoke-test notebook:

```text
notebooks/quick_start.ipynb
```

GitHub source:

```text
https://github.com/moonbuglabs/frustrated-lattice-validation/blob/main/notebooks/quick_start.ipynb
```

Colab users can open the notebook from:

```text
File -> Open notebook -> GitHub
Search: moonbuglabs
Repository: moonbuglabs/frustrated-lattice-validation
Branch: main
Notebook: notebooks/quick_start.ipynb
```

For the latest version, reopen the notebook from GitHub rather than an older saved Drive copy.

First sanity check: bipartite baseline and readout distinction. Primary windowed fraction should be `1.0`; instantaneous diagnostic will be lower because the bipartite attractor is a sign-flipping limit cycle.

```python
import requests

BASE = "https://api.semulo.com"

r = requests.post(f"{BASE}/observe/square/2d/torus", json={
    "width": 32,
    "height": 32,
    "seed": 0,
    "bits": 10,
    "warmup_ticks": 700,
    "scan_ticks": 300,
    "sample_ticks": 200,
}, timeout=120).json()

obs = r["observable"]
print("topology:", r["topology"])
print("primary_readout:", obs["primary_readout"])
print("antiphase_fraction:", obs["antiphase_fraction"])
print("instantaneous_antiphase_fraction_mean:", obs["instantaneous_antiphase_fraction_mean"])
print("site_snapshot_antiphase_fraction:", obs["site_snapshot_antiphase_fraction"])
```

Expected output:

```text
topology: square_2d_torus
primary_readout: window_mean_unsigned_bond_displacement
antiphase_fraction: 1.0
instantaneous_antiphase_fraction_mean: ~0.847
site_snapshot_antiphase_fraction: ~0.851
```

FCC gradient recomputation check (CSV path, deeper audit):

```python
import io
import pandas as pd
import requests

BASE = "https://api.semulo.com"

payload = {
    "nx": 8,
    "ny": 8,
    "nz": 8,
    "seed": 0,
    "material_condition": "z_gradient",
}

summary = requests.post(f"{BASE}/observe/fcc/torus", json=payload).json()
window_csv = requests.post(f"{BASE}/observe/fcc/torus/bonds_window.csv", json=payload)

bonds_window = pd.read_csv(io.StringIO(window_csv.text))

print(summary["observable"]["antiphase_fraction"])
print(bonds_window["window_antiphase_fraction"].mean())
print(bonds_window[[
    "signed_bond_displacement_mean",
    "unsigned_bond_displacement_mean",
    "window_antiphase_fraction",
]].head())
print(abs(
    summary["observable"]["antiphase_fraction"]
    - bonds_window["window_antiphase_fraction"].mean()
))
```

For the current API, the CSV recomputation should match the JSON time-window observable for the same request payload.

## Ownership

Copyright (c) 2026 Moonbug Labs LLC. All rights reserved.

This repository contains validation notebooks and API-client utilities for black-box frustrated-lattice experiments. It does not contain the proprietary execution engine or implementation details.
