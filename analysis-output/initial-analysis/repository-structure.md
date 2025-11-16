# Temporal Server Repository Structure

**Analysis Date:** 2025-11-16
**Commit SHA:** `0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3`

---

## Overview

This document provides a comprehensive breakdown of the Temporal server repository structure, explaining the purpose of each major directory and key files.

---

## Top-Level Directory Structure

```
temporal/
├── api/                      # Generated protocol buffer code
├── chasm/                    # CHASM state machine library
├── client/                   # Inter-service gRPC clients
├── cmd/                      # Command-line entry points
├── common/                   # Shared utilities and libraries (~70 packages)
├── components/               # Pluggable components
├── config/                   # Configuration templates
├── develop/                  # Development environment (docker-compose)
├── docs/                     # Architecture and development documentation
├── proto/                    # Protocol buffer definitions
├── schema/                   # Database schemas
├── service/                  # Core services (Frontend, History, Matching, Worker)
├── temporal/                 # Server bootstrap and dependency injection
├── temporaltest/             # External test utilities
├── tests/                    # Functional and integration tests
├── tools/                    # Development and operational tools
├── .github/                  # GitHub workflows and CI/CD
├── go.mod, go.sum           # Go module dependencies
├── Makefile                  # Build system (200+ targets)
├── README.md                 # Project overview
├── CONTRIBUTING.md           # Contribution guidelines
├── AGENTS.md                 # Best practices for AI agents
└── LICENSE                   # MIT License
```

---

## Core Directories

### `/api` - Generated Protocol Buffer Code

