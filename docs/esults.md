# Experimental Results

## Overview

This project evaluates a complete security pipeline combining:

* Active Directory infrastructure;
* Federated Learning;
* Secure Aggregation;
* Privacy attack analysis;
* Real-time intrusion detection;
* Active Directory attack and hardening validation.

The experiments were conducted on a real virtualized laboratory environment and combine offline datasets with live Windows and Active Directory telemetry.

---

# 1. Dataset Results

The initial dataset was built from Mordor / OTRF security telemetry.

| Metric                     |   Result |
| -------------------------- | -------: |
| Events parsed              | ~826,000 |
| Initial windows extracted  |      656 |
| Windows after downsampling |      227 |
| ATT&CK classes             |        9 |
| Standardized features      |       29 |
| Generated `.npz` files     |        6 |
| FL clients                 |        3 |

The resulting dataset represents nine attack-related classes derived from Windows security telemetry.

The preprocessing pipeline converts raw event data into fixed-size windows suitable for machine-learning training.

```text
Mordor / OTRF Events
        │
        ▼
Event Parsing
        │
        ▼
Feature Extraction
        │
        ▼
Window Generation
        │
        ▼
Downsampling
        │
        ▼
29 Standardized Features
        │
        ▼
Federated Clients
```

---

# 2. Centralized Baseline

Before evaluating Federated Learning, a centralized MLP baseline was trained.

### Model

```text
Input:       29 features
     │
     ▼
Dense:       64
     │
     ▼
Dense:       32
     │
     ▼
Output:      9 classes
```

Training configuration:

| Parameter         |            Value |
| ----------------- | ---------------: |
| Architecture      | 29 → 64 → 32 → 9 |
| Epochs            |              100 |
| Number of classes |                9 |
| Metric            |         Macro F1 |
| Macro F1          |        **0.607** |

The centralized baseline establishes the reference point for the subsequent federated experiments.

---

# 3. Federated Learning

The federated system uses three clients.

Each client trains locally and sends model parameters to the FL server rather than transmitting its raw telemetry.

```text
             ┌───────────────┐
             │  FL Server    │
             │ Flower 1.13   │
             └───────┬───────┘
                     │
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
      Client 1   Client 2   Client 3
          │          │          │
       Local      Local      Local
      Training   Training   Training
```

The system includes:

* Flower 1.13;
* RADIUS pre-authentication;
* mTLS;
* audit logging;
* distributed local training;
* model aggregation.

The infrastructure was deployed on ESXi with redundant Active Directory domain controllers and Windows clients.

---

# 4. Non-IID Data Distribution

The three clients do not observe identical class distributions.

Each client observes approximately **3–4 of the 9 classes**.

This creates a strongly non-IID federated learning environment.

```text
Global Dataset
      │
      ├──────────────┐
      │              │
      ▼              ▼
   Client 1       Client 2       Client 3
   3–4 classes    3–4 classes    3–4 classes
      │              │              │
      └──────────────┼──────────────┘
                     ▼
              Global Model
```

This setup reflects the intended operational scenario in which different machines or sites observe different subsets of attack activity.

---

# 5. Privacy Leakage Without Secure Aggregation

The first privacy experiment evaluated an honest-but-curious FL server.

Without Secure Aggregation, the server receives individual client updates.

The experiment enabled:

```text
ATTACK_LOG_UPDATES=1
```

and stored individual updates for analysis.

For each round, the recorded update contains six model tensors:

```text
fc1.weight
fc1.bias

fc2.weight
fc2.bias

fc3.weight
fc3.bias
```

A total of 30 rounds were generated for the privacy analysis.

---

# 6. Honest-but-Curious Server Analysis

The server-side analysis examined the individual client updates using:

```text
phase3/attacks/attack_curious_server.py
```

The analysis included:

* L2 norms;
* mean;
* standard deviation;
* inter-client distances;
* final-layer deviations.

The individual updates showed measurable differences between clients.

For the final layer, the reported inter-client distances were approximately:

```text
0.026 – 0.035
```

This demonstrates that the server can distinguish client-specific model updates.

---

# 7. Property Inference

A second experiment attempted to infer information about the local data distribution of each client.

The analysis was performed with:

```text
phase3/attacks/attack_property_inference.py
```

The method analyzed deviations in the final layer to estimate which classes were represented locally.

The experiment successfully reconstructed information about the local classes of individual clients.

This is an important result because the raw training data never leaves the client, yet individual model updates can still reveal information about the client's local distribution.

---

# 8. Secure Aggregation

Secure Aggregation was subsequently integrated to prevent the server from directly inspecting individual client updates.

The implementation introduced:

```text
secagg.py
```

