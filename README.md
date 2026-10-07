# 🔐 Secure Active Directory Infrastructure & Federated Learning

A security research laboratory combining **Active Directory hardening**, **Federated Learning**, **Secure Aggregation**, **mTLS/RADIUS authentication**, and **real-time intrusion detection**.

The project was built as a reproducible laboratory environment on **VMware ESXi 7.0**, combining a redundant Active Directory infrastructure with a privacy-preserving federated machine-learning pipeline.

The main objective was not only to build an intrusion detector, but to connect the complete security lifecycle:

```text
Active Directory
      │
      ▼
Offensive Security Assessment
      │
      ▼
Windows Telemetry
      │
      ▼
Federated Learning
      │
      ▼
Privacy Attack
      │
      ▼
Secure Aggregation
      │
      ▼
Real-Time Detection
      │
      ▼
Active Directory Hardening
      │
      ▼
Attack Replay & Verification
```

---

## 📌 Project Overview

The project addresses three complementary security problems:

1. **How can a realistic Active Directory environment be deployed and hardened?**
2. **How can an intrusion-detection model be trained collaboratively without centralizing endpoint telemetry?**
3. **What information can a federated server infer from individual client updates without Secure Aggregation?**

The laboratory therefore combines:

* redundant Active Directory Domain Controllers;
* internal PKI;
* NPS/RADIUS authentication;
* certificate-based mTLS;
* Flower-based Federated Learning;
* non-IID client datasets;
* Secure Aggregation;
* Active Directory attack simulation;
* real-time Windows intrusion detection;
* post-attack hardening and replay validation.

---

# 🏗️ Architecture

The laboratory was deployed on **VMware ESXi 7.0**.

```text
                         ┌──────────────────────────┐
                         │       nord.corp AD       │
                         │                          │
                         │  ┌────────┐  ┌────────┐  │
                         │  │ DC01   │  │ DC02   │  │
                         │  │ AD/DNS │  │ AD/DNS │  │
                         │  └────────┘  └────────┘  │
                         └─────────────┬────────────┘
                                       │
                    ┌──────────────────┼──────────────────┐
                    │                  │                  │
                    ▼                  ▼                  ▼
             ┌────────────┐     ┌────────────┐     ┌────────────┐
             │  CLIENT01  │     │  CLIENT02  │     │  CLIENT03  │
             │  Windows   │     │  Windows   │     │  Windows   │
             │ FL Client  │     │ FL Client  │     │ FL Client  │
             └──────┬─────┘     └──────┬─────┘     └──────┬─────┘
                    │                  │                  │
                    └──────────────────┼──────────────────┘
                                       │
                              Federated Training
                                       │
                                       ▼
                         ┌──────────────────────────┐
                         │      FL Server           │
                         │ Ubuntu + Flower 1.13     │
                         │ RADIUS + mTLS            │
                         │ Secure Aggregation       │
                         └────────────┬─────────────┘
                                      │
                                      ▼
                         Federated Intrusion Model

        ┌────────────────┐
        │ FL-ROOT-CA     │
        │ Internal PKI   │
        └────────────────┘

        ┌────────────────┐
        │ NPS / RADIUS   │
        │ FL-Policy      │
        └────────────────┘

        ┌────────────────┐
        │ Kali Linux     │
        │ Security Tests │
        └────────────────┘
```

The detailed architecture is documented in [`docs/architecture.md`](docs/architecture.md).

---

# 📊 Key Results

| Component                         |                Result |
| --------------------------------- | --------------------: |
| Security events processed         |              ~826,000 |
| Windows time windows extracted    |                   656 |
| Windows after downsampling        |                   227 |
| ATT&CK technique classes          |                     9 |
| Standardized ML features          |                    29 |
| Client datasets                   |        6 `.npz` files |
| Federated clients                 |                     3 |
| Centralized MLP Macro F1          |             **0.607** |
| Real-time detector F1             |            **≈ 0.50** |
| Federated training                |            Successful |
| mTLS authentication               |             Validated |
| Secure Aggregation                | Integrated and tested |
| Property inference without SecAgg |            Successful |
| Property inference with SecAgg    |           Neutralized |
| Real Kali attack detection        |             Validated |

