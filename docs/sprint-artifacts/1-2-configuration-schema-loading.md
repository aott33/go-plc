# Story 1.2: Configuration Schema & Loading

**Status:** ready-for-dev
**Epic:** 1 - Project Foundation & Core Runtime
**Estimated Effort:** Medium (3-4 hours)
**Priority:** Critical Path - Blocks Stories 1.3-1.8

---

## Overview

This story implements the YAML configuration parsing and validation system for go-plc. You'll create the configuration structs that define sources (Modbus devices) and variables, implement a loader that parses and validates the configuration, and integrate it into the main application startup sequence.

**Why This Matters:**
Configuration is the foundation of go-plc. Every other component (variable store, Modbus client, OPC UA server, GraphQL API) depends on this configuration. Getting the schema right now prevents painful migrations later. The type-discriminated source pattern enables future protocol additions without breaking changes.

---

## Context from Previous Work

**Module Path:** `github.com/aott33/go-plc` (defined in go.mod)

**Current main.go Structure:**
The entry point (`cmd/go-plc/main.go`) already has slog configured with JSON output. Your config loading will be called early in the startup sequence - after flag parsing but before variable store initialization. The existing main.go comments outline where config loading fits:
```
// parse command line flags
// load and validate yaml file    <-- THIS STORY
// initialize variable store
// connect to sources
// ...
```

**Development Environment Notes from Story 1.1:**
- Go version is 1.25 (verify with `go version`)
- Windows users: Makefile requires bash - use WSL, Git Bash, or VSCode integrated terminal
- `go mod tidy` should be run after adding yaml.v3 dependency

---

## User Story

> **As a** developer,
> **I want** to define sources and variables in a YAML configuration file that is parsed and validated on startup,
> **So that** I can configure the PLC without code changes and catch configuration errors early.

---

## Acceptance Criteria

### AC1: YAML Configuration Parsing with Type-Discriminated Sources

**Given** a YAML configuration file with the architecture-mandated schema
**When** the application starts with `-config path/to/config.yaml`
**Then** sources are parsed with protocol-specific config blocks (modbus-tcp, modbus-rtu)
**And** variables are parsed with source references, register definitions, and optional scaling

### AC2: Configuration Validation with Clear Error Messages

**Given** a YAML file with invalid configuration (missing required field, invalid type, unknown source reference)
**When** the application attempts to load the config
**Then** a clear, human-readable error message is logged in format: `[config] - [description] (context: [details])`
**And** the application exits with non-zero status before starting the runtime

### AC3: Successful Configuration Loading

**Given** a valid configuration
**When** config loading completes
**Then** all sources and variables are accessible via typed Go structs
**And** INFO level log confirms "Configuration loaded successfully"

---

## Technical Requirements

### Architecture Compliance

You MUST follow these patterns from the architecture document:

**Naming Conventions:**
- Package name: `config` (lowercase, single word)
- File names: `config.go`, `loader.go`, `types.go`, `config_test.go`
- Exported types: `PascalCase` (e.g., `Config`, `Source`, `Variable`)
- Unexported: `camelCase` (e.g., `validateSources`, `loadFile`)
- JSON/YAML tags: `camelCase` (e.g., `json:"pollInterval"` NOT `json:"poll_interval"`)

**Error Message Format:**
```
[Component] - [Human description] (context: [details])
```
Examples:
- `[config] - Missing required field 'host' (source: remoteIO, type: modbus-tcp)`
- `[config] - Unknown source reference (variable: tankLevel, source: unknownDevice)`

**Logging:**
- Use `log/slog` exclusively (NOT `log`, `logrus`, `zap`, or `fmt.Println`)
- JSON structured format
- Appropriate log levels: INFO for success, ERROR for failures

### YAML Configuration Schema

The architecture mandates this exact schema structure. Study it carefully - your structs must match this exactly:

```yaml
# Application-level settings
logLevel: info  # debug, info, warn, error

# Sources with type-discriminated config blocks
sources:
  - name: remoteIO              # Unique identifier
    type: modbus-tcp            # Protocol type
    config:                     # Protocol-specific block
      host: 192.168.1.100
      port: 502
      unitId: 1
      timeout: 1s
      pollInterval: 100ms
      retryInterval: 5s
      byteOrder: big-endian     # big-endian or little-endian
      wordOrder: high-word-first # high-word-first or low-word-first

  - name: serialDevice
    type: modbus-rtu
    config:
      device: /dev/ttyUSB0
      baudRate: 19200
      dataBits: 8
      parity: none              # none, even, odd
      stopBits: 2
      unitId: 1
      timeout: 1s
      pollInterval: 100ms

# Flat variable list with source references
variables:
  - name: tankLevel
    source: remoteIO            # References source by name
    register:
      type: holding             # holding, input, coil, discrete
      address: 0                # 0-based offset
    dataType: uint16            # See supported types below
    scale:                      # Optional - omit for raw values
      rawMin: 0
      rawMax: 65535
      engMin: 0
      engMax: 100
      unit: "%"
    tags: [tank1, level]        # Optional - for filtering/grouping

  - name: pumpRunning
    source: remoteIO
    register:
      type: coil
      address: 0
    dataType: bool
    tags: [tank1, pump]
```

