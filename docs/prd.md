# go-plc Product Requirements Document (PRD)

## Goals and Background Context

### Goals

- Create a simple soft PLC using Go with JavaScript task scripting and YAML configuration
- Enable fast iteration cycles for testing control logic concepts
- Support S88/ISA-88 patterns to enable rapid testing with Acceleer platform

### Background Context

Traditional automation IDEs have slow iteration cycles that hinder rapid concept testing. go-plc addresses this by using YAML files to configure variables and I/O sources, with JavaScript files for task logic. This enables quick testing of S88/ISA-88 control concepts with platforms like Acceleer without the overhead of traditional PLC development environments.

### Change Log

| Date | Version | Description | Author |
|------|---------|-------------|---------|
| 2025-10-13 | 1.0 | Initial PRD creation | John (PM) |
| 2025-10-13 | 1.1 | Added Out of Scope section, deployment target, S88 clarification, checklist validation, and next steps | John (PM) |

## Requirements

### Functional

**FR1:** YAML configuration parser shall read sources (Modbus TCP), variables, and scaling definitions

**FR2:** Modbus TCP driver shall read and write holding registers and coils

**FR3:** JavaScript task engine (Sobek) shall auto-discover task files from `/tasks` directory

**FR4:** Task engine shall expose `plc` object with direct property access (e.g., `plc.pressure = 42`)

**FR5:** OPC UA server shall expose variables for Ignition/SCADA integration

**FR6:** GraphQL API shall provide queries and subscriptions for variable values

**FR7:** WebUI shall display:
- Tasks with execution speed and configured scan rate
- Sources with connection state
- Variables with current values

**FR8:** CLI shall support `plc run config.yaml` command to start the runtime

**FR9:** System shall support S88/ISA-88 state machine patterns in JavaScript tasks

### Non Functional

**NFR1:** Task execution overhead shall be <50µs per task cycle

**NFR2:** GraphQL queries shall respond in <10ms

**NFR3:** Iteration cycle (edit JavaScript task → test) shall be <1 minute

**NFR4:** Single binary deployment with embedded WebUI (no external dependencies)

**NFR5:** In-memory state only (no persistence required for MVP)

## User Interface Design Goals

### Overall UX Vision

WebUI serves as a monitoring dashboard only - the primary interfaces are YAML configuration files, JavaScript task files, and CLI commands. The WebUI provides real-time visibility into system state for debugging and verification.

### Key Interaction Paradigms

- Read-only dashboard (no configuration changes via UI)
- Auto-refresh for real-time data updates
- Simple table/list views for tasks, sources, and variables

### Core Screens and Views

- Single dashboard page displaying all three views (Tasks, Sources, Variables)

### Accessibility

None (internal development tool, MVP focuses on functionality)

### Branding

None (minimal styling, focus on clarity and readability)

### Target Device and Platforms

Web Responsive - primarily desktop browsers for development/commissioning work

## Technical Assumptions

### Repository Structure

Monorepo - Single Go project with embedded WebUI assets

### Service Architecture

Monolith - Single binary with concurrent goroutines for I/O sources, task execution, and API servers

### Testing Requirements

Unit + Integration - Unit tests for core components, integration tests for Modbus/OPC UA/GraphQL end-to-end flows

### Additional Technical Assumptions and Requests

- **Language & Runtime:** Go 1.22+ for core runtime, JavaScript (ES6+) via Sobek for task execution
- **Key Libraries:**
  - `github.com/grafana/sobek` - JavaScript runtime
  - `github.com/99designs/gqlgen` - GraphQL server
  - `github.com/gopcua/opcua` - OPC UA server (preferred over custom Sparkplug B for Ignition integration)
  - `github.com/goburrow/modbus` - Modbus TCP driver
- **Frontend:** React + Apollo Client for GraphQL subscriptions, embedded in Go binary using `embed` package
- **Configuration:** YAML only (using `gopkg.in/yaml.v3`)
- **State Management:** In-memory only (no database, no persistence)
- **Concurrency Model:** Goroutines with channels for inter-component communication
- **Performance Constraint:** Task execution engine must benchmark <50µs overhead (validate Sobek performance early)
- **Deployment Target:** Linux and Windows (cross-platform, single binary distribution)

## Epic List

**Epic 1: Foundation & Core Runtime**
Establish Go project structure, YAML config parser, and basic CLI with initial health check functionality.

**Epic 2: Modbus I/O Integration**
Implement Modbus TCP driver with connection management and variable mapping to enable physical I/O.