and was activated with:

```text
SECAGG=1
```

The server receives a masked aggregate instead of directly observing each client's individual update.

Conceptually:

```text
Client 1 ──┐
Client 2 ──┼──► Masked Aggregation ──► Server
Client 3 ──┘
```

The masking terms cancel during aggregation, allowing the global update to be recovered without exposing each individual contribution.

---

# 9. Secure Aggregation Verification

The Secure Aggregation module was first validated using a unit test.

The resulting implementation was then integrated into the live FL training pipeline.

The server-side observation changed from:

```text
Individual client updates
```

to:

```text
Masked aggregate
```

The privacy attack experiments were then replayed after Secure Aggregation.

The previously observable client-specific information was no longer directly available to the server.

---

# 10. Privacy Metrics Before and After Secure Aggregation

| Metric                      | Without Secure Aggregation | With Secure Aggregation |
| --------------------------- | -------------------------: | ----------------------: |
| Individual updates          |           Directly visible |                  Masked |
| L2 norm differences         |            Distinguishable | Approximately identical |
| Inter-client `fc3` distance |                0.026–0.035 |             0.001–0.004 |
| Top-4 inferred classes      |            Client-specific |               Identical |
| Deviation scores            |                0.015–0.046 |             0.001–0.011 |
| Property inference          |                 Successful |             Neutralized |

These experiments demonstrate the practical security value of Secure Aggregation in the proposed FL architecture.

---

# 11. Replay Attack After Secure Aggregation

The same privacy-analysis methodology was applied after enabling Secure Aggregation.

The objective was to determine whether the information previously extracted from individual updates remained available.

The result was that the client-specific information leakage was neutralized by masking.

This provides an experimental comparison between:

```text
FL without Secure Aggregation
        │
        ▼
Individual update visibility
        │
        ▼
Property inference
```

and:

```text
FL with Secure Aggregation
        │
        ▼
Masked aggregation
        │
        ▼
No individual update visibility
        │
        ▼
Property inference neutralized
```

---

# 12. Secure Aggregation Convergence Observation

During the live Secure Aggregation experiments, an important implementation observation was made.

The aggregation required a uniform average:

```text
1 / N
```

rather than an incorrect weighting configuration.

Using the wrong aggregation weighting caused the federated F1 score to collapse.

After correcting the aggregation to a uniform average, the Secure Aggregation pipeline operated as intended.

This highlights an important practical point:

> Privacy protection must preserve the mathematical behavior of the underlying federated optimization algorithm.

The report also identifies extended Secure Aggregation training as an area for future work.

---

# 13. Real-Time Intrusion Detection

The federated model was subsequently deployed as a real-time detector on the three Windows clients.

The detector processes Windows telemetry in approximately 30-second windows.

The processing pipeline is:

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
Prediction
      │
      ▼
Confidence Threshold
      │
      ▼
Security Alert
```

The deployed model uses:

```text
mlp_federated_best.pt
```

with metadata stored in:

```text
metadata.pkl
```

---

# 14. Real-Time Detection Results

The live detector was tested against attacks launched from Kali Linux against the Windows environment.

The detector successfully produced attack classifications during live execution.

Reported examples included:

```text
ntds_dump
apt29
```

with observed confidence reaching:

```text
1.00
```

The detector therefore demonstrated that the federated model could be connected to live Windows security telemetry rather than being limited to offline evaluation.

---

# 15. Real-Time Performance

The report measured the operational behavior of the detector.

| Metric                       |     Result |
| ---------------------------- | ---------: |
| Inference latency            |     < 5 ms |
| Memory usage                 |   ≈ 120 MB |
| Alerts during normal windows |        ≈ 0 |
| Live attack detection        | Successful |
| Real-time detector F1        |     ≈ 0.50 |

The inference latency is sufficiently small for the demonstrated window-based detection workflow.

However, the overall F1 score remains moderate and indicates that the detector requires additional training data and further optimization before being considered production-ready.

---

# 16. Active Directory Security Validation

The final phase connected offensive security testing with defensive hardening.

The project tested multiple Active Directory attack techniques and then applied corresponding mitigations.

| Attack                       | Hardening                                       | Verification              |
| ---------------------------- | ----------------------------------------------- | ------------------------- |
| AS-REP Roasting              | Kerberos pre-authentication                     | No roastable accounts     |
| Kerberoasting                | AES / gMSA / SPN removal                        | No vulnerable SPN         |
| Pass-the-Hash                | RunAsPPL / Credential Guard / NTLM restrictions | Replay failed             |
| LLMNR / NBT-NS               | Disable protocols + SMB signing                 | Attack path removed       |
| Local admin credential reuse | LAPS                                            | Old hash rejected         |
| Golden Ticket                | Double `krbtgt` rotation                        | Forged ticket rejected    |
| ESC8                         | Remove AD CS Web Enrollment                     | `/certsrv/` unavailable   |
| Privilege escalation         | Tier 0/1/2 separation                           | Cross-tier access blocked |
| DNS attacks                  | DNSSEC                                          | Signed DNS zone           |

This creates a complete attack → mitigation → verification cycle.

---

# 17. Security Telemetry

The Active Directory attack scenarios generate Windows telemetry that is also relevant to the federated detector.

The project particularly uses events such as:

```text
4624  → Successful logon
4625  → Failed logon
4672  → Special privileges assigned
4768  → Kerberos authentication ticket request
4769  → Kerberos service ticket request
```

These events form part of the telemetry used to characterize attack behavior.

The result is a direct relationship between the AD security laboratory and the machine-learning pipeline.

---

# 18. End-to-End Results

The complete project can be summarized as:

```text
                ┌──────────────────────┐
                │ Mordor / OTRF Data   │
                └──────────┬───────────┘
                           │
                           ▼
                 ┌──────────────────┐
                 │ Feature Pipeline │
                 └────────┬─────────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ 3 FL Clients  │
                  └───────┬───────┘
                          │
                          ▼
                  ┌───────────────┐
                  │ FL Aggregator │
                  └───────┬───────┘
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
       Without SecAgg           With SecAgg
              │                       │
              ▼                       ▼
       Privacy Leakage          Masked Updates
              │                       │
              └───────────┬───────────┘
                          │
                          ▼
                 Federated Model
                          │
                          ▼
                 Real-Time Detector
                          │
                          ▼
                  Live AD Attacks
                          │
                          ▼
                  Detection + Alert
                          │
                          ▼
                   AD Hardening
