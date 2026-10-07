# 🔐 Secure Active Directory Infrastructure & Federated Learning

A security research laboratory combining **Active Directory, PKI, RADIUS, mTLS, Federated Learning, Secure Aggregation and offensive security testing** in a reproducible virtualized environment.

The project was implemented on **VMware ESXi 7.0** and integrates enterprise identity infrastructure with privacy-preserving machine learning for federated intrusion detection.

---

## 🎯 Project Overview

This project explores how **Federated Learning can be integrated into a secured Active Directory environment** while protecting both authentication and federated model updates.

The laboratory combines:

* 🏢 Active Directory Domain Services
* 🔐 Active Directory Certificate Services (AD CS)
* 🔑 NPS / RADIUS authentication
* 🔒 Mutual TLS (mTLS)
* 🤖 Federated Learning with Flower
* 🛡️ Secure Aggregation (SecAgg+)
* 🕵️ Federated Learning privacy attacks
* 🚨 Real-time intrusion detection
* ⚔️ Active Directory security assessment
* 🛡️ Security hardening and validation

The project follows an experimental security approach:

```text
Active Directory Infrastructure
            │
            ▼
     Security Telemetry
            │
            ▼
      Federated Learning
            │
       ┌────┴────┐
       │         │
       ▼         ▼
 Without SA    SecAgg+
       │         │
       ▼         ▼
Privacy Attack  Protected
       │         │
       └────┬────┘
            ▼
   Federated Detection
            │
            ▼
     Real-Time Alerts
```

---

## 🏆 Key Results

The experimental environment produced the following results:

| Component                 |                        Result |
| ------------------------- | ----------------------------: |
| Security events processed |                  **826,000+** |
| Extracted time windows    |                       **656** |
| MITRE ATT&CK classes      |                         **9** |
| Standardized features     |                        **29** |
| Federated clients         |                         **3** |
| Centralized MLP Macro F1  |                     **0.607** |
| Real-time detector F1     |                    **≈ 0.50** |
| FL privacy attack         | **Successfully demonstrated** |
| Property inference        | **Successfully demonstrated** |
| Secure Aggregation        |  **Integrated and validated** |
| Real attack validation    | **Performed from Kali Linux** |

The experiments demonstrate both the **privacy risks of federated learning without secure aggregation** and the ability of Secure Aggregation to prevent the server from directly inspecting individual client updates.

---

## 🏗️ Architecture

The laboratory is deployed on a virtualized infrastructure and contains:

* **DC01 / DC02** — redundant Active Directory Domain Controllers
* **Windows clients** — federated learning participants
* **FL Server** — federated coordination and model aggregation
* **FL-ROOT-CA** — internal certificate authority
* **NPS / RADIUS** — centralized authentication
* **Kali Linux** — controlled offensive security testing

```text
                         ┌──────────────────────┐
                         │   Federated Learning │
                         │       Server         │
                         └──────────┬───────────┘
                                    │
                             Secure Aggregation
                                    │
                 ┌──────────────────┼──────────────────┐
                 │                  │                  │
          ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
          │   Windows   │    │   Windows   │    │   Windows   │
          │   Client 1  │    │   Client 2  │    │   Client 3  │
          │   FL Node   │    │   FL Node   │    │   FL Node   │
          └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
                 │                  │                  │
                 └──────────────────┼──────────────────┘
                                    │
                                  mTLS
                                    │
                         ┌──────────▼──────────┐
                         │ Active Directory    │
                         │      Domain         │
                         └──────────┬──────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
        ┌─────▼─────┐        ┌──────▼──────┐       ┌─────▼─────┐
        │   DC01    │        │    AD CS    │       │ NPS/RADIUS│
        │   DC02    │        │  FL-ROOT-CA │       │           │
        └───────────┘        └─────────────┘       └───────────┘

                         ┌──────────────┐
                         │ Kali Linux   │
                         │ Security Lab │
                         └──────────────┘
```
## 📊 Experimental Results

### Dataset & Preprocessing

The federated intrusion detection dataset was generated from **Mordor / OTRF security telemetry**.

The preprocessing pipeline produced:

| Metric                            |             Result |
| --------------------------------- | -----------------: |
| Security events parsed            |       **826,000+** |
| Extracted time windows            |            **656** |
| Final downsampled windows         |            **227** |
| MITRE ATT&CK techniques / classes |              **9** |
| Standardized features             |             **29** |
| Generated dataset files           | **6 `.npz` files** |

The dataset represents multiple attack techniques and is distributed across three federated clients using a **non-IID configuration**.

---

### 🧠 Machine Learning Results

