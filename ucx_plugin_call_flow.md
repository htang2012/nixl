# NIXL UCX Plugin Call Flow Diagram

This document illustrates the complete call flow for how NIXL invokes the UCX plugin, from initialization to data transfer operations.

## 1. Plugin Discovery and Loading

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant PM as nixlPluginManager
    participant DL as Dynamic Loader
    participant UCX_P as UCX Plugin (.so)
    participant UCX_E as nixlUcxEngine

    App->>Agent: nixlAgent(name, config)
    Agent->>Agent: Constructor initialization
    
    App->>Agent: createBackend("UCX", params)
    Agent->>PM: loadPlugin("UCX")
    PM->>DL: dlopen("libplugin_UCX.so")
    DL-->>PM: library handle
    PM->>DL: dlsym(handle, "nixl_plugin_init")
    DL-->>PM: function pointer
    PM->>UCX_P: nixl_plugin_init()
    UCX_P-->>PM: nixlBackendPlugin* (plugin structure)
    PM-->>Agent: nixlPluginHandle*
    
    Agent->>PM: createEngine(&init_params)
    PM->>UCX_P: plugin->create_engine(&init_params)
    UCX_P->>UCX_E: nixlUcxEngine::create(init_params)
    UCX_E->>UCX_E: Initialize UCX context, workers
    UCX_E-->>UCX_P: nixlUcxEngine*
    UCX_P-->>PM: nixlBackendEngine*
    PM-->>Agent: nixlBackendEngine*
    
    Agent->>Agent: Store backend in backendEngines["UCX"]
    Agent-->>App: nixlBackendH* (success)
```

## 2. Memory Registration Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine
    participant UCX_API as UCX API

    App->>Agent: nixl_reg_desc_create(memory_desc)
    Agent->>Agent: Select UCX backend for memory type
    Agent->>UCX_E: registerMem(blob_desc, mem_type)
    
    UCX_E->>UCX_API: ucp_mem_map(context, &params)
    UCX_API-->>UCX_E: ucp_mem_h (memory handle)
    
    UCX_E->>UCX_API: ucp_rkey_pack(context, mem_h, &rkey_buffer)
    UCX_API-->>UCX_E: rkey_buffer (serialized remote key)
    
    UCX_E->>UCX_E: Create nixlUcxPrivateMetadata
    UCX_E->>UCX_E: Store rkey_buffer in metadata
    UCX_E-->>Agent: nixlBackendMD* (metadata handle)
    Agent-->>App: nixl_reg_desc_h* (registration handle)
```

## 3. Connection Establishment Flow

```mermaid
sequenceDiagram
    participant Agent1 as nixlAgent (Node 1)
    participant UCX_E1 as nixlUcxEngine (Node 1)
    participant UCX_API1 as UCX API (Node 1)
    participant Network as Network
    participant UCX_API2 as UCX API (Node 2)
    participant UCX_E2 as nixlUcxEngine (Node 2)
    participant Agent2 as nixlAgent (Node 2)

    Agent1->>UCX_E1: getConnInfo()
    UCX_E1->>UCX_API1: ucp_worker_get_address()
    UCX_API1-->>UCX_E1: worker_address
    UCX_E1-->>Agent1: serialized worker address
    
    Agent1->>Network: Share connection info via ETCD/metadata
    Network->>Agent2: Receive connection info
    
    Agent2->>UCX_E2: loadRemoteConnInfo(remote_agent, conn_info)
    UCX_E2->>UCX_E2: Deserialize worker address
    UCX_E2->>UCX_E2: Store in remoteConnMap[remote_agent]
    
    Agent1->>UCX_E1: connect(remote_agent)
    UCX_E1->>UCX_API1: ucp_ep_create(worker, &ep_params)
    UCX_API1-->>UCX_E1: ucp_ep_h (endpoint)
    UCX_E1->>UCX_E1: Store endpoint in connection
    UCX_E1-->>Agent1: NIXL_SUCCESS
```

## 4. Transfer Descriptor Creation Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine

    App->>Agent: nixl_xfer_desc_create(local_desc, remote_desc, remote_agent, op)
    Agent->>Agent: Validate descriptors and select UCX backend
    Agent->>UCX_E: prepXfer(operation, local_mds, remote_mds, remote_agent)
    
    UCX_E->>UCX_E: Get connection for remote_agent
    UCX_E->>UCX_E: Extract local memory handles
    UCX_E->>UCX_E: Deserialize remote rkeys
    UCX_E->>UCX_E: Create nixlUcxReq transfer request
    UCX_E->>UCX_E: Setup transfer parameters (src, dst, size)
    
    UCX_E-->>Agent: nixlBackendReqH* (transfer handle)
    Agent-->>App: nixl_xfer_desc_h* (transfer descriptor)