```

---

# 19. Main Quantitative Results

| Component                  |         Result |
| -------------------------- | -------------: |
| Events processed           |      **~826K** |
| Initial windows            |        **656** |
| Final windows              |        **227** |
| Features                   |         **29** |
| ATT&CK classes             |          **9** |
| FL clients                 |          **3** |
| Centralized Macro F1       |      **0.607** |
| Real-time detector F1      |     **≈ 0.50** |
| FL privacy rounds analyzed |         **30** |
| Real-time inference        |     **< 5 ms** |
| Detector memory            |   **≈ 120 MB** |
| Normal-window alerts       |        **≈ 0** |
| Live attack detection      | **Successful** |

---

# 20. Limitations

The experimental results must be interpreted in the context of the laboratory configuration.

## Dataset Size

After preprocessing and downsampling, only 227 windows remained.

This limits the statistical robustness of the reported machine-learning results.

## Strong Non-IID Distribution

Each client observes only approximately 3–4 of the 9 classes.

This reflects a realistic distributed setting but makes federated convergence more difficult.

## Real-Time F1

The real-time detector achieves approximately:

```text
F1 ≈ 0.50
```

This indicates that false positives and false negatives remain possible.

Additional telemetry and training data are required to improve generalization.

## Secure Aggregation Training

Secure Aggregation was successfully implemented and validated, but the report identifies the need for an extended training campaign to further evaluate convergence and model performance under the protected aggregation regime.

---

# 21. Future Experimental Directions

The project identifies several extensions:

* consolidate the comparison between FL without SA and FL with SA;
* enrich the training dataset;
* evaluate Differential Privacy with Gaussian noise;
* evaluate Krum;
* evaluate Trimmed Mean;
* integrate the detector with a SIEM;
* evaluate multiple Active Directory domains with trust relationships;
* extend the architecture toward a Zero Trust multi-tenant environment.

These extensions would improve both the privacy guarantees and the robustness of the detection system.

---

# Conclusion

The experiments demonstrate a complete security pipeline rather than an isolated machine-learning model.

The project successfully combines:

```text
Active Directory
       +
Federated Learning
       +
mTLS / RADIUS
       +
Secure Aggregation
       +
Privacy Attack Analysis
       +
Real-Time Detection
       +
Offensive Security Testing
       +
Hardening
```

The strongest experimental result is the demonstration that federated learning alone does not completely prevent information leakage: an honest-but-curious server can inspect individual model updates and infer client-specific information.

Secure Aggregation addresses this specific visibility problem by preventing the server from directly observing individual client contributions.

The final system therefore demonstrates both sides of privacy-preserving security engineering:

```text
Build the system
      ↓
Attack the system
      ↓
Measure the leakage
      ↓
Apply protection
      ↓
Replay the attack
      ↓
Verify the protection
```

This methodology is the central experimental contribution of the project.
