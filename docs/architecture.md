# Architecture Documentation

## High-Level Architecture

Kamailio follows a modular, event-driven architecture designed for high-performance SIP message processing. The system consists of a lightweight core with dynamically loadable modules that extend functionality.

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Kamailio Core                            │
├─────────────────────────────────────────────────────────────┤
│  SIP Message Parser & State Machine                        │
│  ├─ Message Processing Pipeline                            │
│  ├─ Route Script Engine                                    │
│  └─ Transaction Management                                 │
├─────────────────────────────────────────────────────────────┤
│  Module Loading Framework                                   │
│  ├─ Dynamic Library Loader                                 │
│  ├─ Module API & Callbacks                                 │
│  └─ Configuration Interface                                │
├─────────────────────────────────────────────────────────────┤
│  Memory Management                                          │
│  ├─ Shared Memory Pool                                     │
│  ├─ Private Memory Pool                                    │
│  └─ Lock-free Data Structures                             │
├─────────────────────────────────────────────────────────────┤
│  Network I/O Layer                                         │
│  ├─ UDP/TCP/TLS Transport                                  │
│  ├─ WebSocket Support                                      │
│  └─ SCTP Transport                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  Loadable Modules                          │
├───────────────┬───────────────┬───────────────┬─────────────┤
│  Protocol     │  Database     │  Authentication│  Routing    │
│  Modules      │  Modules      │  Modules      │  Modules    │
│               │               │               │             │
│ • websocket   │ • db_mysql    │ • auth        │ • dispatcher│
│ • tls         │ • db_postgres │ • auth_radius │ • drouting  │
│ • xmpp        │ • db_redis    │ • auth_db     │ • lcr       │
│ • mqtt        │ • db_mongodb  │ • permissions │ • rtjson    │
└───────────────┴───────────────┴───────────────┴─────────────┘
```

## Core Components

### SIP Processing Engine
**Location**: `src/core/`

**Key Responsibilities**:
- SIP message parsing and validation
- Transaction state management  
- Route script execution
- Dialog tracking
- Timer management

**Critical Modules**:
- `parser/`: SIP message parsing
- `mem/`: Memory management
- `locking.h`: Synchronization primitives
- `route.c`: Routing logic
- `timer.c`: Timer subsystem

### Module Framework
**Location**: `src/core/mod_*`, `src/modules/`

**Architecture**:
- **Dynamic Loading**: Modules loaded as shared libraries (.so files)
- **API Interface**: Standardized module interface (`mod_exports`)
- **Callback System**: Event-driven hooks for message processing
- **Configuration Integration**: Module parameters via configuration file

**Module Categories**:
1. **Core Extensions** (`tm`, `sl`, `rr`) - Essential SIP functionality
2. **Protocol Handlers** (`tls`, `websocket`, `sctp`) - Transport protocols
3. **Database Connectors** (`db_*`) - Data persistence
4. **Application Logic** (`registrar`, `presence`, `dialog`) - SIP services
5. **Utility Modules** (`textops`, `xlog`, `cfgutils`) - Helper functions

### Memory Management
**Strategy**: Multi-tier memory architecture for performance

**Components**:
- **Shared Memory**: Process-shared data structures
- **Private Memory**: Per-process heap allocation  
- **Package Memory**: Fast allocation for temporary data
- **Lock-free Structures**: Hash tables, queues using atomic operations

**Considerations**:
- Memory pools prevent fragmentation
- Reference counting for shared structures
- Careful synchronization in multi-process mode

### Configuration System
**Location**: `src/core/cfg/`, main configuration files

**Architecture**:
- **Route Scripts**: Domain-specific language for SIP processing logic
- **Module Parameters**: Key-value configuration per module
- **Runtime Reconfiguration**: Hot-reload capabilities via RPC
- **Include System**: Modular configuration files

**Processing Flow**:
1. Configuration parsing at startup
2. Route script compilation to bytecode
3. Runtime execution with context switching
4. Dynamic parameter updates via MI/RPC

## Data Flow Architecture

### Message Processing Pipeline

```
Network Interface
        │
        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Transport     │───▶│  SIP Parser     │───▶│  Route Engine   │
