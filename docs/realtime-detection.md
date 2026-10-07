# Real-Time Intrusion Detection

## Overview

The final stage of the project deploys the federated model as a real-time Host-based Intrusion Detection System (HIDS) on the three Windows clients.

The objective is to move from offline model training to operational detection by continuously analyzing Windows event logs and classifying recent activity into the attack techniques learned during federated training.

The detector operates directly on the monitored Windows hosts and analyzes:

* Windows Security logs
* Sysmon logs
* PowerShell logs

It is therefore a host-based detector rather than a network IDS. It can complement a NIDS but does not replace network-level monitoring.

---

## Detection Architecture

The real-time detector is implemented in:

```text
realtime_detector.py
```

and deployed on each Windows client under:

```text
C:\PFA
```

The detector loads three main components:

```text
┌───────────────────────────────────────────┐
│             Windows Client                │
│                                           │
│  Security ─────┐                          │
│  Sysmon ───────┼──► Event Collection      │
│  PowerShell ───┘         │                │
│                          ▼                │
│                  30-second window         │
│                          │                │
│                          ▼                │
│                 Feature Extraction        │
│                          │                │
│                          ▼                │
│                    StandardScaler         │
│                          │                │
│                          ▼                │
│                 Federated MLP Model       │
│                          │                │
│                          ▼                │
│              Prediction + Confidence     │
│                          │                │
│                          ▼                │
│              Consecutive-window          │
│                   decision                │
│                          │                │
│                          ▼                │
│                  [WARNING] ALERTE         │
└───────────────────────────────────────────┘
```

The deployed model is the global federated checkpoint saved from an extended training run.

According to the project report, the production checkpoint corresponds to round 61 and contains the nine attack classes.

---

## Model and Metadata

The detector loads:

```text
mlp_federated_best.pt
metadata.pkl
```

The metadata file contains the information required to reproduce the preprocessing performed during training:

* `StandardScaler`
* `LabelEncoder`
* the 29 feature names
* the nine class labels

This guarantees that live events pass through the same preprocessing pipeline as the training data.

The complete inference chain is:

```text
Windows Events
      │
      ▼
Feature Extraction
      │
      ▼
29 Features
      │
      ▼
StandardScaler
      │
      ▼
MLP
      │
      ▼
9-Class Prediction
      │
      ▼
Confidence Threshold
      │
      ▼
Consecutive-Window Decision
      │
      ▼
Alert
```

---

## Sliding-Window Detection

The detector continuously collects events using a sliding window of approximately 30 seconds.

For each window:

1. Events are collected from Security, Sysmon and PowerShell.
2. The same 29 features used during training are extracted.
3. The features are standardized using the saved scaler.
4. The MLP performs inference.
5. The predicted class and confidence are evaluated.
6. The alert logic checks the configured confidence threshold and consecutive-window condition.

An alert is generated when an attack class is predicted with sufficient confidence according to the configured detection policy.

---

## Event Collection

The initial implementation used Windows `EvtSubscribe` to receive events as a stream.

During deployment, this approach repeatedly generated Windows error 87 (`ERROR_INVALID_PARAMETER`). The event collection mechanism was therefore changed to periodic polling using:

```text
EvtQuery
EvtNext
EvtRender
```

with deduplication based on `EventRecordID`.

This change allowed the detector to reliably retrieve events from the three monitored channels.

The report records approximately:

```text
Security    ≈ 27,000 events
Sysmon      ≈ 28,000 events
PowerShell  ≈    311 events
```

during the validation phase.

---

## XML Parsing Correction

A second deployment issue concerned Windows event XML parsing.

The parser initially returned empty dictionaries because the namespace removal logic only handled one quotation style for the `xmlns` attribute.

Windows emitted the namespace using single quotes in the observed events.

The parser was therefore updated to handle both quotation styles, restoring event parsing and feature extraction.

This correction was essential for the transition from the offline dataset to live Windows telemetry.

---

## Deployment

The detector is deployed on all three Windows clients.

The required files are:

```text
C:\PFA\
├── realtime_detector.py
├── mlp_federated_best.pt
└── metadata.pkl
```

The environment uses Python 3.10.

The report also notes a `scikit-learn` `InconsistentVersionWarning` caused by a difference between the version used when serializing the scaler and the runtime version. This warning did not prevent inference in the reported deployment.

---

## Configuration

The detector can be configured through environment variables.

The deployment parameters documented in the project include:

```text
MODEL_PATH
META_PATH
ALERT_THRESHOLD
CONSECUTIVE_WINDOWS
```

Example:

```cmd
cd C:\PFA

set MODEL_PATH=mlp_federated_best.pt
set META_PATH=metadata.pkl
set ALERT_THRESHOLD=0.80
set CONSECUTIVE_WINDOWS=2

python realtime_detector.py
```

The report also documents a live deployment configuration using:

```text
ALERT_THRESHOLD=0.90
CONSECUTIVE_WINDOWS=3
```

Therefore, the exact threshold and consecutive-window values should be treated as deployment parameters rather than fixed properties of the model.

---

## Replay Validation

