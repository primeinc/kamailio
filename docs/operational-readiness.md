# Operational Readiness Guide

## Build and Packaging

### Build System Overview

Kamailio supports multiple build systems to accommodate different deployment scenarios and preferences.

#### Make-based Build (Primary)
**Location**: Root `Makefile`, `src/Makefile.*`
**Usage**:
```bash
# Basic build
make cfg
make all

# Install
make install

# Module-specific build
make modules=modules/registrar modules/tm
```

**Configuration Options**:
- `FLAVOUR`: Build variant (kamailio, kamailio-basic, kamailio-all)
- `PREFIX`: Installation prefix (default: /usr/local)
- `CC`: Compiler selection
- `DEFS`: Custom compile-time definitions

#### CMake Build (Alternative)  
**Location**: `CMakeLists.txt`, `cmake/`
**Usage**:
```bash
mkdir build && cd build
cmake ..
make
make install
```

**Advantages**: Better IDE integration, Windows support, dependency management

### Package Targets

#### Distribution Packages
**DEB Packages** (`pkg/kamailio/deb/`):
- **kamailio**: Core package with essential modules
- **kamailio-mysql-modules**: MySQL database support
- **kamailio-postgres-modules**: PostgreSQL database support  
- **kamailio-extra-modules**: Additional protocol and utility modules
- **kamailio-presence-modules**: Presence and messaging modules

**RPM Packages** (`pkg/kamailio/obs/`):
- Similar module organization to DEB packages
- Support for CentOS, RHEL, SUSE, Fedora
- Separate packages for different database backends

#### Container Images
**Official Images**: Available via Docker Hub
```bash
# Basic Kamailio image
docker pull kamailio/kamailio:latest

# Development image with build tools
docker pull kamailio/kamailio-ci:latest
```

**Custom Build**:
```dockerfile
FROM debian:bullseye-slim
# Install dependencies and build Kamailio
# See .devcontainer/Dockerfile for reference
```

### Build Dependencies

#### Core Dependencies
- **Required**: gcc/clang, make, flex, bison, libssl-dev
- **Optional**: libxml2-dev, libcurl-dev, libpq-dev, libmysqlclient-dev

#### Module-specific Dependencies
- **Database**: Client libraries for target databases
- **TLS**: OpenSSL development headers
- **SCTP**: lksctp-tools-dev (Linux)
- **Radius**: libradiusclient-ng-dev or freeradius-client-dev
- **LDAP**: libldap2-dev
- **Lua**: liblua5.x-dev
- **Python**: python3-dev

## Deployment Environments

### Production Deployment Architecture

#### High Availability Setup
```
                     ┌─────────────────┐
                     │  Load Balancer  │ (HAProxy, F5, etc.)
                     │   (SIP-aware)   │
                     └─────────┬───────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼─────────┐ ┌────▼────┐ ┌─────────▼─────────┐
    │   Kamailio-1      │ │   ...   │ │   Kamailio-N      │
    │   (Primary SIP)   │ │         │ │   (Primary SIP)   │
    └─────────┬─────────┘ └─────────┘ └─────────┬─────────┘
              │                                 │
              └────────────────┬────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   Shared Database   │ (MySQL Cluster,
                    │     Cluster         │  PostgreSQL HA)
                    └─────────────────────┘
```

#### Geographic Distribution
- **Regional Deployments**: Multiple Kamailio clusters per region
- **Data Replication**: Database replication across regions  
- **DNS-based Routing**: Geographic load balancing
- **Anycast Addressing**: IP-level geographic routing

### Environment Types

#### Development Environment
- **Single-node**: All components on one machine
- **Database**: Local SQLite or containerized MySQL/PostgreSQL
- **Configuration**: Development-friendly defaults
- **Logging**: Verbose debug logging enabled

#### Staging Environment
- **Multi-node**: Production-like topology
- **Database**: Production-like cluster setup
- **Configuration**: Production configuration with staging overrides
- **Testing**: Full integration and performance testing

#### Production Environment
- **Load Balanced**: Multiple Kamailio instances  
- **Database HA**: Clustered database with failover
- **Monitoring**: Full observability stack
- **Security**: Hardened configuration and network isolation

