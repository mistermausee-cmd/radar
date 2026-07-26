# Implementation Plan: MITM Proxy Player Positions

## Overview

This implementation plan breaks down the MITM proxy feature into four phases: Core Infrastructure, Decryption Pipeline, Integration, and Testing & Documentation. The proxy intercepts Photon traffic between the Albion Online client and server to decrypt player positions in real-time.

**Implementation Language:** Go (as specified in design document)

**Total Estimated Effort:** 15-20 development sessions

---

## Phase 1: Core Infrastructure

### 1. Project Structure and Interfaces

- [ ] 1.1 Create MITM package structure and core interfaces
  - Create `internal/mitm/` directory with subdirectories: `proxy/`, `crypto/`, `session/`
  - Define core interfaces: `PacketHandler`, `EventProcessor`, `KeyProvider`
  - Create error types and constants for MITM-specific errors
  - _Requirements: 1.1, 1.5_
  - **Priority: P0** | **Est: 2h**

- [ ] 1.2 Define data models and configuration structures
  - Create `MITMConfig` struct with validation methods
  - Create `SessionState` struct with thread-safe accessors
  - Create `DecryptedPosition` and `KeySyncEvent` models
  - Create `SessionStats` for metrics tracking
  - _Requirements: 11.1, 11.4_
  - **Priority: P0** | **Est: 1.5h**

### 2. UDP Proxy Core

- [ ] 2.1 Implement UDP listener and connection management
  - Create `MITMProxy` struct with configuration
  - Implement `Start(ctx context.Context) error` method
  - Implement `Stop() error` method with graceful shutdown
  - Bind to localhost (127.0.0.1) only for security
  - _Requirements: 1.1, 10.1_
  - **Priority: P0** | **Est: 3h**

- [ ] 2.2 Implement bidirectional packet forwarding
  - Create client-to-server forwarding goroutine
  - Create server-to-client forwarding goroutine
  - Implement packet channel architecture for async processing
  - Preserve packet ordering and timing (<10ms latency)
  - _Requirements: 1.2, 1.3, 1.4, 9.1_
  - **Priority: P0** | **Est: 3h**

- [ ]* 2.3 Write unit tests for UDP proxy core
  - Test packet forwarding with mock connections
  - Test localhost binding constraint
  - Test graceful shutdown behavior
  - _Requirements: 1.1, 1.2, 1.3, 10.1_

- [ ] 2.4 Implement packet parsing and routing
  - Parse Photon packet headers
  - Route packets based on session state (handshake vs established)
  - Handle packet parsing errors gracefully
  - _Requirements: 3.2_
  - **Priority: P0** | **Est: 2h**

### 3. Session State Management

- [ ] 3.1 Implement SessionState with thread-safe key storage
  - Create `SessionState` struct with `sync.RWMutex`
  - Implement `SetAESKey()`, `SetXorCode()`, `GetXorCode()` methods
  - Implement session lifecycle tracking (connected, active, disconnected)
  - Implement `Reset()` for session cleanup
  - _Requirements: 2.5, 8.3_
  - **Priority: P0** | **Est: 2h**

- [ ]* 3.2 Write unit tests for SessionState
  - Test concurrent key access from multiple goroutines
  - Test state transitions
  - Test reset behavior
  - _Requirements: 2.5, 8.3_

- [ ] 3.3 Checkpoint - Core Infrastructure Complete
  - Ensure all tests pass, ask the user if questions arise.
  - **Priority: P0**

---

## Phase 2: Decryption Pipeline

### 4. Diffie-Hellman Key Interception

- [ ] 4.1 Implement DHKeyInterceptor component
  - Create `DHKeyInterceptor` struct with Oakley 768-bit parameters
  - Initialize prime (Oakley 768-bit MODP) and generator (22)
  - Implement client private exponent generation
  - _Requirements: 2.2_
  - **Priority: P0** | **Est: 2h**

- [ ] 4.2 Implement DH handshake detection and processing
  - Implement `ProcessClientHello(data []byte) error`
  - Implement `ProcessServerHello(data []byte) error`
  - Detect DH initiation packets in packet stream
  - _Requirements: 2.1, 2.2_
  - **Priority: P0** | **Est: 3h**

- [ ] 4.3 Implement shared secret computation and AES key derivation
  - Compute shared secret: `S = B^a mod p`
  - Derive AES key: `SHA256(sharedSecret)`
  - Store derived key in session state
  - _Requirements: 2.3, 2.4_
  - **Priority: P0** | **Est: 2h**