Before performing the live attack demonstration, the detector was tested in replay mode using a previously captured JSON event sequence.

Command:

```cmd
python realtime_detector.py --replay
```

The replay test successfully classified the attack window as:

```text
pred=apt29
conf=1.00
```

This validated the complete inference pipeline:

```text
Captured Events
      │
      ▼
Feature Extraction
      │
      ▼
Scaling
      │
      ▼
MLP Inference
      │
      ▼
Attack Classification
```

Replay mode provides a reproducible way to validate the detector without requiring a live attack.

---

## Live Validation

The detector was subsequently tested against real activity generated from the Kali Linux machine.

The validation scenario consisted of:

1. Network reconnaissance.
2. SMB-related access attempts.
3. Authentication attempts.
4. Kerberoasting activity.
5. Observation of the resulting Windows telemetry by the HIDS.

The target Windows workstation was hardened, including SMB signing and NTLM restrictions. Consequently, some offensive attempts failed at the access layer.

This did not prevent the experiment from generating observable security events.

The objective was not only to obtain successful compromise, but to verify that the detector could identify suspicious activity from the resulting host telemetry.

---

## Observed Detection

During the live attack, the number of events inside a detection window increased significantly.

The report records a transition from normal windows containing only a few dozen events to windows containing more than:

```text
2,400 events
```

The detector subsequently classified the activity as:

```text
ntds_dump
```

and later:

```text
apt29
```

with:

```text
confidence = 1.00
```

Several alerts were therefore generated:

```text
[WARNING] ALERTE
```

This demonstrates the complete operational chain:

```text
Kali Activity
      │
      ▼
Windows Security / Sysmon / PowerShell
      │
      ▼
Live Event Collection
      │
      ▼
Feature Extraction
      │
      ▼
Federated MLP
      │
      ▼
Attack Prediction
      │
      ▼
Real-Time Alert
```

---

## Operational Performance

The report evaluates several operational characteristics of the deployed detector.

### Inference latency

The MLP inference time was measured at less than:

```text
5 ms per window
```

The 30-second event-collection window is therefore the dominant factor in the detection pipeline rather than neural-network inference.

### Memory footprint

The reported memory footprint is approximately:

```text
120 MB per Windows client
```

including PyTorch, the scaler and the event buffer.

### Alert behavior

During the recorded background activity:

```text
Normal windows:       0 alerts
Offensive scenarios:  alerts generated
```

This demonstrates that the configured alerting logic distinguished the recorded normal background activity from the tested offensive scenarios.

---

## Model Performance

The global model used by the real-time detector achieved approximately:

```text
Macro F1 ≈ 0.50
```

on the reported `server_test.npz` evaluation.

The deployed checkpoint corresponds to an extended federated training run and was saved at round 61.

The model is therefore operationally usable for demonstrating live detection, but its classification performance is not yet sufficient to consider the system production-ready.

---

## Limitations

The main limitation is model performance.

A macro F1 of approximately 0.50 indicates that the detector can identify meaningful attack patterns but can still generate:

* false positives
* false negatives
* incorrect attack-class predictions

The underlying dataset is also relatively small after preprocessing and exhibits strong non-IID characteristics across the federated clients.

Consequently, live detection should be considered a proof of concept and research prototype rather than a fully production-hardened HIDS.

Further improvements would require:

* more Windows telemetry
* more attack scenarios
* additional normal activity
* better class balance
* longer federated training
* systematic threshold tuning
* evaluation across additional hosts

---

## Reproduction

### Start the detector

On a Windows client:

```cmd
cd C:\PFA

set MODEL_PATH=mlp_federated_best.pt
set META_PATH=metadata.pkl
set ALERT_THRESHOLD=0.80
set CONSECUTIVE_WINDOWS=2

python realtime_detector.py
```

### Replay a captured attack

```cmd
python realtime_detector.py --replay
```

### Live validation

Run the detector first, then generate controlled security-testing activity from the Kali machine against the isolated laboratory target.

The purpose of the experiment is to verify that the generated Windows telemetry produces detectable feature patterns and corresponding alerts.

---

## Security Positioning

The real-time detector is the final stage of the project's security pipeline:

```text
Active Directory
      │
      ├── Authentication
      ├── PKI
      ├── RADIUS
      └── Windows Event Logging
                  │
                  ▼
          Federated Learning
                  │
                  ├── Privacy Attack
                  │
                  └── Secure Aggregation
                          │
                          ▼
                 Global Federated Model
                          │
                          ▼
                  Real-Time HIDS
                          │
                          ▼
                    Security Alert
```

The project therefore connects infrastructure security, privacy-preserving federated learning and operational intrusion detection in a single experimental environment.

---

## Conclusion

The real-time detection phase demonstrates that the federated model can be deployed directly on Windows endpoints and used to analyze live security telemetry.

The complete pipeline was validated from event collection to feature extraction, model inference and alert generation.

The live Kali validation produced observable detection events and high-confidence predictions, demonstrating the feasibility of using the federated model as a lightweight HIDS.

At the same time, the reported macro F1 of approximately 0.50 highlights the need for additional data, training and evaluation before deployment in a production environment.
