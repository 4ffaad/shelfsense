# ShelfSense architecture

## 1. Mental model

ShelfSense has three distinct responsibilities:

- **Vision** observes what is visible in an image.
- **Inventory** turns observations into counts and business events.
- **RAG** finds relevant trusted context and helps a person understand the data.

The boundaries matter. A language model must not silently change a detected count, and a detector must not be expected to answer questions about store policy.

## 2. System context

```mermaid
flowchart TB
    Operator["Store operator / learner"]
    Camera["Image upload or camera"]
    UI["Dashboard / chat UI"]
    API["FastAPI application"]
    Vision["Vision pipeline"]
    Inventory["Inventory domain"]
    DB[("SQLite -> PostgreSQL")]
    Knowledge["Knowledge sources"]
    RAG["RAG assistant"]
    Model["Local or hosted LLM"]

    Operator --> UI
    Camera --> API
    UI --> API
    API --> Vision
    Vision --> Inventory
    Inventory --> DB
    Knowledge --> RAG
    DB --> RAG
    UI --> RAG
    RAG --> Model
    Model --> RAG
    RAG --> UI
```

## 3. Request flow: image to observation

```mermaid
sequenceDiagram
    actor User
    participant API as FastAPI
    participant Ingest as Image ingestion
    participant Detector as Detector adapter
    participant ID as SKU identifier
    participant Domain as Inventory service
    participant Store as Repository

    User->>API: POST /v1/observations (image)
    API->>Ingest: Validate bytes, type, dimensions, checksum
    Ingest-->>API: Valid image metadata
    API->>Detector: Detect visible objects
    Detector-->>API: Bounding boxes + confidence
    API->>ID: Identify each crop
    ID-->>API: SKU or unknown + evidence
    API->>Domain: Aggregate counts and apply rules
    Domain->>Store: Persist append-only observation/events
    Store-->>API: Observation identifier
    API-->>User: Counts, unknowns, events, versions
```

## 4. RAG flow: question to grounded answer

```mermaid
sequenceDiagram
    actor User
    participant API as FastAPI
    participant Query as Query service
    participant Retriever as Hybrid retriever
    participant Index as Vector + keyword index
    participant Prompt as Context builder
    participant LLM as Local/hosted LLM

    User->>API: Ask a question
    API->>Query: Validate question and filters
    Query->>Retriever: Search query + shelf/SKU/time filters
    Retriever->>Index: Lexical + semantic retrieval
    Index-->>Retriever: Ranked chunks + metadata
    Retriever-->>Prompt: Evidence within token budget
    Prompt->>LLM: Question + evidence + answer rules
    LLM-->>Prompt: Answer with source identifiers
    Prompt-->>API: Answer, citations, abstention reason
    API-->>User: Grounded response
```

## 5. Internal layers

```mermaid
flowchart LR
    subgraph Transport["Transport layer"]
        Routes["FastAPI routes"]
        Schemas["Pydantic schemas"]
    end
    subgraph Application["Application layer"]
        ObsService["Observation service"]
        QueryService["RAG query service"]
        IngestService["Knowledge ingestion service"]
    end
    subgraph Domain["Domain layer"]
        VisionContract["Detector / identifier protocols"]
        InventoryRules["Counts and inventory events"]
        RetrievalPolicy["Citation and abstention policy"]
    end
    subgraph Adapters["Adapter layer"]
        CV["OpenCV / PyTorch / detector"]
        Embed["Embedding provider"]
        LLMAdapter["LLM provider"]
        Repositories["SQL repositories"]
    end

    Routes --> Schemas --> ObsService
    Routes --> Schemas --> QueryService
    Routes --> Schemas --> IngestService
    ObsService --> VisionContract
    ObsService --> InventoryRules
    QueryService --> RetrievalPolicy
    IngestService --> Embed
    VisionContract --> CV
    QueryService --> Embed
    QueryService --> LLMAdapter
    ObsService --> Repositories
    IngestService --> Repositories
    QueryService --> Repositories
```

## 6. Data model

```mermaid
erDiagram
    PRODUCT ||--o{ PRODUCT_ALIAS : has
    SHELF ||--o{ SHELF_SKU_RULE : allows
    PRODUCT ||--o{ SHELF_SKU_RULE : configured_for
    SHELF ||--o{ SHELF_IMAGE : receives
    SHELF_IMAGE ||--|| SHELF_OBSERVATION : produces
    SHELF_OBSERVATION ||--o{ DETECTION : contains
    DETECTION ||--o| PRODUCT_MATCH : receives
    PRODUCT ||--o{ PRODUCT_MATCH : identifies
    SHELF_OBSERVATION ||--o{ INVENTORY_EVENT : emits
    DOCUMENT ||--o{ DOCUMENT_CHUNK : split_into
    DOCUMENT_CHUNK }o--o{ SHELF_OBSERVATION : cites

    PRODUCT {
        string sku PK
        string name
        string barcode
        boolean active
    }
    SHELF {
        string id PK
        string store_id
        string name
    }
    SHELF_IMAGE {
        string id PK
        string sha256
        int width
        int height
    }
    SHELF_OBSERVATION {
        string id PK
        string image_id FK
        string pipeline_version
        string status
    }
    DETECTION {
        string id PK
        string observation_id FK
        float confidence
        string box_json
    }
    PRODUCT_MATCH {
        string id PK
        string detection_id FK
        string sku FK
        string method
        float confidence
    }
    INVENTORY_EVENT {
        string id PK
        string observation_id FK
        string event_type
        int observed_count
    }
    DOCUMENT {
        string id PK
        string source_uri
        string source_version
    }
    DOCUMENT_CHUNK {
        string id PK
        string document_id FK
        string content_hash
        string embedding_model
    }
```

## 7. Trust boundaries

```mermaid
flowchart LR
    Image["Untrusted image"] --> Validate["Size/type/decode validation"]
    Validate --> Vision["Model inference"]
    Vision --> Observation["Structured observation"]
    Observation --> Rules["Deterministic inventory rules"]
    Rules --> Facts["Trusted application facts"]
    Docs["External documents"] --> Sanitize["Parse, normalize, source metadata"]
    Sanitize --> Index["Search index"]
    Facts --> Index
    Question["User question"] --> Retrieve["Bounded retrieval"]
    Index --> Retrieve
    Retrieve --> Prompt["Evidence-only prompt"]
    Prompt --> Answer["Answer + citations or abstention"]
```

Security rule: retrieved documents are data. They are never treated as instructions that can override application policy, call tools, or modify counts.

## 8. Deployment progression

```mermaid
graph LR
    Local["Local dev\nSQLite + files\nMPS/CPU"] --> Compose["Docker Compose\nPostgreSQL + object store"]
    Compose --> Deploy["Optional deployment\nAPI + worker + DB"]
    Local -. "same interfaces" .-> Compose
    Compose -. "only when needed" .-> Deploy
```

We do not start with cloud infrastructure. The first useful system should run on the M3 with local data and explicit model/provider adapters.