---

# 🧠 Dataset & Machine Learning

The machine-learning pipeline is based on **Mordor/OTRF Windows security telemetry**.

The preprocessing pipeline produced:

```text
~826K events
     │
     ▼
656 time windows
     │
     ▼
Downsampling
     │
     ▼
227 windows
     │
     ▼
29 standardized features
     │
     ▼
9 ATT&CK technique classes
```

The dataset was distributed across **three federated clients** with deliberately non-IID distributions.

Each client observes approximately **3–4 of the 9 classes**, creating a challenging federated-learning scenario.

---

## Neural Network

The baseline classifier is a multilayer perceptron:

```text
29 input features
       │
       ▼
    Dense 64
       │
       ▼
    Dense 32
       │
       ▼
     Dense 9
       │
       ▼
  ATT&CK classes
```

Training configuration:

* 100 epochs;
* 29 input features;
* 9 output classes;
* StandardScaler preprocessing;
* LabelEncoder class mapping.

The centralized baseline achieved a **Macro F1 of 0.607**.

The federated version was subsequently trained across three clients.

More details: [`docs/federated-learning.md`](docs/federated-learning.md).

---

# 🔄 Federated Learning

The federated architecture uses **Flower 1.13**.

Instead of sending endpoint telemetry to a central server, each Windows client trains locally and sends model updates.

```text
                ┌──────────────┐
                │   FL Server  │
                └──────┬───────┘
                       │
             Global model distribution
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     ┌────────┐   ┌────────┐   ┌────────┐
     │Client 1│   │Client 2│   │Client 3│
     │ local  │   │ local  │   │ local  │
     │ train  │   │ train  │   │ train  │
     └────┬───┘   └────┬───┘   └────┬───┘
          │            │            │
          └────────────┼────────────┘
                       │
                 Model updates
                       │
                       ▼
                Global aggregation
```

The communication layer was protected using:

* RADIUS/NPS pre-authentication;
* internal PKI;
* mutual TLS;
* certificate-authenticated clients;
* audit logging.

The distinction between **transport security** and **update privacy** is important:

```text
mTLS
  → protects updates while travelling over the network

Secure Aggregation
  → protects individual updates from the aggregation server
```

---

# 🔎 Privacy Attack: Honest-but-Curious Server

The project deliberately evaluated what happens when the federated server is **honest-but-curious**.

Without Secure Aggregation, the server can access individual client model updates after transport decryption.

The experimental configuration enabled:

```text
ATTACK_LOG_UPDATES=1
```

The server stored individual updates such as:

```text
round_NN_updates.npz
```

Each client update contained six model tensors:

```text
fc1.weight
fc1.bias
fc2.weight
fc2.bias
fc3.weight
fc3.bias
```

Thirty federated rounds were analyzed.

The attack scripts examined:

* L2 norms;
* mean and standard deviation;
* inter-client distances;
* final-layer deviations;
* client-specific model behavior.

A property-inference attack was then used to infer information about the classes represented locally by each client.

The experiment demonstrated that the server could reconstruct useful information about client-local class distributions.

Detailed methodology: [`docs/attacks-fl.md`](docs/attacks-fl.md).

---

# 🛡️ Secure Aggregation

Secure Aggregation was integrated to prevent the server from directly observing individual client updates.

Conceptually:

```text
Client 1 ──┐
            │
Client 2 ──┼──> Masked updates ──> Server ──> Aggregate
            │
Client 3 ──┘
```

The implementation was tested using:

```text
secagg.py
SECAGG=1
```

The server observes the masked aggregate rather than the individual client updates.

The replay experiment showed that the information leakage observed without Secure Aggregation was neutralized.

For example, inter-client update distances and final-layer deviation scores became substantially smaller and much less client-specific.

| Metric                |  Without SecAgg | With SecAgg |
| --------------------- | --------------: | ----------: |
| `fc3` client distance |     0.026–0.035 | 0.001–0.004 |
| Deviation score       |     0.015–0.046 | 0.001–0.011 |
| Top inferred classes  | Client-specific |   Identical |
| Property inference    |      Successful | Neutralized |

