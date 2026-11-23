# AVISPA_QRAS-IoD

Formal security verification of **QRAS-IoD** with AVISPA and SPAN.

This repository contains the HLPSL specification and the verification results used in the security analysis of the paper:

> **Quantum-safe and Robust Authentication and Key Agreement Scheme for PUF-enabled Internet of Drones**
> (Joonyoung Lee, Sungjin Yu, and Kisung Park)

The model checks whether a Dolev–Yao intruder who fully controls the public channel can break the secrecy of the session key and credentials, or the mutual authentication among the mobile user, the ground station server, and the drone.

---

## Table of Contents

1. [Protocol Overview](#protocol-overview)
2. [Repository Structure](#repository-structure)
3. [Requirements](#requirements)
4. [Reproducing the Analysis](#reproducing-the-analysis)
5. [HLPSL Model](#hlpsl-model)
6. [Security Goals](#security-goals)
7. [Results](#results)
8. [Scope and Limitations](#scope-and-limitations)
9. [Citation](#citation)
10. [References](#references)

---

## Protocol Overview

QRAS-IoD is an authentication and key agreement (AKA) scheme that combines **ML-KEM** (FIPS 203) with a **physical unclonable function (PUF)** embedded in each drone.

| Entity | Symbol | Role in the protocol |
|---|---|---|
| Mobile user | `MUi` | Starts the AKA phase and encapsulates a shared secret under the GSS public key |
| Ground station server | `GSS` | Trusted registration authority; relays and verifies messages between user and drone |
| Drone | `Dj` | Reconstructs its secret response `Rj = PUF(Cj)` on demand and derives the session key |

The AKA phase exchanges four messages over a public channel:

| Message | Direction | Content |
|---|---|---|
| `Msg1` | `MUi → GSS` | `{ct1, M1, M2, V1, TIDi, T1}` |
| `Msg2` | `GSS → Dj` | `{M3, V2, T2}` |
| `Msg3` | `Dj → GSS` | `{M4, V3, V4, T3}` |
| `Msg4` | `GSS → MUi` | `{M5, V4, V5, T3, T4}` |

Both ends derive the session key `SK = h(PIDi ‖ IDDj ‖ N1 ‖ N2 ‖ N3)`.

---

## Repository Structure

```
AVISPA_QRAS-IoD/
├── code/
│   └── QRAS-IoD.hlpsl     # HLPSL specification of QRAS-IoD
├── result/
│   ├── OFMC               # Output of the OFMC back-end
│   └── CL-AtSe            # Output of the CL-AtSe back-end
└── README.md
```

---

## Requirements

| Component | Purpose |
|---|---|
| [Oracle VirtualBox](https://www.virtualbox.org/) | Runs the SPAN + AVISPA virtual machine |
| [SPAN + AVISPA VirtualBox disk](https://people.irisa.fr/Thomas.Genet/span/) | Bundles AVISPA, its back-ends, and the SPAN animator |

No additional installation is needed inside the virtual machine.

---

## Reproducing the Analysis

### 1. Download the SPAN + AVISPA distribution

Download the VirtualBox disk from <https://people.irisa.fr/Thomas.Genet/span/>. The same page provides video and PDF tutorials on SPAN and AVISPA.

### 2. Import the disk and start the virtual machine

Open VirtualBox, import the downloaded SPAN + AVISPA disk image, and boot the virtual machine.

### 3. Load the specification

Copy `code/QRAS-IoD.hlpsl` into the virtual machine and open it in SPAN (`File → Open`).

### 4. Run the back-ends

In SPAN, select the back-end and execute the analysis once for each of:

- **OFMC** (On-the-Fly Model-Checker)
- **CL-AtSe** (Constraint-Logic-based Attack Searcher)

Alternatively, if the `avispa` script is available on the command line of the virtual machine:

```bash
avispa QRAS-IoD.hlpsl --ofmc
avispa QRAS-IoD.hlpsl --cl-atse
```

### 5. Compare with the provided results

The outputs obtained by the authors are stored in `result/OFMC` and `result/CL-AtSe`.

---

## HLPSL Model

### Roles

| Role | Type | Description |
|---|---|---|
| `mobile_user` | Basic role | Behaviour of `MUi`: user registration, `Msg1` generation, `Msg4` verification, session key derivation |
| `ground_station` | Basic role | Behaviour of `GSS`: drone and user registration, `Msg1`/`Msg3` verification, `Msg2`/`Msg4` generation |
| `drone` | Basic role | Behaviour of `Dj`: drone registration, `Msg2` verification, `Msg3` generation, session key derivation |
| `session` | Composed role | One protocol run; binds the three basic roles to concrete agents, keys, and channels |
| `environment` | Top-level role | Global constants, intruder knowledge, and the sessions to be analysed |

The `goal` section at the end of the file declares the secrecy and authentication properties to be verified.


### Intruder model

The intruder follows the **Dolev–Yao** model on every `channel(dy)`: it can eavesdrop, intercept, modify, replay, delete, and inject messages. Its initial knowledge consists of the agent names, the public key of `GSS`, the hash function, and the public masking functions:

```
intruder_knowledge = {u, gss, d, pkgss, h,
                      mask1, mask2, mask3, mask4, mask5,
                      umaskb, umaskd1, umaskd2, dmaskf, dmaskg}
```

---

## Security Goals

### Secrecy

| Goal | Protected value | Shared among |
|---|---|---|
| `sec_k1` | ML-KEM shared secret `K1` | `MUi`, `GSS` |
| `sec_sk` | Session key `SK` | `MUi`, `GSS`, `Dj` |
| `sec_xi` | User long-term secret `Xi` | `MUi`, `GSS` |
| `sec_rj` | PUF response `Rj` | `GSS`, `Dj` |
| `sec_n1`, `sec_n2`, `sec_n3` | Session nonces `N1`, `N2`, `N3` | `MUi`, `GSS`, `Dj` |

### Authentication

Each goal is expressed with a `witness`/`request` pair and checked as strong authentication (`authentication_on`).

| Goal | Property |
|---|---|
| `user_gss_n1` | `GSS` authenticates `MUi` on `N1` |
| `gss_user_n2` | `MUi` authenticates `GSS` on `N2` |
| `gss_drone_n2` | `Dj` authenticates `GSS` on `N2` |
| `drone_gss_n3` | `GSS` authenticates `Dj` on `N3` |
| `drone_user_sk` | `MUi` authenticates `Dj` on the session key `SK` |

---

## Results

The complete outputs are available in the `result/` folder.


## Citation

If you use this repository, please cite the paper:

```bibtex
@unpublished{lee_qras_iod,
  author = {Joonyoung Lee, Kisung Park, and Sungjin Yu},
  title  = {Quantum-safe and Robust Authentication and Key Agreement Scheme for PUF-enabled Internet of Drones},
  note   = {Manuscript},
  year   = {2026}
}
```

---

## References

- SPAN, the Security Protocol ANimator for AVISPA: <https://people.irisa.fr/Thomas.Genet/span/>
