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
                 │                                     │
        ┌────────▼────────┐                   ┌────────▼────────┐
        │   Client / FL   │                   │   Client / FL   │
        │     Node 01     │                   │     Node 02     │
        └────────┬────────┘                   └────────┬────────┘
                 │                                     │
                 └──────────────┬──────────────────────┘
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
