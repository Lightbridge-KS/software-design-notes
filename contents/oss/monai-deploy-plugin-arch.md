# Plugin Architecture Pattern -- Learning from MONAI Deploy Informatics Gateway

> **Pattern Name:** Runtime Plugin Loading via Reflection + Chain of Responsibility
> **Language:** C# / .NET 8
> **Source Codebase:** MONAI Deploy Informatics Gateway

---

## Table of Contents

- [Plugin Architecture Pattern -- Learning from MONAI Deploy Informatics Gateway](#plugin-architecture-pattern----learning-from-monai-deploy-informatics-gateway)
  - [Table of Contents](#table-of-contents)
  - [1. What Problem Does This Solve?](#1-what-problem-does-this-solve)
  - [2. Pattern Overview](#2-pattern-overview)
  - [3. Architecture Layers](#3-architecture-layers)
  - [4. Layer 1 -- The Contract](#4-layer-1----the-contract)
  - [5. Layer 2 -- The Engine](#5-layer-2----the-engine)
  - [6. Layer 3 -- The Type Resolver](#6-layer-3----the-type-resolver)
  - [7. Layer 4 -- The Factory (Discovery)](#7-layer-4----the-factory-discovery)
  - [8. Layer 5 -- The Caller (Integration Point)](#8-layer-5----the-caller-integration-point)
  - [9. Layer 6 -- DI Registration](#9-layer-6----di-registration)
  - [10. End-to-End Flow](#10-end-to-end-flow)
  - [11. How to Write a Plugin](#11-how-to-write-a-plugin)
    - [Step 1: Create a Class Library](#step-1-create-a-class-library)
    - [Step 2: Implement the Interface](#step-2-implement-the-interface)
    - [Step 3: Build and Deploy](#step-3-build-and-deploy)
    - [Step 4: Configure via REST API](#step-4-configure-via-rest-api)
    - [Plugin Rules (from the source code docs)](#plugin-rules-from-the-source-code-docs)
  - [12. Design Patterns Involved](#12-design-patterns-involved)
    - [12.1 Chain of Responsibility](#121-chain-of-responsibility)
    - [12.2 Strategy Pattern](#122-strategy-pattern)
    - [12.3 Factory Pattern](#123-factory-pattern)
    - [12.4 Service Locator (Controlled)](#124-service-locator-controlled)
    - [12.5 Template Method (Implicit)](#125-template-method-implicit)
  - [13. Key Takeaways](#13-key-takeaways)
    - [What Makes This Plugin Architecture Good](#what-makes-this-plugin-architecture-good)
    - [What to Watch Out For](#what-to-watch-out-for)
    - [The Architecture Compared to Other Approaches](#the-architecture-compared-to-other-approaches)
  - [14. Source File Index](#14-source-file-index)

---

## 1. What Problem Does This Solve?

The Informatics Gateway receives DICOM files from hospital modalities (CT, MR, US scanners). Different AE Titles (endpoints) need **different processing** before files are stored:

- `BRAIN-AI` AE Title: anonymize patient data before sending to external AI vendor
- `LUNG-AI` AE Title: add custom metadata tags for a lung screening workflow
- `ARCHIVE` AE Title: no processing, store as-is

**Without plugins:** you'd hardcode each processing path, requiring recompilation and redeployment for every new requirement.

**With plugins:** processing logic lives in separate DLLs. Hospital IT drops a new DLL into a folder and configures it via REST API -- no gateway restart, no recompilation.

```
Without Plugins                         With Plugins
─────────────────                       ─────────────────
if (aeTitle == "BRAIN-AI")              // Configured per AE Title
    Anonymize(file);                    foreach (plugin in plugins)
else if (aeTitle == "LUNG-AI")              (file, meta) = plugin.Execute(file, meta);
    AddLungTags(file);
// ... grows forever
```

---

## 2. Pattern Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    COMPILE TIME                              │
│                                                              │
│   Api Project              Plugin Project (separate DLL)     │
│   ┌─────────────────┐     ┌──────────────────────────┐       │
│   │ IInputDataPlugIn│◄────│ MyPlugin : IInputDataPlugIn      │
│   │ (interface)     │     │                          │       │
│   └─────────────────┘     └──────────────┬───────────┘       │
│          ▲                               │                   │
│          │ no direct reference           │ build to DLL      │
│          │ from host to plugin           ▼                   │
│          │                       MyPlugin.dll                │
└──────────┼───────────────────────────────┼───────────────────┘
           │                               │
┌──────────┼───────────────────────────────┼────────────────────┐
│          │        RUNTIME                │ deploy             │
│          │                               ▼                    │
│   Host Application               plug-ins/MyPlugin.dll        │
│   ┌─────────────────────┐               │                     │
│   │ PlugInEngine        │── scan *.dll ─┘                     │
│   │                     │                                     │
│   │ 1. Type.GetType()   │── resolve fully-qualified name      │
│   │ 2. ActivatorUtils   │── create instance with DI           │
│   │ 3. foreach Execute  │── chain plugins sequentially        │
│   └─────────────────────┘                                     │
└───────────────────────────────────────────────────────────────┘
```

The host application **never references** the plugin DLL at compile time. It only knows the `IInputDataPlugIn` interface. At runtime, it loads the DLL, finds the type, creates an instance with full DI support, and chains execution.

---

## 3. Architecture Layers

The plugin system is composed of **6 layers**, each with a single responsibility:

```
Layer 6: DI Registration        Program.cs          (wires everything into DI container)
Layer 5: The Caller             ApplicationEntity    (calls engine during DICOM ingestion)
                                Handler
Layer 4: The Factory            DataPlugInEngine     (discovers available plugins from
                                FactoryBase<T>        plug-ins/ directory)
Layer 3: The Type Resolver      TypeExtensions       (turns "Namespace.Class, Assembly"
                                                      string into a live .NET Type)
Layer 2: The Engine             InputDataPlugIn      (loads plugin instances and
                                Engine                chains execution)
Layer 1: The Contract           IInputDataPlugIn     (interface that plugins implement)
```

Each layer is explained below with the actual source code.

---

## 4. Layer 1 -- The Contract

> **File:** `src/Api/PlugIns/IInputDataPlugin.cs`

```csharp
public interface IInputDataPlugIn
{
    string Name { get; }

    Task<(DicomFile dicomFile, FileStorageMetadata fileMetadata)>
        ExecuteAsync(DicomFile dicomFile, FileStorageMetadata fileMetadata);
}
```

**Design decisions:**

| Decision | Why |
|----------|-----|
| Returns a **tuple** `(DicomFile, FileStorageMetadata)` | Enables chaining -- output of one plugin feeds into the next |
| **Async** (`Task<>`) | Plugins may do I/O (database lookups, network calls) |
| `string Name` property | Human-readable identifier for logging and configuration display |
| Lives in the **Api** project (not the host) | Both host and plugins reference Api, creating a shared contract without circular dependencies |

There's also an optional attribute for friendly naming:

> **File:** `src/Api/PlugIns/PluginNameAttribute.cs`

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class PlugInNameAttribute : Attribute
{
    public string Name { get; set; }

    public PlugInNameAttribute(string name)
    {
        Name = name;
    }
}
```

Usage on a plugin class:
```csharp
[PlugInName("Remote App Execution Outgoing")]
public class DicomDeidentifier : IOutputDataPlugIn { ... }
```

**There are 3 parallel plugin contracts** for different data types:

| Interface | When it runs | Signature |
|-----------|-------------|-----------|
| `IInputDataPlugIn` | During ingestion (before upload) | `(DicomFile, FileStorageMetadata) → (DicomFile, FileStorageMetadata)` |
| `IOutputDataPlugIn` | During export (before sending) | `(DicomFile, ExportRequestDataMessage) → (DicomFile, ExportRequestDataMessage)` |
| `IInputHL7DataPlugIn` | During HL7 ingestion | HL7-specific processing |

---

## 5. Layer 2 -- The Engine

> **File:** `src/InformaticsGateway/Services/Common/InputDataPluginEngine.cs`

The engine has two responsibilities: **load** plugin instances and **execute** them in sequence.

```csharp
internal class InputDataPlugInEngine : IInputDataPlugInEngine
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<InputDataPlugInEngine> _logger;
    private IReadOnlyList<IInputDataPlugIn>? _plugins;

    // ── Step 1: Configure (load plugins from type name strings) ──────
    public void Configure(IReadOnlyList<string> pluginAssemblies)
    {
        _plugins = LoadPlugIns(_serviceProvider, pluginAssemblies);
    }

    // ── Step 2: Execute (chain plugins sequentially) ─────────────────
    public async Task<Tuple<DicomFile, FileStorageMetadata>> ExecutePlugInsAsync(
        DicomFile dicomFile, FileStorageMetadata fileMetadata)
    {
        if (_plugins == null)
            throw new PlugInInitializationException("Not configured, call Configure() first.");

        // Chain of Responsibility: each plugin transforms the data
        foreach (var plugin in _plugins)
        {
            _logger.ExecutingInputDataPlugIn(plugin.Name);
            (dicomFile, fileMetadata) = await plugin.ExecuteAsync(dicomFile, fileMetadata);
        }

        return new Tuple<DicomFile, FileStorageMetadata>(dicomFile, fileMetadata);
    }

    // ── Loading: turn type name strings into live objects ─────────────
    private List<IInputDataPlugIn> LoadPlugIns(
        IServiceProvider serviceProvider, IReadOnlyList<string> pluginAssemblies)
    {
        var list = new List<IInputDataPlugIn>();
        foreach (var plugin in pluginAssemblies)
        {
            // This is where the magic happens (see Layer 3)
            list.Add(typeof(IInputDataPlugIn)
                .CreateInstance<IInputDataPlugIn>(serviceProvider, typeString: plugin));
        }
        return list;
    }
}
```

**The chain execution visualized:**

```
Plugin[0].ExecuteAsync(file, meta)
    │
    ├── modifies file (e.g., anonymize patient name)
    ├── modifies meta (e.g., add workflow ID)
    │
    └── returns (modifiedFile, modifiedMeta)
                    │
                    ▼
Plugin[1].ExecuteAsync(modifiedFile, modifiedMeta)
    │
    ├── further transforms...
    │
    └── returns (finalFile, finalMeta)
                    │
                    ▼
            Engine returns to caller
```

**The Output variant** (`OutputDataPlugInEngine`) follows the same pattern but additionally deserializes the DICOM file from `byte[]` before the chain and serializes it back after:

```csharp
// OutputDataPlugInEngine.ExecutePlugInsAsync()
var dicomFile = _dicomToolkit.Load(exportRequestDataMessage.FileContent);  // deserialize

foreach (var plugin in _plugins)
    (dicomFile, exportRequestDataMessage) = await plugin.ExecuteAsync(dicomFile, exportRequestDataMessage);

using var ms = new MemoryStream();
await dicomFile.SaveAsync(ms);                                            // serialize back
exportRequestDataMessage.SetData(ms.ToArray());
```

---

## 6. Layer 3 -- The Type Resolver

> **File:** `src/InformaticsGateway/Common/TypeExtensions.cs`

This is the **reflection core** -- it turns a string like `"MyNamespace.MyPlugin, MyAssembly"` into a live .NET object with injected dependencies.

```csharp
public static class TypeExtensions
{
    // Turn a type string into an instance with DI
    public static T CreateInstance<T>(
        this Type interfaceType, IServiceProvider serviceProvider,
        string typeString, params object[] parameters)
    {
        var type = interfaceType.GetType(typeString);  // resolve Type from string
        var processor = ActivatorUtilities.CreateInstance(serviceProvider, type, parameters);
        return (T)processor;
    }

    // Resolve a Type from a fully-qualified type string
    public static Type GetType(this Type interfaceType, string typeString)
    {
        var type = Type.GetType(
            typeString,
            assemblyResolver: (name) =>
            {
                // 1. First: check already-loaded assemblies
                var assembly = AppDomain.CurrentDomain.GetAssemblies()
                    .FirstOrDefault(z => z.FullName.StartsWith(name.FullName));

                // 2. Fallback: load DLL from plug-ins/ directory
                assembly ??= Assembly.Load(
                    File.ReadAllBytes(
                        Path.Combine(SR.PlugInDirectoryPath, $"{name.Name}.dll")));

                return assembly;
            },
            typeResolver: null,
            throwOnError: true);

        // 3. Validate: must implement the expected interface
        if (type is not null && type.GetInterfaces().Contains(interfaceType))
            return type;

        throw new NotSupportedException($"{typeString} is not a sub-type of {interfaceType.Name}");
    }
}
```

**Three-step resolution:**

```
Input: "Monai.Deploy...DicomDeidentifier, Monai.Deploy...RemoteAppExecution"
                                            ─────────────────────────────────
                                            │                               │
                                            ▼                               ▼
                                      Class Name                     Assembly Name

Step 1: Check AppDomain.CurrentDomain.GetAssemblies()
        ── Is the assembly already loaded?
        ── If yes: return it

Step 2: Assembly.Load(File.ReadAllBytes("plug-ins/{AssemblyName}.dll"))
        ── Load DLL from plug-ins/ directory
        ── File-based loading (not GAC, not NuGet)

Step 3: Validate type implements IInputDataPlugIn
        ── Safety check against loading arbitrary types

Output: System.Type object ready for instantiation
```

**`ActivatorUtilities.CreateInstance`** is the key .NET API that makes DI work for plugins:

```csharp
// This is NOT just Activator.CreateInstance()
// ActivatorUtilities resolves constructor parameters from the DI container

// If plugin has this constructor:
public DicomDeidentifier(
    ILogger<DicomDeidentifier> logger,          // ← resolved from DI
    IServiceScopeFactory serviceScopeFactory,    // ← resolved from DI
    IOptions<PlugInConfiguration> configuration) // ← resolved from DI
{
}

// ActivatorUtilities knows how to fill all these from serviceProvider
ActivatorUtilities.CreateInstance(serviceProvider, type);
```

---

## 7. Layer 4 -- The Factory (Discovery)

> **File:** `src/InformaticsGateway/Services/Common/IInputDataPluginEngineFactory.cs`

The factory scans for **all available plugins** at startup. This powers the REST API that lists registered plugins.

```csharp
public abstract class DataPlugInEngineFactoryBase<T> : IDataPlugInEngineFactory<T>
{
    private readonly Dictionary<string, string> _cachedTypeNames;

    public IReadOnlyDictionary<string, string> RegisteredPlugIns()
    {
        // ── Step 1: Load DLLs from plug-ins/ directory ───────
        LoadAssembliesFromPlugInsDirectory();

        // ── Step 2: Scan ALL loaded assemblies for implementations ───
        var types = AppDomain.CurrentDomain.GetAssemblies()
            .SelectMany(s => s.GetTypes())
            .Where(p => typeof(T).IsAssignableFrom(p) && p != typeof(T))
            .ToList();

        // ── Step 3: Cache friendly-name → fully-qualified-name mapping ──
        AddToCache(types);

        return _cachedTypeNames;
    }

    private void LoadAssembliesFromPlugInsDirectory()
    {
        var files = _fileSystem.Directory.GetFiles(
            SR.PlugInDirectoryPath, "*.dll", SearchOption.TopDirectoryOnly);

        foreach (var file in files)
        {
            var assembly = Assembly.Load(File.ReadAllBytes(file));
            // Find types implementing T (e.g., IInputDataPlugIn)
            var matchingTypes = assembly.GetTypes()
                .Where(p => typeof(T).IsAssignableFrom(p) && p != typeof(T))
                .ToList();
            AddToCache(matchingTypes);
        }
    }

    private void AddToCache(List<Type> types)
    {
        types.ForEach(p =>
        {
            // Use [PlugInName("...")] if present, otherwise class name
            var nameAttribute = p.GetCustomAttribute<PlugInNameAttribute>();
            var name = nameAttribute?.Name ?? p.Name;
            _cachedTypeNames.Add(name, p.GetShortTypeAssemblyName());
        });
    }
}
```

**Concrete factories** are trivial -- they only specify the type parameter:

```csharp
public class InputDataPlugInEngineFactory
    : DataPlugInEngineFactoryBase<IInputDataPlugIn> { }

public class OutputDataPlugInEngineFactory
    : DataPlugInEngineFactoryBase<IOutputDataPlugIn> { }

public class InputHL7DataPlugInEngineFactory
    : DataPlugInEngineFactoryBase<IInputHL7DataPlugIn> { }
```

**Result example** of `RegisteredPlugIns()`:

```
{
  "Remote App Execution Outgoing": "Monai.Deploy...DicomDeidentifier, Monai.Deploy...RemoteAppExecution",
  "Remote App Execution Incoming": "Monai.Deploy...DicomReidentifier, Monai.Deploy...RemoteAppExecution"
}
```

---

## 8. Layer 5 -- The Caller (Integration Point)

> **File:** `src/InformaticsGateway/Services/Scp/ApplicationEntityHandler.cs`

This is where the plugin engine is actually **used** during DICOM file reception:

```csharp
internal class ApplicationEntityHandler : IApplicationEntityHandler
{
    private readonly IInputDataPlugInEngine _pluginEngine;

    // ── Called once when DICOM association starts ─────────────────
    public void Configure(MonaiApplicationEntity monaiApplicationEntity, ...)
    {
        _configuration = monaiApplicationEntity;

        // Load plugins configured for THIS specific AE Title
        _pluginEngine.Configure(_configuration.PlugInAssemblies);
        //                      ───────────────────────────────
        //                      ["Namespace.Plugin1, Assembly1",
        //                       "Namespace.Plugin2, Assembly2"]
    }

    // ── Called for EACH DICOM file received ───────────────────────
    public async Task<string> HandleInstanceAsync(
        DicomCStoreRequest request, string calledAeTitle, ...)
    {
        // Create metadata for this file
        var dicomInfo = new DicomFileStorageMetadata(
            associationId, identifier,
            studyInstanceUid, seriesInstanceUid, sopInstanceUid,
            DataService.DIMSE, callingAeTitle, calledAeTitle);

        // ──── EXECUTE PLUGIN CHAIN ────
        var result = await _pluginEngine.ExecutePlugInsAsync(request.File, dicomInfo);
        //                               ───────────────  ────────
        //                               original file    original metadata
        //
        //   Returns: (transformedFile, transformedMetadata)

        dicomInfo = (result.Item2 as DicomFileStorageMetadata)!;
        var dicomFile = result.Item1;

        // Continue with transformed data: upload, assemble payload, etc.
        await dicomInfo.SetDataStreams(dicomFile, ...);
        await _payloadAssembler.Queue(key, dicomInfo, ...);
        _uploadQueue.Queue(dicomInfo);
    }
}
```

**Key point:** `_pluginEngine.Configure(...)` is called with the `PlugInAssemblies` list from the specific `MonaiApplicationEntity`. This means **each AE Title can have a different plugin pipeline**.

---

## 9. Layer 6 -- DI Registration

> **File:** `src/InformaticsGateway/Program.cs` (lines 118-123)

```csharp
// Plugin engines -- Scoped (one per request/scope)
services.AddScoped<IInputDataPlugInEngine, InputDataPlugInEngine>();
services.AddScoped<IOutputDataPlugInEngine, OutputDataPlugInEngine>();
services.AddScoped<IInputHL7DataPlugInEngine, InputHL7DataPlugInEngine>();

// Plugin factories -- Scoped (for discovery)
services.AddScoped<IDataPlugInEngineFactory<IInputDataPlugIn>, InputDataPlugInEngineFactory>();
services.AddScoped<IDataPlugInEngineFactory<IOutputDataPlugIn>, OutputDataPlugInEngineFactory>();
services.AddScoped<IDataPlugInEngineFactory<IInputHL7DataPlugIn>, InputHL7DataPlugInEngineFactory>();
```

**Why Scoped?** Each DICOM association (connection) gets its own `ApplicationEntityHandler`, which creates a scope and gets a fresh `InputDataPlugInEngine`. This means each association's plugin chain is independent -- no shared state between concurrent connections.

---

## 10. End-to-End Flow

Here's every step from user configuration to plugin execution:

```
                                TIME
                                 │
 ╔═══════════════════════════════╪═══════════════════════════════════╗
 ║  DEPLOYMENT                   │                                   ║
 ║                               │                                   ║
 ║  1. Build plugin DLL          │  MyPlugin.dll                     ║
 ║  2. Copy to plug-ins/         │  → plug-ins/MyPlugin.dll          ║
 ╚═══════════════════════════════╪═══════════════════════════════════╝
                                 │
 ╔═══════════════════════════════╪═══════════════════════════════════╗
 ║  CONFIGURATION (REST API)     │                                   ║
 ║                               │                                   ║
 ║  POST /config/monai           │                                   ║
 ║  {                            │                                   ║
 ║    "aeTitle": "BRAIN-AI",     │                                   ║
 ║    "plugInAssemblies": [      │                                   ║
 ║      "My.Plugin, MyPlugin"    │  Saved to database                ║
 ║    ]                          │  (MonaiApplicationEntity table)   ║
 ║  }                            │                                   ║
 ╚═══════════════════════════════╪═══════════════════════════════════╝
                                 │
 ╔═══════════════════════════════╪═══════════════════════════════════╗
 ║  DICOM ASSOCIATION ARRIVES    │                                   ║
 ║                               │                                   ║
 ║  3. ScpService accepts        │  C-STORE to AE "BRAIN-AI"         ║
 ║     connection                │                                   ║
 ║                               │                                   ║
 ║  4. ApplicationEntityManager  │  Looks up MonaiApplicationEntity  ║
 ║     creates handler           │  from database by AE Title        ║
 ║                               │                                   ║
 ║  5. handler.Configure(entity) │                                   ║
 ║     └── pluginEngine          │                                   ║
 ║         .Configure(           │                                   ║
 ║           ["My.Plugin,        │                                   ║
 ║            MyPlugin"])        │                                   ║
 ║                               │                                   ║
 ║  6. LoadPlugIns() for each    │                                   ║
 ║     type string:              │                                   ║
 ║     a. Type.GetType()         │  Resolve type from string         ║
 ║        └── AppDomain check    │  (check loaded assemblies)        ║
 ║        └── Assembly.Load()    │  (fallback: load from plug-ins/)  ║
 ║     b. Validate interface     │  (must implement IInputDataPlugIn)║
 ║     c. ActivatorUtilities     │  (create instance with DI)        ║
 ║        .CreateInstance()      │                                   ║
 ╚═══════════════════════════════╪═══════════════════════════════════╝
                                 │
 ╔═══════════════════════════════╪═══════════════════════════════════╗
 ║  FOR EACH DICOM FILE          │                                   ║
 ║                               │                                   ║
 ║  7. HandleInstanceAsync()     │  C-STORE request with file        ║
 ║                               │                                   ║
 ║  8. pluginEngine              │                                   ║
 ║     .ExecutePlugInsAsync(     │                                   ║
 ║       dicomFile, metadata)    │                                   ║
 ║                               │                                   ║
 ║     foreach plugin:           │                                   ║
 ║       (file, meta) =          │  Plugin transforms DICOM data     ║
 ║         plugin.ExecuteAsync(  │  and/or metadata                  ║
 ║           file, meta)         │                                   ║
 ║                               │                                   ║
 ║  9. Upload transformed file   │  → MinIO                          ║
 ║ 10. Queue to PayloadAssembler │  → eventually → RabbitMQ          ║
 ╚═══════════════════════════════╪═══════════════════════════════════╝
```

---

## 11. How to Write a Plugin

### Step 1: Create a Class Library

```bash
dotnet new classlib -n MyDicomPlugin
cd MyDicomPlugin
dotnet add reference ../src/Api/Monai.Deploy.InformaticsGateway.Api.csproj
```

### Step 2: Implement the Interface

```csharp
using FellowOakDicom;
using Monai.Deploy.InformaticsGateway.Api.PlugIns;
using Monai.Deploy.InformaticsGateway.Api.Storage;
using Microsoft.Extensions.Logging;

namespace MyDicomPlugin;

[PlugInName("Patient Name Anonymizer")]
public class PatientNameAnonymizer : IInputDataPlugIn
{
    private readonly ILogger<PatientNameAnonymizer> _logger;

    // Constructor dependencies are injected via ActivatorUtilities
    public PatientNameAnonymizer(ILogger<PatientNameAnonymizer> logger)
    {
        _logger = logger;
    }

    public string Name => "Patient Name Anonymizer";

    public Task<(DicomFile dicomFile, FileStorageMetadata fileMetadata)> ExecuteAsync(
        DicomFile dicomFile, FileStorageMetadata fileMetadata)
    {
        var originalName = dicomFile.Dataset.GetSingleValueOrDefault(DicomTag.PatientName, "");
        dicomFile.Dataset.AddOrUpdate(DicomTag.PatientName, "ANONYMOUS");

        _logger.LogInformation("Anonymized patient name: {Original} → ANONYMOUS", originalName);

        return Task.FromResult((dicomFile, fileMetadata));
    }
}
```

### Step 3: Build and Deploy

```bash
dotnet build -c Release
cp bin/Release/net8.0/MyDicomPlugin.dll /path/to/gateway/plug-ins/
```

### Step 4: Configure via REST API

```bash
curl -X POST http://localhost:5000/config/monai \
  -H "Content-Type: application/json" \
  -d '{
    "name": "anon-pipeline",
    "aeTitle": "ANONAI",
    "plugInAssemblies": [
      "MyDicomPlugin.PatientNameAnonymizer, MyDicomPlugin"
    ],
    "timeout": 10
  }'
```

### Plugin Rules (from the source code docs)

1. Plugins **MUST be lightweight** -- do not hinder the upload process
2. Incoming data is processed **one file at a time** -- do not wait for entire study
3. Plugins **SHALL NOT accumulate files** in memory or storage for bulk processing
4. Multiple plugins execute **in the order listed** in `PlugInAssemblies`

---

## 12. Design Patterns Involved

This plugin architecture weaves together **5 design patterns**:

### 12.1 Chain of Responsibility

Each plugin processes data and passes the result to the next. Any plugin can modify the data, and the engine doesn't know what each plugin does.

```
Plugin A → Plugin B → Plugin C
   │           │           │
   └── Each transforms (DicomFile, FileStorageMetadata)
       and passes the result forward
```

### 12.2 Strategy Pattern

Different AE Titles are configured with different plugin lists. The same engine executes different strategies based on configuration.

```
AE "BRAIN-AI"  → [Anonymizer, BrainMetadataAdder]
AE "LUNG-AI"   → [Anonymizer, LungScreeningValidator]
AE "ARCHIVE"   → []  (no plugins)
```

### 12.3 Factory Pattern

`DataPlugInEngineFactoryBase<T>` discovers and caches plugin types. The generic `<T>` parameter allows one factory implementation for three plugin interfaces.

```csharp
// One base, three concrete factories via generic specialization
InputDataPlugInEngineFactory    : DataPlugInEngineFactoryBase<IInputDataPlugIn>
OutputDataPlugInEngineFactory   : DataPlugInEngineFactoryBase<IOutputDataPlugIn>
InputHL7DataPlugInEngineFactory : DataPlugInEngineFactoryBase<IInputHL7DataPlugIn>
```

### 12.4 Service Locator (Controlled)

`ActivatorUtilities.CreateInstance(serviceProvider, type)` acts as a service locator to inject constructor dependencies. This is intentional here -- plugins are loaded dynamically, so they can't be registered in DI at compile time.

### 12.5 Template Method (Implicit)

All plugins follow the same execution contract. The engine defines the algorithm skeleton (load → validate → iterate → execute), and each plugin fills in the `ExecuteAsync` step.

---

## 13. Key Takeaways

### What Makes This Plugin Architecture Good

| Quality | How It's Achieved |
|---------|-------------------|
| **Decoupled deployment** | Plugins are separate DLLs in `plug-ins/` directory |
| **No recompilation needed** | Host loads plugins via `Assembly.Load` at runtime |
| **Full DI support** | `ActivatorUtilities.CreateInstance` injects logger, config, repos |
| **Per-endpoint configuration** | Each AE Title has its own `PlugInAssemblies` list |
| **Type safety** | Validated against interface at load time (`GetInterfaces().Contains(...)`) |
| **Composable** | Multiple plugins chain via sequential execution |
| **Testable** | Plugins are plain classes with interface dependencies |

### What to Watch Out For

| Risk | Mitigation in This Codebase |
|------|----------------------------|
| Plugin crashes take down host | Aggregate exception collection in `LoadPlugIns()` |
| Slow plugins block ingestion | Documented rule: plugins MUST be lightweight |
| Version mismatch (Api DLL) | Plugins reference the same Api project version |
| Circular dependencies | Plugin DLLs only reference Api, never the host |

### The Architecture Compared to Other Approaches

```
Approach               Coupling    DI Support    Hot Deploy    Complexity
──────────────────     ────────    ──────────    ──────────    ──────────
Hardcoded if/else      High        N/A           No            Low
Interface + DI         Medium      Yes           No            Low
MEF (Managed Ext.)     Low         Partial       Yes           Medium
THIS APPROACH          Low         Full          Yes*          Medium
  (Reflection + DI)
Plugin Host (AppDomain) Very Low   Isolated      Yes           High

* Requires engine reconfiguration, not full app restart
```

---

## 14. Source File Index

| Layer | File | Purpose |
|-------|------|---------|
| Contract | `src/Api/PlugIns/IInputDataPlugin.cs` | Input plugin interface |
| Contract | `src/Api/PlugIns/IOutputDataPlugin.cs` | Output plugin interface |
| Contract | `src/Api/PlugIns/IInputHL7DataPlugIn.cs` | HL7 plugin interface |
| Contract | `src/Api/PlugIns/IInputDataPluginEngine.cs` | Engine interface |
| Contract | `src/Api/PlugIns/IOutputDataPluginEngine.cs` | Engine interface |
| Contract | `src/Api/PlugIns/PluginNameAttribute.cs` | Friendly name attribute |
| Contract | `src/Api/PlugIns/SR.cs` | Plugin directory path constant |
| Engine | `src/InformaticsGateway/Services/Common/InputDataPluginEngine.cs` | Input plugin chain executor |
| Engine | `src/InformaticsGateway/Services/Common/OutputDataPluginEngine.cs` | Output plugin chain executor |
| Resolver | `src/InformaticsGateway/Common/TypeExtensions.cs` | Reflection-based type loading |
| Factory | `src/InformaticsGateway/Services/Common/IInputDataPluginEngineFactory.cs` | Plugin discovery + all 3 factories |
| Caller | `src/InformaticsGateway/Services/Scp/ApplicationEntityHandler.cs` | Integration point (DICOM SCP) |
| DI | `src/InformaticsGateway/Program.cs` (lines 118-123) | Service registration |
| Config | `src/Api/Models/MonaiApplicationEntity.cs` (line 80) | `PlugInAssemblies` property |
| Example | `src/Plug-ins/RemoteAppExecution/DicomDeidentifier.cs` | Built-in output plugin |
| Example | `src/Plug-ins/RemoteAppExecution/DicomReidentifier.cs` | Built-in input plugin |

---

*Learning material generated from source code analysis of the MONAI Deploy Informatics Gateway repository.*