One important limitation was observed during Secure Aggregation training: stable convergence required a **uniform `1/N` aggregation weighting**. An extended Secure Aggregation training campaign remains future work.

More details: [`docs/secure-aggregation.md`](docs/secure-aggregation.md).

---

# 🖥️ Real-Time Intrusion Detection

The federated model was deployed as a host-based intrusion detector on the three Windows clients.

The detector is implemented in:

```text
realtime_detector.py
```

It processes approximately **30-second windows** of Windows telemetry.

Data sources include:

* Windows Security logs;
* Sysmon;
* PowerShell-related events.

The inference pipeline is:

```text
Windows Events
      │
      ▼
30-second window
      │
      ▼
Feature extraction
      │
      ▼
StandardScaler
      │
      ▼
Federated MLP
      │
      ▼
9-class prediction
      │
      ▼
Confidence threshold
      │
      ▼
Real-time alert
```

The deployed model uses:

```text
mlp_federated_best.pt
metadata.pkl
```

The detector achieved:

* inference time below 5 ms;
* approximately 120 MB memory usage;
* approximately zero alerts during normal observed windows;
* successful detection during live offensive testing;
* confidence reaching 1.00 for observed attack predictions.

Observed attack predictions included:

```text
ntds_dump
apt29
```

The real-time detector achieved an F1 score of approximately **0.50**.

This result is useful as a laboratory validation, but it also highlights the need for additional data and further model improvement.

Detailed implementation: [`docs/realtime-detection.md`](docs/realtime-detection.md).

---

# ⚔️ Active Directory Security Assessment

The laboratory was intentionally configured with controlled weaknesses to reproduce representative Active Directory attack techniques.

The assessment followed:

```text
Attack
  │
  ▼
Observation
  │
  ▼
Hardening
  │
  ▼
Replay
  │
  ▼
Verification
```

The main techniques included:

* BloodHound reconnaissance;
* AS-REP Roasting;
* Kerberoasting;
* Pass-the-Hash;
* LLMNR/NBT-NS poisoning;
* local administrator credential reuse;
* Golden Ticket;
* AD CS / ESC8.

The complete assessment is documented in [`docs/ad-attacks.md`](docs/ad-attacks.md).

---

# 🔐 Active Directory Hardening

The project did not stop at attack reproduction.

Each major attack was followed by a corresponding hardening measure and a replay test.

Key controls included:

| Attack               | Hardening                                       | Verification               |
| -------------------- | ----------------------------------------------- | -------------------------- |
| AS-REP Roasting      | Kerberos pre-authentication                     | No roastable accounts      |
| Kerberoasting        | AES + gMSA + SPN removal                        | No vulnerable SPN          |
| Pass-the-Hash        | RunAsPPL + Credential Guard + NTLM restrictions | `STATUS_NOT_SUPPORTED`     |
| LLMNR Poisoning      | Disable LLMNR/NBT-NS + SMB signing              | Poisoning path removed     |
| Local admin reuse    | LAPS                                            | Old hash rejected          |
| Golden Ticket        | Double `krbtgt` rotation                        | `KRB_AP_ERR_BAD_INTEGRITY` |
| ESC8                 | Remove Web Enrollment                           | `/certsrv/` → 404          |
| DNS poisoning        | DNSSEC                                          | Signed zone                |
| Privilege escalation | Tier 0/1/2 model                                | Cross-tier login blocked   |
| AD reconnaissance    | RestrictRemoteSAM + session restrictions        | Reduced enumeration        |

The hardening architecture combines:

```text
Identity
   │
   ├── Kerberos hardening
   ├── gMSA / AES
   ├── krbtgt rotation
   └── Tier model

Network
   │
   ├── NTLM restrictions
   ├── SMB signing
   ├── LLMNR disabled
   └── NBT-NS disabled

Endpoint
   │
   ├── RunAsPPL
   ├── Credential Guard
   ├── LAPS
   └── Windows auditing

DNS / PKI
   │
   ├── DNSSEC
   └── AD CS hardening
```

Detailed hardening documentation: [`docs/hardening.md`](docs/hardening.md).

---

# 🔗 From Attack to Detection

One of the main objectives of the project was to connect offensive security activity with machine-learning-based detection.

The complete pipeline is:

