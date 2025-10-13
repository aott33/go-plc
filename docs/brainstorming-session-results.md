# Brainstorming Session Results

**Session Date:** 2025-10-13
**Facilitator:** Business Analyst Mary
**Participant:** Andrew Ott

---

## Executive Summary

**Topic:** Reducing verbosity in Tentacle PLC implementation for faster iteration cycles

**Session Goals:** Design a new, less verbose soft PLC implementation that maintains critical features (Sparkplug B, GraphQL, WebUI, Modbus) while dramatically simplifying variable definitions and task creation.

**Techniques Used:** First Principles Thinking, Analogical Thinking, SCAMPER Method, Assumption Reversal, What If Scenarios

**Total Ideas Generated:** 25+ concepts across architecture, language choice, configuration, and features

**Key Themes Identified:**
- Configuration-driven simplicity (YAML + code separation)
- Direct property access (eliminating `.value` verbosity)
- Go core + JavaScript tasks for performance + ease
- Auto-discovery patterns (tasks, variables)
- Developer experience features (hot-reload, browser IDE, auto-logging)

---

## Technique Sessions

### First Principles Thinking - Core Architecture Decisions

**Description:** Stripped away current Tentacle PLC assumptions to identify absolute fundamentals of what a soft PLC must do.

**Ideas Generated:**

1. **Core Requirements Identified**
   - Execute code in loops at defined intervals
   - Unified "Source" abstraction for all I/O (Modbus, MQTT, OPC UA, Ethernet/IP)
   - Read/write to any source without excessive boilerplate
   - Maintain state between loops
   - Expose data externally (MQTT/Sparkplug B, GraphQL, WebUI)

2. **Direct Property Access Model**
   - Eliminate verbose `variables.count.value` pattern
   - Adopt simple direct access: `plc.suctionPressure = 42`
   - Inspired by Davy Demers' C# demo: `Global.M001.command = true`

3. **YAML Configuration Approach**
   - Separate concerns: configuration in YAML, logic in JavaScript tasks
   - Middle ground between minimal and explicit
   - Example structure:
     ```yaml
     sources:
       remoteIO:
         type: modbus
         host: 192.168.1.10

     variables:
       suctionPressure:
         source: remoteIO.holding.2000
         scale: [0, 32000, 0, 100]
         units: PSI

     sparkplug:
       broker: mqtt://localhost:1883
       group: pump-station
     ```

4. **Task Convention over Configuration**
   - Tasks auto-discovered from `/tasks` folder
   - Each task declares its own scan rate
   - No explicit registration needed
   - Example: `sensor-monitor.js` exports `scanRate = 100`

5. **Language Architecture: Go Core + JavaScript Tasks**
   - **Go Core** handles:
     - I/O drivers (Modbus, OPC UA, Ethernet/IP)
     - YAML config parsing & code generation
     - Sparkplug B publishing
     - GraphQL API server
     - WebUI serving
     - Task scheduling & execution engine
     - JavaScript runtime (Sobek/goja)

   - **JavaScript Tasks** for:
     - Control logic (frequently changing code)
     - Fast iteration without recompiling
     - Maximum AI agent compatibility (most training data)
     - Developer familiarity