### Supported Data Types

From the simonvetter/modbus library - you'll need these for validation:

| dataType | Registers | Description |
|----------|-----------|-------------|
| `bool` | N/A | Coils and discrete inputs only |
| `uint16` | 1 | Unsigned 16-bit integer |
| `int16` | 1 | Signed 16-bit integer |
| `uint32` | 2 | Unsigned 32-bit integer |
| `int32` | 2 | Signed 32-bit integer |
| `float32` | 2 | 32-bit floating point |
| `uint64` | 4 | Unsigned 64-bit integer |
| `int64` | 4 | Signed 64-bit integer |
| `float64` | 4 | 64-bit floating point |

### Register Types

| type | Function Codes | Use Case |
|------|----------------|----------|
| `holding` | FC03/FC06/FC16 | Read/write registers |
| `input` | FC04 | Read-only registers |
| `coil` | FC01/FC05/FC15 | Read/write single bits |
| `discrete` | FC02 | Read-only single bits |

---

## Implementation Guide

### Step 1: Create the Configuration Types

**Location:** `internal/config/types.go`

**What to implement:**

1. **Create the root Config struct** containing:
   - `LogLevel` field (string with validation for debug/info/warn/error)
   - `Sources` slice
   - `Variables` slice

2. **Implement the type-discriminated Source struct** using Go's type embedding or interface pattern:
   - Think about how to handle different protocol configs (modbus-tcp vs modbus-rtu)
   - Option A: Use `map[string]interface{}` for the config block and type-assert based on `Type`
   - Option B: Use embedded structs with yaml unmarshaling hooks
   - Option C: Define separate structs and use a custom UnmarshalYAML method

   **Guidance:** Option C (custom UnmarshalYAML) is cleanest for type safety. Your Source struct should hold the protocol-specific config in a typed struct after unmarshaling.

3. **Create ModbusTCPConfig struct** with fields:
   - `Host` (string, required)
   - `Port` (int, default 502)
   - `UnitId` (uint8, default 1)
   - `Timeout` (time.Duration, default 1s)
   - `PollInterval` (time.Duration, default 100ms)
   - `RetryInterval` (time.Duration, default 5s)
   - `ByteOrder` (string: "big-endian" or "little-endian")
   - `WordOrder` (string: "high-word-first" or "low-word-first")

4. **Create ModbusRTUConfig struct** with fields:
   - `Device` (string, required - e.g., "/dev/ttyUSB0")
   - `BaudRate` (int, required)
   - `DataBits` (int, default 8)
   - `Parity` (string: "none", "even", "odd")
   - `StopBits` (int, default 1)
   - `UnitId` (uint8, default 1)
   - `Timeout` (time.Duration)
   - `PollInterval` (time.Duration)

5. **Create Variable struct** with fields:
   - `Name` (string, required, unique)
   - `Source` (string, required - must reference existing source)
   - `Register` (RegisterConfig)
   - `DataType` (string - must be one of the supported types)
   - `Scale` (optional *ScaleConfig)
   - `Tags` ([]string, optional)

6. **Create RegisterConfig struct:**
   - `Type` (string: "holding", "input", "coil", "discrete")
   - `Address` (uint16 - 0-based offset)

7. **Create ScaleConfig struct:**
   - `RawMin`, `RawMax`, `EngMin`, `EngMax` (float64)
   - `Unit` (string - display unit like "%", "PSI", "degC")

**Key Decision Point:**
How will you validate that `bool` dataType is only used with `coil` or `discrete` register types? Consider building a validation map or switch statement.

### Step 2: Implement the Configuration Loader

**Location:** `internal/config/loader.go`

**What to implement:**

1. **Create a Load function** with signature:
   ```go
   func Load(path string) (*Config, error)
   ```

2. **File reading and YAML parsing:**
   - Use `os.ReadFile` to read the config file
   - Use `gopkg.in/yaml.v3` for YAML parsing (add to go.mod)
   - Handle file-not-found with clear error message

