# NIXL System Component Architecture Diagram

This document provides a comprehensive component architecture diagram for the NIXL (NVIDIA Inference Xfer Library) system, showing how different components interact and the role of plugins like UCX.

## Overall System Architecture

```mermaid
graph TB
    subgraph "Application Layer"
        APP1[Python Application<br/>PyTorch/JAX]
        APP2[C++ Application<br/>CUDA Applications]
        APP3[Rust Application<br/>ML Frameworks]
    end

    subgraph "Language Bindings"
        PYBIND[Python Bindings<br/>pybind11]
        RUSTBIND[Rust Bindings<br/>FFI Layer]
        CPPAPI[C++ Public API<br/>nixl.h]
    end

    subgraph "NIXL Core"
        AGENT[nixlAgent<br/>Main Orchestrator]
        PLUGMGR[nixlPluginManager<br/>Plugin Discovery]
        MEMSEC[Memory Sections<br/>nixl_memory_section.cpp]
        DESC[Descriptors<br/>nixl_descriptors.cpp]
        LISTENER[nixlListener<br/>Event Handling]
    end

    subgraph "Plugin Interface Layer"
        BACKENDAPI[Backend Engine API<br/>backend_engine.h]
        PLUGINAPI[Plugin Interface<br/>backend_plugin.h]
    end

    subgraph "Communication Backends"
        UCX[UCX Plugin<br/>High-perf Networking]
        UCXMO[UCX_MO Plugin<br/>Multi-object UCX]
        POSIX[POSIX Plugin<br/>File System I/O]
        GDS[CUDA GDS Plugin<br/>GPU Direct Storage]
        GDSMT[GDS_MT Plugin<br/>Multi-threaded GDS]
        GPUNETIO[GPUNetIO Plugin<br/>BlueField DPU]
        MOONCAKE[Mooncake Plugin<br/>Distributed Storage]
        HF3FS[HF3FS Plugin<br/>HuggingFace 3FS]
        OBJ[Object Storage Plugin<br/>S3/Cloud Storage]
    end

    subgraph "External Dependencies"
        UCXLIB[UCX Library<br/>v1.18.0+]
        CUDA[CUDA Toolkit<br/>GPU Operations]
        ETCD[ETCD Server<br/>Coordination]
        FILESYSTEM[File System<br/>POSIX/NFS]
        CLOUD[Cloud Storage<br/>S3/GCS/Azure]
    end

    subgraph "Memory Types"
        DRAM[System Memory<br/>DRAM_SEG]
        VRAM[GPU Memory<br/>VRAM_SEG]
        FILE[File Storage<br/>FILE_SEG]
        BLOCK[Block Storage<br/>BLK_SEG]
        OBJECT[Object Storage<br/>OBJ_SEG]
    end

    %% Application to Bindings
    APP1 --> PYBIND
    APP2 --> CPPAPI
    APP3 --> RUSTBIND

    %% Bindings to Core
    PYBIND --> AGENT
    RUSTBIND --> AGENT
    CPPAPI --> AGENT

    %% Core Internal Dependencies
    AGENT --> PLUGMGR
    AGENT --> MEMSEC
    AGENT --> DESC
    AGENT --> LISTENER

    %% Plugin Management
    PLUGMGR --> PLUGINAPI
    AGENT --> BACKENDAPI

    %% Plugin Implementations
    PLUGINAPI --> UCX
    PLUGINAPI --> UCXMO
    PLUGINAPI --> POSIX
    PLUGINAPI --> GDS
    PLUGINAPI --> GDSMT
    PLUGINAPI --> GPUNETIO
    PLUGINAPI --> MOONCAKE
    PLUGINAPI --> HF3FS
    PLUGINAPI --> OBJ

    %% Backend to External Dependencies
    UCX --> UCXLIB
    UCXMO --> UCXLIB
    GDS --> CUDA
    GDSMT --> CUDA
    GPUNETIO --> CUDA
    POSIX --> FILESYSTEM
    OBJ --> CLOUD
    AGENT --> ETCD

    %% Memory Type Support
    UCX --> DRAM
    UCX --> VRAM
    POSIX --> FILE
    GDS --> VRAM
    GDS --> FILE
    OBJ --> OBJECT

    %% Styling
    classDef appLayer fill:#e1f5fe
    classDef bindingLayer fill:#f3e5f5
    classDef coreLayer fill:#e8f5e8
    classDef pluginLayer fill:#fff3e0
    classDef backendLayer fill:#fce4ec
    classDef externalLayer fill:#f5f5f5
    classDef memoryLayer fill:#e0f2f1

    class APP1,APP2,APP3 appLayer
    class PYBIND,RUSTBIND,CPPAPI bindingLayer
    class AGENT,PLUGMGR,MEMSEC,DESC,LISTENER coreLayer
    class BACKENDAPI,PLUGINAPI pluginLayer
    class UCX,UCXMO,POSIX,GDS,GDSMT,GPUNETIO,MOONCAKE,HF3FS,OBJ backendLayer
    class UCXLIB,CUDA,ETCD,FILESYSTEM,CLOUD externalLayer
    class DRAM,VRAM,FILE,BLOCK,OBJECT memoryLayer
```

