# Project Brief: go-plc

**Version:** 1.0
**Date:** 2025-10-13
**Author:** Andrew Ott

---

## Executive Summary

**go-plc** is a simple, fast soft PLC that eliminates verbose configuration through direct property access (`plc.pressure = 42`). It combines a Go core with JavaScript tasks, YAML config, and auto-discovery patterns to enable sub-minute iteration cycles. Core features: Modbus I/O, Sparkplug B publishing, GraphQL API, and basic WebUI. Target users: automation engineers frustrated with traditional PLC verbosity and IoT developers needing industrial protocol support.

---

## Problem Statement

Existing PLC development is too verbose and slow:

**Current pain points:**
- Verbose patterns: `variables.count.value++` instead of `plc.count++`
- Explicit registration boilerplate for tasks and variables
- 15-30 minute iteration cycles (compile → deploy → test)
- Duplicate definitions across GraphQL, MQTT, and runtime

**Why it matters:**
- Slow commissioning and field debugging
- Poor AI code generation due to complexity
- High barrier for new developers

---

## Proposed Solution

**Architecture:** YAML config + JavaScript tasks + Go runtime

```yaml
# config.yaml - Define once, use everywhere
sources:
  remoteIO: { type: modbus, host: 192.168.1.10 }

variables:
  pressure: { source: remoteIO.holding.2000, scale: [0,32000,0,100] }
  pumpOn: { source: remoteIO.coil.2502 }
```

```javascript
// tasks/pump-control.js - Auto-discovered, direct access
export const scanRate = 100;
export function execute() {
  plc.pumpOn = (plc.pressure > 5); // That's it!
}
```

**Key features:**
- **Direct access:** `plc.pressure = 42` (no `.value` noise)
- **Auto-discovery:** Drop files in `/tasks`, they just work
- **Single definition:** Variables auto-published to GraphQL + Sparkplug B
- **Fast iteration:** JavaScript tasks, no recompilation
- **Proven tech:** Go + Sobek (used by Grafana)

---

## Target Users

**Primary:** Automation engineers tired of verbose PLC programming (Ladder Logic, Structured Text)
- Want fast field debugging and modern version control (Git)
- Need cloud integration (MQTT, GraphQL) without complexity

**Secondary:** IoT/edge developers needing industrial protocols
- Know JavaScript, want to avoid learning IEC 61131-3
- Need Modbus/OPC UA without deep protocol expertise

---

## Goals & Success Metrics

**MVP Success = Complete example project running end-to-end**

**Key metrics:**
- Iteration cycle: <1 minute (edit → test)
- Time to "Hello World": <30 minutes for new user
- Code reduction: 80% less boilerplate vs. Tentacle PLC
- Performance: <50µs task execution, <10ms GraphQL queries

---

## MVP Scope

### Must Have (v1.0)

1. **YAML config parser** - Sources (Modbus TCP only), variables with scaling
2. **Modbus TCP driver** - Read/write holding registers and coils
3. **JavaScript task engine (Sobek)** - Auto-discover from `/tasks`, expose `plc` object
4. **Direct property access** - `plc.variable = value` with auto-sync
5. **Sparkplug B publishing** - NBIRTH + NDATA for Ignition integration
6. **GraphQL API** - Query + subscription for current values
7. **Simple WebUI** - Live variable display
8. **CLI** - `plc run config.yaml`

### Explicitly Out (Keep it simple!)

- OPC UA, Ethernet/IP, other protocols
- Browser IDE
- Variable history/trending
- Auto file watching (manual reload only)
- Authentication
- Advanced error recovery

**Done = Working example + README + <30min quickstart**

---

## Future Ideas (Not MVP)

**Phase 2 (if MVP succeeds):**
- Auto file watching (`plc dev config.yaml`)
- OPC UA support
- Browser IDE for remote commissioning
- Variable history/trending

**Later:**
- Multi-protocol gateway
- AI code generation
- Community task library

---

## Technical Stack

**Backend:**
- Go 1.22+ (runtime, I/O drivers, APIs)
- Sobek for JavaScript execution (github.com/grafana/sobek)
- gqlgen for GraphQL (github.com/99designs/gqlgen)
- paho.mqtt.golang for Sparkplug B
- goburrow/modbus for Modbus TCP

**Frontend:**
- React + Apollo Client (GraphQL subscriptions)
- Simple dashboard only for MVP

**Architecture:**
- Single binary (Go embed for WebUI)
- In-memory state (no persistence in MVP)
- Goroutines for concurrent I/O + task execution

---

## Constraints & Assumptions

**Constraints:**
- Solo developer, part-time (~20 hrs/week)
- 2-3 month MVP timeline
- Must maintain Tentacle PLC feature parity (Sparkplug B, Modbus, GraphQL)

**Key assumptions:**
- Sobek is fast enough (<50µs task execution) - **needs validation**
- Users prefer JavaScript over Ladder Logic
- YAML scales to 100+ variables
- Non-safety-critical deployments for MVP

---

## Risks & Open Questions

**Top risks:**
1. **Performance:** Sobek too slow for real-time control → Mitigation: Benchmark early
2. **Adoption:** PLC programmers resist JavaScript → Mitigation: Target IoT devs first
3. **Scope creep:** Adding features delays MVP → Mitigation: Strict scope discipline

**Open questions:**
- Best Sparkplug B library for Go? (Eclipse Tahu vs. custom)
- Modbus error handling strategy? (retry logic, fallback values)
- How to debug JavaScript tasks? (DevTools integration?)
- Init task pattern needed?

---

## References

**Based on:** [Brainstorming session 2025-10-13](brainstorming-session-results.md)

**Key libraries:**
- Sobek (JS runtime): https://github.com/grafana/sobek
- Sparkplug B: https://sparkplug.eclipse.org/
- goburrow/modbus: https://github.com/goburrow/modbus
- gqlgen: https://github.com/99designs/gqlgen

**Inspiration:**
- Davy Demers' C# PLC demo (direct property access pattern)
- Web framework ORMs (Django, Rails) for variable mapping
- Game engine hot-reload patterns

---

## Next Steps

1. **Benchmark Sobek** - Validate <50µs task execution assumption
2. **Spike: Go + Sobek + Modbus** - Proof of concept (1 week)
3. **Design YAML schema** - Finalize config structure
4. **Setup repo** - Init Go project, basic structure
5. **Build MVP** - 8 core features, 2-3 months
6. **Beta test** - 3 users, gather feedback

**Ready for PRD** - This brief provides context for detailed requirements doc