3. **Implement custom UnmarshalYAML** for the Source struct:
   - First unmarshal to get the `type` field
   - Based on type, unmarshal the `config` block into the appropriate struct
   - Store the typed config in the Source struct

   **Hint:** You might need an intermediate struct like:
   ```go
   type rawSource struct {
       Name   string
       Type   string
       Config yaml.Node  // Delay parsing until we know the type
   }
   ```

4. **Apply default values** after unmarshaling:
   - Modbus TCP port defaults to 502
   - UnitId defaults to 1
   - Timeout defaults to 1s
   - PollInterval defaults to 100ms
   - RetryInterval defaults to 5s

### Step 3: Implement Validation

**Location:** `internal/config/loader.go` (or separate `validation.go`)

**Validation rules to implement:**

1. **Source validation:**
   - Name must be non-empty and unique across all sources
   - Type must be "modbus-tcp" or "modbus-rtu"
   - For modbus-tcp: host is required
   - For modbus-rtu: device and baudRate are required
   - ByteOrder must be "big-endian" or "little-endian" (default big-endian)
   - WordOrder must be "high-word-first" or "low-word-first" (default high-word-first)

2. **Variable validation:**
   - Name must be non-empty and unique across all variables
   - Source must reference an existing source name
   - DataType must be one of the supported types
   - Register type must be valid ("holding", "input", "coil", "discrete")
   - Boolean dataType can only be used with "coil" or "discrete" registers
   - Multi-register types (uint32, float32, etc.) should only be used with "holding" or "input"

3. **Scaling validation (if present):**
   - rawMin < rawMax
   - engMin and engMax must be set if any scaling field is set

4. **LogLevel validation:**
   - Must be "debug", "info", "warn", or "error"
   - Default to "info" if not specified

**Error reporting guidance:**
Create a validation function that collects ALL errors, not just the first one. Return a multi-error so the user can fix all problems at once. Consider using a pattern like:

```go
var errors []error
// ... validation checks that append to errors
if len(errors) > 0 {
    return combineErrors(errors)
}
```

### Step 4: Integrate with Main Application

**Location:** `cmd/go-plc/main.go`

**Current State:** The existing main.go already initializes slog with JSON output. You'll be adding to this file, not replacing it.

**What to implement:**

1. **Add command-line flag parsing:**
   - Add `-config` flag for config file path
   - Use Go's standard `flag` package (no external dependencies)
   - Example: `./go-plc -config config.yaml`
   - Place flag parsing after slog setup, before config loading

2. **Update main() function:**
   - Parse the config flag at the start (after logger setup)
   - If no config provided, log error and exit with `os.Exit(1)`
   - Import your config package: `"github.com/aott33/go-plc/internal/config"`
   - Call `config.Load(configPath)`
   - On error: log structured error and exit with code 1
   - On success: log "Configuration loaded successfully" with source count and variable count

3. **Structured logging on success:**
   ```go
   slog.Info("Configuration loaded successfully",
       "sources", len(cfg.Sources),
       "variables", len(cfg.Variables),
       "logLevel", cfg.LogLevel,
   )
   ```

4. **Import path reminder:** Your import will be `github.com/aott33/go-plc/internal/config` (module path from go.mod + package path)

### Step 5: Create Example Configuration File

**Location:** `config.yaml` (project root)