A centralized MLP was first trained as a baseline before federated training.

**Architecture:**

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
   MITRE ATT&CK classes
```

Training configuration:

| Parameter            |       Value |
| -------------------- | ----------: |
| Input features       |      **29** |
| Hidden layers        | **64 → 32** |
| Output classes       |       **9** |
| Training epochs      |     **100** |
| Centralized Macro F1 |   **0.607** |
| Federated clients    |       **3** |

The centralized model provides a baseline for evaluating the federated learning experiments.

---

### 🤖 Federated Learning

The federated environment uses **three Windows clients** connected to a centralized Flower server.

Each client trains locally on its own dataset and sends model updates to the federated server.

The data distribution is intentionally **non-IID**: each client observes only a subset of the nine attack classes.

This configuration makes the experiment closer to a realistic distributed security environment, but also makes federated convergence more difficult.

---

### 🕵️ Privacy Attack Without Secure Aggregation

The project demonstrates a security weakness of standard Federated Learning when the aggregation server can inspect individual client updates.

An **honest-but-curious server** was implemented to record individual model updates during training.

For each round, the server could access the individual tensors of every client:

```text
fc1.weight
fc1.bias
fc2.weight
fc2.bias
fc3.weight
fc3.bias
```

The experiment generated **30 rounds of individual client updates**.

The recorded updates were then analyzed using:

* L2 norms
* mean / standard deviation
* client distance matrices
* final-layer deviation analysis

A **property inference attack** was also performed.

The attack successfully reconstructed information about the local classes present on individual clients.

This demonstrates that **mTLS protects model updates during transport, but does not prevent a trusted aggregation server from inspecting decrypted updates at its endpoint**.

---

### 🛡️ Secure Aggregation

To mitigate this information leakage, **Secure Aggregation (SecAgg+)** was integrated into the federated training pipeline.

With Secure Aggregation enabled:

```text
Client 1 ──┐
Client 2 ──┼──► Masked Updates ──► Aggregation
Client 3 ──┘
```

The server receives only the protected aggregate rather than the individual client model updates.

The SecAgg implementation was validated with a dedicated unit test and integrated into the live federated training environment.

The replay of previously demonstrated privacy attacks after masking no longer provided the same direct visibility into individual client updates.

---

### 🚨 Real-Time Detection

The federated model was subsequently deployed as a **real-time intrusion detection component** on the Windows clients.

The detector processes Windows security telemetry and evaluates activity against the federated model.

The system was validated against a **real attack launched from Kali Linux**.

The resulting real-time detector achieved an approximate:

> **Macro F1 ≈ 0.50**

This result demonstrates the feasibility of using the federated model for live detection, while also showing that additional data and model optimization are required before considering the detector production-ready.

---

### ⚔️ Active Directory Security Validation

The laboratory was also used to reproduce several Active Directory attack techniques in a controlled environment:

| Attack / Technique | Validation |
| ------------------ | ---------- |
| BloodHound         | ✅          |
| Pass-the-Hash      | ✅          |
| AS-REP Roasting    | ✅          |
| Kerberoasting      | ✅          |
| LLMNR poisoning    | ✅          |
| AD CS ESC8         | ✅          |
| Golden Ticket      | ✅          |

The corresponding defensive measures were then evaluated, including:

* Credential Guard / RunAsPPL
* NTLM restriction
* LLMNR disabling
* SMB signing
* LAPS
* gMSA / AES
* removal of unnecessary SPNs
* AD CS web enrollment hardening
* double `krbtgt` rotation
* DNSSEC
* tiered administration

---

## ⚠️ Interpretation of the Results

The experiments demonstrate three complementary points:

**1. Federated Learning improves data locality**

Raw security telemetry does not need to be centralized for model training.

**2. Federated Learning alone does not guarantee privacy**

Without Secure Aggregation, an aggregation server can inspect individual model updates and potentially infer information about the local training data.

**3. Secure Aggregation reduces this visibility**

SecAgg+ changes what the server can observe by preventing direct inspection of individual client updates.

At the same time, the experiments revealed limitations, particularly the difficulty of training under the strongly non-IID data distribution and the relatively low F1 of the real-time detector.

## 🎯 Project Objectives

The main objectives of this project are to:

- Design and deploy a secure Active Directory infrastructure
- Implement centralized identity and access management
- Deploy an internal PKI using Active Directory Certificate Services
- Configure certificate-based authentication and mutual TLS (mTLS)
- Implement centralized authentication using NPS / RADIUS
- Build a Federated Learning environment with multiple clients
- Protect model updates using Secure Aggregation
- Perform controlled Active Directory security assessments
- Identify common security weaknesses and apply hardening measures
- Monitor and evaluate the security of the infrastructure

## 🛡️ Active Directory Security

The lab includes a structured Active Directory environment designed to practice enterprise identity management and security.

### Infrastructure

- Windows Server Domain Controllers
- Active Directory Domain Services (AD DS)
- Organizational Units (OUs)
- Users and Security Groups
- Group Policy Objects (GPOs)
- Windows DNS
- Kerberos authentication
- LDAP directory services

### Security Assessment

Controlled security assessments are performed in an isolated laboratory environment using tools and techniques such as:

- BloodHound
- Kerberoasting
- AS-REP Roasting
- Pass-the-Hash
- LLMNR / NBT-NS poisoning
- Active Directory Certificate Services (AD CS) security assessment

### Hardening

The project also focuses on improving the security posture of the infrastructure through:

- Group Policy hardening
- Secure authentication
- Privileged access management
- Network segmentation
- Secure service configuration
- Monitoring and security validation

> ⚠️ All security testing is performed in an isolated laboratory environment for educational and research purposes.

## 🔑 PKI, RADIUS & Secure Authentication

The infrastructure integrates several security mechanisms to provide strong and centralized authentication.

### 🔐 Public Key Infrastructure (PKI)

An internal PKI is deployed using **Active Directory Certificate Services (AD CS)**.

The implementation includes:

- Internal Certificate Authority (CA)
- Certificate templates
- Certificate enrollment
- Digital certificates
- Certificate-based authentication
- Certificate lifecycle management

### 🔑 NPS / RADIUS

**Network Policy Server (NPS)** is used to provide centralized authentication and authorization.

The lab explores:

- RADIUS authentication
- Network access policies
- Centralized authentication
- Integration with Active Directory
- Secure authentication mechanisms

### 🔒 Mutual TLS (mTLS)

Mutual TLS is implemented to provide secure communication between trusted services.

The mechanism provides:

- Server authentication
- Client authentication
- Certificate-based identity verification
- Encrypted communication
- Protection against unauthorized clients

The combination of **AD, PKI, RADIUS and mTLS** provides a layered approach to identity and access security.

## 🤖 Federated Learning & Secure Aggregation

The project explores a Federated Learning environment designed to study privacy-preserving machine learning in a distributed infrastructure.

### 🧠 Federated Learning

Instead of sending raw datasets to a central server, participating clients perform local model training and share model updates with the Federated Learning server.

The architecture includes:

- Federated Learning server
- Multiple participating clients
- Local model training
- Model update exchange
- Centralized model aggregation

### 🔒 Secure Aggregation

Secure Aggregation is used to protect individual client updates during the aggregation process.

The objective is to allow the server to obtain the combined model update without directly accessing each client's individual contribution.

This approach helps improve:

- Data privacy
- Confidentiality of client updates
- Protection against unauthorized access
- Security of distributed machine learning

### 🧪 Security Research

The Federated Learning environment is also used as a research laboratory to study potential security threats against distributed learning systems.

Security experiments are conducted in an isolated environment for educational and research purposes.

## 🛠️ Technologies & Tools

### 🌐 Networking

- Cisco IOS
- VLANs
- Inter-VLAN Routing
- Routing Protocols
- Network Segmentation

### 🔐 Identity & Security

- Active Directory
- Kerberos
- LDAP
- Group Policy
- AD CS / PKI
- NPS / RADIUS
- mTLS
- BloodHound
- Kali Linux

### 🖥️ Systems

- Windows Server
- Linux
- DNS
- Windows Administration

### 🤖 Federated Learning

- Federated Learning
- Secure Aggregation
- Distributed Machine Learning
- Privacy-Preserving Techniques

### ☁️ Virtualization

- VMware ESXi
- VMware vCenter
- VMware Horizon

### 🧰 Tools

- GNS3
- Wireshark
- Packet Tracer
- Git
- GitHub

## 📁 Project Structure

```text
secure-ad-federated-learning/
│
├── README.md
│
├── docs/
│   ├── architecture/
│   ├── active-directory/
│   ├── pki/
│   ├── radius/
│   ├── federated-learning/
│   └── security-testing/
│
├── diagrams/
│   ├── network-topology/
│   ├── ad-architecture/
│   └── federated-learning/
│
├── scripts/
│   ├── powershell/
│   ├── python/
│   └── linux/
│
├── configurations/
│   ├── active-directory/
│   ├── network/
│   └── security/
│
├── screenshots/
│   ├── active-directory/
│   ├── pki/
│   ├── radius/
│   ├── federated-learning/
│   └── security-testing/
│
└── LICENSE
