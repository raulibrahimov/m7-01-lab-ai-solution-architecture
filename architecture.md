# Architecture — Scenario A: Real-time Fraud Scoring

Serving pattern: **online (synchronous request/response)** on the payment path,
with **streaming** feature updates and **batch** training off the critical path.

```mermaid
flowchart LR
    %% ---------- Online path (in the 80 ms budget) ----------
    subgraph ONLINE["Online — synchronous, p95 ≤ 80 ms"]
        direction LR
        PAY["Payments App<br/>(checkout)"]
        GW["Scoring API / Gateway"]
        FS_ON["Feature Store<br/>(online: low-latency KV)"]
        MODEL["Fraud Model Service<br/>(model loaded from registry)"]
        DEC["Decision Engine<br/>block / allow / step-up"]

        PAY -- "score request<br/>(txn + device fingerprint)" --> GW
        GW -- "lookup keys<br/>(account_id, device_id)" --> FS_ON
        FS_ON -- "account history +<br/>device features" --> GW
        GW -- "feature vector" --> MODEL
        MODEL -- "risk score 0–1" --> GW
        GW -- "score + threshold" --> DEC
        DEC -- "block / allow / step-up" --> PAY
    end

    %% ---------- Streaming / ingestion ----------
    subgraph STREAM["Streaming — feature freshness"]
        direction TB
        BUS["Event Bus<br/>(transaction + auth events)"]
        FEAT["Feature Pipeline<br/>(stream aggregations)"]
    end

    PAY -. "transaction events" .-> BUS
    DEC -. "decision events" .-> BUS
    BUS -- "raw events" --> FEAT
    FEAT -- "write online features" --> FS_ON
    FEAT -- "write offline features" --> FS_OFF["Feature Store<br/>(offline: warehouse)"]

    %% ---------- Offline / batch (off critical path) ----------
    subgraph OFFLINE["Offline — training & governance"]
        direction TB
        LABELS["Label Store<br/>(chargebacks, fraud confirmations)"]
        TRAIN["Training Pipeline<br/>(batch, scheduled + triggered)"]
        REG["Model Registry<br/>(versioned, staged)"]
    end

    FS_OFF -- "training features<br/>(point-in-time correct)" --> TRAIN
    LABELS -- "labels" --> TRAIN
    TRAIN -- "candidate model" --> REG
    REG -- "promoted model<br/>(staged rollout)" --> MODEL

    %% ---------- Monitoring & feedback loop ----------
    subgraph MON["Monitoring"]
        MONIT["Monitoring<br/>(latency, drift, score dist.,<br/>block rate, false-positive rate)"]
    end

    MODEL -. "predictions + latency" .-> MONIT
    DEC -. "decision outcomes" .-> MONIT
    LABELS -. "delayed ground truth" .-> MONIT
    MONIT == "drift / degradation signal<br/>(triggers retrain)" ==> TRAIN
    MONIT == "alert<br/>(latency SLO breach)" ==> GW
```

## Reading the diagram

- **Serving boundary.** Everything in the `ONLINE` box runs inside the 80 ms budget
  on the synchronous payment path. `STREAM` keeps the online feature store fresh but
  is not blocking. `OFFLINE` (training, registry, labels) runs entirely off the
  critical path.
- **Contracts on the arrows.** Solid arrows = synchronous request/response in the
  latency budget. Dotted arrows = asynchronous events. Bold arrows = control signals
  (retrain trigger, SLO alert).
- **Two-tier feature store.** The same features are materialized twice: an online
  low-latency KV store for serving and an offline warehouse copy for point-in-time
  correct training. The pipeline is the single writer to both, which keeps
  train/serve skew controlled.
- **Feedback loop.** Monitoring watches live predictions and decisions, joins them
  against delayed ground truth from the label store (chargebacks land days later),
  and emits a drift/degradation signal that triggers the training pipeline. New
  models flow back to serving only through the registry via a staged rollout.

## Component inventory

| Component | Plane | Responsibility |
|---|---|---|
| Payments App | online | Issues score request at checkout, applies the decision |
| Scoring API / Gateway | online | Orchestrates feature lookup + inference, enforces timeout/fallback |
| Feature Store (online) | online | Low-latency KV reads of account history + device features |
| Fraud Model Service | online | Stateless inference, model pulled from registry |
| Decision Engine | online | Maps score → block / allow / step-up via thresholds |
| Event Bus | streaming | Transports transaction, auth, and decision events |
| Feature Pipeline | streaming | Computes aggregations, writes online + offline stores |
| Feature Store (offline) | offline | Warehouse copy for point-in-time-correct training |
| Label Store | offline | Confirmed-fraud and chargeback labels (delayed) |
| Training Pipeline | offline | Scheduled + drift-triggered retraining |
| Model Registry | offline | Versioned models, staged promotion to serving |
| Monitoring | cross-cutting | Latency/SLO, score drift, FP rate; emits retrain + alert signals |