**Purpose:** Auto-generated Go code from Protocol Buffer definitions (from [temporalio/api](https://github.com/temporalio/api) repo)

**Structure:**
```
api/
├── adminservice/v1/          # Admin service APIs
├── historyservice/v1/        # History service internal APIs
├── matchingservice/v1/       # Matching service internal APIs
├── persistence/v1/           # Persistence layer types
├── enums/v1/                 # Enumeration types
├── common/v1/                # Common message types
├── workflowservice/v1/       # User-facing workflow APIs
├── operatorservice/v1/       # Operator/admin operations
└── ...                       # 30+ subdirectories
```

**Key Files:**
- `service.pb.go` - Service definitions
- `message.pb.go` - Message types
- `request_response.pb.go` - Request/response messages

**DO NOT EDIT:** Files are generated via `make proto` from upstream API repository

---

### `/chasm` - Coordinated Heterogeneous State Machines

**Purpose:** Library for building complex state machines with coordinated transitions

**Usage:**
- Scheduler workflows
- Advanced workflow patterns
- State machine-based components

**Key Concepts:**
- State definitions
- Transition rules
- Event handling
- Coordination between multiple state machines

**Entry Point:** [`chasm/state_machine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/chasm/state_machine.go)

---

### `/client` - Inter-Service gRPC Clients

**Purpose:** Client implementations for internal service-to-service communication

**Structure:**
```
client/
├── admin/                    # Admin service client
├── frontend/                 # Frontend service client
├── history/                  # History service client
├── matching/                 # Matching service client
├── factory.go                # Client factory for dependency injection
└── clientbean.go             # Client bean for Fx DI
```

**Key Responsibilities:**
- gRPC connection management
- Request routing and retries
- Service discovery via membership
- Load balancing

**Example:**
```go
// Create history client
historyClient := s.clientFactory.NewHistoryClient()
response, err := historyClient.StartWorkflowExecution(ctx, request)
```

---

### `/cmd` - Command-Line Entry Points

**Purpose:** Main executable entry points

**Structure:**
```
cmd/
├── server/                   # Main Temporal server
│   ├── main.go              # Entry point for temporal-server binary
│   └── config.go            # Config loading
└── tools/                    # Build and operational tools
    ├── cassandra/           # Cassandra schema tool
    ├── sql/                 # SQL schema tool
    ├── elasticsearch/       # Elasticsearch setup tool
    ├── copyright/           # License header checker
    ├── gendocs/             # Documentation generator
    ├── genrequestid/        # Request ID generator
    └── ...
```

**Main Server Entry Point:**
[`cmd/server/main.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/cmd/server/main.go#L61)

```go
func main() {
    app := buildCLI()
    // Commands: start, validate-dynamic-config, render-config
    _ = app.Run(os.Args)
}
```

**CLI Commands:**
- `start` - Start Temporal server
- `--config` - Config file path
- `--env` - Environment (development, production)
- `--service` - Service to run (all, frontend, history, matching, worker)
- `--allow-no-auth` - Disable authentication (dev only)

---

### `/common` - Shared Utilities (~70 packages)

**Purpose:** Reusable libraries and utilities used across all services

**Major Packages:**

#### Persistence
```
common/persistence/
├── client/                   # Persistence client with rate limiting
├── cassandra/                # Cassandra implementation
├── sql/                      # SQL implementations (MySQL, PostgreSQL, SQLite)
├── nosql/                    # NoSQL abstraction
├── visibility/               # Visibility store (Elasticsearch, SQL)
├── serialization/            # Protobuf serialization
└── tests/                    # Persistence test suite
```

**Key Files:**
- [`persistence/interfaces.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/common/persistence/interfaces.go) - Core persistence interfaces
- `persistence/client/factory.go` - Factory for creating persistence clients

#### Metrics & Telemetry
```
common/metrics/
├── config.go                 # Metrics configuration
├── tally_metrics_handler.go  # Tally framework implementation
├── otel_metrics_handler.go   # OpenTelemetry implementation
├── defs.go                   # Metric definitions
└── tags.go                   # Metric tags
```

#### Dynamic Configuration
```
common/dynamicconfig/
├── file_based_client.go      # File-based dynamic config
├── memory_client.go          # In-memory client (testing)
├── config.go                 # Config structures
└── constants.go              # Config key constants
```

**Purpose:** Runtime configuration changes without server restart

#### Membership & Clustering
```
common/membership/
├── ringpop/                  # Ringpop-based membership
├── interfaces.go             # Membership interfaces
└── monitor.go                # Membership monitoring
```

**Purpose:** Service discovery and cluster membership using Ringpop

#### Other Critical Packages
| Package | Purpose |
|---------|---------|
| `common/log/` | Structured logging (Zap-based) |
| `common/rpc/` | gRPC utilities and interceptors |
| `common/namespace/` | Namespace management and caching |
| `common/clock/` | Time abstraction (mockable for tests) |
| `common/cache/` | LRU and size-based caching |
| `common/quotas/` | Rate limiting and quotas |
| `common/authorization/` | RBAC and authorization |
| `common/archiver/` | Workflow history archival |
| `common/searchattribute/` | Custom search attributes |
| `common/primitives/` | Primitive types and utilities |
| `common/backoff/` | Retry backoff strategies |
| `common/headers/` | gRPC metadata and headers |
| `common/locks/` | Distributed locking primitives |
| `common/tasks/` | Task management abstractions |

---

### `/components` - Pluggable Components

**Purpose:** Optional, pluggable functionality built on top of core services

**Structure:**
```
components/
├── callbacks/                # Nexus callback handling
├── nexusoperations/          # Nexus operation execution
└── dummy/                    # Placeholder/test components
```

**Design Pattern:** Component-based architecture for extensibility

---

### `/config` - Configuration Templates

**Purpose:** YAML configuration templates for different environments

**Key Files:**

| File | Purpose |
|------|---------|
| `development.yaml` | SQLite in-memory (default for local dev) |
| `development-sqlite-file.yaml` | SQLite with persistent file |
| `development-cass.yaml` | Cassandra-based development |
| `development-mysql8.yaml` | MySQL 8.0+ development |
| `development-postgres12.yaml` | PostgreSQL 12+ development |
| `docker.yaml` | Docker container configuration |
| `dynamicconfig/*.yaml` | Dynamic config samples |

**Example Configuration Sections:**
```yaml
log:
  stdout: true
  level: info

persistence:
  numHistoryShards: 4
  defaultStore: sqlite-default
  datastores:
    sqlite-default: { ... }

services:
  frontend:
    rpc:
      grpcPort: 7233
      httpPort: 7243
```

---

### `/service` - Core Services

**Purpose:** The four main services that comprise the Temporal server

#### Structure
```
service/
├── frontend/                 # User-facing API service
├── history/                  # Workflow execution engine
├── matching/                 # Task queue management
├── worker/                   # Background processing
└── fx.go                     # Shared service utilities
```

---

#### `/service/frontend` - User-Facing API

**Responsibilities:**
- Handle all user API calls (StartWorkflowExecution, SignalWorkflow, etc.)
- Admin operations (namespace management, cluster operations)
- Request validation and routing
- Rate limiting and quota enforcement
- HTTP/REST gateway
- Nexus endpoints

**Key Files:**
- [`handler.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/frontend/handler.go) - Main workflow API handlers
- `admin_handler.go` - Admin API handlers
- `namespace_handler.go` - Namespace operations
- `nexus_handler.go` - Nexus HTTP endpoints
- `configs/quotas.go` - Rate limiting configuration

**API Priority Levels:**
```
P0: System APIs (health checks, describe)
P1: Critical user operations (start workflow, activity complete)
P2: State changes (terminate, cancel)
P3: Queries (describe execution, get workflow)
P4-5: Polls and info (poll task queue, OpenAPI)
```

---

#### `/service/history` - Workflow Execution Engine

**Responsibilities:**
- Maintain workflow execution state
- Append-only event history
- Shard ownership and management
- Transfer task processing (schedule activities, child workflows)
- Timer task processing (workflow timeouts, activity timeouts)
- Cross-datacenter replication
- Visibility updates

**Key Components:**
```
service/history/
├── handler.go                # gRPC API handlers
├── historyEngine.go          # Core workflow execution logic
├── shard/                    # Shard management
│   ├── controller.go         # Shard ownership
│   └── context_impl.go       # Shard context
├── workflow/                 # Workflow state management
│   ├── mutable_state.go      # In-memory workflow state
│   ├── context.go            # Workflow context
│   └── cache/                # Workflow cache
├── queues/                   # Task queue processors
│   ├── queue.go              # Base queue
│   ├── executable_task.go    # Task execution
│   └── scheduler.go          # Task scheduling
├── replication/              # Cross-DC replication
├── events/                   # Event generation and validation
└── ndc/                      # Multi-datacenter consistency
```

**Critical Files:**
- [`historyEngine.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/historyEngine.go) - Core engine
- [`workflow/mutable_state_impl.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/history/workflow/mutable_state_impl.go) - Workflow state machine

**Sharding Architecture:**
- Each shard manages subset of workflow executions
- Shard count fixed at cluster creation
- Workflows hashed to shard by WorkflowID
- Each shard has RangeID for fencing (prevents split-brain)

---

#### `/service/matching` - Task Queue Management

**Responsibilities:**
- Maintain in-memory task queues
- Handle worker polls (PollWorkflowTaskQueue, PollActivityTaskQueue)
- Task dispatching and load balancing
- Sync matching (immediate dispatch to waiting worker)
- Task queue backlog management
- Deployment-aware routing (worker versioning)

**Key Components:**
```
service/matching/
├── handler.go                # gRPC API handlers
├── matcher.go                # Task matching logic
├── db.go                     # Task queue persistence
├── taskreader.go             # Read tasks from queue
├── taskwriter.go             # Write tasks to queue
├── physical_task_queue_manager.go  # Task queue lifecycle
├── fair/                     # Fair task distribution
└── versioning/               # Worker versioning support
```

**Critical Files:**
- [`matcher.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/matcher.go) - Sync match algorithm
- [`physical_task_queue_manager.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/physical_task_queue_manager.go) - Queue management

**Task Queue Architecture:**
- Task queues can be partitioned for parallelism
- Sync match: worker waiting → immediate dispatch
- Async match: task persisted → worker polls later
- Backlog tracking for monitoring

**Documentation:**
- [`fairness.md`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/service/matching/fairness.md) - Multi-partition fairness algorithm

---

#### `/service/worker` - Background Processing

**Responsibilities:**
- Cross-datacenter replication processing
- Scheduler workflows (CHASM-based)
- System workflows (data cleanup, archival)
- Callback processing (Nexus)
- Migration workflows

**Key Components:**
```
service/worker/
├── service.go                # Worker service setup
├── replicator/               # Cross-DC replication
├── scheduler/                # Scheduled workflows
├── callbacks/                # Nexus callbacks
├── archiver/                 # History archival
├── batcher/                  # Batch operations
└── migration/                # Data migration workflows
```

**Design:** Worker service is itself a Temporal worker, running system workflows

---

### `/temporal` - Server Bootstrap

**Purpose:** Server initialization and dependency injection setup

**Key Files:**
- [`fx.go`](https://github.com/temporalio/temporal/blob/0ae5fb7f5d92364acb793aabadb2eb7e82f8bbe3/temporal/fx.go#L194) - Uber Fx dependency injection module
- `server.go` - Server interface definition
- `server_impl.go` - Server implementation

**Initialization Flow:**
```go
1. Load configuration
2. Initialize logger
3. Setup metrics provider
4. Create persistence clients
5. Initialize service providers (Frontend, History, Matching, Worker)
6. Wire up dependencies via Fx
7. Start gRPC servers
8. Handle graceful shutdown
```

**Fx Modules:**
- `ServerFx` - Main server module
- `ServiceProviders` - Service initialization
- `PersistenceRateLimiting` - Rate limiting for persistence
- `DynamicConfigClientProvider` - Dynamic configuration
- `NamespaceLoggerProvider` - Namespace-aware logging

---

### `/schema` - Database Schemas

**Purpose:** Database schemas for all supported persistence backends

**Structure:**
```
schema/
├── cassandra/                # Cassandra CQL schemas
│   ├── temporal/             # Main execution data
│   │   └── versioned/        # Versioned migrations (v0.1 - v1.18)
│   └── visibility/           # Visibility data
│       └── versioned/
├── mysql/                    # MySQL schemas
│   ├── v8/                   # MySQL 8.0+
│   │   ├── temporal/
│   │   └── visibility/
│   └── versioned/            # Migrations (v0.1 - v1.13)
├── postgresql/               # PostgreSQL schemas
│   ├── v12/                  # PostgreSQL 12+
│   │   ├── temporal/
│   │   └── visibility/
│   └── versioned/            # Migrations
├── sqlite/                   # SQLite schemas
│   ├── temporal/
│   └── visibility/
└── elasticsearch/            # Elasticsearch index templates
    └── visibility/           # Search indices
```

**Schema Management:**
- Versioned migrations for schema evolution
- Tools: `temporal-cassandra-tool`, `temporal-sql-tool`
- Commands: `setup-schema`, `update-schema`, `create-database`

**Example Migration:**
```sql
-- cassandra/temporal/versioned/v1.0/manifest.json
{
  "CurrVersion": "1.0",
  "MinCompatibleVersion": "1.0",
  "Description": "base version of schema",
  "SchemaUpdateCqlFiles": ["schema.cql"]
}
```

---

### `/tests` - Functional Tests

**Purpose:** End-to-end functional tests validating complete features

**Structure:**
```
tests/
├── functional_test_base.go  # Base test suite
├── ndc/                      # Multi-datacenter tests
├── xdc/                      # Cross-datacenter replication
├── workflow_*.go             # Workflow-specific tests
├── activity_*.go             # Activity tests
├── namespace_*.go            # Namespace tests
└── ...                       # Feature-specific tests
```

**Test Categories:**
- Workflow execution tests
- Activity execution tests
- Timer and cron tests
- Signal and query tests
- Child workflow tests
- Namespace operations
- Versioning and deployment tests
- Replication and failover tests

**Test Infrastructure:**
- `FunctionalTestBase` - Base test suite with cluster setup
- `TaskPoller` - Worker simulation
- Mock admin clients for multi-cluster scenarios

---

### `/tools` - Development & Operational Tools

**Purpose:** Utilities for development, testing, and operations

**Key Tools:**
```
tools/
├── cassandra/                # Cassandra schema tool
├── sql/                      # SQL schema tool
├── cli/                      # Command-line utilities
├── tdbg/                     # Temporal debugger
├── codegen/                  # Code generation (mocks, protobufs)
├── copyright/                # License header management
└── ...
```

**Notable Tools:**
- `tdbg` - Debug workflows, inspect state
- Schema tools - Database setup and migrations
- Code generators - Mocks, protobuf, gRPC

---

### `/.github` - CI/CD Workflows

**Purpose:** GitHub Actions workflows for continuous integration

**Key Workflows:**
```
.github/workflows/
├── run-tests.yml             # Main test workflow (unit, integration, functional)
├── create-tag.yml            # Release tagging
├── docker.yml                # Docker image builds
├── scorecards.yml            # Security scorecard
└── ...
```

**Test Strategy:**
- Parallel test execution across 3 shards
- Multi-database testing (SQLite, Cassandra, MySQL, PostgreSQL)
- Test retries for flaky tests
- Coverage reporting to Codecov

---

## File Naming Conventions

### Go Files
- `*_test.go` - Test files
- `*_mock.go` - Mock implementations (often auto-generated)
- `fx.go` - Uber Fx dependency injection modules
- `config.go` - Configuration structures
- `metrics.go` - Metrics definitions

### Protocol Buffers
- `*.proto` - Protocol buffer definitions
- `*.pb.go` - Generated Go code
- `*_grpc.pb.go` - Generated gRPC service code

---

## Module Organization Principles

1. **Layered Architecture**
   - `/api` - API contracts (generated)
   - `/service` - Business logic (core services)
   - `/common` - Shared utilities
   - `/temporal` - Application layer (bootstrap)

2. **Separation of Concerns**
   - Each service in separate directory
   - Common utilities extracted to `/common`
   - Test code co-located with implementation

3. **Dependency Direction**
   - Services depend on `/common`, not each other directly
   - Use `/client` for inter-service communication
   - gRPC APIs as service boundaries

4. **Testing Strategy**
   - Unit tests alongside implementation
   - Integration tests in `/common/persistence/tests`
   - Functional tests in `/tests`

---

## Key Metrics

| Metric | Value |
|--------|-------|
| Total Go Files | 2,363 |
| Test Files | 674 (28.5%) |
| Packages in `/common` | ~70 |
| Services in `/service` | 4 |
| Schema Versions (Cassandra) | 18 |
| Schema Versions (MySQL) | 13 |
| Supported Databases | 4 |
| Total Lines of Code | ~392,360 |

---

## Navigation Tips

### Finding Features
1. **User-facing APIs** → `/service/frontend/handler.go`
2. **Workflow execution logic** → `/service/history/historyEngine.go`
3. **Task queue logic** → `/service/matching/matcher.go`
4. **Persistence layer** → `/common/persistence/`
5. **Metrics definitions** → `/common/metrics/defs.go`

### Understanding a Feature
1. Start with API definition in `/api`
2. Find handler in `/service/frontend`
3. Follow to business logic in `/service/history`
4. Check persistence layer in `/common/persistence`

### Contributing
1. Read `/CONTRIBUTING.md`
2. Check `/docs/development/testing.md`
3. Review `/AGENTS.md` for best practices
4. Study relevant code in `/service` or `/common`

---

**Next:** [Dependency Graph](dependency-graph.md) | [Back to Quick Start](00-quick-start.md)