**Insights Discovered:**
- Verbosity stems from explicit type definitions and source configurations
- Direct property access (like Davy's C#) is key to reducing verbosity
- Separation of YAML config and JS logic provides clean architecture
- Go + JavaScript (Sobek) gives performance + ease of development

**Notable Connections:**
- Davy's C# implementation proves simple, direct access works well
- Configuration-driven approach mirrors Docker Compose patterns
- JavaScript maximizes AI agent assistance (GitHub's #1 language)

---

### Analogical Thinking - Learning from Similar Systems

**Description:** Explored how other successful systems solve similar problems (state management, real-time loops, I/O abstraction, configuration-driven behavior).

**Ideas Generated:**

1. **Game Engine Patterns (Unity, Godot)**
   - Component/entity pattern for equipment modeling
   - Update loops at different rates (like PLC scan cycles)
   - Hot-reload for fast iteration
   - Connection: User input → PLC input, Update loop → Scan cycle

2. **Web Framework Patterns (Express, Django)**
   - ORM-like pattern for variable mapping
   - Direct property access: `user.email` → `plc.suctionPressure`
   - Auto-sync to data sources (Database → Modbus/OPC UA)
   - Middleware pipeline → Task execution chain
   - Configuration patterns: Database config → Source config

   **Key insight:** Web frameworks solved verbosity with ORMs!
   ```python
   # Django ORM
   user.email = "new@example.com"
   user.save()  # Auto-writes to database

   # PLC equivalent
   plc.suctionPressure = 42  # Auto-writes to Modbus
   ```

3. **Node-RED Architecture**
   - Visual editor for simple logic (not pursuing for this project)
   - **Key takeaway:** Go backend handles complex processes
   - WebUI for monitoring and control
   - Code-first approach with visual status dashboard

4. **Docker Compose Multi-Source Orchestration**
   - Sources defined like Docker services
   - Variables reference sources by name
   - Clean, readable multi-source configuration
   ```yaml
   sources:
     remoteIO: { type: modbus, host: 192.168.1.10 }
     cloudMQTT: { type: mqtt, broker: mqtt://cloud.io }
   ```

**Insights Discovered:**
- Web framework ORM pattern is perfect analog for PLC variable mapping
- Game engine hot-reload aligns with fast iteration goal
- Node-RED demonstrates value of powerful backend + simple frontend
- Docker Compose patterns make complex configurations readable

**Notable Connections:**
- GraphQL + Sparkplug B parallels REST API + WebSockets in web frameworks
- Real-time loops exist in games, web servers, robotics - proven patterns
- Configuration-driven systems (Docker, K8s) use YAML successfully

---

### SCAMPER Method - Systematic Improvements

**Description:** Applied SCAMPER framework (Substitute, Combine, Adapt, Modify, Put to other use, Eliminate, Reverse) to systematically improve the design.

**Ideas Generated:**

**Top 3 Selected Ideas:**

5. **Combine: Single Data Model (GraphQL + Sparkplug B unified)**
   - Define variables once in YAML
   - Automatically available via:
     - GraphQL query/subscription
     - Sparkplug B MQTT topics
     - WebUI display
     - JavaScript task access (direct property)
   - Eliminates duplicate definitions and "publish" steps

6. **Eliminate: Auto-discover Tasks**
   - Remove explicit task registration
   - Go core auto-discovers tasks from `/tasks` folder
   - Each task exports its own `scanRate`
   - Zero configuration needed
   ```
   /tasks
     ├── sensor-monitor.js    (export scanRate = 100)
     ├── pump-control.js      (export scanRate = 500)
     └── data-logger.js       (export scanRate = 1000)
   ```

8. **Eliminate: Source/Variable Distinction**
   - Everything is just a variable
   - Source is just metadata (where it comes/goes)
   - YAML adds source mapping, code sees only variables
   ```javascript
   plc.suctionPressure        // Read (auto-syncs from Modbus)
   plc.pumpRunning = true     // Write (auto-syncs to Modbus)
   ```

**Additional Ideas Explored:**

9. **Modify: Convention over Configuration**
   - Compact notation: `suctionPressure: modbus.2000[0-100]`
   - Framework infers sensible defaults
   - Reduces YAML verbosity further

11. **Adapt: Docker Compose-style Multi-Source** (Selected as interesting)
   - Named sources referenced like Docker services
   - Clean orchestration of multiple protocols
   - Example in "Docker Compose" section above

**Insights Discovered:**
- Single data model eliminates duplication across GraphQL/MQTT/WebUI
- Auto-discovery patterns reduce configuration overhead
- Unifying source and variable concepts simplifies mental model

**Notable Connections:**
- Web frameworks use auto-discovery (Rails, Django)
- Convention over configuration is proven pattern (Ruby on Rails)
- Docker Compose demonstrates readable multi-service config

---

### Assumption Reversal - Challenging Core Beliefs

**Description:** Identified and reversed core assumptions to explore alternative approaches and validate decisions.

**Ideas Generated:**

**Key Decisions Made:**

- **Fixed Scan Rates (NOT Reversed)**
  - Decision: Keep deterministic fixed scan rates
  - Rationale: PLCs must be deterministic for safety/reliability
  - Event-driven architectures not appropriate for control systems

- **YAML Configuration (NOT Reversed)**
  - Decision: Keep YAML for configuration
  - Rationale: Clean separation of config and logic
  - Single view of all I/O mappings
  - Version control friendly

- **Variable Scoping (ADOPTED)**
  - Global variables: Defined in YAML, published to Sparkplug B/GraphQL
  - Local variables: Declared in task code, not published, low overhead
  - Clear distinction: Important data global, task state local
  ```yaml
  # config.yaml - Global variables
  variables:
    suctionPressure: modbus.2000[0-100]
    pumpRunning: modbus.2502
  ```
  ```javascript
  // pump-control.js - Local + global
  let startupTimer = 0  // Local, not published
  let faultCount = 0    // Local, not published

  export function execute() {
    if (plc.suctionPressure < 5) {  // Global access
      plc.pumpRunning = false
      faultCount++
    }
  }
  ```

**Deferred to Roadmap:**

- **Multi-PLC Instances:** One runtime managing multiple virtual PLCs
  - Decision: Too complex for initial version
  - Future: Could enable multi-tenant deployments

- **Go Plugin System:** User-extensible Go core with plugins
  - Decision: Keep simple for v1, JavaScript tasks only
  - Future: Custom protocol drivers as Go plugins

**Insights Discovered:**
- Deterministic execution is non-negotiable for PLC safety
- YAML provides better overview than code-as-config for I/O
- Variable scoping prevents unnecessary data publishing overhead

---

### What If Scenarios - Exploring Alternatives

**Description:** Explored provocative "what if" questions to push boundaries and discover innovative features.

**Ideas Generated:**

**Features Selected for Implementation:**

- **Hot-Reload with Safety Gates**
  - Development mode: Auto-reload on file changes
  - Production mode: Confirmation required before reload
  ```bash
  # Dev mode
  $ plc dev config.yaml
  [DEV] Hot-reload enabled - changes apply immediately

  # Production mode
  $ plc run config.yaml
  [PROD] Change detected: pump-control.js
  [PROD] Press 'y' to reload, 'n' to ignore: _
  ```

- **Multi-Language Task Support**
  - Tasks can be JavaScript or Python (future)
  - CLI shows performance impact warnings
  - Example:
  ```
  /tasks
    ├── sensor-monitor.js       [~50µs execution]
    ├── pid-control.py          [~200µs execution] ⚠️ Python overhead
    └── safety-interlock.js     [~50µs execution]
  ```

- **Browser-Based IDE**
  - WebUI includes code/config editor (Monaco/VSCode in browser)
  - Live variable dashboard
  - Hot-reload trigger button
  - Useful for remote commissioning and field debugging

- **Built-in Variable History/Trending**
  - Every variable auto-logs for diagnostics
  - Time-series API built into variable access
  ```javascript
  plc.suctionPressure              // Current: 42
  plc.suctionPressure.history(60)  // Last 60 seconds
  plc.suctionPressure.avg(60)      // Average over 60s
  plc.suctionPressure.changed()    // True if changed this scan
  ```

**Additional Ideas Explored:**

- **Code-as-Config:** Variables/sources defined inline in JavaScript
  - Decision: Less appealing than YAML separation
  - YAML provides better overview of all I/O

- **Auto-Generated Simulator:** Framework generates hardware simulator from config
  - Interesting for future exploration
  - Accelerates testing without real hardware

**Insights Discovered:**
- Hot-reload critical for Davy's "fast iteration" goal
- Multi-language support enables best tool per task (NumPy for math)
- Browser IDE removes dependency on local development tools
- Built-in trending eliminates need for external historians

**Notable Connections:**
- Hot-reload mirrors game engine development experience
- Browser IDE follows VS Code online, Jupyter notebook patterns
- Auto-logging addresses common PLC debugging pain point

---

## Idea Categorization

### Immediate Opportunities
*Ideas ready to implement now in v1*

1. **Go Core + JavaScript (Sobek) Runtime**
   - Description: Use Go for I/O drivers, task scheduling, APIs; Sobek for task execution
   - Why immediate: Sobek is production-ready (used by Grafana), pure Go, no CGO
   - Resources needed: Go development environment, Sobek library
   - Implementation:
     - Set up Go project with Sobek integration
     - Create task execution engine
     - Expose `plc` global object to JavaScript

2. **YAML Configuration with Direct Property Access**
   - Description: YAML defines sources/variables, JavaScript tasks access via `plc.variableName`
   - Why immediate: Core verbosity reduction, well-defined pattern
   - Resources needed: YAML parser (Go stdlib), code generation for JS types
   - Implementation:
     - Parse YAML config
     - Generate TypeScript definitions for IDE autocomplete
     - Expose variables as direct properties on `plc` object

3. **Auto-Discovery Task Loading**
   - Description: Scan `/tasks` folder, load all .js files, read `scanRate` export
   - Why immediate: Simple file system operation, zero-config benefit
   - Resources needed: Go file system scanning, JavaScript module loading
   - Implementation:
     - Directory watcher in Go
     - Task metadata extraction from JS exports
     - Dynamic task registration

4. **Single Data Model (Unified GraphQL + Sparkplug B)**
   - Description: Variables defined once in YAML, auto-published to all interfaces
   - Why immediate: Eliminates duplication, core simplification
   - Resources needed: GraphQL library (gqlgen), Sparkplug B library
   - Implementation:
     - Generate GraphQL schema from YAML
     - Map variables to Sparkplug B metrics
     - Auto-publish on variable changes

5. **Variable Scoping (Global + Local)**
   - Description: YAML variables are global (published), task variables are local (not published)
   - Why immediate: Simple to implement, clear performance benefit
   - Resources needed: Scope isolation in JavaScript runtime
   - Implementation:
     - Global `plc` object with YAML variables
     - Task-local scope for internal state
     - No publication for local variables

### Future Innovations
*Ideas requiring development/research*

6. **Hot-Reload with Safety Gates**
   - Description: Dev mode auto-reloads, production mode requires confirmation
   - Development needed: File watching, safe task swapping, state preservation
   - Timeline estimate: v1.1 (1-2 months after v1.0)

7. **Browser-Based IDE**
   - Description: Monaco editor in WebUI for editing config/tasks with hot-reload
   - Development needed: Monaco integration, file system API, authentication
   - Timeline estimate: v1.2 (3-4 months after v1.0)

8. **Built-in Variable History/Trending**
   - Description: Time-series data collection with query API (`.history()`, `.avg()`)
   - Development needed: Time-series database (embedded), retention policies, API design
   - Timeline estimate: v1.3 (4-6 months after v1.0)

9. **Multi-Language Task Support (Python)**
   - Description: Tasks can be .py files, executed via Starlark or gRPC bridge
   - Development needed: Python runtime integration, performance profiling, error handling
   - Timeline estimate: v2.0 (6-9 months after v1.0)

10. **Docker Compose-Style Multi-Source Orchestration**
    - Description: Named sources with advanced dependency management
    - Development needed: Source lifecycle management, health checks, failover
    - Timeline estimate: v1.2 (3-4 months after v1.0)

### Moonshots
*Ambitious, transformative concepts*

11. **Auto-Generated Hardware Simulator**
    - Description: `plc generate-simulator config.yaml` creates Modbus server + WebUI
    - Transformative potential: Zero-hardware development, instant testing environment
    - Challenges to overcome:
      - Realistic physics simulation for sensors
      - State machine modeling for equipment behavior
      - Multi-protocol simulation (Modbus + OPC UA + Ethernet/IP)
    - Timeline: v2.0+ (9-12 months)

12. **Multi-PLC Instance Runtime**
    - Description: Single Go process manages multiple virtual PLCs with isolated configs
    - Transformative potential: Multi-tenant deployments, edge computing efficiency
    - Challenges to overcome:
      - Resource isolation and scheduling
      - Inter-PLC communication patterns
      - Performance impact of virtualization
    - Timeline: v2.0+ (12+ months)

13. **Go Plugin System for Custom Protocols**
    - Description: Users write Go plugins for proprietary I/O drivers
    - Transformative potential: Extensible to any industrial protocol
    - Challenges to overcome:
      - Plugin ABI stability
      - Security sandboxing
      - Hot-loading compiled plugins
    - Timeline: v2.0+ (12+ months)

### Insights & Learnings

- **Verbosity reduction is about directness:** The `.value` pattern and string-based setters create cognitive overhead. Direct property access (`plc.x = 42`) is the key win.

- **Configuration vs. code separation matters:** YAML for I/O mapping provides overview and version control benefits. Code for logic keeps tasks readable.

- **AI agent compatibility drives language choice:** JavaScript's ubiquity on GitHub means maximum LLM training data, leading to better code generation and assistance.

- **Performance is rarely the bottleneck:** Even "slow" JavaScript (50µs/task) fits easily in 100ms scan cycles. I/O latency (Modbus, OPC UA) dominates. Language choice should prioritize developer experience.

- **Convention over configuration reduces verbosity:** Auto-discovery, sensible defaults, and unified abstractions eliminate boilerplate without sacrificing power.

- **Fast iteration requires hot-reload:** Davy's main goal is rapid testing. Development mode with instant feedback is critical. Production safety gates preserve reliability.

- **Web framework patterns translate well:** ORMs for data access, middleware for execution flow, configuration-driven setup—these patterns solve the same problems PLCs face.

---

## Action Planning

### Top 3 Priority Ideas

#### #1 Priority: Core Architecture - Go + Sobek + YAML

**Rationale:**
- Foundation for entire project
- Directly addresses verbosity problem (direct property access)
- Proven technology stack (Sobek used by Grafana)
- Enables fast iteration (Davy's main goal)

**Next Steps:**
1. Initialize Go project with Sobek dependency
2. Create YAML config parser for sources and variables
3. Implement `plc` global object with dynamic property access
4. Build basic Modbus driver integration
5. Create proof-of-concept: read sensor, run task, write output

**Resources Needed:**
- Go development environment
- Sobek library: `github.com/grafana/sobek`
- YAML parser: Go stdlib `gopkg.in/yaml.v3`
- Modbus library: `github.com/goburrow/modbus`

**Timeline:** 2-3 weeks for MVP

#### #2 Priority: Auto-Discovery Task System

**Rationale:**
- Eliminates boilerplate task registration
- Convention over configuration philosophy
- Simple to implement once core exists
- High impact on developer experience

**Next Steps:**
1. Implement directory watcher for `/tasks` folder
2. JavaScript module loader in Sobek
3. Extract `scanRate` export from task modules
4. Create task scheduler with configurable scan rates
5. Test with multiple tasks at different rates

**Resources Needed:**
- Go file system watching
- Sobek module system understanding
- Task scheduling algorithm (ticker-based)

**Timeline:** 1 week after core architecture

#### #3 Priority: Unified Data Model (GraphQL + Sparkplug B)

**Rationale:**
- Maintains feature parity with Tentacle PLC
- Eliminates duplicate definitions
- Critical for Ignition integration (Sparkplug B)
- Enables real-time monitoring (GraphQL subscriptions)

**Next Steps:**
1. Generate GraphQL schema from YAML config
2. Implement GraphQL resolver for variable queries
3. Add GraphQL subscription support for real-time updates
4. Integrate Sparkplug B library for MQTT publishing
5. Map variables to Sparkplug B metrics (NBIRTH, NDATA)
6. Test with HiveMQ broker and Tentacle UI

**Resources Needed:**
- GraphQL library: `github.com/99designs/gqlgen`
- Sparkplug B library: Research/select or implement
- MQTT client: `github.com/eclipse/paho.mqtt.golang`

**Timeline:** 1-2 weeks after task system

---

## Reflection & Follow-up

### What Worked Well

- First Principles Thinking immediately clarified core requirements and eliminated inherited complexity
- Analogical Thinking (especially web frameworks) provided concrete patterns to borrow
- Examining Davy's C# demo gave real-world validation of "less verbose" approach
- SCAMPER forced systematic exploration beyond initial ideas
- What If scenarios stretched thinking to innovative features (browser IDE, auto-logging)

### Areas for Further Exploration

- **Performance profiling:** Benchmark Sobek execution time with realistic PLC tasks
- **Error handling strategy:** How should runtime errors in JavaScript tasks be handled safely?
- **State persistence:** Should PLC state survive restarts? How?
- **Security model:** Authentication for WebUI/GraphQL, authorization for variable writes
- **Testing approach:** Unit tests for tasks, integration tests for I/O, simulation strategies
- **Documentation generation:** Auto-generate API docs from YAML config
- **Migration path:** How do existing Tentacle PLC users migrate to new system?

### Recommended Follow-up Techniques

- **Prototyping:** Build minimal Go + Sobek spike to validate core architecture
- **User Story Mapping:** Define user journeys for commissioning engineer, operator, developer
- **Competitive Analysis:** Deep dive into similar projects (OpenPLC, Beremiz, etc.)
- **Technical Spike:** Evaluate Sparkplug B libraries and GraphQL code generation

### Questions That Emerged

- What's the best Sparkplug B library for Go? Build custom or use existing?
- How to handle Modbus connection failures gracefully (retry logic, fallback values)?
- Should variables support units of measure conversion (PSI ↔ bar, °F ↔ °C)?
- What's the WebUI technology stack? React (like Tentacle UI) or something lighter?
- How to debug JavaScript tasks? Chrome DevTools protocol integration?
- Should there be a CLI for common operations (`plc init`, `plc validate`, `plc run`)?
- How granular should permissions be (per-variable write control)?
- What logging/telemetry should the runtime produce for operations teams?

### Next Session Planning

**Suggested Topics:**
1. **Technical Architecture Deep-Dive:** Detailed component design, API contracts, data flow
2. **Developer Experience Research:** Interview Davy about pain points, workflow, tooling needs
3. **Implementation Roadmap:** Break down v1.0 into sprints with deliverables
4. **Competitive Analysis:** Study OpenPLC, Beremiz, Node-RED architecture decisions

**Recommended Timeframe:** 1-2 weeks (after initial prototype validates core concepts)

**Preparation Needed:**
- Build Go + Sobek proof-of-concept (validate JavaScript execution)
- Research Sparkplug B Go libraries
- Set up development environment and repository structure
- Create initial project documentation (README, CONTRIBUTING)

---

*Session facilitated using the BMAD-METHOD™ brainstorming framework*
