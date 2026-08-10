# 🏗️ Infrastructure Architecture

## 🎯 Overview

This project implements a security-focused infrastructure combining enterprise identity management, PKI, centralized authentication, and Federated Learning.

The laboratory environment is designed to simulate a realistic enterprise infrastructure while providing an isolated environment for cybersecurity testing and research.

## 🧩 Main Components

### Active Directory

- Domain Controllers
- Organizational Units (OUs)
- Users and Security Groups
- Group Policy Objects (GPOs)
- DNS
- Kerberos
- LDAP

### PKI

- Active Directory Certificate Services (AD CS)
- Internal Certificate Authority
- Certificate enrollment
- Certificate-based authentication
- Mutual TLS (mTLS)

### Authentication

- NPS
- RADIUS
- Active Directory integration

### Federated Learning

- Federated Learning Server
- Multiple Clients
- Local Model Training
- Model Aggregation
- Secure Aggregation

### Security Testing

- BloodHound
- Kerberoasting
- AS-REP Roasting
- Pass-the-Hash
- LLMNR / NBT-NS security testing
- AD CS security assessment

## 🔐 Security Approach

The infrastructure follows a layered security approach combining:

- Identity security
- Network security
- Certificate-based authentication
- Access control
- Secure communications
- Security assessment
- Infrastructure hardening

> ⚠️ All security testing is performed in an isolated laboratory environment for educational and research purposes.
