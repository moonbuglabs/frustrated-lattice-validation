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
- compare final-state and time-window readouts

The initial focus is FCC antiferromagnetic validation, with room for additional frustrated and control lattices as the protocol matures.

## Current API

Base URL:

```text
https://api.semulo.com
```

Current endpoints include:

```text
GET  /health
GET  /system
GET  /observe/fcc/topologies
POST /observe/fcc/torus
POST /observe/fcc/torus/sites.csv
POST /observe/fcc/torus/bonds.csv
POST /observe/fcc/torus/bonds_window.csv
```

The JSON summary endpoint reports the experiment metadata, CPU/runtime information, timing, and the time-window nearest-neighbor antiphase observable. The CSV endpoints expose site-level and bond-level data so the reported observable can be recomputed independently.

## Validation Posture

This repository does not ask readers to trust a claimed internal mechanism. It exposes the experiment inputs and returned observables so that researchers can test whether the reported behavior survives independent recomputation and control comparisons.

Important distinctions:

- The implementation is black-box.
- The validation harness is open.
- Site state and site material rate are configurable through the API.
- Bonds are fixed nearest-neighbor relations, not configurable weights.
- Bond observables are derived from endpoint site states.
- Raw per-tick state is diagnostic; the primary reported observable is time-window based.

## Quick Check

First smoke-test notebook:

```text
notebooks/fcc_smoke_test.ipynb
```

GitHub source:

```text
https://github.com/moonbuglabs/frustrated-lattice-validation/blob/main/notebooks/fcc_smoke_test.ipynb
```

Colab users can open the notebook from:

```text
File -> Open notebook -> GitHub
Search: moonbuglabs
Repository: moonbuglabs/frustrated-lattice-validation
Branch: main
Notebook: notebooks/fcc_smoke_test.ipynb
```

For the latest version, reopen the notebook from GitHub rather than an older saved Drive copy.

Example Python check:

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
print(abs(
    summary["observable"]["antiphase_fraction"]
    - bonds_window["window_antiphase_fraction"].mean()
))
```

For the current API, the CSV recomputation should match the JSON time-window observable for the same request payload.

## Ownership

Copyright (c) 2026 Moonbug Labs LLC.

This repository contains validation notebooks and API-client utilities for black-box frustrated-lattice experiments. It does not contain the proprietary execution engine or implementation details.

Semulo is the intended commercialization brand/entity, pending formation and formal IP arrangements.