- [ ]* 4.4 Write property tests for DH key derivation
  - **Property 5: DH Shared Secret Computation** - Verify B^a mod p computation
  - **Property 6: AES Key Derivation** - Verify SHA256 output is 32 bytes
  - _Requirements: 2.3, 2.4_

- [ ] 4.5 Implement DH handshake timeout and error handling
  - Implement 30-second timeout for handshake completion
  - Log failures and continue in fallback mode
  - Trigger fallback mode on handshake failure
  - _Requirements: 2.6, 8.2_
  - **Priority: P1** | **Est: 1.5h**

### 5. AES-256-CBC Decryption

- [ ] 5.1 Implement AESDecryptor component
  - Create `AESDecryptor` struct with key and IV (16 null bytes)
  - Implement `SetKey(key []byte) error` with key validation
  - Implement `Decrypt(data []byte) ([]byte, error)`
  - Handle CBC padding correctly
  - _Requirements: 3.1, 3.2_
  - **Priority: P0** | **Est: 3h**

- [ ] 5.2 Implement event-specific decryption
  - Implement `DecryptEvent(event *EventData) (*EventData, error)`
  - Parse decrypted event code and parameters
  - Handle decryption failures gracefully
  - _Requirements: 3.2, 3.4_
  - **Priority: P0** | **Est: 2h**

- [ ]* 5.3 Write property tests for AES decryption
  - **Property 7: AES Round-Trip Preservation** - Verify encrypt→decrypt returns original
  - **Property 8: Event Parameter Parsing** - Verify parameter extraction
  - _Requirements: 3.1, 3.2_

- [ ] 5.4 Implement decryption error recovery
  - Handle invalid session key errors with re-derivation attempt
  - Skip corrupted packets and continue processing
  - Log errors with timestamps and error codes
  - _Requirements: 3.3, 3.4, 8.6_
  - **Priority: P1** | **Est: 1.5h**

### 6. XOR Position Decryption

- [ ] 6.1 Implement XORDecryptor component
  - Create `XORDecryptor` struct with XorCode storage
  - Implement `SetXorCode(code []byte)` with 8-byte validation
  - Implement `DecryptFloat(encrypted []byte) float32`
  - _Requirements: 4.1, 5.4, 5.5_
  - **Priority: P0** | **Est: 2h**

- [ ] 6.2 Implement KeySync event processing
  - Implement `ExtractXorCodeFromKeySync(event *EventData) ([]byte, error)`
  - Identify the KeySync event by resolving the real event code from `params[252]` (== `eventcodes.KeySync`), NOT the wire dispatch byte
  - Resolve event codes symbolically via the generated `eventcodes` package — never hardcode (KeySync is currently 600 but shifts across patches)
  - Extract XorCode from Parameters[0] as ByteArray
  - Validate XorCode is exactly 8 bytes
  - Update stored XorCode on each new KeySync event
  - _Requirements: 4.1, 4.2, 4.3_
  - **Priority: P0** | **Est: 2h**

- [ ] 6.3 Implement position decryption for Move events
  - Implement `DecryptPosition(event *EventData) *EventData`
  - Decrypt X coordinate using XorCode bytes 0-3
  - Decrypt Y coordinate using XorCode bytes 4-7
  - Convert to float32 little-endian
  - Note: the Move layout (params[1] offsets 9/13 → params[4]/[5]) is the VERIFIED layout
  - _Requirements: 5.1, 5.2, 5.4, 5.5_
  - **Priority: P0** | **Est: 2h**

- [ ] 6.4 Implement position decryption for NewCharacter events
  - **OPEN ITEM:** the event-29 spawn-position parameter/offset is UNVERIFIED — first identify the correct parameter/offset from a live capture as a prerequisite; do NOT assume the Move layout
  - Implement `DecryptSpawnPosition(event *EventData) *EventData` once the layout is confirmed
  - Decrypt spawn position coordinates using XorCode
  - Inject decrypted positions into Parameters[4] and Parameters[5]
  - Validate decrypted coordinates are finite (not NaN/Inf)
  - _Requirements: 5.3, 5.5_
  - **Priority: P0** | **Est: 1.5h**