```

## 5. Transfer Execution Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine
    participant UCX_API as UCX API
    participant UCX_Worker as UCX Worker Thread

    App->>Agent: nixl_xfer_submit(xfer_desc)
    Agent->>UCX_E: postXfer(operation, local_mds, remote_mds, remote_agent, handle)
    
    alt WRITE Operation
        UCX_E->>UCX_API: ucp_put_nb(ep, local_addr, size, remote_addr, rkey, callback)
    else READ Operation
        UCX_E->>UCX_API: ucp_get_nb(ep, local_addr, size, remote_addr, rkey, callback)
    end
    
    UCX_API-->>UCX_E: ucs_status_ptr_t (request status)
    UCX_E->>UCX_E: Store request in nixlUcxReq
    UCX_E-->>Agent: NIXL_SUCCESS
    Agent-->>App: NIXL_SUCCESS
    
    loop Background Processing
        UCX_Worker->>UCX_API: ucp_worker_progress(worker)
        UCX_API->>UCX_API: Process network operations
        UCX_API->>UCX_E: completion_callback(request, status)
        UCX_E->>UCX_E: Update transfer status
    end
```

## 6. Transfer Status Polling Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine
    participant UCX_API as UCX API

    loop Status Checking
        App->>Agent: nixl_xfer_query(xfer_desc)
        Agent->>UCX_E: checkXfer(handle)
        
        UCX_E->>UCX_E: Get nixlUcxReq from handle
        
        alt Request Complete
            UCX_E->>UCX_E: Check completion status
            UCX_E-->>Agent: NIXL_SUCCESS
        else Request In Progress
            UCX_E->>UCX_API: ucp_worker_progress(worker)
            UCX_API-->>UCX_E: progress_count
            UCX_E-->>Agent: NIXL_IN_PROGRESS
        else Request Error
            UCX_E-->>Agent: NIXL_ERROR
        end
        
        Agent-->>App: nixl_status_t
    end
```

## 7. Cleanup and Resource Management Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine
    participant UCX_API as UCX API
    participant PM as nixlPluginManager

    App->>Agent: nixl_xfer_desc_destroy(xfer_desc)
    Agent->>UCX_E: releaseReqH(handle)
    UCX_E->>UCX_E: Cleanup nixlUcxReq
    UCX_E-->>Agent: NIXL_SUCCESS

    App->>Agent: nixl_reg_desc_destroy(reg_desc)
    Agent->>UCX_E: deregisterMem(metadata)
    UCX_E->>UCX_API: ucp_mem_unmap(context, mem_h)
    UCX_E->>UCX_E: Cleanup nixlUcxPrivateMetadata
    UCX_E-->>Agent: NIXL_SUCCESS

    App->>Agent: ~nixlAgent() [destructor]
    Agent->>UCX_E: disconnect(remote_agents...)
    UCX_E->>UCX_API: ucp_ep_close_nb(ep, UCP_EP_CLOSE_MODE_FLUSH)
    UCX_E->>UCX_API: ucp_worker_destroy(worker)
    UCX_E->>UCX_API: ucp_cleanup(context)
    
    Agent->>PM: destroyEngine(ucx_engine)
    PM->>UCX_E: delete ucx_engine
    UCX_E->>UCX_E: ~nixlUcxEngine() [destructor]
```

## 8. Error Handling Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant Agent as nixlAgent
    participant UCX_E as nixlUcxEngine
    participant UCX_API as UCX API

    App->>Agent: Any UCX operation
    Agent->>UCX_E: Backend operation
    UCX_E->>UCX_API: UCX API call
    
    alt UCX Success
        UCX_API-->>UCX_E: UCS_OK
        UCX_E-->>Agent: NIXL_SUCCESS
        Agent-->>App: NIXL_SUCCESS
    else UCX Error
        UCX_API-->>UCX_E: UCS_ERR_*
        UCX_E->>UCX_E: Log error with NIXL_ERROR
        UCX_E-->>Agent: NIXL_ERR_BACKEND
        Agent-->>App: NIXL_ERR_BACKEND
    else UCX Exception
        UCX_E->>UCX_E: catch std::exception
        UCX_E->>UCX_E: Log error with NIXL_ERROR
        UCX_E-->>Agent: NIXL_ERR_BACKEND
        Agent-->>App: NIXL_ERR_BACKEND
    end
```

## Key Components Summary

### Core Classes and Their Roles:

1. **nixlAgent** (`src/core/nixl_agent.cpp`): Main orchestrator that manages backends and routes operations
2. **nixlPluginManager** (`src/core/nixl_plugin_manager.cpp`): Handles dynamic loading of plugin shared libraries  
3. **nixlUcxEngine** (`src/plugins/ucx/ucx_backend.cpp`): UCX-specific backend implementation
4. **nixlBackendPlugin** (`src/api/cpp/backend/backend_plugin.h`): Standard plugin interface structure
5. **UCX Plugin** (`src/plugins/ucx/ucx_plugin.cpp`): Plugin entry point and factory functions

### Key Data Structures:

- **nixlBackendInitParams**: Configuration passed to backend during creation
- **nixlUcxPrivateMetadata**: Local memory registration data with UCX handles
- **nixlUcxPublicMetadata**: Remote memory access data with UCX rkeys  
- **nixlUcxConnection**: Connection state and UCX endpoints for remote agents
- **nixlUcxReq**: Transfer request tracking UCX operations

This architecture provides a clean separation between the NIXL core and UCX-specific implementation, allowing for easy extensibility with other communication backends.