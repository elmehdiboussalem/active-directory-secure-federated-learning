# 🔐 Secure Active Directory Infrastructure & Federated Learning

A cybersecurity infrastructure project combining **Active Directory security, PKI, RADIUS, mutual TLS, Federated Learning, Secure Aggregation, and offensive security testing**.

## 🎯 Project Overview

This project focuses on designing and implementing a secure infrastructure environment integrating enterprise identity services with modern cybersecurity and privacy-preserving machine learning techniques.

The lab combines:

- Active Directory
- DNS
- Group Policy
- Active Directory Certificate Services (AD CS)
- NPS / RADIUS
- PKI
- Mutual TLS (mTLS)
- Federated Learning
- Secure Aggregation
- Active Directory security assessment
- Security monitoring and hardening

## 🏗️ Architecture

The infrastructure includes:

```text
                         ┌──────────────────────┐
                         │   Federated Learning │
                         │       Server         │
                         └──────────┬───────────┘
                                    │
                              Secure Aggregation
                                    │
                 ┌──────────────────┴──────────────────┐
                 │                                      │
        ┌────────▼────────┐                    ┌────────▼────────┐
        │   Client / FL   │                    │   Client / FL   │
        │     Node 01     │                    │     Node 02     │
        └────────┬────────┘                    └────────┬────────┘
                 │                                      │
                 └──────────────┬───────────────────────┘
                                 │
                          mTLS / PKI
                                 │
                     ┌───────────▼───────────┐
                     │    Active Directory   │
                     │        Domain         │
                     └───────────┬───────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               │                 │                 │
           DNS Server         AD CS             NPS/RADIUS
```

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