- [ ]* 6.5 Write property tests for XOR decryption
  - **Property 9: XorCode Extraction** - Verify 8-byte extraction from the KeySync event
  - **Property 10: XorCode Update** - Verify latest XorCode is stored
  - **Property 11: X Coordinate Decryption** - Verify XOR bytes 0-3
  - **Property 12: Y Coordinate Decryption** - Verify XOR bytes 4-7
  - **Property 14: XOR Round-Trip** - Verify (b XOR k) XOR k = b
  - _Requirements: 4.1, 4.3, 5.1, 5.2, 5.4_

- [ ] 6.6 Checkpoint - Decryption Pipeline Complete
  - Ensure all tests pass, ask the user if questions arise.
  - **Priority: P0**

---

## Phase 3: Integration

### 7. OpenRadar Backend Integration

- [ ] 7.1 Integrate MITM proxy with Capture Manager
  - Add MITM mode to `CaptureManager`
  - Implement mode switching between MITM and passive capture
  - Handle proxy lifecycle (start/stop) from Capture Manager
  - _Requirements: 6.2, 7.2, 7.4_
  - **Priority: P0** | **Est: 3h**

- [ ] 7.2 Connect decrypted events to existing Photon parser
  - Route decrypted events to existing `internal/photon` deserializer
  - Preserve event structure for downstream processing
  - Handle Event 3 (Move) and Event 29 (NewCharacter) specifically
  - Note: event identification for KeySync/NewCharacter must go through `params[252]` (per EventRouter.js / PROTOCOL18_PARAM_LAYOUTS.md), consistent with Protocol18
  - _Requirements: 6.1_
  - **Priority: P0** | **Est: 2h**

- [ ] 7.3 Implement WebSocket broadcasting for decrypted positions
  - Broadcast decrypted position updates to frontend
  - Update Player_Record with new posX and posY values
  - Ensure position appears on radar within 50ms
  - _Requirements: 6.1, 6.2, 6.5_
  - **Priority: P0** | **Est: 2h**

- [ ] 7.4 Implement fallback to passive mode
  - Detect fatal MITM errors and switch to fallback mode
  - Continue passive capture for spawns and identities
  - Maintain player detection without position decryption
  - _Requirements: 7.1, 7.2, 7.3_
  - **Priority: P0** | **Est: 2h**

- [ ]* 7.5 Write integration tests for backend integration
  - Test MITM → passive mode transition
  - Test WebSocket event broadcasting
  - Test position update flow end-to-end
  - _Requirements: 6.1, 6.2, 7.1, 7.2, 7.3_

### 8. Frontend Integration

- [ ] 8.1 Add MITM status indicator to UI
  - Display current mode: Active, Inactive, Error, or Fallback
  - Show session key derivation timestamp when active
  - Add error status with troubleshooting hints
  - _Requirements: 7.5, 11.4, 11.5_
  - **Priority: P1** | **Est: 2h**

- [ ] 8.2 Implement MITM enable/disable toggle in settings
  - Add toggle switch in Settings UI
  - Handle toggle state changes with backend API calls
  - Display confirmation dialog with TOS risk warning
  - _Requirements: 11.1, 11.2, 11.3_
  - **Priority: P1** | **Est: 2h**

- [ ] 8.3 Update PlayersDrawing to render decrypted positions
  - Process decrypted player positions identically to other entities
  - Apply same visual styling as spawn positions
  - Handle position updates in real-time
  - _Requirements: 6.3, 6.4_
  - **Priority: P0** | **Est: 1.5h**

### 9. Error Handling and Recovery

- [ ] 9.1 Implement comprehensive error handling
  - Handle port binding failure with user-friendly message
  - Handle DH handshake timeout with session reset
  - Handle AES decryption failure with forwarding continuation
  - Handle XorCode missing with position marking as encrypted
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.6_
  - **Priority: P1** | **Est: 3h**

- [ ] 9.2 Implement auto-recovery mechanisms
  - Re-intercept DH exchange on next connection attempt
  - Restart proxy automatically within 5 seconds on crash
  - Clear session state on client disconnect
  - Reinitialize on client reconnect
  - _Requirements: 8.1, 8.3, 8.4, 8.5_
  - **Priority: P1** | **Est: 2h**

- [ ] 9.3 Implement logging and diagnostics
  - Log all errors with timestamps and error codes
  - Log session key derivation events (without key values)
  - Log XorCode updates (hex format for debugging)
  - _Requirements: 8.6_
  - **Priority: P2** | **Est: 1.5h**

- [ ] 9.4 Checkpoint - Integration Complete
  - Ensure all tests pass, ask the user if questions arise.
  - **Priority: P0**