## Configuration Management

### Configuration Structure

#### Main Configuration File
**Location**: `/etc/kamailio/kamailio.cfg`
**Structure**:
```
#!KAMAILIO
# Global parameters
debug=2
log_stderror=no
log_facility=LOG_LOCAL0

# Module loading
loadmodule "tm.so"
loadmodule "sl.so" 
loadmodule "rr.so"

# Module parameters
modparam("tm", "fr_timer", 30000)
modparam("rr", "enable_full_lr", 1)

# Route blocks
route {
    # Main routing logic
}

route[LOCATION] {
    # Location-specific routing
}
```

#### Modular Configuration
**Approach**: Split configuration into functional modules
```
/etc/kamailio/
├── kamailio.cfg          # Main configuration
├── kamailio-local.cfg    # Local overrides
├── modules/              # Module-specific configuration
│   ├── routing.cfg       # Routing logic
│   ├── authentication.cfg # Auth configuration
│   └── database.cfg      # Database settings
└── tls/                  # TLS certificates and keys
    ├── kamailio.pem
    └── kamailio.key
```

### Environment-Specific Configuration

#### Configuration Templates
Use templating systems (Jinja2, Helm, etc.) for environment-specific values:
```bash
# Database connection template
{{#if mysql_enabled}}
modparam("db_mysql", "ping_interval", {{mysql_ping_interval}})
{{/if}}

# Environment-specific values
listen=udp:{{kamailio_ip}}:{{kamailio_port}}
```

#### Secrets Management
- **Environment Variables**: Database passwords, API keys
- **External Secret Stores**: HashiCorp Vault, Kubernetes secrets
- **File-based**: Restricted file permissions (0600)

```bash
# Environment variable usage
modparam("db_mysql", "db_url", 
    "mysql://kamailio:${DB_PASSWORD}@${DB_HOST}/kamailio")
```

### Configuration Validation

#### Pre-deployment Validation
```bash
# Syntax validation
kamailio -c -f /etc/kamailio/kamailio.cfg

# Module loading validation  
kamailio -M -f /etc/kamailio/kamailio.cfg

# Database connectivity test
kamctl db ping
```

#### Automated Validation Pipeline
1. **Syntax Check**: Configuration file parsing
2. **Module Validation**: Module loading and parameter checking
3. **Integration Test**: Database and external service connectivity
4. **Security Scan**: Configuration security review

## Upgrade and Migration

### Upgrade Strategy

#### Rolling Upgrade Process
1. **Preparation**
   - Database schema migration (if required)
   - Configuration file updates
   - Module compatibility verification

2. **Execution**
   - Remove nodes from load balancer rotation
   - Stop Kamailio service gracefully  
   - Update packages/binaries
   - Update configuration
   - Start service and validate
   - Add back to load balancer rotation

3. **Validation**
   - Service health checks
   - SIP functionality validation
   - Performance baseline verification
   - Monitoring alert validation

#### Database Migration
```bash
# Backup before migration
mysqldump kamailio > kamailio_backup_$(date +%Y%m%d).sql

# Schema migration
kamdbctl migrate

# Data migration (if required)
mysql kamailio < migration_script.sql
```

### Version Compatibility

#### Major Version Upgrades
- **Configuration Changes**: Review changelog for breaking changes
- **Module Compatibility**: Verify module API compatibility
- **Database Schema**: Run migration scripts
- **Testing**: Full integration testing required

#### Minor Version Upgrades  
- **Configuration**: Usually backward compatible
- **Hot Fixes**: Security and bug fixes
- **Testing**: Smoke tests sufficient

### Rollback Procedures

#### Service Rollback
1. **Immediate**: Revert to previous package version
2. **Configuration**: Restore previous configuration files
3. **Database**: Restore database backup (if schema changed)
4. **Validation**: Verify service functionality

#### Rollback Triggers
- **Service Failure**: Service won't start or crashes
- **Performance Degradation**: Response time or throughput issues
- **Functional Issues**: SIP protocol violations or feature regressions

## Health and Readiness Checks

### Service Health Monitoring