**Epic 3: JavaScript Task Engine**
Build Sobek-based task runtime with auto-discovery, direct property access (`plc.variable`), and scan rate scheduling.

**Epic 4: OPC UA Server**
Expose variables via OPC UA for Ignition/SCADA integration.

**Epic 5: GraphQL API & WebUI**
Create GraphQL API with subscriptions and simple monitoring dashboard for debugging and visibility.

## Out of Scope (MVP)

The following features are explicitly excluded from the MVP to maintain focus on core functionality and fast iteration cycles:

- **OPC UA Advanced Features:** Historical data access, alarms/events, complex data types (arrays, structs)
- **Authentication & Authorization:** No security for OPC UA, GraphQL, or WebUI (internal development tool)
- **File Watching/Hot-Reload:** Manual restart required to pick up config or task changes
- **Variable Persistence:** No database, no history, in-memory state only
- **Browser IDE:** No web-based code editor for tasks
- **Advanced Scaling:** Only linear scaling supported (no logarithmic, polynomial, or custom transforms)
- **Additional Protocols:** No Ethernet/IP, Profinet, BACnet (Modbus TCP only)
- **Auto-Discovery of I/O:** Manual YAML configuration required
- **Task Debugging Tools:** No JavaScript debugger integration (console logging only)

## Epic 1: Foundation & Core Runtime

**Goal:** Establish the foundational Go project with proper structure, YAML configuration parsing, and a basic CLI that can load and validate configuration. This epic delivers a working binary that demonstrates the "config-driven" architecture and provides a health check endpoint or simple output to verify the system is running correctly.

### Story 1.1: Project Setup and Repository Structure

**As a** developer,
**I want** a properly structured Go project with module configuration and basic directory layout,
**so that** I can begin implementing features with a clean foundation.

**Acceptance Criteria:**

1. Go module initialized with `go.mod` using Go 1.22+
2. Directory structure created: `/cmd/plc`, `/internal`, `/tasks`, `/config`
3. README.md with project description and basic setup instructions
4. `.gitignore` configured for Go projects
5. Initial `main.go` with placeholder CLI that prints "go-plc starting..."

### Story 1.2: YAML Configuration Parser

**As a** developer,
**I want** to parse YAML configuration files for sources and variables,
**so that** the PLC runtime can be configured without code changes.

**Acceptance Criteria:**

1. YAML parser using `gopkg.in/yaml.v3` reads `config.yaml` file
2. Configuration structure supports `sources` section with Modbus TCP source definitions (host, port)
3. Configuration structure supports `variables` section with name, source reference, and optional scaling
4. Parser validates required fields and returns clear error messages for invalid config
5. Unit tests cover valid config parsing and error cases
6. Example `config.yaml` file created in `/config` directory

### Story 1.3: CLI with Config Loading

**As a** user,
**I want** to run `plc run config.yaml` from the command line,
**so that** I can start the PLC runtime with my configuration.

**Acceptance Criteria:**

1. CLI accepts `run` command with config file path argument
2. CLI loads and parses the specified YAML config file
3. CLI displays parsed configuration summary (number of sources, variables)
4. CLI handles file not found and parse errors with clear messages
5. CLI prints "PLC runtime started successfully" and waits (graceful shutdown on Ctrl+C)
6. Integration test verifies CLI can load example config and start

### Story 1.4: In-Memory Variable Registry

**As a** developer,
**I want** an in-memory registry for storing variable metadata and current values,
**so that** other components can read and write variable data.

**Acceptance Criteria:**

1. Variable registry struct stores variable definitions from config (name, type, source, scaling)
2. Registry provides `GetValue(name)` and `SetValue(name, value)` methods (internal Go API)
3. Registry is thread-safe (uses mutex or channels)
4. Registry initialized from parsed YAML config on startup
5. Unit tests cover concurrent read/write operations
6. Initial values default to zero/nil
7. Note: Direct property access (`plc.variable = value`) for JavaScript tasks will be implemented in Epic 3

## Epic 2: Modbus I/O Integration

**Goal:** Implement Modbus TCP client to read and write I/O from remote devices. This epic delivers the ability to define Modbus sources in YAML, connect to Modbus TCP servers, map holding registers and coils to variables with scaling, and continuously poll/update values. The PLC can now interact with physical hardware.

### Story 2.1: Modbus TCP Connection Manager