---

## Phase 4: Testing & Documentation

### 10. Performance Testing

- [ ] 10.1 Implement performance benchmarks
  - Benchmark packet forwarding latency (target: <1ms)
  - Benchmark decryption overhead (target: <0.5ms)
  - Benchmark total additional latency (target: <10ms)
  - _Requirements: 9.1_
  - **Priority: P1** | **Est: 2h**

- [ ] 10.2 Implement throughput benchmarks
  - Benchmark events per second processing (target: 500+)
  - Benchmark WebSocket broadcast rate (target: 100+/sec)
  - Benchmark packet handling without loss
  - _Requirements: 9.4_
  - **Priority: P1** | **Est: 2h**

- [ ] 10.3 Implement resource usage profiling
  - Profile memory usage (target: <100MB)
  - Profile CPU usage (target: <10% on quad-core 2.5GHz)
  - Identify and fix memory leaks
  - _Requirements: 9.2, 9.3_
  - **Priority: P1** | **Est: 2h**

### 11. Edge Case and Security Testing

- [ ]* 11.1 Write edge case tests
  - Test corrupted packet data handling
  - Test missing XorCode scenario
  - Test invalid XorCode length in the KeySync event
  - Test malformed KeySync event parameters
  - _Requirements: 3.4, 4.4, 5.6, 8.6_

- [ ]* 11.2 Write security verification tests
  - **Property 15: Packet Content Preservation** - Verify forwarded content equals original or properly decrypted
  - **Property 16: Localhost Binding** - Verify listener binds only to 127.0.0.1
  - Test no plain-text data logged to disk
  - _Requirements: 10.1, 10.2, 10.5_

### 12. Documentation

- [ ] 12.1 Create technical documentation
  - Document MITM architecture and components
  - Document DH key derivation process
  - Document encryption/decryption flow
  - Add diagrams for troubleshooting
  - _Priority: P2_ | **Est: 2h**

- [ ] 12.2 Create user documentation
  - Write setup guide for enabling MITM mode
  - Document TOS risk warning and implications
  - Create troubleshooting guide for common issues
  - Document error status meanings
  - _Requirements: 11.1, 11.2, 11.3_
  - **Priority: P2** | **Est: 2h**

- [ ] 12.3 Final checkpoint - All phases complete
  - Ensure all tests pass, ask the user if questions arise.
  - **Priority: P0**

---

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties
- Unit tests validate specific examples and edge cases
- Priority levels: **P0** (critical), **P1** (important), **P2** (nice-to-have)
- Event codes are resolved symbolically via the generated `eventcodes` package (KeySync currently 600); Albion shifts codes across patches — never hardcode.

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2"] },
    { "id": 1, "tasks": ["2.1", "3.1"] },
    { "id": 2, "tasks": ["2.2", "2.4", "3.2"] },
    { "id": 3, "tasks": ["2.3", "4.1"] },
    { "id": 4, "tasks": ["4.2", "4.3"] },
    { "id": 5, "tasks": ["4.4", "4.5", "5.1"] },
    { "id": 6, "tasks": ["5.2", "5.4"] },
    { "id": 7, "tasks": ["5.3", "6.1"] },
    { "id": 8, "tasks": ["6.2", "6.3"] },
    { "id": 9, "tasks": ["6.4", "6.5"] },
    { "id": 10, "tasks": ["7.1"] },
    { "id": 11, "tasks": ["7.2", "7.3", "7.4"] },
    { "id": 12, "tasks": ["7.5", "8.1", "8.2"] },
    { "id": 13, "tasks": ["8.3", "9.1", "9.2"] },
    { "id": 14, "tasks": ["9.3", "10.1", "11.1"] },
    { "id": 15, "tasks": ["10.2", "10.3", "11.2", "12.1", "12.2"] }
  ]
}
```

## Summary

| Phase | Tasks | Critical (P0) | Important (P1) | Optional (*) |
|-------|-------|---------------|----------------|--------------|
| Phase 1: Core Infrastructure | 9 | 6 | 0 | 2 |
| Phase 2: Decryption Pipeline | 14 | 9 | 1 | 4 |
| Phase 3: Integration | 11 | 6 | 4 | 1 |
| Phase 4: Testing & Documentation | 8 | 0 | 5 | 3 |
| **Total** | **42** | **21** | **10** | **10** |

**Minimum MVP Path (P0 tasks only):** ~21 tasks, ~40 hours estimated effort