```text
             Active Directory Attack
                       │
                       ▼
          Windows Security / Sysmon
                       │
                       ▼
             Local Feature Extraction
                       │
                       ▼
                FL Client
                       │
                       ▼
              Federated Model
                       │
                       ▼
              Attack Classification
                       │
                       ▼
               Real-Time Alert
                       │
                       ▼
                Hardening
                       │
                       ▼
                Attack Replay
                       │
                       ▼
                 Verification
```

This creates a closed security-engineering loop rather than treating penetration testing, machine learning, and Active Directory administration as separate activities.

---

# 📈 Results

## Dataset

```text
~826,000 events
656 windows
227 windows after downsampling
29 features
9 ATT&CK classes
6 .npz datasets
3 federated clients
```

## Machine Learning

```text
Architecture: 29 → 64 → 32 → 9
Epochs:       100
Centralized:  Macro F1 = 0.607
Realtime:     F1 ≈ 0.50
```

## Privacy

```text
Without SecAgg
→ individual updates visible
→ client-specific statistics
→ local class information inferred

With SecAgg
→ masked aggregation
→ individual updates protected
→ replayed inference attack neutralized
```

## Infrastructure

The project successfully validated:

* redundant Active Directory Domain Controllers;
* internal PKI;
* NPS/RADIUS authentication;
* certificate-based mTLS;
* federated training;
* Secure Aggregation;
* Windows real-time detection;
* live Kali attack validation;
* Active Directory hardening and replay verification.

Detailed quantitative results are available in [`docs/results.md`](docs/results.md).

---

# 🗂️ Repository Structure

```text
active-directory-secure-federated-learning/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── lab-environment.md
│   ├── active-directory.md
│   ├── pki.md
│   ├── radius.md
│   ├── federated-learning.md
│   ├── secure-aggregation.md
│   ├── attacks-fl.md
│   ├── realtime-detection.md
│   ├── ad-attacks.md
│   ├── hardening.md
│   ├── results.md
│   └── troubleshooting.md
│
├── diagrams/
│
├── screenshots/
│
├── phase1-data/
├── phase2-baseline/
├── phase3-fl/
├── phase4-secagg/
├── phase5-detection/
├── phase6-ad-security/
│
├── scripts/
├── configs/
│
└── ...
```

The README provides the project overview, while the `docs/` directory contains the detailed technical documentation.

---

# 🔬 Reproducibility

The project was designed as a reproducible laboratory workflow.

A typical federated-learning run follows:

### FL Server

```bash
cd ~/projet_pfa
source venv/bin/activate

export RADIUS_SECRET="<RADIUS_SECRET>"
export MTLS_CERT_DIR="./certs"

python -m phase3.server.fl_server
```

### Windows Client

```powershell
cd C:\PFA\code
.\venv\Scripts\activate

python -m phase3.client.fl_client
```

### Honest-but-Curious Experiment

```bash
ATTACK_LOG_UPDATES=1 python -m phase3.server.fl_server
```

Then analyze the captured updates:

```bash
python phase3/attacks/attack_curious_server.py --round 30
```

Property inference:

```bash
python phase3/attacks/attack_property_inference.py --rounds 1-30 --top 4
```

### Real-Time Detector

```powershell
set MODEL_PATH=mlp_federated_best.pt
set META_PATH=metadata.pkl
set ALERT_THRESHOLD=0.80
set CONSECUTIVE_WINDOWS=2

python realtime_detector.py
```

Replay mode:

```powershell
python realtime_detector.py --replay
```

**Credentials, certificates, hashes, and secrets must never be committed to the repository.**

---

# ⚠️ Limitations

The project has several important limitations.

### Dataset size

After downsampling, only **227 windows** remained.

This limits the statistical robustness of the reported detection performance.

### Extreme non-IID distribution

Each federated client observes only approximately **3–4 of the 9 classes**.

This creates a difficult learning environment and contributes to convergence challenges.

### Secure Aggregation training

Secure Aggregation successfully protected individual updates, but stable convergence required a uniform `1/N` aggregation strategy.

A larger dedicated training campaign remains future work.

### Real-Time detector

The real-time detector achieved an F1 score of approximately **0.50**.

