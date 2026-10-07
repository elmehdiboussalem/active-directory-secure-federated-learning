# Troubleshooting

## Overview

This project combines Windows event collection, Active Directory, PKI, RADIUS, mTLS, Federated Learning, Secure Aggregation and real-time inference.

Several implementation issues appeared during integration.

This document records the main problems encountered during the project, their causes, and the corresponding solutions.

---

# 1. Windows Event Collection — Error 87

## Problem

The first implementation of the real-time detector used the Windows Event Log subscription mechanism:

```text
EvtSubscribe
```

The event collection component returned:

```text
ERROR_INVALID_PARAMETER
Error 87
```

As a result, the detector could not reliably consume live Windows Security events.

---

## Diagnosis

The issue was related to the parameters used by the subscription-based event collection path.

The original implementation was therefore replaced by a more explicit query-and-enumeration workflow.

---

## Solution

The event collector was migrated to:

```text
EvtQuery
    │
    ▼
EvtNext
    │
    ▼
EvtRender
```

The resulting flow became:

```text
Windows Event Log
       │
       ▼
EvtQuery
       │
       ▼
EvtNext
       │
       ▼
EvtRender
       │
       ▼
XML Event
       │
       ▼
Feature Extraction
```

This provided a stable mechanism for reading the required Windows events.

---

# 2. Windows Event XML Parsing

## Problem

After switching to `EvtQuery` / `EvtNext` / `EvtRender`, the detector received XML event records.

The parser initially failed on some event structures because of XML namespace handling.

The resulting issue was caused by namespace-qualified XML elements rather than by the event data itself.

---

## Solution

The XML parser was modified to correctly handle namespace-qualified elements.

The parser now extracts the relevant event fields independently of the namespace prefix.

Conceptually:

```text
Rendered XML
     │
     ▼
Namespace-aware parsing
     │
     ▼
Event fields
     │
     ├── Event ID
     ├── Timestamp
     ├── Provider
     └── Security fields
```

This allowed the event-processing pipeline to continue with normalized Windows telemetry.

---

# 3. Real-Time Detection Pipeline

## Problem

The detector needed to operate continuously on Windows clients while using the same feature representation as the federated training pipeline.

A mismatch between training metadata and live inference could result in incorrect predictions even if the model itself was valid.

---

## Solution

The deployed detector uses the trained model together with its associated metadata:

```text
mlp_federated_best.pt
metadata.pkl
```

The metadata provides the information required to reproduce the preprocessing and class representation used during training.

The inference pipeline therefore remains:

```text
Windows Events
      │
      ▼
Feature Extraction
      │
      ▼
StandardScaler
      │
      ▼
Federated MLP
      │
      ▼
Class Prediction
```

This avoids independently recreating the preprocessing logic during deployment.

---

# 4. Model and Metadata Synchronization

## Problem

The trained model cannot be treated as an isolated `.pt` file.

The model expects the same feature ordering, scaling and class mapping used during training.

Changing these values during deployment can produce invalid predictions.

---

## Solution

The real-time detector loads both:

```text
MODEL_PATH
META_PATH
```

Example:

```text
set MODEL_PATH=mlp_federated_best.pt
set META_PATH=metadata.pkl
```

The deployment therefore keeps:

```text
Model
  +
Feature metadata
  +
Class mapping
```

as a single inference package.

---

# 5. Alert Threshold Tuning

## Problem

The model produces a probability/confidence value for each prediction.

A prediction alone should not automatically generate an operational alert because low-confidence classifications can produce unnecessary alerts.

---

## Solution

The detector supports an alert threshold:

```text
ALERT_THRESHOLD
```

and a consecutive-window requirement:

```text
CONSECUTIVE_WINDOWS
```

For example:

```text
ALERT_THRESHOLD=0.80
CONSECUTIVE_WINDOWS=2
```

The deployment configuration documented in the project also experimented with:

```text
ALERT_THRESHOLD=0.90
CONSECUTIVE_WINDOWS=3
```

The principle is:

```text
Prediction
    │
    ▼
Confidence Check
    │
    ├── Low confidence → Ignore
    │
    └── High confidence
              │
              ▼
       Consecutive windows
              │
              ▼
             Alert
```

This reduces isolated false alerts.

---

# 6. Real-Time Replay Mode

## Problem

Testing the complete Windows event collection pipeline for every modification is slow and difficult to reproduce.

A deterministic way to replay previously collected telemetry was therefore useful during development.

---

## Solution

The detector provides a replay mode:

```text
python realtime_detector.py --replay
```

This allows the inference stage to be tested independently from live event collection.

