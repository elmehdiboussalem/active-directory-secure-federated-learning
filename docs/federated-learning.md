# 🤖 Federated Learning

## Objective

The Federated Learning component allows several Windows clients to collaboratively train an intrusion-detection model without centralizing their raw security telemetry.

The implementation uses **Flower 1.13** and a supervised **Multi-Layer Perceptron (MLP)**.

Three Windows clients participate in the training process.

## Dataset

The training data originates from **Mordor / OTRF** security telemetry.

The preprocessing pipeline uses Windows Security and Sysmon events to construct time-window-based features.

The project produced:

| Metric                     |    Value |
| -------------------------- | -------: |
| Parsed security events     | 826,000+ |
| Extracted windows          |      656 |
| Windows after downsampling |      227 |
| MITRE ATT&CK classes       |        9 |
| Standardized features      |       29 |
| Dataset files              | 6 `.npz` |

## Preprocessing Pipeline

```text
Mordor / OTRF
      │
      ▼
Windows Security + Sysmon Events
      │
      ▼
Event Parsing
      │
      ▼
Time-Window Extraction
      │
      ▼
Feature Engineering
      │
      ▼
Standardization
      │
      ▼
Downsampling
      │
      ▼
Federated Dataset
```

## Non-IID Distribution

The dataset is distributed between three clients using a **non-IID configuration**.

Each client observes only a subset of the nine attack classes.

This creates a deliberately challenging federated-learning scenario because the local datasets are not statistically identical.

The report notes that each client sees approximately **3–4 classes out of the 9 available classes**.

## Model Architecture

The MLP used by the project has the following structure:

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

The centralized baseline was trained for **100 epochs**.

The centralized experiment achieved:

```text
Macro F1 = 0.607
```

This baseline provides a reference for evaluating the federated experiments.

## Federated Training

The federated training process follows the standard iterative workflow:

```text
             Global Model
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
     Client 1  Client 2  Client 3
        │         │         │
   Local train Local train Local train
        │         │         │
        └─────────┼─────────┘
                  ▼
          Model Updates
                  │
                  ▼
             Aggregation
                  │
                  ▼
             Global Model
                  │
                  └──────► Next Round
```

At each round, the server provides the current global model to the participating clients.

Each client performs local training on its own data and returns an updated model.

The server aggregates the client contributions to construct the next global model.

## Authentication and Transport Security

Before participating in training, clients are authenticated through the project's security infrastructure.

The communication channel is protected with **mutual TLS (mTLS)**.

The authentication flow combines:

```text
Client
  │
  ├── Certificate authentication
  │
  └── RADIUS pre-authentication
          │
          ▼
      FL Server
```

This prevents an arbitrary machine or account from simply joining the federated-training process.

## Federated Learning and Active Directory

The FL system is directly connected to the Active Directory security environment.

Security telemetry generated on Windows clients is transformed into features locally and used for model training.

This creates the following pipeline:

```text
Active Directory Activity
          │
          ▼
Windows Security / Sysmon
          │
          ▼
Local Feature Extraction
          │
          ▼
Local Federated Training
          │
          ▼
Protected Model Update
          │
          ▼
Global Federated Model
```

## Experimental Validation

Several live training runs were successfully performed with the three Windows clients.

The project also tested the privacy implications of exposing individual model updates to the aggregation server.

This experiment led to the implementation of the Secure Aggregation component described in [`secure-aggregation.md`](secure-aggregation.md).