#### Process-level Health
```bash
# Service status
systemctl status kamailio

# Process monitoring
ps aux | grep kamailio

# Port availability
netstat -tlnp | grep :5060
ss -tlnp | grep :5060
```

#### Application-level Health
```bash
# Management interface check
kamctl stats get core:uptime

# SIP OPTIONS ping
kamctl monitor

# Database connectivity
kamctl db ping
```

### Health Check Endpoints

#### HTTP Health Checks (via xhttp module)
```
GET /health
Response: 200 OK
{
    "status": "healthy",
    "uptime": "72h34m12s", 
    "active_dialogs": 1250,
    "memory_usage": "45%",
    "database": "connected"
}
```

#### SIP-based Health Checks
```
OPTIONS sip:health@kamailio.example.com SIP/2.0
Response: 200 OK
```

### Readiness Checks

#### Startup Readiness
1. **Configuration Loaded**: All modules loaded successfully
2. **Database Connected**: Database connectivity established
3. **Ports Bound**: All configured network interfaces listening
4. **Routes Compiled**: Route scripts compiled and validated

#### Runtime Readiness
1. **Memory Available**: Sufficient memory for operations
2. **Database Responsive**: Database queries completing successfully
3. **Network Connectivity**: External dependencies reachable
4. **Performance Threshold**: Response times within SLA

### Monitoring Integration

#### Kubernetes Probes
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:  
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### Traditional Monitoring
- **Nagios/Icinga**: Custom check scripts
- **Zabbix**: Template-based monitoring
- **Prometheus**: Metrics-based alerting
- **Datadog**: APM and infrastructure monitoring

## Operational Procedures

### Service Management

#### Systemd Service Control
```bash
# Service operations
systemctl start kamailio
systemctl stop kamailio  
systemctl restart kamailio
systemctl reload kamailio

# Status and logs
systemctl status kamailio
journalctl -u kamailio -f

# Enable/disable automatic startup
systemctl enable kamailio
systemctl disable kamailio
```

#### Graceful Shutdown
```bash
# Graceful shutdown with call completion
kamctl monitor
kamctl stop

# Force shutdown (emergency)
systemctl kill --signal=SIGKILL kamailio
```

### Troubleshooting Procedures

#### Common Issues

1. **Service Won't Start**
   - Check configuration syntax: `kamailio -c`
   - Verify module loading: `kamailio -M`
   - Check port conflicts: `netstat -tlnp`
   - Review error logs: `journalctl -u kamailio`

2. **Database Connectivity**
   - Test connection: `kamctl db ping`
   - Check credentials and permissions
   - Verify network connectivity
   - Review database server logs

3. **Memory Issues**
   - Monitor memory usage: `kamctl stats get shmem:`
   - Check for memory leaks: `kamctl stats get core:`
   - Review process memory: `ps aux | grep kamailio`
   - Analyze core dumps if available

4. **Performance Issues**
   - Check system resources: CPU, memory, disk I/O
   - Monitor SIP message rates
   - Review database query performance
   - Analyze network latency

#### Diagnostic Tools
```bash
# Real-time statistics
kamctl stats

# Active dialogs
kamctl ul show

# Transaction monitoring  
kamctl tm stats

# Memory analysis
kamctl pkg.stats

# Network capture
tcpdump -i any -s 0 -w kamailio.pcap port 5060
```

### Performance Tuning

#### System-level Tuning
```bash
# Network buffers
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf

# File descriptors
echo 'fs.file-max = 1000000' >> /etc/sysctl.conf

# Connection tracking
echo 'net.netfilter.nf_conntrack_max = 1000000' >> /etc/sysctl.conf
```

#### Kamailio-specific Tuning
```bash
# Process configuration
children=16              # Match CPU cores
tcp_children=8           # For TCP workloads

# Memory configuration  
shm_mem=256             # Shared memory (MB)
pkg_mem=16              # Private memory per process (MB)

# Network configuration
tcp_connection_lifetime=300  # TCP connection timeout
tcp_max_connections=4096     # Maximum TCP connections
```

This operational readiness guide should be customized based on specific deployment requirements, infrastructure constraints, and organizational procedures.