│   Handler       │    │                 │    │                 │
│ • UDP/TCP/TLS   │    │ • Header parse  │    │ • Script exec   │
│ • WebSocket     │    │ • Body parse    │    │ • Module calls  │
│ • SCTP          │    │ • Validation    │    │ • State mgmt    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                                       │
                                                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Response      │◀───│   Transaction   │◀───│    Modules      │
│   Generation    │    │   Manager       │    │                 │
│                 │    │                 │    │ • Auth check    │
│ • Status codes  │    │ • TU handling   │    │ • DB operations │
│ • Header build  │    │ • Retrans logic │    │ • Routing logic │
│ • Send response │    │ • Timer mgmt    │    │ • Custom logic  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Multi-Process Architecture

**Process Model**:
- **Main Process**: Configuration, module loading, process management
- **Worker Processes**: SIP message processing (typically CPU count)
- **Timer Process**: Handles timeout events and maintenance
- **MI Process**: Management interface for runtime control

**Inter-Process Communication**:
- Shared memory for common data structures
- FIFO queues for message passing
- TCP/UDP for MI (Management Interface)
- Signals for process control

## Module Architecture

### Module Interface Definition
```c
struct module_exports {
    char* name;                    /* Module name */
    unsigned int dlflags;          /* Dynamic loading flags */
    cmd_export_t* cmds;           /* Exported functions */
    param_export_t* params;       /* Configuration parameters */
    rpc_export_t* rpc_methods;    /* RPC methods */
    pv_export_t* pv_items;       /* Pseudo-variables */
    proc_export_t* procs;         /* Child processes */
    init_function init_f;         /* Init function */
    response_function response_f;  /* Response processing */
    destroy_function destroy_f;   /* Cleanup function */
    child_init_function child_init_f; /* Child init */
};
```

### Critical Module Dependencies

**Core Dependencies** (tightly coupled):
- **tm** (Transaction Module): State management, retransmissions
- **sl** (Stateless): Stateless reply generation
- **rr** (Record Route): Route header management

**Common Integration Patterns**:
- Database modules → Application modules (registrar, presence)
- Authentication → Database → Permission checking
- Routing → Database → Load balancing logic

## Configuration Architecture

### Route Script Language
Domain-specific scripting language optimized for SIP processing:

**Core Constructs**:
- Route blocks: `route[NAME]`, `onreply_route`, `failure_route`
- Conditional logic: `if/else`, pattern matching
- Module function calls: `function(parameters)`
- Variable manipulation: pseudo-variables (`$var`, `$rU`, `$td`)

**Performance Considerations**:
- Compiled to bytecode for fast execution
- Minimal string operations
- Direct memory access for SIP headers
- Efficient regular expression engine

### Runtime Reconfiguration
**Capabilities**:
- Module parameter updates via RPC
- Routing table reloads (dispatcher, drouting)
- Certificate reloading (TLS module)
- Statistics and debugging control

**Constraints**:
- Core configuration requires restart
- Module loading/unloading not supported at runtime
- Memory layout changes require restart

## Scalability Architecture

### Horizontal Scaling Patterns
1. **Load Balancing**: Multiple Kamailio instances behind load balancer
2. **Functional Partitioning**: Separate proxy/registrar/presence servers  
3. **Geographic Distribution**: Regional deployments with federation
4. **Database Clustering**: Shared/partitioned database backends

### Performance Optimizations
- **Zero-copy**: Direct buffer manipulation where possible
- **Memory pools**: Pre-allocated memory for common structures
- **Hash tables**: Fast lookups for dialogs, transactions, users
- **Asynchronous operations**: Non-blocking database and HTTP operations

## Known Constraints and Design Trade-offs

### Tight Coupling Areas
1. **Core-TM Integration**: Transaction management deeply embedded
2. **Memory Management**: Shared structures limit some flexibility  
3. **Configuration Parsing**: Static configuration model
4. **Module Dependencies**: Some modules have circular dependencies

### Recommended Abstractions
1. **Database Layer**: Consistent connection pooling across DB modules
2. **Message Queue Interface**: Standardized async messaging
3. **Service Discovery**: Dynamic backend configuration
4. **Monitoring Interface**: Unified metrics collection
5. **Security Framework**: Centralized certificate and key management

### Architectural Debt
- Legacy code patterns in some older modules
- Mixed synchronous/asynchronous patterns
- Inconsistent error handling across modules
- Platform-specific code in core components