## UCX Plugin Internal Architecture

```mermaid
graph TB
    subgraph "UCX Plugin (libplugin_UCX.so)"
        UCXPLUGIN[UCX Plugin Entry<br/>ucx_plugin.cpp]
        UCXENGINE[nixlUcxEngine<br/>Main Implementation]
        UCXCONN[nixlUcxConnection<br/>Connection Management]
        UCXMETA[Metadata Classes<br/>Private/Public MD]
        UCXREQ[nixlUcxReq<br/>Request Tracking]
        UCXTHREAD[nixlUcxThread<br/>Progress Thread]
    end

    subgraph "UCX Utilities"
        UCXUTILS[UCX Utils<br/>Helper Functions]
        UCXRKEY[Remote Key Mgmt<br/>rkey.cpp]
        UCXCONFIG[UCX Configuration<br/>config.cpp]
    end

    subgraph "UCX Library Integration"
        UCXCONTEXT[UCP Context<br/>ucp_context_h]
        UCXWORKER[UCP Worker<br/>ucp_worker_h]
        UCXEP[UCP Endpoint<br/>ucp_ep_h]
        UCXMEM[UCP Memory<br/>ucp_mem_h]
        UCXRKEY2[UCP Remote Key<br/>ucp_rkey_h]
    end

    subgraph "CUDA Integration (Optional)"
        CUDACTX[CUDA Context<br/>CUcontext]
        CUDAMEM[CUDA Memory<br/>Device Pointers]
        GPUDIRECT[GPU Direct<br/>P2P Access]
    end

    %% Plugin Internal Flow
    UCXPLUGIN --> UCXENGINE
    UCXENGINE --> UCXCONN
    UCXENGINE --> UCXMETA
    UCXENGINE --> UCXREQ
    UCXENGINE --> UCXTHREAD

    %% Utilities
    UCXENGINE --> UCXUTILS
    UCXENGINE --> UCXRKEY
    UCXENGINE --> UCXCONFIG

    %% UCX Library Integration
    UCXENGINE --> UCXCONTEXT
    UCXCONTEXT --> UCXWORKER
    UCXWORKER --> UCXEP
    UCXCONTEXT --> UCXMEM
    UCXMEM --> UCXRKEY2

    %% CUDA Integration
    UCXENGINE --> CUDACTX
    UCXMEM --> CUDAMEM
    UCXEP --> GPUDIRECT

    classDef pluginCore fill:#fce4ec
    classDef utilities fill:#fff3e0
    classDef ucxLib fill:#e3f2fd
    classDef cudaIntegration fill:#e8f5e8

    class UCXPLUGIN,UCXENGINE,UCXCONN,UCXMETA,UCXREQ,UCXTHREAD pluginCore
    class UCXUTILS,UCXRKEY,UCXCONFIG utilities
    class UCXCONTEXT,UCXWORKER,UCXEP,UCXMEM,UCXRKEY2 ucxLib
    class CUDACTX,CUDAMEM,GPUDIRECT cudaIntegration
```

## Data Flow Architecture