False positives and false negatives therefore remain possible.

More representative telemetry and additional attack scenarios are required before considering a stronger operational deployment.

### Laboratory scope

The results were obtained inside a controlled ESXi laboratory.

They should therefore be interpreted as **research and laboratory validation**, not as a guarantee of security in arbitrary production environments.

---

# 🚀 Future Work

Potential extensions include:

* expanding the Windows telemetry dataset;
* reducing the impact of the non-IID distribution;
* extending Secure Aggregation training;
* evaluating additional aggregation strategies;
* improving real-time detector F1;
* adding more ATT&CK techniques;
* evaluating additional privacy attacks;
* extending automated hardening verification;
* improving the visualization of federated-training metrics;
* adding more comprehensive automated tests.

---

# 🧰 Technologies

### Infrastructure

* VMware ESXi 7.0
* Windows Server
* Windows clients
* Ubuntu Server
* Kali Linux
* Active Directory Domain Services
* DNS
* NPS / RADIUS
* Active Directory Certificate Services

### Security

* BloodHound CE
* RustHound-CE
* Impacket
* Mimikatz
* CrackMapExec
* Responder
* Hashcat
* Certipy-ad
* LAPS
* Credential Guard
* RunAsPPL
* DNSSEC

### Machine Learning

* Python
* Flower 1.13
* PyTorch
* StandardScaler
* LabelEncoder
* NumPy

### Federated Security

* RADIUS pre-authentication
* mTLS
* Internal PKI
* Secure Aggregation
* Honest-but-curious server analysis

---

# 📚 Documentation

| Topic                  | Documentation                                              |
| ---------------------- | ---------------------------------------------------------- |
| Architecture           | [`docs/architecture.md`](docs/architecture.md)             |
| Lab environment        | [`docs/lab-environment.md`](docs/lab-environment.md)       |
| Active Directory       | [`docs/active-directory.md`](docs/active-directory.md)     |
| PKI                    | [`docs/pki.md`](docs/pki.md)                               |
| RADIUS                 | [`docs/radius.md`](docs/radius.md)                         |
| Federated Learning     | [`docs/federated-learning.md`](docs/federated-learning.md) |
| Secure Aggregation     | [`docs/secure-aggregation.md`](docs/secure-aggregation.md) |
| FL Privacy Attacks     | [`docs/attacks-fl.md`](docs/attacks-fl.md)                 |
| Real-Time Detection    | [`docs/realtime-detection.md`](docs/realtime-detection.md) |
| AD Security Assessment | [`docs/ad-attacks.md`](docs/ad-attacks.md)                 |
| Hardening              | [`docs/hardening.md`](docs/hardening.md)                   |
| Results                | [`docs/results.md`](docs/results.md)                       |
| Troubleshooting        | [`docs/troubleshooting.md`](docs/troubleshooting.md)       |

---

# 🔐 Security Philosophy

The project follows a defense-in-depth approach:

```text
Prevent
  ↓
Detect
  ↓
Protect Privacy
  ↓
Respond
  ↓
Harden
  ↓
Replay
  ↓
Verify
```

The central idea is that security controls should be **tested empirically whenever possible**.

A configuration is therefore not considered fully validated simply because it is enabled:

```text
Configuration
     ↓
Attack Replay
     ↓
Observed Behavior
     ↓
Verification
```

This methodology was applied to both the Active Directory hardening controls and the federated-learning security model.

---

# ⚠️ Disclaimer

This repository documents a controlled cybersecurity research laboratory.

All offensive security techniques were performed against intentionally configured laboratory systems under controlled conditions.

Do not reproduce these techniques against systems for which you do not have explicit authorization.

---

# 📌 Project Summary

This project demonstrates an end-to-end security laboratory combining:

```text
Active Directory
      +
Offensive Security
      +
Windows Telemetry
      +
Federated Learning
      +
mTLS / RADIUS
      +
Secure Aggregation
      +
Real-Time Detection
      +
Active Directory Hardening
```

The main contribution is the integration of these components into a single experimental workflow where **attacks generate telemetry, telemetry feeds federated learning, privacy attacks expose the need for Secure Aggregation, and the resulting detection and hardening mechanisms are validated through replay**.
