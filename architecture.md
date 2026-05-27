# Architecture — Image Moderation Pipeline (Scenario C)

```mermaid
flowchart TD
    subgraph ingest["① INGESTION — synchronous · ACK immediately"]
        UA[Upload API] -->|store image| OS[(Object Storage)]
        UA -->|enqueue image ref| MQ[[Message Queue]]
    end

    subgraph pipeline["② MODERATION PIPELINE — async · auto-path p95 < 600 ms"]
        MQ -->|image ref| HF[Hash Filter]
        HF <-->|hash lookup| KVD[(Known Violations DB)]
        HF -->|no match| MS[ML Serving]
        MS <-->|fetch active version| MR[(Model Registry)]
        MS -->|confidence score| DR{Decision Router\ntwo-threshold}
    end

    HF -->|hash match · known violation| AR[Auto-Reject]
    DR -->|score ge 0.9 · safe| AA[Auto-Approve]
    DR -->|score le 0.3 · violation| AR
    DR -->|0.3 lt score lt 0.9\nor model down| HRQ[[Human Review Queue]]

    subgraph human["③ HUMAN REVIEW — async · minutes to hours"]
        HRQ --> RUI[Review UI]
    end

    AA & AR & RUI -->|decision + rationale| AL[(Audit Log)]
    AA -->|approved| NS[Notification Service]
    AR -->|removed| NS
    RUI -->|outcome| NS

    subgraph offline["④ OFFLINE — periodic retraining"]
        MON[Monitoring] -->|drift signal| TP[Training Pipeline]
    end

    AL -->|accuracy metrics · escalation rate| MON
    RUI -->|labeled example| TP
    TP -->|register new version| MR
```

> **Note on layout:** Mermaid's auto-layout algorithm struggles with graphs that combine multiple subgraphs and cross-boundary feedback loops. The feedback path from Training Pipeline back to Model Registry (inside the pipeline subgraph) creates a crossing that forces the layout to sprawl. The diagram is architecturally correct - all components, contracts, and serving boundaries are as intended. A tool like draw.io or Excalidraw would allow manual placement for a cleaner visual, at the cost of version control.