```mermaid
graph LR
    subgraph "Node A"
        subgraph "Application A"
            APPA[App Process A]
        end
        subgraph "NIXL Agent A"
            AGENTA[nixlAgent A]
            UCXA[UCX Engine A]
        end
        subgraph "Memory A"
            MEMA[GPU Memory A<br/>VRAM]
            SYSMEMA[System Memory A<br/>DRAM]
        end
    end

    subgraph "Network Layer"
        NETWORK[High-Speed Network<br/>InfiniBand/Ethernet]
        ETCDNET[ETCD Coordination<br/>Metadata Exchange]
    end

    subgraph "Node B"
        subgraph "Application B"
            APPB[App Process B]
        end
        subgraph "NIXL Agent B"
            AGENTB[nixlAgent B]
            UCXB[UCX Engine B]
        end
        subgraph "Memory B"
            MEMB[GPU Memory B<br/>VRAM]
            SYSMEMB[System Memory B<br/>DRAM]
        end
    end

    %% Data Flow
    APPA --> AGENTA
    AGENTA --> UCXA
    UCXA --> MEMA
    UCXA --> SYSMEMA

    UCXA <--> NETWORK
    NETWORK <--> UCXB

    UCXB --> MEMB
    UCXB --> SYSMEMB
    UCXB --> AGENTB
    AGENTB --> APPB

    %% Coordination
    AGENTA <--> ETCDNET
    ETCDNET <--> AGENTB

    classDef application fill:#e1f5fe
    classDef agent fill:#e8f5e8
    classDef memory fill:#fff3e0
    classDef network fill:#fce4ec

    class APPA,APPB application
    class AGENTA,UCXA,AGENTB,UCXB agent
    class MEMA,SYSMEMA,MEMB,SYSMEMB memory
    class NETWORK,ETCDNET network
```

## Plugin Lifecycle Management

```mermaid
stateDiagram-v2
    [*] --> PluginDiscovery

    PluginDiscovery --> PluginLoading : Plugin found in directory
    PluginLoading --> PluginInitialization : dlopen() success
    PluginInitialization --> EngineCreation : nixl_plugin_init() success
    EngineCreation --> EngineReady : Backend engine created

    EngineReady --> MemoryRegistration : registerMem()
    EngineReady --> ConnectionSetup : connect()
    EngineReady --> TransferOperations : prepXfer()/postXfer()

    MemoryRegistration --> EngineReady : Success
    ConnectionSetup --> EngineReady : Success
    TransferOperations --> EngineReady : Success

    EngineReady --> EngineDestroy : Agent destruction
    EngineDestroy --> PluginCleanup : destroyEngine()
    PluginCleanup --> PluginUnload : nixl_plugin_fini()
    PluginUnload --> [*] : dlclose()

    PluginLoading --> PluginError : dlopen() failed
    PluginInitialization --> PluginError : nixl_plugin_init() failed
    EngineCreation --> PluginError : Engine creation failed
    PluginError --> [*] : Error cleanup
```

## Memory Management Architecture

```mermaid
graph TB
    subgraph "Application Memory"
        APPMEM[Application Buffers<br/>malloc/cudaMalloc]
    end

    subgraph "NIXL Memory Management"
        MEMDESC[Memory Descriptors<br/>nixlBlobDesc]
        MEMSEC[Memory Sections<br/>nixlMemorySection]
        REGLIST[Registration List<br/>reg_desc_list_t]
    end

    subgraph "Backend Memory Abstraction"
        BACKENDMD[Backend Metadata<br/>nixlBackendMD]
        PRIVMD[Private Metadata<br/>Local handles]
        PUBMD[Public Metadata<br/>Remote keys]
    end

    subgraph "UCX Memory Management"
        UCXMEM["UCX Memory Map<br/>ucp_mem_map()"]
        UCXRKEY["UCX Remote Key<br/>ucp_rkey_pack()"]
        UCXHANDLE["UCX Memory Handle<br/>ucp_mem_h"]
    end

    subgraph "Physical Memory"
        HOSTMEM[Host Memory<br/>DRAM]
        DEVMEM[Device Memory<br/>GPU VRAM]
        STORAGE[Storage<br/>NVMe/Network]
    end

    APPMEM --> MEMDESC
    MEMDESC --> MEMSEC
    MEMSEC --> REGLIST
    REGLIST --> BACKENDMD

    BACKENDMD --> PRIVMD
    BACKENDMD --> PUBMD

    PRIVMD --> UCXMEM
    UCXMEM --> UCXHANDLE
    UCXHANDLE --> UCXRKEY

    PUBMD --> UCXRKEY

    UCXMEM --> HOSTMEM
    UCXMEM --> DEVMEM
    UCXMEM --> STORAGE

    classDef appLayer fill:#e1f5fe
    classDef nixlLayer fill:#e8f5e8
    classDef backendLayer fill:#fff3e0
    classDef ucxLayer fill:#fce4ec
    classDef physicalLayer fill:#f5f5f5

    class APPMEM appLayer
    class MEMDESC,MEMSEC,REGLIST nixlLayer
    class BACKENDMD,PRIVMD,PUBMD backendLayer
    class UCXMEM,UCXRKEY,UCXHANDLE ucxLayer
    class HOSTMEM,DEVMEM,STORAGE physicalLayer
```

