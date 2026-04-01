# MONAI Deploy Informatics Gateway -- System Architecture & OOP/UML

> **Repository:** [Project-MONAI/monai-deploy-informatics-gateway](https://github.com/Project-MONAI/monai-deploy-informatics-gateway)
> **Tech Stack:** .NET 8 / ASP.NET Core / C# / Entity Framework Core / MongoDB / fo-dicom 5.x
> **License:** Apache 2.0

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
  - [1.1 High-Level Architecture](#11-high-level-architecture)
- [2. System Context](#2-system-context)
- [3. Solution Structure](#3-solution-structure)
- [4. Core Domain Model](#4-core-domain-model)
- [5. Service Architecture](#5-service-architecture)
- [6. Data Flow](#6-data-flow)
- [7. Repository Pattern](#7-repository-pattern)
- [8. Plugin System](#8-plugin-system)
- [9. Configuration Architecture](#9-configuration-architecture)
- [10. REST API Controllers](#10-rest-api-controllers)
- [11. Dependency Injection](#11-dependency-injection)
- [12. Design Patterns Catalog](#12-design-patterns-catalog)
- [13. SOLID Principles Mapping](#13-solid-principles-mapping)
- [14. External Dependencies](#14-external-dependencies)

---

## 1. Executive Summary

The **MONAI Deploy Informatics Gateway** (MIG) is a medical imaging data gateway that bridges clinical systems (PACS, RIS, FHIR servers) with AI inference workflows. It ingests data via four healthcare protocols, batches files into payloads, uploads them to object storage, and publishes workflow events to a message broker for downstream consumption.

| Metric | Value |
|--------|-------|
| Projects in solution | 14 |
| Hosted services (background workers) | 11 |
| Ingestion protocols | DICOM DIMSE (SCP), DICOMweb (STOW-RS), HL7/MLLP, FHIR |
| Export protocols | DICOM DIMSE (SCU), DICOMweb, HL7/MLLP, External App |
| Database backends | SQLite (EF Core), MongoDB |
| Object storage | MinIO (S3-compatible) |
| Message broker | RabbitMQ |

**Architecture style:** Modular monolith with hexagonal (ports & adapters) influence. All external systems are accessed through abstraction interfaces, and the core domain has no direct dependency on infrastructure.

### 1.1 High-Level Architecture

```mermaid
flowchart LR
    subgraph ingest["INGEST"]
        direction TB
        SCP["DICOM SCP\n(DIMSE C-STORE)"]
        STOW["DICOMweb\n(STOW-RS)"]
        MLLP["HL7 MLLP\n(v2 Messages)"]
        FHIR["FHIR\n(Resources)"]
    end

    subgraph process["PROCESS"]
        direction TB
        PLUGIN_IN["Input Plugin\nEngine"]
        UPLOAD["Object Upload\nService"]
        PA["Payload\nAssembler"]
        NOTIFY["Payload\nNotification"]
    end

    subgraph export["EXPORT"]
        direction TB
        SCU["DICOM SCU\n(C-STORE)"]
        DW_EXP["DICOMweb\n(STOW-RS)"]
        HL7_EXP["HL7 MLLP\n(Send)"]
        EXT_EXP["External App\n(SCU)"]
    end

    subgraph infra["INFRASTRUCTURE"]
        direction LR
        MINIO["MinIO\n(Object Storage)"]
        RABBIT["RabbitMQ\n(Message Broker)"]
        DB["SQLite / MongoDB\n(Persistence)"]
    end

    API["REST API\n(Configuration\n& Health)"]

    SCP & STOW & MLLP & FHIR -->|"Files +\nMetadata"| PLUGIN_IN
    PLUGIN_IN -->|Transformed| UPLOAD
    UPLOAD -->|Store files| MINIO
    UPLOAD --> PA
    PA -->|"Timeout-based\nbatching"| NOTIFY
    NOTIFY -->|"Publish\nWorkflowRequestEvent"| RABBIT
    RABBIT -->|"Subscribe\nExportRequestEvent"| SCU & DW_EXP & HL7_EXP & EXT_EXP

    PA -.->|Persist state| DB
    API -.->|CRUD config| DB

    style ingest fill:#e3f2fd,stroke:#1565c0
    style process fill:#fff3e0,stroke:#ef6c00
    style export fill:#e8f5e9,stroke:#2e7d32
    style infra fill:#f3e5f5,stroke:#7b1fa2
```

The gateway operates as a **three-phase pipeline**: Ingest, Process, and Export.

**Ingest** -- Four protocol-specific services (DICOM SCP, DICOMweb STOW-RS, HL7 MLLP, FHIR) accept data from clinical systems. Each service creates a `FileStorageMetadata` record for every file received. Despite the different protocols, all ingestion paths converge into the same processing pipeline -- this is the key architectural insight. A CT scanner sending via DIMSE and a web client posting via DICOMweb both end up in the same `PayloadAssembler`.

**Process** -- The Input Plugin Engine runs per-AE-Title plugin chains (e.g., anonymization, metadata enrichment) on each file. The `ObjectUploadService` uploads files to MinIO. The `PayloadAssembler` groups files by a correlation key (typically DICOM Study Instance UID) and uses timeout-based batching -- it waits for a configurable quiet period (default 5 seconds) before transitioning the Payload through its state machine (`Created` → `Move` → `Notify`). The `PayloadNotificationService` publishes a `WorkflowRequestEvent` to RabbitMQ, making the data available to downstream systems like the MONAI Workflow Manager.

**Export** -- Export services subscribe to `ExportRequestEvent` messages from RabbitMQ. Each export type (DICOM SCU, DICOMweb, HL7, External App) extends `ExportServiceBase`, which provides a 4-stage TPL Dataflow pipeline: Download → Output Plugins → Export → Report. This decoupled, event-driven design means the export path operates independently of ingestion -- they communicate only through the message broker.

**Infrastructure** cuts across all phases: MinIO stores file content, RabbitMQ decouples ingestion from export, and the database (SQLite or MongoDB, selected at deployment) persists configuration entities and payload state. The REST API provides CRUD operations for AE Titles, destinations, and health monitoring.

---

## 2. System Context

```mermaid
flowchart TB
    subgraph sources["Ingestion Sources"]
        MOD["DICOM Modalities\n(CT, MR, US)"]
        PACS["PACS / VNA"]
        RIS["RIS / HIS\n(HL7 v2)"]
        FHIR_SRC["FHIR Server"]
    end

    subgraph MIG["MONAI Deploy Informatics Gateway"]
        SCP["DICOM SCP\nPort 104"]
        STOW["DICOMweb STOW-RS\nHTTP API"]
        MLLP["HL7 MLLP\nPort 2575"]
        FHIR_EP["FHIR Endpoint\nHTTP API"]
        CORE["Core Processing\n& Payload Assembly"]
        EXP["Export Services"]
        API["REST Config API\nPort 5000"]
    end

    subgraph infra["Infrastructure"]
        MINIO["MinIO\n(Object Storage)"]
        RABBIT["RabbitMQ\n(Message Broker)"]
        DB["SQLite / MongoDB"]
    end

    subgraph downstream["Downstream"]
        WM["Workflow Manager"]
        DEST_PACS["Destination PACS"]
        DEST_HL7["HL7 Destination"]
    end

    subgraph clients["Management"]
        CLI["CLI Tool"]
        CLIENT["Client Library"]
    end

    MOD -->|C-STORE| SCP
    PACS -->|C-STORE / DICOMweb| SCP & STOW
    RIS -->|HL7 v2| MLLP
    FHIR_SRC -->|FHIR Resources| FHIR_EP

    SCP & STOW & MLLP & FHIR_EP --> CORE
    CORE -->|Upload files| MINIO
    CORE -->|Persist state| DB
    CORE -->|Publish events| RABBIT
    RABBIT -->|Export requests| EXP
    EXP -->|C-STORE| DEST_PACS
    EXP -->|HL7 v2| DEST_HL7

    CLI & CLIENT -->|HTTP| API
    WM -->|Subscribe| RABBIT
```

---

## 3. Solution Structure

### 3.1 Project Dependency Map

```mermaid
flowchart LR
    subgraph presentation["Presentation Layer"]
        IG["InformaticsGateway\n(ASP.NET Host)"]
        CLI_P["CLI"]
        CLIENT_P["Client"]
        CLIENT_COMMON["Client.Common"]
    end

    subgraph domain["Domain Layer"]
        API_P["Api\n(Models, Interfaces,\nPlugin Contracts)"]
    end

    subgraph infrastructure["Infrastructure Layer"]
        DB_API["Database.Api\n(Repository Interfaces)"]
        DB_EF["Database.\nEntityFramework"]
        DB_MONGO["Database.\nMongoDB"]
        CONFIG["Configuration"]
        COMMON["Common\n(Utilities)"]
        DWC["DicomWebClient"]
    end

    subgraph plugins["Plugins"]
        RAE["RemoteApp\nExecution"]
    end

    IG --> API_P & CONFIG & COMMON & DB_API & DWC
    CLI_P --> CLIENT_P
    CLIENT_P --> CLIENT_COMMON & API_P
    DB_EF --> DB_API & API_P
    DB_MONGO --> DB_API & API_P
    RAE --> API_P & DB_API
    DWC --> API_P
```

### 3.2 Dependency Inversion at Database Boundary

A key architectural decision: the host project (`InformaticsGateway`) and domain (`Api`) never reference a concrete database implementation. The `Database.Api` project defines repository interfaces, and the actual ORM is selected at runtime via configuration.

```mermaid
flowchart TB
    HOST["InformaticsGateway\n(Composition Root)"]
    DB_API["Database.Api\n(IPayloadRepository,\nIMonaiApplicationEntityRepository, ...)"]
    DB_EF["Database.EntityFramework\n(EF Core + SQLite)"]
    DB_MONGO["Database.MongoDB\n(MongoDB Driver)"]

    HOST -->|depends on| DB_API
    DB_EF -.->|implements| DB_API
    DB_MONGO -.->|implements| DB_API
    HOST -.->|runtime selection\nvia ConfigureDatabase()| DB_EF & DB_MONGO

    style DB_API fill:#e1f5fe,stroke:#0288d1
    style DB_EF fill:#fff3e0,stroke:#ef6c00
    style DB_MONGO fill:#fff3e0,stroke:#ef6c00
```

---

## 4. Core Domain Model

### 4.1 Application Entity Hierarchy

These entities represent the DICOM and virtual application entities that the gateway manages. Note that `MonaiApplicationEntity` and `VirtualApplicationEntity` extend `MongoDBEntityBase` directly (not `BaseApplicationEntity`), because they are MONAI-specific concepts rather than standard DICOM AEs.

```mermaid
classDiagram
    class MongoDBEntityBase {
        <<abstract>>
    }

    class BaseApplicationEntity {
        +string Name
        +string HostIp
        +string? CreatedBy
        +string? UpdatedBy
        +DateTime? DateTimeUpdated
        +SetDefaultValues()
        +SetAuthor(ClaimsPrincipal, EditMode)
    }

    class SourceApplicationEntity {
        +string AeTitle
        +SetDefaultValues()
    }

    class DestinationApplicationEntity {
        +string AeTitle
        +int Port
        +SetDefaultValues()
    }

    class MonaiApplicationEntity {
        +string Name
        +string AeTitle
        +string Grouping = "0020,000D"
        +List~string~ Workflows
        +List~string~ PlugInAssemblies
        +List~string~ IgnoredSopClasses
        +List~string~ AllowedSopClasses
        +uint Timeout = 5
        +string? CreatedBy
        +string? UpdatedBy
    }

    class VirtualApplicationEntity {
        +string Name
        +string VirtualAeTitle
        +List~string~ Workflows
        +List~string~ PlugInAssemblies
        +string? CreatedBy
        +string? UpdatedBy
    }

    MongoDBEntityBase <|-- BaseApplicationEntity
    BaseApplicationEntity <|-- SourceApplicationEntity
    BaseApplicationEntity <|-- DestinationApplicationEntity
    MongoDBEntityBase <|-- MonaiApplicationEntity
    MongoDBEntityBase <|-- VirtualApplicationEntity
```

**Key distinction:**
- `SourceApplicationEntity` / `DestinationApplicationEntity` -- standard DICOM AE concepts (remote systems the gateway talks to)
- `MonaiApplicationEntity` -- the gateway's own SCP AE title, with workflow mapping and plugin configuration
- `VirtualApplicationEntity` -- DICOMweb-only virtual AE for STOW-RS endpoints

### 4.2 File Storage Metadata Hierarchy

`FileStorageMetadata` is an abstract record (value-type semantics) that tracks each ingested file. Subtypes add protocol-specific fields.

```mermaid
classDiagram
    class FileStorageMetadata {
        <<abstract record>>
        +string Id
        +string CorrelationId
        +DataOrigin DataOrigin
        +List~string~ Workflows
        +DateTime DateReceived
        +string? WorkflowInstanceId
        +string? TaskId
        +string? PayloadId
        +abstract string DataTypeDirectoryName
        +abstract StorageObjectMetadata File
        +bool IsUploaded
        +bool IsUploadFailed
        +bool IsMoveCompleted
        +SetWorkflows(string[])
        +SetFailed()
    }

    class StorageObjectMetadata {
        +string TemporaryPath
        +string UploadPath
        +string ContentType
        +bool IsUploaded
        +bool IsUploadFailed
        +bool IsMoveCompleted
        +SetFailed()
    }

    class DicomFileStorageMetadata {
        +string StudyInstanceUid
        +string SeriesInstanceUid
        +string SopInstanceUid
        +StorageObjectMetadata File
        +StorageObjectMetadata JsonFile
        +string DataTypeDirectoryName = "dcm"
    }

    class Hl7FileStorageMetadata {
        +StorageObjectMetadata File
        +string DataTypeDirectoryName = "hl7"
    }

    class FhirFileStorageMetadata {
        +StorageObjectMetadata File
        +string DataTypeDirectoryName = "fhir"
    }

    FileStorageMetadata <|-- DicomFileStorageMetadata
    FileStorageMetadata <|-- Hl7FileStorageMetadata
    FileStorageMetadata <|-- FhirFileStorageMetadata
    FileStorageMetadata --> StorageObjectMetadata : File
    DicomFileStorageMetadata --> StorageObjectMetadata : JsonFile
```

### 4.3 Payload -- State Machine

The `Payload` is the central batching unit. All ingestion protocols converge here. Files are grouped by a correlation key (e.g., DICOM Study UID), and a timeout triggers state transitions.

```mermaid
classDiagram
    class Payload {
        +Guid PayloadId
        +string Key
        +string CorrelationId
        +string? WorkflowInstanceId
        +string? TaskId
        +uint Timeout
        +int RetryCount
        +PayloadState State
        +List~FileStorageMetadata~ Files
        +DataOrigin DataTrigger
        +HashSet~DataOrigin~ DataOrigins
        +int Count
        +bool HasTimedOut
        +int FilesUploaded
        +int FilesFailedToUpload
        +Add(FileStorageMetadata)
        +ElapsedTime() TimeSpan
        +ResetRetry()
        +Dispose()
    }

    class PayloadState {
        <<enumeration>>
        Created
        Move
        Notify
    }

    Payload --> PayloadState
    Payload --> "0..*" FileStorageMetadata : Files
```

**State Transitions:**

```mermaid
stateDiagram-v2
    [*] --> Created : PayloadAssembler creates\nnew payload for key

    Created --> Move : Timeout expires AND\nall files uploaded

    Move --> Notify : Files moved to\npayload directory

    Move --> Move : Retry on failure\n(max 3 retries)

    Notify --> [*] : Published to\nmessage broker

    Notify --> Notify : Retry on failure\n(max 3 retries)

    note right of Created
        Files are being received
        and added to the payload.
        Stopwatch resets on each Add().
    end note

    note right of Move
        MAX_RETRY = 3
        RetryCount tracks attempts
    end note
```

### 4.4 Export Domain Model

```mermaid
classDiagram
    class ExportRequestDataMessage {
        +byte[] FileContent
        +bool IsFailed
        +List~string~ Messages
        +FileExportStatus ExportStatus
        +string Filename
        +List~string~ PlugInAssemblies
        +string ExportTaskId
        +string WorkflowInstanceId
        +string CorrelationId
        +string FilePayloadId
        +List~string~ Destinations
        +SetFailed(FileExportStatus, string)
    }

    class ExportRequestEvent {
        <<from Monai.Deploy.Messaging>>
        +string ExportTaskId
        +string WorkflowInstanceId
        +string[] Destinations
        +string[] Files
        +List~string~ PlugInAssemblies
    }

    ExportRequestEvent <|-- ExportRequestDataMessage : extends
```

---

## 5. Service Architecture

### 5.1 Hosted Services Topology

All background services implement `IHostedService` and `IMonaiService` (which adds `ServiceStatus` and `ServiceName`). They are registered in `Program.cs` (lines 157-167).

```mermaid
flowchart TB
    subgraph ingestion["Ingestion Services"]
        SCP_SVC["ScpService\n(DICOM SCP)"]
        EXT_SCP["ExternalAppScpService\n(External App SCP)"]
        MLLP_HOST["MllpServiceHost\n(HL7 MLLP Listener)"]
    end

    subgraph http_ingestion["HTTP Ingestion (not hosted)"]
        STOW_CTRL["StowController\n→ StowService"]
        FHIR_CTRL["FhirController\n→ FhirService"]
    end

    subgraph processing["Processing Services"]
        UPLOAD["ObjectUploadService\n(File → MinIO)"]
        RETRIEVAL["DataRetrievalService\n(Inference Requests)"]
        NOTIFY["PayloadNotificationService\n(State Machine Driver)"]
    end

    subgraph export["Export Services"]
        SCU_EXP["ScuExportService\n(DICOM C-STORE)"]
        EXT_EXP["ExtAppScuExportService\n(External App)"]
        DW_EXP["DicomWebExportService\n(STOW-RS)"]
        HL7_EXP["Hl7ExportService\n(HL7 MLLP)"]
    end

    subgraph utility["Utility Services"]
        SCU_SVC["ScuService\n(On-demand DICOM SCU)"]
    end

    subgraph singletons["Shared Singletons"]
        PA["PayloadAssembler"]
        OUQ["ObjectUploadQueue"]
        SCU_Q["ScuQueue"]
        SIP["StorageInfoProvider"]
        AEM["ApplicationEntityManager"]
    end

    SCP_SVC & EXT_SCP --> AEM
    SCP_SVC & EXT_SCP & MLLP_HOST & STOW_CTRL & FHIR_CTRL --> OUQ & PA
    OUQ --> UPLOAD
    PA --> NOTIFY
    SCU_Q --> SCU_SVC

    UPLOAD -->|Upload to MinIO| UPLOAD
    NOTIFY -->|Publish to RabbitMQ| NOTIFY
    SCU_EXP & EXT_EXP & DW_EXP & HL7_EXP -->|Subscribe from RabbitMQ| export
```

### 5.2 SCP Service Inheritance (Template Method)

```mermaid
classDiagram
    class IHostedService {
        <<interface>>
        +StartAsync(CancellationToken)
        +StopAsync(CancellationToken)
    }

    class IMonaiService {
        <<interface>>
        +ServiceStatus Status
        +string ServiceName
    }

    class ScpServiceBase {
        <<abstract>>
        #ITcpListenerFactory TcpListenerFactory
        #IApplicationEntityManager AeManager
        +StartAsync(CancellationToken)
        +StopAsync(CancellationToken)
    }

    class ScpService {
        +string ServiceName = "SCP Service"
    }

    class ExternalAppScpService {
        +string ServiceName = "External App SCP"
    }

    class ApplicationEntityManager {
        +HandleCStoreRequest(DicomCStoreRequest, ...)
        +IsAeTitleConfiguredAsync(string)
        +IsValidSourceAsync(string, string)
    }

    IHostedService <|.. ScpServiceBase
    IMonaiService <|.. ScpServiceBase
    ScpServiceBase <|-- ScpService
    ScpServiceBase <|-- ExternalAppScpService
    ScpServiceBase --> ApplicationEntityManager : uses
```

### 5.3 Export Service Inheritance (Template Method + TPL Dataflow)

```mermaid
classDiagram
    class ExportServiceBase {
        <<abstract>>
        #IMessageBrokerSubscriberService MessageSubscriber
        #IMessageBrokerPublisherService MessagePublisher
        #IServiceScopeFactory ServiceScopeFactory
        +abstract string RoutingKey
        +abstract string ServiceName
        #abstract ushort Concurrency
        #abstract ExportDataBlockCallback(ExportRequestDataMessage, CancellationToken)*
        #abstract ProcessMessage(MessageReceivedEventArgs)*
        #SetupActionBlocks() (TransformManyBlock, ActionBlock)
        +StartAsync(CancellationToken)
        +StopAsync(CancellationToken)
    }

    class ScuExportService {
        +string RoutingKey = "export.request.monaiscu"
        +string ServiceName = "SCU Export"
        #ExportDataBlockCallback()
    }

    class DicomWebExportService {
        +string RoutingKey = "export.request.monaidicomweb"
        +string ServiceName = "DICOMweb Export"
        #ExportDataBlockCallback()
    }

    class ExtAppScuExportService {
        +string RoutingKey = "export.request.external"
        +string ServiceName = "Ext App SCU Export"
        #ExportDataBlockCallback()
    }

    class Hl7ExportService {
        +string RoutingKey = "export.request.hl7"
        +string ServiceName = "HL7 Export"
        #ExportDataBlockCallback()
    }

    ExportServiceBase <|-- ScuExportService
    ExportServiceBase <|-- DicomWebExportService
    ExportServiceBase <|-- ExtAppScuExportService
    ExportServiceBase <|-- Hl7ExportService
```

**TPL Dataflow Pipeline** (inside `SetupActionBlocks()`):

```mermaid
flowchart LR
    A["TransformManyBlock\nDownloadPayload\n(fetch files from MinIO)"]
    B["TransformBlock\nOutputDataEngine\n(execute output plugins)"]
    C["TransformBlock\nExportAction\n(protocol-specific export)"]
    D["ActionBlock\nReporting\n(aggregate status,\npublish completion)"]

    A -->|ExportRequestDataMessage| B
    B -->|ExportRequestDataMessage| C
    C -->|ExportRequestDataMessage| D

    style A fill:#e3f2fd
    style B fill:#fff3e0
    style C fill:#e8f5e9
    style D fill:#fce4ec
```

Each block runs with `MaxDegreeOfParallelism = Concurrency`. If a file fails at any stage, `IsFailed` is set and subsequent blocks skip processing (pass-through).

---

## 6. Data Flow

### 6.1 DICOM SCP Ingestion

```mermaid
sequenceDiagram
    participant Modality
    participant ScpService
    participant AEManager as ApplicationEntityManager
    participant PluginEngine as InputDataPlugInEngine
    participant UploadQueue as ObjectUploadQueue
    participant UploadSvc as ObjectUploadService
    participant MinIO
    participant PayloadAsm as PayloadAssembler
    participant NotifySvc as PayloadNotificationService
    participant RabbitMQ

    Modality->>ScpService: DICOM Association (C-STORE)
    ScpService->>AEManager: HandleCStoreRequest(request, calledAE, callingAE)
    AEManager->>AEManager: Validate AE Title (MonaiApplicationEntity lookup)
    AEManager->>AEManager: Validate Source (SourceApplicationEntity lookup)
    AEManager->>AEManager: Check SOP Class (Allowed/Ignored)

    AEManager->>PluginEngine: Configure(plugInAssemblies)
    AEManager->>PluginEngine: ExecutePlugInsAsync(dicomFile, metadata)
    PluginEngine-->>AEManager: (transformedFile, transformedMetadata)

    AEManager->>UploadQueue: Queue(fileStorageMetadata)
    UploadQueue-->>UploadSvc: Dequeue
    UploadSvc->>MinIO: PutObjectAsync(file)
    UploadSvc->>UploadSvc: Update IsUploaded status

    AEManager->>PayloadAsm: Queue(bucket, file, DataOrigin.DIMSE, timeout)
    PayloadAsm->>PayloadAsm: Create/update Payload, reset Stopwatch

    Note over PayloadAsm: Timeout expires (default 5s)

    PayloadAsm-->>NotifySvc: Dequeue completed Payload
    NotifySvc->>NotifySvc: State: Created → Move → Notify
    NotifySvc->>RabbitMQ: Publish WorkflowRequestEvent
```

### 6.2 DICOMweb STOW-RS Ingestion

```mermaid
sequenceDiagram
    participant Client as HTTP Client
    participant StowCtrl as StowController
    participant StowSvc as StowService
    participant Reader as StowRequestReader
    participant Writer as StreamsWriter
    participant UploadQueue as ObjectUploadQueue
    participant PayloadAsm as PayloadAssembler

    Client->>StowCtrl: POST /dicomweb/studies/{studyUid}
    StowCtrl->>StowSvc: StoreAsync(request, studyUid, aet, workflowName)

    StowSvc->>Reader: Read(request) [multipart or single]
    Reader-->>StowSvc: IAsyncEnumerable<DicomFile>

    loop Each DICOM file
        StowSvc->>Writer: Save(dicomFile, correlationId)
        Writer->>UploadQueue: Queue(fileStorageMetadata)
        Writer->>PayloadAsm: Queue(bucket, file, DataOrigin.DICOMWeb, timeout)
    end

    StowSvc-->>StowCtrl: StowResult (success/failure per instance)
    StowCtrl-->>Client: HTTP 200 with DICOM response
```

### 6.3 Export Flow (SCU Example)

```mermaid
sequenceDiagram
    participant RabbitMQ
    participant ExportSvc as ScuExportService
    participant MinIO
    participant PluginEngine as OutputDataPlugInEngine
    participant DicomClient as DicomClient (fo-dicom)
    participant DestPACS as Destination PACS

    RabbitMQ->>ExportSvc: ExportRequestEvent (routing key)
    ExportSvc->>ExportSvc: ProcessMessage → SetupActionBlocks

    Note over ExportSvc: Stage 1: DownloadPayload
    ExportSvc->>MinIO: GetObjectAsync(file)
    MinIO-->>ExportSvc: byte[] FileContent

    Note over ExportSvc: Stage 2: OutputDataEngine
    ExportSvc->>PluginEngine: ExecutePlugInsAsync(exportDataMessage)
    PluginEngine-->>ExportSvc: Transformed message

    Note over ExportSvc: Stage 3: ExportAction
    ExportSvc->>DicomClient: AddRequestAsync(DicomCStoreRequest)
    DicomClient->>DestPACS: C-STORE
    DestPACS-->>DicomClient: C-STORE Response

    Note over ExportSvc: Stage 4: Reporting
    ExportSvc->>RabbitMQ: Publish ExportCompleteEvent
    ExportSvc->>RabbitMQ: Acknowledge original message
```

---

## 7. Repository Pattern

### 7.1 Dual ORM Implementation

Each repository interface has two implementations. Only 3 representative interfaces are shown below; the full system has **12 repository interfaces**.

```mermaid
classDiagram
    class IPayloadRepository {
        <<interface>>
        +ToListAsync() List~Payload~
        +AddAsync(Payload) Payload
        +UpdateAsync(Payload) Payload
        +RemoveAsync(Payload) Payload
        +GetPayloadsInStateAsync(PayloadState[]) List~Payload~
        +RemovePendingPayloadsAsync()
    }

    class IMonaiApplicationEntityRepository {
        <<interface>>
        +ToListAsync() List~MonaiApplicationEntity~
        +FindByNameAsync(string) MonaiApplicationEntity
        +AddAsync(MonaiApplicationEntity)
        +UpdateAsync(MonaiApplicationEntity)
        +RemoveAsync(MonaiApplicationEntity)
        +ContainsAsync(Expression)
    }

    class IStorageMetadataRepository {
        <<interface>>
        +AddOrUpdateAsync(FileStorageMetadata)
        +GetFileStorageMetdataAsync(string)
        +DeleteAsync(string)
    }

    class EfPayloadRepository {
        -InformaticsGatewayContext _context
    }

    class MongoPayloadRepository {
        -IMongoCollection _collection
    }

    class EfMonaiAeRepository
    class MongoMonaiAeRepository

    IPayloadRepository <|.. EfPayloadRepository : EF Core
    IPayloadRepository <|.. MongoPayloadRepository : MongoDB
    IMonaiApplicationEntityRepository <|.. EfMonaiAeRepository : EF Core
    IMonaiApplicationEntityRepository <|.. MongoMonaiAeRepository : MongoDB
```

### 7.2 Database Provider Selection

```mermaid
flowchart TB
    CONFIG["appsettings.json\nConnectionStrings:Type"]
    CONFIG -->|"sqlite"| EF["Register EF Core Repositories\n(InformaticsGatewayContext)"]
    CONFIG -->|"mongodb"| MONGO["Register MongoDB Repositories\n(MongoClient)"]
    EF --> DI["DI Container\n(IPayloadRepository, etc.)"]
    MONGO --> DI

    style CONFIG fill:#e8eaf6
    style DI fill:#e1f5fe
```

**All 12 repository interfaces:**

| Interface | Entity |
|-----------|--------|
| `IPayloadRepository` | Payload |
| `IMonaiApplicationEntityRepository` | MonaiApplicationEntity |
| `ISourceApplicationEntityRepository` | SourceApplicationEntity |
| `IDestinationApplicationEntityRepository` | DestinationApplicationEntity |
| `IVirtualApplicationEntityRepository` | VirtualApplicationEntity |
| `IInferenceRequestRepository` | InferenceRequest |
| `IStorageMetadataRepository` | FileStorageMetadata |
| `IDicomAssociationInfoRepository` | DicomAssociationInfo |
| `IHL7ApplicationConfigRepository` | Hl7ApplicationConfigEntity |
| `IHL7DestinationEntityRepository` | HL7DestinationEntity |
| `IExternalAppDetailsRepository` | ExternalAppDetails |
| `IRemoteAppExecutionRepository` | (Plugin-specific) |

---

## 8. Plugin System

Plugins provide a **Chain of Responsibility** pattern for data transformation. Each plugin receives the output of the previous one, enabling composable processing pipelines.

```mermaid
classDiagram
    class IInputDataPlugIn {
        <<interface>>
        +string Name
        +ExecuteAsync(DicomFile, FileStorageMetadata) (DicomFile, FileStorageMetadata)
    }

    class IOutputDataPlugIn {
        <<interface>>
        +string Name
        +ExecuteAsync(DicomFile, ExportRequestDataMessage) (DicomFile, ExportRequestDataMessage)
    }

    class IInputHL7DataPlugIn {
        <<interface>>
        +string Name
    }

    class IInputDataPlugInEngine {
        <<interface>>
        +Configure(IReadOnlyList~string~ pluginAssemblies)
        +ExecutePlugInsAsync(DicomFile, FileStorageMetadata) Tuple
    }

    class IOutputDataPlugInEngine {
        <<interface>>
        +Configure(IReadOnlyList~string~ pluginAssemblies)
        +ExecutePlugInsAsync(ExportRequestDataMessage) ExportRequestDataMessage
    }

    class IDataPlugInEngineFactory~T~ {
        <<interface>>
        +GetInstance() T
    }

    IInputDataPlugInEngine --> "0..*" IInputDataPlugIn : loads & chains
    IOutputDataPlugInEngine --> "0..*" IOutputDataPlugIn : loads & chains
    IDataPlugInEngineFactory --> IInputDataPlugInEngine : creates
    IDataPlugInEngineFactory --> IOutputDataPlugInEngine : creates
```

**Plugin execution flow:**
1. `Configure(assemblyNames)` -- dynamically loads plugin types from assembly names
2. `ExecutePlugInsAsync()` -- chains plugins sequentially, passing the transformed `(DicomFile, metadata)` tuple through each

**Built-in plugin:** `RemoteAppExecution` (in `src/Plug-ins/RemoteAppExecution/`) provides DICOM de-identification and re-identification for external AI applications.

---

## 9. Configuration Architecture

```mermaid
classDiagram
    class InformaticsGatewayConfiguration {
        +DicomConfiguration Dicom
        +StorageConfiguration Storage
        +DicomWebConfiguration DicomWeb
        +FhirConfiguration Fhir
        +Hl7Configuration Hl7
        +DataExportConfiguration Export
        +MessageBrokerConfiguration Messaging
    }

    class DicomConfiguration {
        +ScpConfiguration Scp
        +ScuConfiguration Scu
    }

    class ScpConfiguration {
        +int Port = 104
        +string AeTitle
        +int MaxAssociations
        +bool LogDimseDatasets
    }

    class ScuConfiguration {
        +string AeTitle
        +bool LogDataPdus
    }

    class StorageConfiguration {
        +string StorageServiceBucketName
        +string TemporaryDataStorage
        +RetryConfiguration Retries
    }

    class MessageBrokerConfiguration {
        +string PublisherServiceAssemblyName
        +string SubscriberServiceAssemblyName
        +TopicConfiguration Topics
    }

    InformaticsGatewayConfiguration --> DicomConfiguration
    InformaticsGatewayConfiguration --> StorageConfiguration
    InformaticsGatewayConfiguration --> MessageBrokerConfiguration
    DicomConfiguration --> ScpConfiguration
    DicomConfiguration --> ScuConfiguration
```

**Bound in `Program.cs` via:**
```csharp
services.AddOptions<InformaticsGatewayConfiguration>()
    .Bind(hostContext.Configuration.GetSection("InformaticsGateway"));
```

Validated at startup by `ConfigurationValidator` (implements `IValidateOptions<InformaticsGatewayConfiguration>`).

---

## 10. REST API Controllers

```mermaid
flowchart LR
    subgraph ae_config["AE Configuration"]
        MONAI_CTRL["MonaiAeTitleController\n/config/monai"]
        SRC_CTRL["SourceAeTitleController\n/config/source"]
        DEST_CTRL["DestinationAeTitleController\n/config/destination"]
        VAE_CTRL["VirtualAeTitleController\n/config/virtual"]
    end

    subgraph hl7_config["HL7 Configuration"]
        HL7_CFG["Hl7ApplicationConfigController\n/config/hl7"]
        HL7_DEST["HL7DestinationController\n/config/hl7/destination"]
    end

    subgraph operations["Operations"]
        INF_CTRL["InferenceController\n/inference"]
        ASSOC_CTRL["DicomAssociationInfoController\n/dicom/associations"]
    end

    subgraph data_ingestion["Data Ingestion"]
        STOW_CTRL["StowController\n/dicomweb"]
        FHIR_CTRL["FhirController\n/fhir"]
    end

    subgraph health["Health"]
        HEALTH_CTRL["HealthController\n/health"]
    end
```

All controllers follow standard ASP.NET Core patterns with constructor-injected dependencies, `[ApiController]` attribute, and model validation.

---

## 11. Dependency Injection

All services are registered in `Program.cs` (the **composition root**). Lifetimes are chosen based on service statefulness.

### Transient Services (new instance per injection)

| Interface | Implementation | Purpose |
|-----------|---------------|---------|
| `IFileSystem` | `FileSystem` | File I/O abstraction |
| `IDicomToolkit` | `DicomToolkit` | fo-dicom wrapper |
| `IStowService` | `StowService` | DICOMweb STOW-RS |
| `IFhirService` | `FhirService` | FHIR resource handling |
| `IStreamsWriter` | `StreamsWriter` | DICOM stream persistence |
| `IApplicationEntityHandler` | `ApplicationEntityHandler` | Per-association DICOM handler |
| `IMllpExtract` | `MllpExtract` | HL7 message extraction |

### Scoped Services (per-request / per-scope)

| Interface | Implementation | Purpose |
|-----------|---------------|---------|
| `IPayloadMoveActionHandler` | `PayloadMoveActionHandler` | Move payload files |
| `IPayloadNotificationActionHandler` | `PayloadNotificationActionHandler` | Publish payload events |
| `IInputDataPlugInEngine` | `InputDataPlugInEngine` | Input DICOM plugin chain |
| `IOutputDataPlugInEngine` | `OutputDataPlugInEngine` | Output plugin chain |
| `IInputHL7DataPlugInEngine` | `InputHL7DataPlugInEngine` | Input HL7 plugin chain |
| `IDataPlugInEngineFactory<T>` | `*PlugInEngineFactory` | Plugin engine creation |
| All repository interfaces | EF/MongoDB implementations | Data access |

### Singleton Services (one instance, application lifetime)

| Interface | Implementation | Purpose |
|-----------|---------------|---------|
| `IObjectUploadQueue` | `ObjectUploadQueue` | File upload work queue |
| `IPayloadAssembler` | `PayloadAssembler` | In-memory payload batching |
| `IMonaiServiceLocator` | `MonaiServiceLocator` | Service discovery |
| `IStorageInfoProvider` | `StorageInfoProvider` | Disk space monitoring |
| `IMonaiAeChangedNotificationService` | `MonaiAeChangedNotificationService` | AE change events |
| `ITcpListenerFactory` | `TcpListenerFactory` | TCP socket factory |
| `IMllpClientFactory` | `MllpClientFactory` | MLLP client factory |
| `IApplicationEntityManager` | `ApplicationEntityManager` | AE validation & routing |
| `IScuQueue` | `ScuQueue` | SCU work queue |
| `IMllpService` | `MllpService` | HL7 MLLP server |

### Hosted Services (11 background workers)

| Service | Category |
|---------|----------|
| `ObjectUploadService` | Processing |
| `DataRetrievalService` | Processing |
| `ScpService` | Ingestion |
| `ExternalAppScpService` | Ingestion |
| `ScuService` | Utility |
| `ExtAppScuExportService` | Export |
| `ScuExportService` | Export |
| `DicomWebExportService` | Export |
| `PayloadNotificationService` | Processing |
| `MllpServiceHost` | Ingestion |
| `Hl7ExportService` | Export |

---

## 12. Design Patterns Catalog

| Pattern | Where Applied | Key Classes |
|---------|---------------|-------------|
| **Template Method** | SCP and Export service hierarchies | `ScpServiceBase`, `ExportServiceBase` |
| **Repository** | Dual ORM data access | `Database.Api` interfaces, EF/MongoDB impls |
| **Chain of Responsibility** | Plugin execution pipelines | `IInputDataPlugIn`, `IOutputDataPlugIn` |
| **Factory** | Plugin engines, TCP listeners, MLLP clients | `IDataPlugInEngineFactory<T>`, `ITcpListenerFactory` |
| **State Machine** | Payload lifecycle | `Payload.PayloadState` (Created → Move → Notify) |
| **Observer** | AE configuration change notifications | `IMonaiAeChangedNotificationService` |
| **TPL Dataflow Pipeline** | Export processing with backpressure | `ExportServiceBase.SetupActionBlocks()` |
| **Dependency Injection** | All service composition | `Program.cs` (composition root) |
| **Strategy** | Database provider selection | `ConfigureDatabase()` selects EF or MongoDB |
| **Adapter** | Network abstractions | `ITcpClientAdapter`, `ITcpListenerFactory` |

---

## 13. SOLID Principles Mapping

### S -- Single Responsibility

Each hosted service has one focused responsibility. For example:
- `ObjectUploadService` -- only uploads files to MinIO
- `PayloadNotificationService` -- only drives the Payload state machine and publishes events
- `ExportServiceBase` handles pipeline orchestration; subclasses implement only protocol-specific export logic

### O -- Open/Closed

- **Plugin system:** New data transformations are added by implementing `IInputDataPlugIn` or `IOutputDataPlugIn` -- no modification to existing code required
- **Database providers:** New ORM backends can be added by implementing `Database.Api` interfaces without changing consumers

### L -- Liskov Substitution

- `SourceApplicationEntity` and `DestinationApplicationEntity` are fully substitutable for `BaseApplicationEntity` in repository and controller operations
- All `FileStorageMetadata` subtypes (`Dicom`, `Hl7`, `Fhir`) are handled uniformly by `ObjectUploadService` and `PayloadAssembler`

### I -- Interface Segregation

- Separate repository interfaces per aggregate root (`IPayloadRepository`, `IMonaiApplicationEntityRepository`, etc.) rather than one mega-repository
- Plugin interfaces are split by direction and protocol: `IInputDataPlugIn`, `IOutputDataPlugIn`, `IInputHL7DataPlugIn`

### D -- Dependency Inversion

- All services depend on abstractions registered in the DI container
- `Database.Api` defines interfaces; `Database.EntityFramework` and `Database.MongoDB` provide implementations
- External systems accessed via `IStorageService` (MinIO) and `IMessageBrokerPublisherService` / `IMessageBrokerSubscriberService` (RabbitMQ) -- the gateway never references concrete SDK types

---

## 14. External Dependencies

```mermaid
flowchart TB
    MIG["MONAI Deploy\nInformatics Gateway"]

    subgraph dicom["DICOM Protocol"]
        FODICOM["fo-dicom 5.x\n(FellowOakDicom)"]
    end

    subgraph messaging["Messaging"]
        MSG_LIB["Monai.Deploy.Messaging\n(abstraction)"]
        RABBIT_IMPL["Monai.Deploy.Messaging.\nRabbitMQ"]
    end

    subgraph storage["Storage"]
        STG_LIB["Monai.Deploy.Storage\n(abstraction)"]
        MINIO_IMPL["Monai.Deploy.Storage.\nMinIO"]
    end

    subgraph data["Data Access"]
        EF["Microsoft.\nEntityFrameworkCore 8.x"]
        MONGO_DRV["MongoDB.Driver"]
    end

    subgraph health_proto["Healthcare Protocols"]
        HL7["HL7-dotnetcore"]
    end

    subgraph web["Web Framework"]
        ASPNET["ASP.NET Core 8"]
        SWAGGER["Swashbuckle\n(Swagger/OpenAPI)"]
    end

    subgraph utilities["Utilities"]
        NLOG["NLog"]
        POLLY["Polly\n(Resilience)"]
        GUARD["Ardalis.GuardClauses"]
    end

    MIG --> FODICOM & MSG_LIB & STG_LIB & EF & MONGO_DRV & HL7 & ASPNET & SWAGGER & NLOG & POLLY & GUARD
    MSG_LIB --> RABBIT_IMPL
    STG_LIB --> MINIO_IMPL
```

---

*Document generated from source code analysis of the MONAI Deploy Informatics Gateway repository.*