The workflow becomes:

```text
Recorded Events
      │
      ▼
Replay
      │
      ▼
Feature Extraction
      │
      ▼
Model Inference
      │
      ▼
Prediction / Alert
```

Replay mode was particularly useful for validating detector behavior before performing live attack scenarios.

---

# 7. mTLS Configuration

## Problem

The Federated Learning infrastructure requires authenticated and encrypted communication between the clients and the FL server.

Simply enabling TLS encryption would not provide the intended client authentication model.

---

## Solution

Mutual TLS was deployed using certificates issued by the internal PKI.

The architecture uses:

```text
FL-ROOT-CA
     │
     ├── FL Server certificate
     │
     └── FL Client certificates
```

The server validates client certificates and clients validate the server certificate.

The resulting communication model is:

```text
Client
  │
  │  Client certificate
  │
  ▼
mTLS
  │
  │  Server certificate
  │
  ▼
FL Server
```

mTLS was validated as part of the live FL infrastructure.

---

# 8. RADIUS Authentication

## Problem

The FL infrastructure also required an authentication layer integrated with the Active Directory environment.

The objective was to authenticate users/clients through the existing identity infrastructure rather than maintaining an isolated authentication database.

---

## Solution

NPS RADIUS was deployed in the Active Directory environment.

The resulting architecture was:

```text
FL Client
    │
    ▼
 RADIUS
    │
    ▼
NPS
    │
    ▼
Active Directory
```

A dedicated network policy was used for the FL environment:

```text
FL-Policy
```

This integrated FL authentication with the existing AD identity infrastructure.

---

# 9. Secure Aggregation Integration

## Problem

Federated Learning without Secure Aggregation exposes individual client updates to the server.

This allowed the project's honest-but-curious server and property-inference experiments to recover client-specific information.

However, adding masking to the aggregation process also introduced additional mathematical and implementation constraints.

---

## Solution

A dedicated implementation was introduced:

```text
secagg.py
```

and enabled through:

```text
SECAGG=1
```

The server then operates on a masked aggregate instead of directly inspecting each individual client update.

The implementation was first validated with a unit test before being integrated into live FL training.

---

# 10. Secure Aggregation Convergence Issue

## Problem

During the Secure Aggregation training campaign, the model's F1 score could collapse when the aggregation weighting was configured incorrectly.

The problem was not the masking mechanism itself, but the mathematical weighting used during aggregation.

---

## Diagnosis

The protected aggregation required a uniform average over the participating clients:

```text
1 / N
```

Using an incorrect weighting configuration altered the optimization behavior.

---

## Solution

The aggregation was corrected to use the uniform average.

The corrected pipeline became:

```text
Client 1 ──┐
Client 2 ──┼──► Masked Updates
Client 3 ──┘
                 │
                 ▼
           Uniform Average
                 │
                 ▼
           Global Model
```

This allowed Secure Aggregation to operate while preserving the intended FedAvg-style averaging behavior.

The report identifies extended Secure Aggregation training as a remaining area for further evaluation.

---

# 11. Honest-but-Curious Server Testing

## Problem

Federated Learning was initially treated as if keeping raw telemetry on clients automatically guaranteed privacy.

The project demonstrated that this assumption is incomplete.

Without Secure Aggregation, the FL server receives individual model updates.

---

## Solution

The server was instrumented using:

```text
ATTACK_LOG_UPDATES=1
```

Individual updates were saved for analysis.

The resulting files followed the structure:

```text
round_NN_updates.npz
```

The project then analyzed:

* tensor norms;
* mean and standard deviation;
* inter-client distances;
* final-layer deviations;
* inferred local classes.

This experiment established the need for Secure Aggregation.

---

# 12. Property Inference Testing

## Problem

The individual model updates contained enough information to distinguish the local behavior of different clients.

This created a privacy risk even though raw telemetry never left the clients.

---

## Solution

The project implemented a dedicated analysis script:

```text
phase3/attacks/attack_property_inference.py
```

The analysis examined final-layer deviations across rounds.

The experiment successfully inferred information about client-local classes.

After Secure Aggregation was enabled, the same type of analysis was no longer able to directly exploit individual client updates.

---

# 13. Active Directory Hardening Validation

## Problem

Simply applying a security configuration does not prove that the corresponding attack path has disappeared.

For example, changing a GPO does not by itself demonstrate that Pass-the-Hash or LLMNR poisoning is no longer effective.

---

## Solution

The project used an attack → harden → replay methodology.

Examples:

```text
AS-REP Roasting
        ↓
Enable pre-authentication
        ↓
Replay
        ↓
No roastable accounts
```