## Build System Integration

```mermaid
graph TB
    subgraph "Build Configuration"
        MESON[meson.build<br/>Root Configuration]
        OPTIONS[meson_options.txt<br/>Build Options]
        PKGCONFIG[nixl.pc.in<br/>Package Config]
    end

    subgraph "Source Organization"
        SRCCORE[src/core/<br/>Agent Implementation]
        SRCAPI[src/api/<br/>Public Headers]
        SRCPLUGINS[src/plugins/<br/>Plugin Sources]
        SRCBINDINGS[src/bindings/<br/>Language Bindings]
    end

    subgraph "Plugin Build System"
        PLUGINMESON[Plugin meson.build<br/>Conditional Building]
        UCXBUILD[UCX Plugin Build<br/>Dependency Check]
        SHAREDLIB[Shared Libraries<br/>libplugin_*.so]
    end

    subgraph "Installation Layout"
        LIBDIR[lib/<br/>Core Libraries]
        INCLUDEDIR[include/<br/>Headers]
        PLUGINDIR[lib/plugins/<br/>Plugin Libraries]
        BINDIR[bin/<br/>Tools/Examples]
    end

    MESON --> SRCCORE
    MESON --> SRCAPI
    MESON --> SRCPLUGINS
    MESON --> SRCBINDINGS
    OPTIONS --> PLUGINMESON

    SRCPLUGINS --> PLUGINMESON
    PLUGINMESON --> UCXBUILD
    UCXBUILD --> SHAREDLIB

    SHAREDLIB --> PLUGINDIR
    SRCCORE --> LIBDIR
    SRCAPI --> INCLUDEDIR
    SRCBINDINGS --> BINDIR

    classDef buildConfig fill:#e3f2fd
    classDef sourceCode fill:#e8f5e8
    classDef pluginBuild fill:#fff3e0
    classDef installation fill:#fce4ec

    class MESON,OPTIONS,PKGCONFIG buildConfig
    class SRCCORE,SRCAPI,SRCPLUGINS,SRCBINDINGS sourceCode
    class PLUGINMESON,UCXBUILD,SHAREDLIB pluginBuild
    class LIBDIR,INCLUDEDIR,PLUGINDIR,BINDIR installation
```

## Key Architectural Principles

### 1. **Modular Plugin Architecture**
- Core NIXL agent remains backend-agnostic
- Plugins implement standardized `nixlBackendEngine` interface
- Dynamic loading allows runtime backend selection
- Clean separation between core and backend-specific code

### 2. **Memory Type Abstraction**
- Unified descriptor interface for different memory types
- Backend-specific metadata handling (private/public)
- Efficient serialization for remote memory access
- Support for heterogeneous memory hierarchies

### 3. **Asynchronous Operations**
- Non-blocking transfer operations
- Progress-driven completion checking
- Optional progress thread support
- Efficient polling mechanisms

### 4. **Multi-Language Support**
- C++ core with stable ABI
- Python bindings via pybind11
- Rust bindings with FFI layer
- Consistent API across languages

### 5. **Scalable Connection Management**
- ETCD-based coordination for distributed systems
- Connection pooling and reuse
- Automatic endpoint management
- Support for multi-node deployments

This component architecture demonstrates how NIXL provides a flexible, high-performance data movement library that can efficiently handle various memory types and communication patterns while maintaining clean abstractions and extensibility.