**As a** developer,
**I want** to establish and manage Modbus TCP connections to remote devices,
**so that** the PLC can communicate with I/O hardware.

**Acceptance Criteria:**

1. Modbus connection manager using `github.com/goburrow/modbus` package
2. Manager creates TCP client for each source defined in config (host:port)
3. Connection state tracked (connected, disconnected, error) for each source
4. Automatic reconnection logic with exponential backoff on connection failures
5. Unit tests with mock Modbus server verify connection/reconnection behavior
6. Manager exposes connection state via getter method for monitoring

### Story 2.2: Modbus Register Read Operations

**As a** developer,
**I want** to read holding registers and coils from Modbus devices,
**so that** the PLC can retrieve input values from I/O hardware.

**Acceptance Criteria:**

1. Read holding registers (function code 0x03) from specified addresses
2. Read coils (function code 0x01) from specified addresses
3. Configurable polling interval per source (default 100ms)
4. Error handling for Modbus exceptions and timeouts
5. Integration test using pump-station-modbus-sim Docker image (https://github.com/aott33/pump-station-modbus-sim) verifies reads
6. Read values logged at debug level for troubleshooting

### Story 2.3: Modbus Register Write Operations

**As a** developer,
**I want** to write holding registers and coils to Modbus devices,
**so that** the PLC can send output values to I/O hardware.

**Acceptance Criteria:**

1. Write single holding register (function code 0x06)
2. Write single coil (function code 0x05)
3. Write operations triggered when variable value changes in registry
4. Error handling for Modbus exceptions with retry logic (max 3 attempts)
5. Integration test using pump-station-modbus-sim Docker image verifies writes
6. Write operations logged at info level for auditing

### Story 2.4: Variable Mapping with Scaling

**As a** developer,
**I want** to map Modbus registers to variables with scaling transformations,
**so that** raw register values are converted to engineering units.

**Acceptance Criteria:**

1. Variable config supports source mapping (e.g., `source: remoteIO.holding.2000`)
2. Variable config supports linear scaling: `scale: [inputMin, inputMax, outputMin, outputMax]`
3. Scaling applied on read: raw register value → scaled variable value
4. Inverse scaling applied on write: scaled variable value → raw register value
5. Unit tests verify scaling calculations for various ranges
6. Variables without scaling use raw register values directly
7. Example config demonstrates scaled analog input (0-32000 → 0-100 PSI) using pump station simulator

### Story 2.5: Continuous Polling Loop

**As a** developer,
**I want** Modbus sources to continuously poll configured registers,
**so that** variable values stay synchronized with I/O hardware.

**Acceptance Criteria:**

1. Goroutine per source runs polling loop at configured scan rate
2. Each poll cycle reads all registers mapped to variables for that source
3. Read values update variable registry after scaling
4. Polling continues on errors (logged but not fatal)
5. Graceful shutdown stops all polling loops
6. Integration test with pump-station-modbus-sim verifies continuous polling updates registry values
7. Polling performance logged (cycle time, errors per minute)

## Epic 3: JavaScript Task Engine

**Goal:** Build the Sobek-based JavaScript runtime that auto-discovers task files from the `/tasks` directory, exposes the `plc` object with direct property access to variables, and executes tasks on configurable scan rates. This epic delivers the core user experience: writing control logic in JavaScript with simple syntax like `plc.pumpOn = (plc.pressure > 5)`. S88/ISA-88 state machine patterns can be implemented in tasks.

### Story 3.1: Sobek JavaScript Runtime Integration

**As a** developer,
**I want** to embed the Sobek JavaScript engine in the Go runtime,
**so that** JavaScript code can be executed within the PLC.

**Acceptance Criteria:**

1. Sobek package (`github.com/grafana/sobek`) integrated as dependency
2. JavaScript runtime instance created on PLC startup
3. Simple "Hello World" JavaScript executed and output logged
4. Runtime supports ES6+ syntax (arrow functions, const/let, template literals)
5. Unit tests verify runtime can execute basic JavaScript expressions
6. Error handling for JavaScript syntax errors with clear error messages

### Story 3.2: Task File Auto-Discovery

**As a** user,
**I want** JavaScript task files to be automatically discovered from the `/tasks` directory,
**so that** I can add control logic without modifying configuration.

**Acceptance Criteria:**

1. On startup, scan `/tasks` directory for `.js` files
2. Each task file loaded and parsed for required exports: `scanRate` (number in ms) and `execute` (function)
3. Invalid task files logged as warnings (missing exports, syntax errors) but don't prevent startup
4. Task metadata stored (filename, scanRate, execute function reference)
5. Integration test verifies tasks are discovered and metadata extracted
6. Example task file created in `/tasks` directory (e.g., `example-task.js`)

### Story 3.3: Direct Property Access via `plc` Object

**As a** user,
**I want** to access and modify PLC variables using direct property syntax (`plc.pressure`, `plc.setpoint = 4`),
**so that** I can write concise control logic without verbose API calls.

**Acceptance Criteria:**

1. `plc` JavaScript object exposed to task runtime with dynamic property getters/setters
2. Reading `plc.variableName` calls Go registry's `GetValue("variableName")` and returns scaled value
3. Writing `plc.variableName = value` calls Go registry's `SetValue("variableName", value)` with scaled value
4. Accessing undefined variables returns `undefined` (no runtime error)
5. Unit tests verify property access bridges to registry correctly
6. Example task demonstrates: `plc.pumpOn = (plc.pressure > 5);`
7. Performance benchmark confirms <50µs overhead for property access

### Story 3.4: Task Scheduling and Execution

**As a** developer,
**I want** tasks to execute on their configured scan rates,
**so that** control logic runs at appropriate frequencies.

**Acceptance Criteria:**

1. Goroutine per task runs execution loop at configured `scanRate` (milliseconds)
2. Each loop iteration calls task's `execute()` function with `plc` object in scope
3. Task execution errors logged (with task filename and line number) but don't crash runtime
4. Execution time per task cycle measured and logged
5. Graceful shutdown stops all task execution loops
6. Integration test verifies task executes multiple times at correct interval
7. Tasks can have different scan rates (e.g., 100ms for fast control, 1000ms for logging)

### Story 3.5: S88 State Machine Example

**As a** user,
**I want** example code demonstrating S88/ISA-88 state machine patterns in JavaScript tasks,
**so that** I can implement batch control logic for testing with Acceleer platform.

**Acceptance Criteria:**

1. Example S88 state machine task created demonstrating state transitions (e.g., Idle → Running → Complete)
2. Task uses simple JavaScript patterns: state variable, switch/case or if/else for state logic
3. Documentation (in README or example comments) shows how to implement S88 phases
4. No custom S88 framework provided - examples use plain JavaScript patterns only
5. Example integrates with pump station simulator (state transitions trigger Modbus writes)
6. Integration test verifies state machine progresses through states correctly
7. Note: S88 support means enabling implementation via JavaScript, not providing an ISA-88 framework

## Epic 4: OPC UA Server

**Goal:** Expose PLC variables via OPC UA server for integration with Ignition, Acceleer, and other SCADA/MES systems. This epic delivers the external integration capability, allowing real-time data consumption by enterprise systems.

### Story 4.1: OPC UA Server Initialization

**As a** developer,
**I want** to start an OPC UA server on PLC startup,
**so that** external systems can connect and browse available variables.

**Acceptance Criteria:**

1. OPC UA server using `github.com/gopcua/opcua` package initialized on startup
2. Server listens on configurable port (default 4840)
3. Server endpoint URL logged on startup (e.g., `opc.tcp://localhost:4840`)
4. Server supports anonymous authentication (no security for MVP)
5. Unit tests verify server starts and accepts connections
6. Graceful shutdown stops server cleanly

### Story 4.2: Variable Namespace and Address Space

**As a** developer,
**I want** PLC variables automatically added to the OPC UA address space,
**so that** clients can browse and discover available data points.

**Acceptance Criteria:**

1. Each variable from registry added to OPC UA address space under `ns=2;s=variableName` pattern
2. Variables organized under root node "PLCVariables"
3. Variable metadata includes DisplayName (human-readable) and DataType (inferred from value)
4. Address space updated dynamically if variables added during runtime (future-proofing)
5. Integration test with OPC UA client verifies variables are browsable
6. Example: `pressure`, `pumpOn`, `setpoint` appear in OPC UA browser

### Story 4.3: Real-Time Variable Value Reads

**As a** SCADA operator,
**I want** to read current variable values via OPC UA,
**so that** I can monitor PLC state in real-time.

**Acceptance Criteria:**

1. OPC UA Read request returns current value from variable registry
2. Values updated reflect latest data from Modbus polling and task execution
3. Read response includes StatusCode (Good/Bad) and SourceTimestamp
4. Multiple variables can be read in single request (batch read)
5. Integration test with OPC UA client verifies reads return correct values
6. Read latency <10ms (aligns with NFR2 for GraphQL)

### Story 4.4: Variable Value Writes via OPC UA

**As a** SCADA operator,
**I want** to write variable values via OPC UA,
**so that** I can adjust setpoints and control outputs remotely.

**Acceptance Criteria:**

1. OPC UA Write request updates variable in registry
2. Write triggers Modbus write operation if variable mapped to output register
3. Write response includes StatusCode indicating success or error
4. Multiple variables can be written in single request (batch write)
5. Integration test with OPC UA client verifies writes update variables
6. Writes logged at info level for auditing

### Story 4.5: OPC UA Subscriptions (Optional MVP Stretch)

**As a** SCADA system,
**I want** to subscribe to variable changes via OPC UA,
**so that** I receive updates only when values change (efficient polling).

**Acceptance Criteria:**

1. OPC UA server supports MonitoredItem creation for variables
2. Value changes trigger notifications to subscribed clients
3. Configurable publishing interval per subscription
4. Integration test verifies client receives change notifications
5. Subscription performance does not impact task execution (<50µs overhead)
6. Note: This story is optional for MVP - can be deferred if time-constrained

## Epic 5: GraphQL API & WebUI

**Goal:** Create a GraphQL API with real-time subscriptions and a simple React-based monitoring dashboard. This epic delivers developer-focused tools for debugging and visibility: query variable values, subscribe to changes, and view system state (tasks, sources, variables) in a web interface.

### Story 5.1: GraphQL Schema and Server Setup

**As a** developer,
**I want** a GraphQL server with schema for PLC entities,
**so that** I can query and subscribe to runtime data programmatically.

**Acceptance Criteria:**

1. GraphQL server using `github.com/99designs/gqlgen` initialized on startup
2. Server listens on configurable port (default 8080, endpoint `/graphql`)
3. Schema defines types: `Variable`, `Task`, `Source`
4. Schema includes queries: `variables`, `variable(name: String!)`, `tasks`, `sources`
5. GraphQL playground available at `/graphql` for interactive testing
6. Unit tests verify schema validation and server startup
7. Server runs concurrently with OPC UA server and task execution

### Story 5.2: Variable Queries

**As a** developer,
**I want** to query current variable values via GraphQL,
**so that** I can inspect PLC state during development.

**Acceptance Criteria:**

1. Query `variables` returns list of all variables with name, value, source mapping
2. Query `variable(name: "pressure")` returns single variable details
3. Values reflect current state from variable registry
4. Query response includes metadata (data type, scaling config if applicable)
5. Integration test verifies queries return correct data
6. Query latency <10ms (per NFR2)

### Story 5.3: Variable Subscriptions

**As a** developer,
**I want** to subscribe to variable value changes via GraphQL,
**so that** I can monitor real-time updates in web applications.

**Acceptance Criteria:**

1. Subscription `variableUpdated(name: String!)` streams value changes for specific variable
2. Subscription `variablesUpdated` streams changes for all variables
3. Updates pushed whenever variable value changes in registry (from Modbus, tasks, or OPC UA writes)
4. WebSocket transport for subscriptions using Apollo Server conventions
5. Integration test verifies subscription receives updates
6. Subscription overhead does not impact task execution (<50µs)

### Story 5.4: Task and Source Queries

**As a** developer,
**I want** to query task and source status via GraphQL,
**so that** I can monitor system health and performance.

**Acceptance Criteria:**

1. Query `tasks` returns list with: filename, scanRate, lastExecutionTime, executionCount, errors
2. Query `sources` returns list with: name, type (modbus), connectionState, lastPollTime, errorCount
3. Data sourced from task scheduler and Modbus connection manager
4. Integration test verifies queries return accurate runtime statistics
5. Query latency <10ms

### Story 5.5: React WebUI with GraphQL Subscriptions

**As a** user,
**I want** a web dashboard showing live PLC state,
**so that** I can monitor variables, tasks, and sources during development and commissioning.

**Acceptance Criteria:**

1. React application with Apollo Client for GraphQL communication
2. Dashboard displays three sections: Tasks, Sources, Variables (per FR7)
3. Tasks section shows: filename, scan rate, execution speed (avg cycle time)
4. Sources section shows: name, connection state (connected/disconnected/error)
5. Variables section shows: name, current value, updates in real-time via subscriptions
6. WebUI assets embedded in Go binary using `embed` package
7. WebUI served at `http://localhost:8080/` (root path)
8. Integration test verifies UI loads and displays data
9. UI updates automatically when values change (via GraphQL subscriptions)

## Checklist Results Report

### Executive Summary

**Overall PRD Completeness:** 85%
**MVP Scope Appropriateness:** Just Right
**Readiness for Architecture Phase:** **READY**

This PRD has been validated against the PM Checklist and is ready for the Architect. Key strengths include excellent epic/story structure, clear technical direction, and appropriate KISS principles. Minor enhancements have been incorporated (Out of Scope section, deployment target specification, S88 clarification).

### Category Validation Results

| Category                         | Status | Notes |
| -------------------------------- | ------ | ----- |
| 1. Problem Definition & Context  | PASS   | Clear problem and target users defined |
| 2. MVP Scope Definition          | PASS   | Out of scope section added |
| 3. User Experience Requirements  | PASS   | Appropriate for tool-focused product |
| 4. Functional Requirements       | PASS   | Clear, testable requirements (FR1-FR9) |
| 5. Non-Functional Requirements   | PASS   | Performance targets quantified |
| 6. Epic & Story Structure        | PASS   | Excellent sequencing and decomposition |
| 7. Technical Guidance            | PASS   | Comprehensive technical stack defined |
| 8. Cross-Functional Requirements | PASS   | Integration and testing well-defined |
| 9. Clarity & Communication       | PASS   | Well-structured, consistent terminology |

### Key Areas for Architect Investigation

The following areas require validation or detailed design during the architecture phase:

1. **Sobek Performance Validation** - Benchmark <50µs task execution overhead (Critical NFR1)
2. **OPC UA Server Mode** - Verify `gopcua/opcua` library supports server implementation (most examples show client mode)
3. **GraphQL Subscription Architecture** - WebSocket transport and change notification mechanism
4. **Single Binary Build Pipeline** - React build and Go embed strategy for NFR4
5. **Graceful Shutdown Coordination** - Managing concurrent goroutines across Modbus, Tasks, OPC UA, GraphQL servers
6. **Logging Framework** - Selection and integration strategy (slog, zap, logrus?)
7. **Error Propagation Strategy** - Channels vs return values for component communication

### Recommendations for Implementation

**Epic 1 (Foundation):**
- Establish logging framework early (needed across all epics)
- Create Sobek performance benchmark in Story 1.1 or 1.4 to validate NFR1 early

**Epic 3 (Task Engine):**
- Story 3.3 benchmark is critical - if Sobek doesn't meet <50µs target, major architecture pivot required

**Epic 4 (OPC UA):**
- Investigate `gopcua/opcua` server capabilities before starting Epic 4
- Consider alternative libraries if server mode not supported

**Epic 5 (WebUI):**
- Frontend build pipeline should be established early in Story 5.1
- Consider deferred Story 5.5 (WebUI) if timeline pressures occur - other epics deliver core value

## Next Steps

### Architect Handoff

The PRD is complete and ready for the Architect to begin detailed design. The Architect should create:

1. **High-level architecture diagram** - Component interactions, data flow, concurrency model
2. **Detailed directory structure** - `/internal` package organization
3. **Interface contracts** - Registry, Modbus, Task Engine, OPC UA, GraphQL APIs
4. **Error handling strategy** - Propagation, logging, recovery patterns
5. **Build and deployment pipeline** - Cross-platform compilation, React embedding
6. **Testing strategy details** - Unit test framework, integration test infrastructure

### UX Expert Prompt

*Note: UX involvement is minimal for this MVP as the primary interfaces are YAML files, JavaScript files, and CLI. The WebUI is a simple monitoring dashboard only.*

If UX review is desired, focus on:
- WebUI information architecture (Tasks, Sources, Variables layout)
- Error message clarity for CLI and JavaScript task errors
- Example YAML/JavaScript file structure and comments

### Architect Prompt

**Task:** Create architecture document for go-plc soft PLC based on the PRD

**Key Requirements:**
- Review PRD at [docs/prd.md](docs/prd.md)
- Design component architecture for monolithic Go application with concurrent goroutines
- Define interfaces between: Config Parser, Variable Registry, Modbus Driver, Task Engine, OPC UA Server, GraphQL Server
- Specify error handling and logging strategy
- Address critical investigation areas: Sobek performance validation, OPC UA server library capabilities, GraphQL subscriptions
- Create architecture document following project conventions

**Deliverable:** Architecture document with component diagrams, interface definitions, and implementation guidance for the 5 epics

