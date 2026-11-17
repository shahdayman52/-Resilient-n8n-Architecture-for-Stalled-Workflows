# n8n Queue Mode Architecture

An incoming webhook event arrives on the main n8n web node. The node saves the execution in PostgreSQL, pushes the job ID to the Redis queue, and acknowledges the caller right away. Worker instances pull jobs from Redis and process them asynchronously. Payloads that still fail after 3 retries go to the DLQ table in PostgreSQL.

```mermaid
flowchart LR
    Client["External caller<br/>(booking API, OTA, approval link)"]

    subgraph Main["n8n main (web process)"]
        WH["Webhook endpoint"]
    end

    Q[("Redis<br/>Bull queue (AOF)")]

    subgraph Workers["n8n-worker instances (async)"]
        W1["Worker 1"]
        W2["Worker 2"]
        WN["Worker N"]
    end

    subgraph PG["PostgreSQL"]
        EX[("Executions +<br/>Wait state")]
        DLQ[("dlq_failed_executions")]
    end

    Client -->|HTTP POST| WH
    WH -->|save execution| EX
    WH -->|enqueue job ID| Q
    WH -.->|immediate 200 ack| Client
    Q -->|pull job| W1 & W2 & WN
    W1 & W2 & WN -->|read/write execution data| EX
    W1 & W2 & WN -->|"failed after 3 retries"| DLQ
```
