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
