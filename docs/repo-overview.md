# Repository Overview

## Project Summary

**Kamailio** is an open source SIP (Session Initiation Protocol) signaling server, originally started in 2001 as SIP Express Router (SER) by Fraunhofer Fokus. The project is designed for scalability, targeting large deployments for IP telephony operators and carriers, but is also suitable for enterprise and personal VoIP, Instant Messaging, and Presence services.

## Technical Stack

### Primary Languages
- **C/C++**: Core implementation (1,391 .c files, 1,363 .h files)
- **Shell Scripts**: Build and utility scripts (109 .sh files)  
- **Python**: Utility scripts and tools (20 .py files)
- **Lua**: Scripting support (1 .lua file)

### Build System
- **Primary**: GNU Make-based build system (`Makefile`, `src/Makefile.*`)
- **Alternative**: CMake support (`CMakeLists.txt`, `cmake/`)
- **Root Makefile**: Forwards commands to `src/` subfolder

### Package Management
- **System Packages**: RPM/DEB packaging support in `pkg/`
- **Dependencies**: External libraries integrated via Makefiles

### Container Support
- **Docker**: Multiple Dockerfiles present
  - `.devcontainer/Dockerfile` (development container)
  - `src/modules/rtp_media_server/docker/Dockerfile` (RTP media server)

## Repository Structure

```
kamailio/
├── .github/                    # GitHub workflows and templates
│   ├── workflows/             # CI/CD pipelines
│   ├── CONTRIBUTING.md        # Contribution guidelines
│   └── dependabot.yml         # Dependency updates
├── cmake/                     # CMake build configuration
├── doc/                       # Project documentation
├── etc/                       # Configuration examples
├── misc/                      # Miscellaneous utilities
├── pkg/                       # Packaging configurations
│   ├── kamailio/             # Main package specs
│   └── [distro-specific]/    # Distribution packages
├── src/                       # Source code
│   ├── core/                 # Core SIP server implementation
│   ├── lib/                  # Shared libraries
│   ├── modules/              # Loadable modules (200+ modules)
│   └── main.c                # Application entry point
├── test/                     # Test suites
└── utils/                    # Utility scripts and tools
```

### Key Components

#### Core Server (`src/core/`)
- SIP message processing engine
- Memory management and data structures
- Configuration parser and runtime
- Module loading framework

#### Modules (`src/modules/`)
Over 200 loadable modules providing functionality including:
- **Authentication**: auth, auth_db, auth_radius, auth_diameter
- **Database**: db_mysql, db_postgres, db_redis, db_mongodb, etc.
- **Protocols**: websocket, tls, sctp, xmpp
- **Routing**: dispatcher, drouting, carrierroute
- **Presence**: presence, presence_xml, pua
- **Media**: rtpproxy, rtpengine, mediaproxy
- **Scripting**: app_lua, app_python, app_perl, app_ruby
- **Monitoring**: snmpstats, statsd, xhttp_prom

#### Configuration
- Text-based configuration files
- Modular configuration approach
- Runtime configuration via RPC

## License Information

- **Primary License**: GPL v2
- **Mixed Licensing**: Some components under BSD license
- **New Contributions**: Core and main modules require BSD license
- **GPL-OpenSSL Exception**: Required for GPL contributions

## Release and Version Management

- **Default Branch**: Detected as current working branch
- **Release Process**: Git tags for version releases
- **Packaging**: Multi-distro packaging (DEB, RPM)

## Integration Points

### External Services
- **Databases**: MySQL, PostgreSQL, Redis, MongoDB, Cassandra
- **Message Queues**: RabbitMQ, Kafka, NATS
- **Monitoring**: InfluxDB, Prometheus, SNMP
- **Authentication**: RADIUS, Diameter, LDAP
- **Security**: TLS/SSL, STIR/SHAKEN, IPSec

### Runtime Dependencies
- Core C libraries (libc, libssl, libcrypto)
- Optional module-specific dependencies
- Database client libraries
- Protocol-specific libraries (SCTP, WebSocket, etc.)

## Current Workflow Status

The repository includes active GitHub Actions workflows:
- **main.yml**: Primary CI/CD pipeline
- **codeql.yml**: Security analysis
- **cmake_build.yml**: CMake-based builds  
- **alpine.yml**: Alpine Linux builds
- **rpm.yml**: RPM packaging
- **cifuzz.yml**: Fuzzing tests

## Development Environment

- **Dev Container**: `.devcontainer/` configuration for consistent development
- **Pre-commit**: Code quality hooks (`.pre-commit-config.yaml`)
- **Code Formatting**: Clang-format configuration (`.clang-format`)
- **LGTM**: Legacy code quality checks (`.lgtm.yml`)