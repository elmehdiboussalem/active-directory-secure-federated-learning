# 🏗️ Laboratory Architecture

## Overview

The laboratory combines a realistic **Active Directory infrastructure** with a secured **Federated Learning environment**.

The infrastructure was deployed on **VMware ESXi 7.0** and includes redundant domain controllers, an internal PKI, NPS/RADIUS authentication, three Windows federated clients, an Ubuntu FL server and a Kali Linux security-testing machine.

The laboratory domain is `nord.corp`.

## Main Components

| Component    | Role                                                 |
| ------------ | ---------------------------------------------------- |
| DC01         | Primary Domain Controller, AD DS, DNS, AD CS and NPS |
| DC02         | Secondary Domain Controller and AD/DNS replication   |
| FL Server    | Ubuntu server running Flower 1.13                    |
| CLIENT01     | Windows federated client                             |
| CLIENT02     | Windows federated client                             |
| CLIENT03     | Windows federated client                             |
| FL-ROOT-CA   | Internal Certificate Authority                       |
| NPS / RADIUS | Pre-authentication of FL participants                |
| Kali Linux   | Controlled offensive-security testing                |

## Active Directory

The Active Directory environment is based on two domain controllers:

```text
                 ┌──────────────────────┐
                 │       nord.corp      │
                 └──────────┬───────────┘
                            │
                 AD DS / DNS / Kerberos
                            │
              ┌─────────────┴─────────────┐
              │                           │
       ┌──────▼──────┐             ┌──────▼──────┐
       │    DC01     │◄────────────►│    DC02     │
       │ Primary DC  │ Replication  │ Secondary DC│
       └──────┬──────┘             └─────────────┘
              │
       ┌──────┼───────────┐
       │      │           │
      AD CS  NPS/RADIUS  DNS
       │
       ▼
   FL-ROOT-CA
```

DC02 provides redundancy and AD/DNS replication.

The report confirms that replication was validated using `repadmin /replsummary`.

## PKI

The internal PKI is based on **Active Directory Certificate Services (AD CS)**.

The Certificate Authority is named:

```text
FL-ROOT-CA
```

Certificates are used to establish trust between participating machines and support the mutual TLS communication used by the Federated Learning system.

Client certificates are deployed through the Active Directory environment.

## RADIUS Authentication

NPS provides the RADIUS authentication layer used to pre-authenticate federated-learning participants.

The FL server sends a RADIUS `Access-Request` to NPS, which validates the participant before the FL session proceeds.

The project uses:

```text
RADIUS
  │
  └── PAP pre-authentication
          │
          ▼
    FL-Participants
```

Only accounts belonging to the `FL-Participants` group are authorized to participate in training.

## Federated Learning Infrastructure

The Ubuntu FL server runs **Flower 1.13**.

The three Windows clients perform local training and communicate with the FL server using secured application channels.

```text
                       ┌─────────────────────┐
                       │     FL Server       │
                       │       Ubuntu        │
                       │                     │
                       │     Flower 1.13     │
                       │   Model Aggregation │
                       └──────────┬──────────┘
                                  │
                              mTLS / gRPC
                                  │
               ┌──────────────────┼──────────────────┐
               │                  │                  │
        ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
        │  CLIENT01   │    │  CLIENT02   │    │  CLIENT03   │
        │   Windows   │    │   Windows   │    │   Windows   │
        │             │    │             │    │             │
        │ Local data  │    │ Local data  │    │ Local data  │
        │ Local train │    │ Local train │    │ Local train │
        └─────────────┘    └─────────────┘    └─────────────┘
```

## Security Flows

The laboratory contains several independent security flows:

### Federated Learning

```text
Windows Client
      │
      │ mTLS / gRPC
      ▼
 FL Server
```

### RADIUS Pre-Authentication

```text
FL Server
    │
    │ RADIUS Access-Request
    │ UDP 1812
    ▼
NPS / RADIUS
    │
    │ Access-Accept
    ▼
FL Server
```

### Active Directory

```text
Windows Clients
      │
      ├── Kerberos
      ├── NTLM
      ├── SMB
      └── LLMNR
             │
             ▼
        Active Directory
```

### Offensive Security Testing

```text
Kali Linux
     │
     │ Controlled attacks
     ▼
Active Directory / Windows Clients
     │
     ▼
Security Events + Sysmon
     │
     ▼
Federated Detection Pipeline
```

## Security Model

The architecture uses several complementary controls:

* **AD DS** for identity and access management
* **AD CS / PKI** for certificate-based trust
* **NPS / RADIUS** for participant pre-authentication
* **mTLS** for secure FL communication
* **Secure Aggregation** for protecting individual model updates
* **Windows Security Logs and Sysmon** for security telemetry

The architecture therefore combines identity security, transport security and model-update privacy rather than relying on a single security mechanism.
