# 🕵️ Federated Learning Privacy Attacks

## Overview

Federated Learning keeps the raw training data on the participating clients, but the model updates exchanged during training can still reveal information about local datasets.

This project experimentally evaluates this threat using an **honest-but-curious aggregation server**.

Two attacks were executed:

1. **Honest-but-curious server analysis**
2. **Property inference**

The experiments were first performed **without Secure Aggregation**, then replayed after integrating **SecAgg+**.

The objective was to measure whether individual client updates expose distinguishable information about their local data.

---

## 🎯 Threat Model

The aggregation server is assumed to be **honest-but-curious**.

This means that the server:

* follows the Federated Learning protocol correctly;
* performs the expected aggregation;
* does not modify the client updates;
* but inspects individual updates received from each client;
* attempts to infer information about the local training distribution.

For the experiments described here, the attacker is therefore the **FL server itself**.

No external attack tool is required.

---

# 1. Honest-but-Curious Server

## Principle

Without Secure Aggregation, the server receives the individual model updates:

```text
Client 1 ──────► w₁
Client 2 ──────► w₂
Client 3 ──────► w₃
                    │
                    ▼
               FL Server
```

Although these values are model parameters rather than raw log records, they contain measurable information related to the local training distribution.

The project investigates:

* parameter values;
* L2 norms;
* means;
* standard deviations;
* differences between clients;
* final-layer distances.

---

## Attack Instrumentation

The attack mode is enabled using:

```bash
ATTACK_LOG_UPDATES=1
```

When enabled, the server stores the individual client model updates at each training round.

Each round produces a file of the form:

```text
round_NN_updates.npz
```

The recorded model contains six tensors for each client:

```text
fc1.weight
fc1.bias

fc2.weight
fc2.bias

fc3.weight
fc3.bias
```

The experiment generated individual-update logs over **30 training rounds**.

---

## Update Analysis

The analysis script is:

```text
phase3/attacks/attack_curious_server.py
```

It can inspect an individual round and report information such as:

```text
Layer parameters
     │
     ├── Weight values
     ├── L2 norm
     ├── Mean
     └── Standard deviation
```

The analysis also computes a **3 × 3 client distance matrix** for the final layer.

The final layer is particularly interesting because it represents the mapping between learned features and the nine output classes.

---

# 2. Property Inference

## Objective

The property-inference experiment investigates whether the server can infer properties of a client's local dataset from its individual model update.

In this project, the target property is the distribution of local attack classes.

The attack therefore follows:

```text
Individual Client Update
          │
          ▼
Final-Layer Analysis
          │
          ▼
Deviation Scores
          │
          ▼
Class Ranking
          │
          ▼
Inferred Local Classes
```

---

## Attack Script

The property-inference experiment is implemented in:

```text
phase3/attacks/attack_property_inference.py
```

The experiment analyzes multiple rounds and ranks the classes according to their inferred contribution.

The analysis can be executed over rounds 1–30.

---

## Experimental Result

The property-inference attack was successfully executed.

The experiment reconstructed information about the local class distribution of the three clients.

This demonstrates that the individual model updates contain information that can distinguish the local datasets.

The leakage is particularly visible under the project's strongly **non-IID** partitioning, where each client observes only a subset of the nine classes.

---

# 3. Quantifying the Leakage

The project compares several indicators between the unprotected and protected configurations.

| Indicator                     | Without SecAgg   | With SecAgg+            |
| ----------------------------- | ---------------- | ----------------------- |
| Individual updates            | Directly visible | Masked                  |
| L2 norm per client            | Distinguishable  | Approximately identical |
| Inter-client distance (`fc3`) | **0.026–0.035**  | **0.001–0.004**         |
| Top-4 inferred classes        | Client-specific  | Identical               |
| Deviation scores              | **0.015–0.046**  | **0.001–0.011**         |
| Property inference            | Successful       | Neutralized             |

The report shows that the inter-client distances decrease by approximately an order of magnitude after masking.

The class fingerprints also become indistinguishable between clients.

---

# 4. Replay After Secure Aggregation

After the initial attacks were demonstrated, the same analysis was replayed with Secure Aggregation enabled.

The comparison uses the same attack scripts:

```text
attack_curious_server.py
attack_property_inference.py
```

The goal is to determine whether the signals used by the attacks remain available to the server.

---

## Without SecAgg

```text
Client 1 ─────────────┐
Client 2 ─────────────┼──► Server
Client 3 ─────────────┘

Individual updates
       │
       ▼
Weight analysis
       │
       ▼
Client differences
       │
       ▼
Property inference
```

The server can isolate each client's contribution.

---

## With SecAgg+

```text
Client 1 ──► Mask ──┐
Client 2 ──► Mask ──┼──► Aggregation
Client 3 ──► Mask ──┘
                         │
                         ▼
                    Global update
```

The individual contributions are masked before aggregation.

The server therefore no longer receives an individually readable update for each client.

---

# 5. Experimental Conclusion

The experiments demonstrate a clear distinction between **transport security** and **privacy against the aggregation server**.

### mTLS

mTLS protects the communication channel between clients and the FL server.

It does **not** prevent the server endpoint from reading the parameters after they have been received.

### Secure Aggregation

SecAgg+ changes what the server can observe by preventing direct access to individual client contributions.

The replay experiments show that:

```text
Without SecAgg
    │
    ├── Individual weights visible
    ├── Client differences measurable
    └── Local classes inferable
             ↓
          Leakage

With SecAgg+
    │
    ├── Contributions masked
    ├── Client differences flattened
    └── Class fingerprint neutralized
             ↓
       Leakage reduced
```

The experiment therefore provides an empirical demonstration of why **mTLS and Secure Aggregation address different security layers and should be used together**.

---

# 6. Reproduction

Start the FL server in honest-but-curious mode:

```bash
ATTACK_LOG_UPDATES=1 python -m phase3.server.fl_server
```

Analyze a specific round:

```bash
python phase3/attacks/attack_curious_server.py --round 30
```

Run property inference across the recorded rounds:

```bash
python phase3/attacks/attack_property_inference.py --rounds 1-30 --top 4
```

For the protected experiment, enable Secure Aggregation:

```bash
SECAGG=1
```

and repeat the analysis on the resulting experiment output.

> Do not commit RADIUS secrets, private keys, certificates, passwords or other credentials to the repository. Use environment variables or an `.env.example` file for reproducibility.