```text
Pass-the-Hash
        ↓
RunAsPPL + Credential Guard + NTLM restrictions
        ↓
Replay
        ↓
STATUS_NOT_SUPPORTED
```

```text
Golden Ticket
        ↓
Double krbtgt rotation
        ↓
Replay
        ↓
KRB_AP_ERR_BAD_INTEGRITY
```

```text
ESC8
        ↓
Remove Web Enrollment
        ↓
Replay
        ↓
/certsrv/ → 404
```

This approach provided direct verification of the hardening measures.

---

# 14. Non-IID Federated Data

## Problem

The three FL clients do not observe the same class distribution.

Each client observes approximately 3–4 of the 9 attack classes.

This makes the federated optimization problem significantly more difficult than centralized training.

---

## Solution

The non-IID distribution was preserved intentionally.

Rather than artificially making every client identical, the project used the heterogeneous distribution to represent a more realistic federated environment.

The architecture therefore evaluates:

```text
Local specialization
       +
Federated aggregation
       =
Global model
```

This also makes the privacy experiments more meaningful because client-specific updates contain information about their local data distribution.

---

# 15. Small Dataset After Downsampling

## Problem

The preprocessing pipeline reduced the original telemetry to:

```text
~826K events
      ↓
656 windows
      ↓
227 windows
```

The resulting dataset is relatively small for a nine-class classification problem.

---

## Impact

The limited number of windows contributes to uncertainty in model performance and makes generalization more difficult.

This is reflected in the real-time detector's approximate:

```text
F1 ≈ 0.50
```

---

## Current Mitigation

The project identifies dataset enrichment as a priority for future work.

Future experiments should increase:

* the number of attack windows;
* the diversity of attack scenarios;
* the amount of normal activity;
* the distribution of classes across clients.

---

# 16. Practical Debugging Workflow

For future modifications, the recommended debugging sequence is:

```text
1. Validate the individual component
          │
          ▼
2. Validate local integration
          │
          ▼
3. Validate authentication / transport
          │
          ▼
4. Validate model input
          │
          ▼
5. Validate aggregation
          │
          ▼
6. Validate end-to-end execution
          │
          ▼
7. Replay the security scenario
```

This is particularly important for a project combining Windows infrastructure and machine learning because a failure in one layer can appear as a failure in another.

---

# 17. Troubleshooting Checklist

Before launching a complete FL or detection experiment:

### Windows

* [ ] Windows Event Log service is available.
* [ ] Security event collection works.
* [ ] XML event parsing succeeds.
* [ ] Required event IDs are available.
* [ ] Feature extraction produces valid vectors.

### Machine Learning

* [ ] Model checkpoint exists.
* [ ] Metadata file exists.
* [ ] Feature ordering matches training.
* [ ] StandardScaler configuration matches training.
* [ ] Label mapping matches training.

### Federated Learning

* [ ] FL server is reachable.
* [ ] Client authentication succeeds.
* [ ] mTLS certificates are valid.
* [ ] RADIUS authentication succeeds.
* [ ] Client updates are accepted.
* [ ] Aggregation produces a valid global model.

### Secure Aggregation

* [ ] `secagg.py` unit tests pass.
* [ ] `SECAGG=1` is correctly applied.
* [ ] Masked aggregation completes.
* [ ] Uniform averaging is preserved.
* [ ] Global model parameters remain valid.

### Active Directory

* [ ] Kerberos pre-authentication is enabled where required.
* [ ] Service-account SPNs are reviewed.
* [ ] NTLM restrictions are applied.
* [ ] LLMNR/NBT-NS configuration is verified.
* [ ] LAPS is operational.
* [ ] `krbtgt` rotation is verified.
* [ ] AD CS Web Enrollment exposure is checked.
* [ ] Tier restrictions are enforced.

---

# Conclusion

The troubleshooting process was an important part of the project because the final architecture spans several independent technologies:

```text
Windows
   +
Active Directory
   +
PKI
   +
RADIUS
   +
mTLS
   +
Flower
   +
Secure Aggregation
   +
PyTorch
   +
Real-Time Detection
```

The main implementation lessons were:

1. validate Windows event collection independently before connecting the ML pipeline;
2. keep training metadata synchronized with deployment;
3. treat mTLS and RADIUS as separate authentication/transport layers;
4. validate Secure Aggregation mathematically before live deployment;
5. preserve correct aggregation weights when adding privacy mechanisms;
6. verify AD hardening by replaying the original attack;
7. use replay mode to separate inference problems from event-collection problems.

These practices made it possible to move from individual components to a reproducible end-to-end security platform.