Create a valid example configuration file that demonstrates:
- At least one modbus-tcp source
- At least one modbus-rtu source (commented out if you don't have serial hardware)
- Variables of different types (uint16, bool, float32)
- Variables with and without scaling
- Variables with different register types

This file serves as documentation and testing reference.

### Step 6: Write Unit Tests

**Location:** `internal/config/config_test.go`

**Test cases to implement:**

1. **Happy path tests:**
   - Load valid config with all fields
   - Load minimal valid config (only required fields)
   - Verify default values are applied
   - Verify all fields are correctly parsed

2. **Error case tests:**
   - Missing config file
   - Invalid YAML syntax
   - Missing required source fields (name, type, host)
   - Unknown source type
   - Variable referencing non-existent source
   - Invalid dataType
   - Bool dataType with holding register (should fail)
   - Duplicate source names
   - Duplicate variable names
   - Invalid logLevel value

3. **Test table pattern:**
   Use Go's table-driven tests for validation cases:
   ```go
   func TestValidation(t *testing.T) {
       tests := []struct {
           name    string
           yaml    string
           wantErr string
       }{
           // ... test cases
       }
       for _, tt := range tests {
           t.Run(tt.name, func(t *testing.T) {
               // ... test implementation
           })
       }
   }
   ```

---

## File Structure After Completion

```
internal/config/
├── doc.go          # Already exists from Story 1.1
├── types.go        # Config struct definitions
├── loader.go       # Load() function and custom unmarshaling
└── config_test.go  # Unit tests

cmd/go-plc/
└── main.go         # Updated with flag parsing and config loading

config.yaml         # Example configuration file (project root)
go.mod              # Updated with gopkg.in/yaml.v3 dependency
```

---

## Dependencies to Add

You'll need to add the YAML parsing library:

```bash
go get gopkg.in/yaml.v3
```

This is the recommended YAML library for Go. It supports the `yaml.Node` type which is useful for delayed/custom unmarshaling of type-discriminated configs.

---

## Common Pitfalls to Avoid

1. **Don't use `map[string]interface{}`** for the final config representation. Type-assert and convert to typed structs during unmarshaling for compile-time safety.

2. **Don't validate in the Load function**. Separate parsing from validation - parse first, then validate the parsed result. This makes testing easier.

3. **Don't ignore time.Duration parsing**. YAML strings like "100ms" need special handling. Consider using `yaml.v3`'s ability to unmarshal into `time.Duration` or write a custom type.

4. **Don't forget byte order defaults**. If byteOrder/wordOrder aren't specified, they should default to big-endian/high-word-first (standard Modbus convention).

5. **Don't exit on first error**. Collect all validation errors and report them together. Users hate fixing one error only to discover another.

6. **Handle time.Duration correctly**. YAML strings like "100ms" or "5s" don't automatically unmarshal to `time.Duration`. You have two options:

   **Option A: Custom Duration Type**
   ```go
   type Duration time.Duration

   func (d *Duration) UnmarshalYAML(value *yaml.Node) error {
       var s string
       if err := value.Decode(&s); err != nil {
           return err
       }
       duration, err := time.ParseDuration(s)
       if err != nil {
           return err
       }
       *d = Duration(duration)
       return nil
   }
   ```

   **Option B: String field with post-parse conversion**
   Store as string in config struct, then parse to `time.Duration` in a separate validation/conversion step.

   Option A is cleaner - the duration is typed correctly from the start.

---

## Testing Your Implementation

### Manual Testing Steps

1. **Build and run with valid config:**
   ```bash
   make build
   ./go-plc -config config.yaml
   ```
   Expected: "Configuration loaded successfully" log message

2. **Run with missing config flag:**
   ```bash
   ./go-plc
   ```
   Expected: Error message about missing config

3. **Run with invalid config:**
   Create a `bad-config.yaml` with errors (missing host, invalid type, etc.)
   ```bash
   ./go-plc -config bad-config.yaml
   ```
   Expected: Clear error messages for each problem

### Automated Testing

```bash
go test -v ./internal/config/...
go test -race ./internal/config/...
```

---

## Definition of Done Checklist

- [ ] `internal/config/types.go` created with all struct definitions
- [ ] `internal/config/loader.go` created with Load() function
- [ ] Custom YAML unmarshaling works for type-discriminated sources
- [ ] All validation rules implemented with clear error messages
- [ ] Default values applied correctly
- [ ] `cmd/go-plc/main.go` updated with `-config` flag
- [ ] `config.yaml` example file created in project root
- [ ] Unit tests cover happy paths and error cases
- [ ] All tests pass with `go test -race ./...`
- [ ] `go vet ./...` passes with no warnings
- [ ] Logging uses slog with correct format
- [ ] Error messages follow `[Component] - [Description] (context)` format

---

## References

- [Architecture: Configuration Schema](docs/architecture.md#data-architecture) - The authoritative schema specification
- [Architecture: Naming Patterns](docs/architecture.md#naming-patterns) - Naming conventions
- [Architecture: Error Message Format](docs/architecture.md#format-patterns) - Error formatting rules
- [yaml.v3 Documentation](https://pkg.go.dev/gopkg.in/yaml.v3) - YAML library reference
- [Previous Story 1.1](docs/sprint-artifacts/1-1-project-initialization-structure.md) - Project structure context

---

## Next Steps After This Story

Once this story is complete, Story 1.3 (Variable Store Implementation) will use these config structs to initialize the variable store. The `Variable` and `ScaleConfig` structs you create here will be passed to the variable store for registration.

---

## Dev Agent Record

### Agent Model Used
Claude Opus 4.5 (claude-opus-4-5-20251101)

### Completion Notes
- Story created by create-story workflow
- Optimized for human developer implementation (detailed steps, not code)
- All architecture patterns incorporated from docs/architecture.md
- Previous story (1.1) context included

### Context References
- docs/architecture.md - Primary technical reference
- docs/epics.md - Story requirements and acceptance criteria
- docs/sprint-artifacts/1-1-project-initialization-structure.md - Previous story context
