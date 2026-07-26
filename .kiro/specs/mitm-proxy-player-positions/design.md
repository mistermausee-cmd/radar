# Design Document: MITM Proxy for Player Position Decryption

## Overview

OpenRadar currently operates as a passive network capture tool that cannot display live player positions because they are encrypted with a double layer: Photon Protocol18 AES-256-CBC encryption on all UDP 5056 traffic, and an additional Albion XOR encryption on position coordinates. The XorCode needed for position decryption is transmitted via the KeySync event (currently event code 600; Albion shifts event codes across patches — always resolve via the generated `eventcodes` package, never hardcode), which is itself AES-encrypted. This two-layer encryption model is confirmed by the project's own analysis in `docs/technical/PLAYER_POSITIONS_MITM.md`.

This design introduces a Photon Protocol18 Man-in-the-Middle (MITM) proxy that intercepts the Diffie-Hellman key exchange between the Albion client and server, derives the AES session key, decrypts the KeySync event to extract the XorCode, and decrypts player positions in Move (dispatch byte 3) and NewCharacter (real event code 29) events. The proxy integrates with the existing OpenRadar architecture while maintaining backward compatibility with passive capture mode.

Under Protocol18 the wire carries only two dispatch bytes — `3` (Move, the hot path) and `1` (generic) — while the authoritative Albion event code is carried in `params[252]` as an int16 for events (and `params[253]` for operations). The frontend router (`web/scripts/core/EventRouter.js`) dispatches on `params[252]`. Consequently, events such as KeySync and NewCharacter cannot be identified from the wire dispatch byte alone; the proxy must read the real code from `params[252]`.

## Architecture

```mermaid
graph TB
    subgraph "Albion Client"
        AC[Albion Online Client]
    end
    
    subgraph "OpenRadar MITM Proxy"
        direction TB
        UDP[UDP Listener<br/>Port 5056]
        MITM[MITM Proxy Core]
        DH[DH Key Interceptor]
        AES[AES Decryptor]
        XOR[XOR Decryptor]
        
        UDP --> MITM
        MITM --> DH
        DH --> AES
        AES --> XOR
    end
    
    subgraph "Albion Server"
        AS[Albion Game Server<br/>UDP 5056]
    end
    
    subgraph "OpenRadar Backend"
        CAP[Capture Manager]
        PHOTON[Photon Parser]
        WS[WebSocket Broadcaster]
    end
    
    subgraph "OpenRadar Frontend"
        PD[PlayersDrawing.js]
        PH[PlayersHandler.js]
    end
    
    AC <-->|UDP 5056| UDP
    MITM <-->|Forwarded UDP| AS
    XOR -->|Decrypted Events| PHOTON
    PHOTON --> WS
    WS --> PH
    PH --> PD
    
    CAP -.->|Passive Mode Fallback| PHOTON
```

### Component Overview

| Component | Purpose |
|-----------|---------|
| UDP Listener | Binds to port 5056, intercepts client packets |
| MITM Proxy Core | Routes packets between client and server |
| DH Key Interceptor | Captures Diffie-Hellman key exchange |
| AES Decryptor | Derives session key, decrypts Photon events |
| XOR Decryptor | Applies XorCode to decrypt position coordinates |

## Sequence Diagrams

### Connection Establishment and Key Derivation

```mermaid
sequenceDiagram
    participant C as Albion Client
    participant P as MITM Proxy
    participant S as Albion Server
    
    Note over C,S: Initial Connection
    C->>P: UDP connect to localhost:5056
    P->>S: Forwarded connection to server:5056
    P->>C: ACK
    
    Note over C,S: Diffie-Hellman Key Exchange
    C->>P: DH ClientHello (public key A, generator g, prime p)
    P->>P: Store client public key A
    P->>S: Forward DH ClientHello
    
    S->>P: DH ServerHello (public key B)
    P->>P: Store server public key B
    P->>P: Compute shared secret: S = B^a mod p
    P->>P: Derive AES key: SHA256(S)
    P->>C: Forward DH ServerHello
    
    Note over C,S: Session Established
    C->>P: Encrypted Photon traffic
    P->>P: Decrypt with AES-256-CBC
    P->>S: Forward decrypted traffic
```

### KeySync Event Processing

```mermaid
sequenceDiagram
    participant S as Albion Server
    participant P as MITM Proxy
    participant D as XOR Decryptor
    participant W as WebSocket
    
    Note over S,W: KeySync Event Reception
    S->>P: Generic event (dispatch byte 1) - AES encrypted
    P->>P: Decrypt AES layer
    P->>P: Read real code from params[252] (int16)
    P->>P: real code == eventcodes.KeySync (currently 600)?
    P->>P: Parse KeySync parameters
    
    Note over P: Extract XorCode
    P->>P: Parameters[0] = XorCode (8 bytes)
    P->>D: Store XorCode for session
    
    Note over S,W: Position Events
    S->>P: Move (dispatch byte 3) - AES + XOR encrypted
    P->>P: Decrypt AES layer
    P->>D: XOR decrypt player positions
    D->>W: Broadcast decrypted Move event
    
    S->>P: Generic event (dispatch byte 1) - AES encrypted
    P->>P: Read real code from params[252] (int16)
    P->>P: real code == eventcodes.NewCharacter (29)?
    P->>D: XOR decrypt spawn position (OPEN ITEM - layout unverified)
    D->>W: Broadcast decrypted NewCharacter event
```

### Passive vs MITM Mode Selection

```mermaid
sequenceDiagram
    participant U as User
    participant UI as Settings UI
    participant M as Capture Manager
    participant P as MITM Proxy
    participant C as Capturer
    
    U->>UI: Toggle MITM Mode
    UI->>M: SetMode(mitm)
    
    alt MITM Mode Enabled
        M->>P: Start MITM Proxy
        P->>P: Bind UDP 5056
        P->>P: Enable DH interception
        M->>C: Stop passive capture
    else Passive Mode
        M->>P: Stop MITM Proxy
        P->>P: Release UDP 5056
        M->>C: Start passive capture
    end
```

## Components and Interfaces

### Component 1: MITMProxy

**Purpose**: Core proxy that intercepts and forwards UDP traffic between client and server.

**Interface**:
```go
// MITMProxy handles packet interception and forwarding
type MITMProxy struct {
    config       ProxyConfig
    conn         *net.UDPConn
    serverAddr   *net.UDPAddr
    session      *SessionState
    packetChan   chan []byte
    forwardChan  chan []byte
}

type ProxyConfig struct {
    ListenPort    int    // Default: 5056
    ServerHost    string // Albion server hostname
    ServerPort    int    // Default: 5056
    EnableAES     bool   // Enable AES decryption
    EnableXOR     bool   // Enable XOR decryption
}

func (p *MITMProxy) Start(ctx context.Context) error
func (p *MITMProxy) Stop() error
func (p *MITMProxy) OnPacket(handler PacketHandler)
```

**Responsibilities**:
- Listen on UDP 5056 for client connections
- Forward packets to actual Albion server
- Route decrypted events to existing WebSocket handler
- Manage session state and key lifecycle

### Component 2: DHKeyInterceptor

**Purpose**: Captures and processes Diffie-Hellman key exchange to derive session key.

**Interface**:
```go
// DHKeyInterceptor handles Diffie-Hellman key exchange interception
type DHKeyInterceptor struct {
    prime      *big.Int  // Oakley 768-bit prime
    generator  *big.Int  // Generator: 22
    clientPub  *big.Int  // Client public key A
    serverPub  *big.Int  // Server public key B
    sharedKey  *big.Int  // Shared secret S = B^a mod p
    aesKey     []byte    // SHA256(sharedKey)
}

func (d *DHKeyInterceptor) ProcessClientHello(data []byte) error
func (d *DHKeyInterceptor) ProcessServerHello(data []byte) error
func (d *DHKeyInterceptor) DeriveAESKey() ([]byte, error)
func (d *DHKeyInterceptor) GetSessionKey() []byte
```

**DH Parameters**:
- Prime (p): Oakley 768-bit MODP group
- Generator (g): 22
- Client sends: A = g^a mod p
- Server sends: B = g^b mod p
- Shared secret: S = B^a mod p = A^b mod p
- AES key: SHA256(S)

**Responsibilities**:
- Identify DH handshake packets
- Extract public keys from handshake
- Compute shared secret
- Derive AES-256 session key

### Component 3: AESDecryptor

**Purpose**: Decrypts Photon AES-256-CBC encrypted events.

**Interface**:
```go
// AESDecryptor handles AES-256-CBC decryption of Photon traffic
type AESDecryptor struct {
    key       []byte  // 32-byte AES key
    iv        []byte  // 16 null bytes
    cipher    cipher.Block
    enabled   bool
}

func (a *AESDecryptor) SetKey(key []byte) error
func (a *AESDecryptor) Decrypt(data []byte) ([]byte, error)
func (a *AESDecryptor) DecryptEvent(event *EventData) (*EventData, error)
```

**Encryption Details**:
- Algorithm: AES-256-CBC
- IV: 16 null bytes (0x00 * 16)
- Key: SHA256 of DH shared secret (32 bytes)
- Applied to all Photon events after DH handshake

**Responsibilities**:
- Initialize cipher with derived key
- Decrypt Photon event payloads
- Handle CBC padding

### Component 4: XORDecryptor

**Purpose**: Decrypts XOR-encrypted player position coordinates.

**Interface**:
```go
// XORDecryptor handles XOR decryption of position coordinates
type XORDecryptor struct {
    xorCode   []byte  // 8-byte XorCode from the KeySync event
    enabled   bool
}

func (x *XORDecryptor) SetXorCode(code []byte)
func (x *XORDecryptor) DecryptFloat(encrypted []byte) float32
func (x *XORDecryptor) DecryptPosition(event *EventData) (*EventData, error)      // Move (verified layout)
func (x *XORDecryptor) DecryptSpawnPosition(event *EventData) (*EventData, error) // NewCharacter (OPEN ITEM - layout unverified)
func (x *XORDecryptor) ExtractXorCodeFromKeySync(event *EventData) ([]byte, error)
```

**XOR Algorithm**:
```go
// DecryptFloat XOR-decrypts 4 bytes into a float32
func (x *XORDecryptor) DecryptFloat(encrypted []byte) float32 {
    decrypted := make([]byte, 4)
    for i := 0; i < 4; i++ {
        decrypted[i] = encrypted[i] ^ x.xorCode[i]
    }
    return math.Float32frombits(binary.LittleEndian.Uint32(decrypted))
}
```

**Responsibilities**:
- Store XorCode from the KeySync event
- Decrypt Move position X coordinate (bytes 0-3)
- Decrypt Move position Y coordinate (bytes 4-7)
- Handle relative vs absolute position conversion
- (OPEN ITEM) Decrypt NewCharacter spawn position once the event-29 layout is verified from a live capture

### Component 5: SessionState

**Purpose**: Maintains session state including keys and connection info.

**Interface**:
```go
// SessionState tracks the current MITM session state
type SessionState struct {
    mu            sync.RWMutex
    sessionID     string
    clientAddr    *net.UDPAddr
    serverAddr    *net.UDPAddr
    aesKey        []byte
    xorCode       []byte
    connectedAt   time.Time
    lastActivity  time.Time
    stats         SessionStats
}

type SessionStats struct {
    PacketsIntercepted int64
    PacketsForwarded   int64
    EventsDecrypted    int64
    PositionsDecrypted int64
}

func (s *SessionState) SetAESKey(key []byte)
func (s *SessionState) SetXorCode(code []byte)
func (s *SessionState) GetXorCode() []byte
func (s *SessionState) Reset()
```

**Responsibilities**:
- Track session lifecycle
- Store encryption keys securely
- Provide thread-safe key access
- Track decryption statistics

## Data Models

### Model 1: KeySyncEvent

```go
// KeySyncEvent represents the decrypted KeySync event (real code resolved from
// params[252]; currently eventcodes.KeySync == 600, but Albion shifts codes
// across patches).
type KeySyncEvent struct {
    XorCode    []byte  `json:"xorCode"`    // 8 bytes, Parameters[0]
    Timestamp  int64   `json:"timestamp"`
    SequenceID int     `json:"sequenceId"`
}
```

**Validation Rules**:
- XorCode must be exactly 8 bytes
- Event must arrive after DH handshake completes
- XorCode changes on zone transitions

### Model 2: DecryptedPosition

```go
// DecryptedPosition represents a decrypted player position
type DecryptedPosition struct {
    ID         int64   `json:"id"`         // Entity ID from event
    X          float32 `json:"x"`          // Decrypted X coordinate
    Y          float32 `json:"y"`          // Decrypted Y coordinate
    RelativeX  float32 `json:"relativeX"`  // Relative to local player
    RelativeY  float32 `json:"relativeY"`  // Relative to local player
    Timestamp  int64   `json:"timestamp"`
    EventType  byte    `json:"eventType"`  // 3=Move, 29=NewCharacter
}
```

**Validation Rules**:
- X and Y must be finite (not NaN/Inf)
- Coordinates are relative to cluster origin
- Must match entity ID from spawn event

### Model 3: MITMConfig

```go
// MITMConfig stores user-configurable MITM settings
type MITMConfig struct {
    Enabled          bool   `json:"enabled"`
    ListenPort       int    `json:"listenPort"`
    ServerOverride   string `json:"serverOverride"` // Optional: custom server
    AutoRestart      bool   `json:"autoRestart"`
    LogDecrypted     bool   `json:"logDecrypted"`
    WarnTOSRisk      bool   `json:"warnTOSRisk"`    // Show TOS warning
}
```

**Validation Rules**:
- ListenPort must be available
- ServerOverride must be valid hostname
- WarnTOSRisk defaults to true

## Algorithmic Pseudocode

### Main Processing Algorithm

```go
// ProcessPacket handles an intercepted UDP packet
// INPUT: packet of type []byte
// OUTPUT: error or nil
// PRECONDITION: proxy is started, connection is established
// POSTCONDITION: packet is decrypted if applicable and forwarded
func (p *MITMProxy) ProcessPacket(packet []byte) error {
    // Step 1: Parse packet header
    header, err := ParsePhotonHeader(packet)
    if err != nil {
        return fmt.Errorf("invalid packet: %w", err)
    }
    
    // Step 2: Check if DH handshake phase
    if p.session.GetState() == StateHandshake {
        if err := p.dhInterceptor.Process(packet); err != nil {
            return fmt.Errorf("DH error: %w", err)
        }
        // Forward unmodified during handshake
        p.forwardPacket(packet)
        return nil
    }
    
    // Step 3: Decrypt AES layer
    decrypted, err := p.aesDecryptor.Decrypt(packet)
    if err != nil {
        // Forward encrypted if decryption fails
        p.forwardPacket(packet)
        return nil
    }
    
    // Step 4: Parse decrypted event
    event, err := photon.DeserializeEvent(decrypted)
    if err != nil {
        p.forwardPacket(packet)
        return nil
    }
    
    // Step 5: Resolve the real Albion event code.
    //
    // Protocol18 reality: the wire dispatch byte (event.DispatchCode) is only
    // ever 3 (Move, hot path) or 1 (generic). The authoritative Albion event
    // code is carried in params[252] as int16 for events. Move is the only
    // event identifiable by dispatch byte alone; every other event (KeySync,
    // NewCharacter, ...) MUST be resolved from params[252]. See
    // docs/technical/PROTOCOL18_PARAM_LAYOUTS.md and EventRouter.js, which
    // routes on params[252].
    if event.DispatchCode == 3 {
        // Move hot path: matched directly on the wire dispatch byte.
        if p.session.GetXorCode() != nil {
            decrypted := p.xorDecryptor.DecryptPosition(event)
            p.broadcastEvent(decrypted)
        }
    } else {
        // Generic dispatch byte (1): decode the real code from params[252].
        // Never hardcode raw numbers; resolve via the generated eventcodes.
        realCode, ok := readRealCode(event) // params[252] as int16
        if ok {
            switch realCode {
            case eventcodes.KeySync: // currently 600; shifts across patches
                xorCode := p.xorDecryptor.ExtractXorCodeFromKeySync(event)
                p.session.SetXorCode(xorCode)
                p.logger.Info("KeySync received", "xorCode", xorCode)
                
            case eventcodes.NewCharacter: // 29
                if p.session.GetXorCode() != nil {
                    // OPEN ITEM: the event-29 spawn-position layout differs
                    // from Move and is unverified (see DecryptSpawnPosition).
                    decrypted := p.xorDecryptor.DecryptSpawnPosition(event)
                    p.broadcastEvent(decrypted)
                }
            }
        }
    }
    
    // Step 6: Forward to server
    p.forwardPacket(packet)
    return nil
}
```

**Preconditions**:
- Proxy is started and listening
- UDP connection is established
- Packet is valid Photon protocol

**Postconditions**:
- Packet is forwarded to destination
- Events are decrypted if applicable
- No packet loss occurs

**Loop Invariants**:
- Session state remains consistent
- Forwarding continues regardless of decryption errors

### Diffie-Hellman Key Derivation Algorithm

```go
// DeriveAESKey computes the AES session key from DH exchange
// INPUT: none (uses stored public keys)
// OUTPUT: 32-byte AES key
// PRECONDITION: client and server public keys are set
// POSTCONDITION: returned key is SHA256 of shared secret
func (d *DHKeyInterceptor) DeriveAESKey() ([]byte, error) {
    // ASSERT: d.clientPub and d.serverPub are set
    if d.clientPub == nil || d.serverPub == nil {
        return nil, errors.New("missing public keys")
    }
    
    // Step 1: Compute shared secret
    // S = serverPub^clientPrivate mod prime
    // (We use client's stored private exponent)
    sharedSecret := new(big.Int).Exp(
        d.serverPub,           // base
        d.clientPrivate,       // exponent
        d.prime,               // modulus
    )
    
    // Step 2: Convert to bytes
    secretBytes := sharedSecret.Bytes()
    
    // Step 3: Derive AES key via SHA256
    hash := sha256.Sum256(secretBytes)
    
    // ASSERT: hash is 32 bytes
    d.aesKey = hash[:]
    
    return d.aesKey, nil
}
```

**Preconditions**:
- Client public key A received
- Server public key B received
- Client private exponent stored

**Postconditions**:
- AES key is valid 32-byte key
- Key matches server's derived key

### Position XOR Decryption Algorithm

> **Verified (Move only):** the `params[1]` ByteArray carries posX at offset 9 and posY at offset 13, which `PostProcessEvent` injects into `params[4]` (X) and `params[5]` (Y). Only player Move blobs are XOR-encrypted at those offsets; mobs/resources share the same `params[1][0] == 3` shape but their positions are unencrypted. The `len(raw) < 17` guard correctly skips 22-byte mode-4 blobs, which never carry positions (see `docs/technical/PROTOCOL18_PARAM_LAYOUTS.md`). This pseudocode is verified accurate and unchanged.

```go
// DecryptPosition extracts and decrypts position from a Move event
// INPUT: event of type *EventData
// OUTPUT: decrypted event with position parameters
// PRECONDITION: xorCode is set (8 bytes)
// POSTCONDITION: Parameters[4] and Parameters[5] contain valid floats
func (x *XORDecryptor) DecryptPosition(event *EventData) *EventData {
    // ASSERT: x.xorCode is 8 bytes
    if len(x.xorCode) != 8 {
        return event // No decryption possible
    }
    
    // Extract encrypted position bytes from Parameters[1]
    raw, ok := event.Parameters[1].(photon.ByteArray)
    if !ok || len(raw) < 17 {
        return event // Invalid position data
    }
    
    // Decrypt X coordinate (bytes 9-12)
    encryptedX := raw[9:13]
    decryptedX := make([]byte, 4)
    for i := 0; i < 4; i++ {
        decryptedX[i] = encryptedX[i] ^ x.xorCode[i]
    }
    xCoord := math.Float32frombits(binary.LittleEndian.Uint32(decryptedX))
    
    // Decrypt Y coordinate (bytes 13-16)
    encryptedY := raw[13:17]
    decryptedY := make([]byte, 4)
    for i := 0; i < 4; i++ {
        decryptedY[i] = encryptedY[i] ^ x.xorCode[i+4]
    }
    yCoord := math.Float32frombits(binary.LittleEndian.Uint32(decryptedY))
    
    // Validate decrypted coordinates
    if !isFiniteFloat32(xCoord) || !isFiniteFloat32(yCoord) {
        return event // Decryption failed, return original
    }
    
    // Inject decrypted positions
    event.Parameters[4] = xCoord
    event.Parameters[5] = yCoord
    
    return event
}
```

**Preconditions**:
- XorCode is 8 bytes from Event 600
- Event contains ByteArray at Parameters[1]
- ByteArray is at least 17 bytes

**Postconditions**:
- Parameters[4] contains valid X coordinate
- Parameters[5] contains valid Y coordinate
- Original event is preserved if decryption fails

**Loop Invariants**:
- XOR operation is deterministic
- Same XorCode always produces same result

### NewCharacter (Event 29) Spawn Position — OPEN ITEM

> **Unverified layout.** The event-29 (NewCharacter) parameter layout differs
> from Move and does **not** reuse the Move `params[1]` byte-array shape. Per
> `docs/technical/PROTOCOL18_PARAM_LAYOUTS.md`, event 29 uses `params[1]` for the
> character name (string), ByteArrays at `params[5..7,16,17]`, and float stats at
> `params[19..37]`. Which parameter carries the XOR-encrypted spawn position, and
> at what offset, has **not** been confirmed. The design MUST NOT assume the Move
> offset 9/13 layout for event 29.
>
> `DecryptSpawnPosition(event)` is therefore an OPEN ITEM: the encrypted
> spawn-position parameter and its byte offsets must be identified from a live
> capture (e.g. via the analyzer/pcap workflow used to produce the Protocol18
> layout snapshot) before implementation. Until then, no specific offset is
> claimed for event 29, and the XOR key material (the KeySync XorCode) is the
> only confirmed input.

### KeySync Event Processing Algorithm

```go
// ExtractXorCode extracts the 8-byte XorCode from the KeySync event
// INPUT: event of type *EventData, whose real code (from params[252]) is KeySync
// OUTPUT: 8-byte XorCode
// PRECONDITION: real code decoded from params[252] == eventcodes.KeySync
// POSTCONDITION: returned slice is exactly 8 bytes
func (x *XORDecryptor) ExtractXorCodeFromKeySync(event *EventData) ([]byte, error) {
    // ASSERT: the real Albion code (params[252], not the wire dispatch byte) is
    // KeySync. Resolve via the generated eventcodes; never hardcode raw numbers.
    if realCode, ok := readRealCode(event); !ok || realCode != eventcodes.KeySync {
        return nil, errors.New("not KeySync event")
    }
    
    // XorCode is in Parameters[0] as ByteArray
    xorCode, ok := event.Parameters[0].(photon.ByteArray)
    if !ok {
        return nil, errors.New("XorCode not found in Parameters[0]")
    }
    
    // ASSERT: XorCode is exactly 8 bytes
    if len(xorCode) != 8 {
        return nil, fmt.Errorf("invalid XorCode length: %d", len(xorCode))
    }
    
    // Store and return
    x.xorCode = xorCode
    return xorCode, nil
}
```

**Preconditions**:
- Event is decrypted (AES layer removed)
- Event code is 600

**Postconditions**:
- XorCode is stored in XORDecryptor
- Future position events can be decrypted

## Key Functions with Formal Specifications

### Function 1: Start()

```go
func (p *MITMProxy) Start(ctx context.Context) error
```

**Preconditions:**
- `p.config.ListenPort` is available (not in use)
- `p.config.ServerHost` is resolvable
- Network is reachable

**Postconditions:**
- UDP listener is bound to configured port
- Forwarding goroutine is running
- Proxy is ready to intercept packets
- Returns `nil` on success, error on failure

**Loop Invariants:**
- Packet forwarding continues until context is cancelled
- Each packet is processed exactly once

### Function 2: DeriveAESKey()

```go
func (d *DHKeyInterceptor) DeriveAESKey() ([]byte, error)
```

**Preconditions:**
- `d.clientPub` is set (received from client)
- `d.serverPub` is set (received from server)
- `d.clientPrivate` is stored from handshake
- `d.prime` and `d.generator` are initialized

**Postconditions:**
- Returns 32-byte AES key derived from SHA256 of shared secret
- Key matches the server's derived key
- Returns error if keys are missing

**Loop Invariants:** N/A

### Function 3: DecryptPosition()

```go
func (x *XORDecryptor) DecryptPosition(event *EventData) *EventData
```

**Preconditions:**
- `x.xorCode` is set and exactly 8 bytes
- `event.Parameters[1]` exists and is ByteArray
- ByteArray length >= 17 bytes

**Postconditions:**
- `event.Parameters[4]` contains decrypted X coordinate (float32)
- `event.Parameters[5]` contains decrypted Y coordinate (float32)
- Coordinates are finite (not NaN/Inf)
- Original event preserved if decryption fails

**Loop Invariants:**
- XOR decryption is deterministic for same XorCode

### Function 4: ExtractXorCodeFromKeySync()

```go
func (x *XORDecryptor) ExtractXorCodeFromKeySync(event *EventData) ([]byte, error)
```

**Preconditions:**
- `event` is not nil
- The real Albion code decoded from `params[252]` (int16) equals `eventcodes.KeySync` (currently 600; resolve via the generated `eventcodes`, never hardcode — Albion shifts codes across patches)
- Event is AES-decrypted

**Postconditions:**
- Returns 8-byte XorCode from `Parameters[0]`
- XorCode is stored for future position decryption
- Returns error if XorCode not found or invalid length

**Loop Invariants:** N/A

## Example Usage

```go
// Example 1: Starting the MITM proxy
func main() {
    ctx := context.Background()
    
    config := MITMConfig{
        Enabled:       true,
        ListenPort:    5056,
        AutoRestart:   true,
        WarnTOSRisk:   true,
    }
    
    proxy := NewMITMProxy(config)
    
    // Handle decrypted events.
    // Protocol18: Move is matched on the wire dispatch byte (3); every other
    // event is resolved from the real code in params[252]. Resolve via the
    // generated eventcodes package - never hardcode raw numbers.
    proxy.OnPacket(func(event *photon.EventData) {
        if event.DispatchCode == 3 { // Move hot path
            x := event.Parameters[4].(float32)
            y := event.Parameters[5].(float32)
            fmt.Printf("Player moved to (%.2f, %.2f)\n", x, y)
            return
        }
        realCode, ok := readRealCode(event) // params[252] as int16
        if !ok {
            return
        }
        switch realCode {
        case eventcodes.NewCharacter: // 29
            name := event.Parameters[1].(string) // params[1] is the name string for event 29
            // OPEN ITEM: the event-29 encrypted spawn-position parameter/offset
            // is unverified (layout differs from Move); confirm from a live
            // capture before extracting coordinates.
            fmt.Printf("Player %s spawned (position pending event-29 layout verification)\n", name)
        }
    })
    
    if err := proxy.Start(ctx); err != nil {
        log.Fatalf("Failed to start proxy: %v", err)
    }
    defer proxy.Stop()
    
    <-ctx.Done()
}

// Example 2: Mode selection in settings
func (m *CaptureManager) SetMode(mode CaptureMode) error {
    if mode == ModeMITM {
        // Stop passive capture
        m.StopPassiveCapture()
        
        // Start MITM proxy
        if err := m.proxy.Start(m.ctx); err != nil {
            // Fallback to passive on failure
            m.StartPassiveCapture()
            return err
        }
    } else {
        // Stop MITM proxy
        m.proxy.Stop()
        
        // Start passive capture
        m.StartPassiveCapture()
    }
    return nil
}

// Example 3: KeySync handling (real code resolved from params[252])
func (h *EventHandler) HandleKeySync(event *photon.EventData) {
    xorCode, err := h.xorDecryptor.ExtractXorCodeFromKeySync(event)
    if err != nil {
        h.logger.Warn("Failed to extract XorCode", "error", err)
        return
    }
    
    h.session.SetXorCode(xorCode)
    h.logger.Info("XorCode updated", "xorCode", fmt.Sprintf("%x", xorCode))
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system-essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Client Packet Forwarding

*For any* UDP packet received from the Albion client, the MITM proxy SHALL forward the packet to the configured Albion server address (live.albiononline.com:5056).

**Validates: Requirements 1.2**

### Property 2: Server Packet Forwarding

*For any* UDP packet received from the Albion server, the MITM proxy SHALL forward the packet to the connected Albion client address.

**Validates: Requirements 1.3**

### Property 3: DH ClientHello Detection

*For any* valid DH ClientHello packet initiating a Photon connection, the DH interceptor SHALL detect and flag the packet as a handshake initiation.

**Validates: Requirements 2.1**

### Property 4: DH Response Parameters

*For any* valid DH initiation packet, the MITM proxy SHALL respond using Oakley 768-bit MODP group with generator 22.

**Validates: Requirements 2.2**

### Property 5: DH Shared Secret Computation

*For any* valid DH key pair (client public A, server public B, client private exponent a), the computed shared secret SHALL equal B^a mod p using the Oakley 768-bit prime.

**Validates: Requirements 2.3**

### Property 6: AES Key Derivation

*For any* shared secret S, the derived AES key SHALL equal SHA256(S) and be exactly 32 bytes.

**Validates: Requirements 2.4**

### Property 7: AES Round-Trip Preservation

*For any* Photon event data, encrypting with the derived AES key and then decrypting with the same key SHALL produce the original event data.

**Validates: Requirements 3.1, 3.2**

### Property 8: Event Parameter Parsing

*For any* valid decrypted Photon event bytes, the deserializer SHALL extract the event code and all parameter key-value pairs.

**Validates: Requirements 3.2**

### Property 9: XorCode Extraction

*For any* valid KeySync event (currently code 600) with XorCode in Parameters[0], the XOR decryptor SHALL extract exactly 8 bytes.

**Validates: Requirements 4.1**

### Property 10: XorCode Update

*For any* sequence of KeySync event (currently code 600) values arriving over time, the stored XorCode SHALL always equal the most recently received valid XorCode.

**Validates: Requirements 4.3**

### Property 11: X Coordinate Decryption

*For any* 4-byte encrypted X coordinate and XorCode, the decryption SHALL equal the byte-wise XOR of encrypted bytes with XorCode bytes 0-3, interpreted as little-endian float32.

**Validates: Requirements 5.1, 5.4, 5.5**

### Property 12: Y Coordinate Decryption

*For any* 4-byte encrypted Y coordinate and XorCode, the decryption SHALL equal the byte-wise XOR of encrypted bytes with XorCode bytes 4-7, interpreted as little-endian float32.

**Validates: Requirements 5.2, 5.4, 5.5**

### Property 13: Spawn Position Decryption

*For any* Event 29 (NewCharacter) with an encrypted spawn position, once the event-29 parameter layout is verified from a live capture (see the NewCharacter Spawn Position OPEN ITEM), the XOR decryptor SHALL decrypt the spawn coordinates using the stored XorCode. The exact parameter/offset for event 29 is not yet confirmed and MUST NOT be assumed to match the Move layout.

**Validates: Requirements 5.3**

### Property 14: XOR Round-Trip

*For any* byte value b and XorCode byte k, applying XOR decryption twice SHALL return the original value: (b XOR k) XOR k = b.

**Validates: Requirements 5.4**

### Property 15: Packet Content Preservation

*For any* packet forwarded by the MITM proxy, the forwarded content SHALL equal either the original packet or the properly decrypted packet content.

**Validates: Requirements 10.2**

### Property 16: Localhost Binding

*For any* MITM proxy startup, the UDP listener SHALL bind only to localhost (127.0.0.1) and not to external network interfaces.

**Validates: Requirements 10.1**

## Error Handling

### Error Scenario 1: Port Binding Failure

**Condition**: UDP port 5056 is already in use by another process
**Response**: Log error, display user-friendly message, suggest closing conflicting application
**Recovery**: Retry with alternative port or wait for port release

### Error Scenario 2: DH Handshake Timeout

**Condition**: Diffie-Hellman handshake does not complete within timeout period
**Response**: Reset session, log warning, restart handshake listener
**Recovery**: User must reconnect to trigger new handshake

### Error Scenario 3: AES Decryption Failure

**Condition**: Packet cannot be decrypted with derived AES key
**Response**: Forward encrypted packet, log debug message, increment error counter
**Recovery**: Continue forwarding; wait for next session/key sync

### Error Scenario 4: XorCode Missing

**Condition**: Position event received before Event 600 KeySync
**Response**: Log warning, forward encrypted event, skip position extraction
**Recovery**: Positions will decrypt after KeySync arrives

### Error Scenario 5: Server Connection Lost

**Condition**: Connection to Albion server is interrupted
**Response**: Clean up session, notify frontend, enter reconnect state
**Recovery**: Auto-reconnect if AutoRestart enabled, else wait for manual restart

### Error Scenario 6: Invalid XorCode Length

**Condition**: Event 600 contains XorCode that is not exactly 8 bytes
**Response**: Log error, skip XorCode storage, continue forwarding
**Recovery**: Wait for next KeySync event

## Testing Strategy

### Test Type Classification Summary

Based on acceptance criteria analysis, tests are classified as follows:

| Test Type | Count | Examples |
|-----------|-------|----------|
| PROPERTY | 16 | Packet forwarding, DH computation, AES/XOR decryption |
| EXAMPLE | 14 | Error handling, mode switching, cleanup behavior |
| EDGE_CASE | 3 | Corrupted packets, missing XorCode, invalid Event 600 |
| INTEGRATION | 12 | Performance, timing, UI, end-to-end flows |
| SMOKE | 3 | Port binding, localhost verification, data security |

### Unit Testing Approach

Unit tests will verify individual component behavior with specific examples:

- **DHKeyInterceptor**: Test key derivation with known inputs and expected outputs
- **AESDecryptor**: Test decryption with known AES-256-CBC ciphertext
- **XORDecryptor**: Test position decryption with known XorCode and encrypted values
- **SessionState**: Test concurrent key access, state transitions

Coverage goal: 80%+ on all crypto components

### Property-Based Testing Approach

Property-based tests will validate universal correctness properties using Go's `testing/quick` or `gopter` library:

**Configuration**: Minimum 100 iterations per property test

**Property Test Cases**:

| Property | Test Strategy |
|----------|---------------|
| Client Packet Forwarding | Generate random UDP packets, verify forwarding to server address |
| Server Packet Forwarding | Generate random UDP packets, verify forwarding to client address |
| DH ClientHello Detection | Generate valid DH initiation packets, verify detection flag |
| DH Shared Secret | Generate random DH key pairs, verify B^a mod p computation |
| AES Key Derivation | Generate random shared secrets, verify SHA256 output is 32 bytes |
| AES Round-Trip | Generate random event data, verify encrypt→decrypt returns original |
| XorCode Extraction | Generate valid Event 600 with random 8-byte values, verify extraction |
| XorCode Update | Generate sequence of Event 600 values, verify latest is stored |
| X Coordinate Decryption | Generate random encrypted bytes and XorCode, verify XOR math |
| Y Coordinate Decryption | Generate random encrypted bytes and XorCode, verify XOR math |
| XOR Round-Trip | For any byte b and XorCode k, verify (b XOR k) XOR k = b |

### Edge Case Testing

Edge case tests verify graceful handling of boundary conditions:

- **Corrupted Packet Data**: AES decryption skips event, processing continues
- **Missing XorCode**: Position decryption skipped, position marked as encrypted
- **Invalid XorCode Length**: Event 600 with wrong length, previous XorCode retained
- **Malformed Event 600**: Invalid parameters, warning logged

### Integration Testing Approach

Integration tests verify end-to-end flows and external dependencies:

- **DH → AES → XOR Flow**: Complete key derivation and decryption pipeline
- **WebSocket Integration**: Verify decrypted events reach frontend
- **Mode Switching**: Test passive ↔ MITM mode transitions
- **Error Recovery**: Simulate failures and verify fallback behavior
- **Performance Benchmarks**: Latency (<10ms), throughput (500+ events/sec), memory (<100MB)

### Smoke Testing

Smoke tests verify infrastructure setup:

- **Port Binding**: UDP listener binds to port 5056 on startup
- **Localhost Binding**: Listener binds only to 127.0.0.1
- **No Plain-Text Logging**: Verify decrypted data not written to disk

### Test Tags

Each test must be tagged with feature and property reference:

```go
// Feature: mitm-proxy-player-positions, Property 7: AES Round-Trip Preservation
func TestAESRoundTrip(t *testing.T) {
    // ...
}
```

## Performance Considerations

### Latency Requirements

- Packet forwarding latency: < 1ms per packet
- Decryption overhead: < 0.5ms per event
- Total additional latency: < 2ms (acceptable for real-time gameplay)

### Throughput Requirements

- Handle up to 1000 packets/second during intense gameplay
- Decrypt up to 500 events/second
- WebSocket broadcast: up to 100 events/second to frontend

### Optimization Strategies

- Use sync.Pool for byte buffer reuse
- Pre-allocate decryption buffers
- Parallel event processing for bulk Move events
- Cache parsed Photon headers

## Security Considerations

### Increased Detection Risk

The MITM proxy modifies the network path between client and server, which increases detection risk compared to passive capture:

1. **Traffic Analysis**: Server sees connection from proxy IP, not client IP
2. **Timing Analysis**: Additional latency may be detectable
3. **Behavioral Analysis**: Proxy behavior may differ from normal client

### Mitigation Strategies

- Document TOS implications clearly in UI
- Warn users before enabling MITM mode
- Keep latency minimal to avoid timing detection
- Consider adding random jitter to packet forwarding

### Data Security

- Clear AES keys from memory on session end
- Never log raw encryption keys
- Secure XorCode storage (memory only, no disk)

### Protocol Volatility & Scope

- **Event codes shift across patches.** Albion renumbers event codes with game
  patches (for example, a +2 shift was observed after the 2026-06-29 patch, and
  older project docs referenced KeySync as 593 whereas 593 is now
  `TerritoryAnnouncePlayerEjection` and KeySync is 600). The implementation MUST
  resolve every event code through the generated `eventcodes` package (sourced
  from `web/scripts/utils/EventCodes.js`) and MUST NOT hardcode raw numeric
  codes anywhere in the decryption or routing path. The wire dispatch byte (only
  `3` or `1` under Protocol18) is never sufficient on its own to identify KeySync
  or NewCharacter — those are resolved from `params[252]`.
- **This feature knowingly diverges from upstream scope.** The upstream
  OpenRadar project deliberately scoped the MITM approach *out*, as documented in
  `docs/technical/PLAYER_POSITIONS_MITM.md`. Its stated reasons are (a) modifying
  the game's network path increases detection risk and changes the threat model,
  and (b) a Photon MITM proxy is an estimated three to four weeks of focused work
  (DH interception, AES decryption, XOR plumbing, replay safety) for a capability
  the project's primarily PvE use cases do not need. This design consciously
  diverges from that stance; the increased detection risk and effort are accepted
  trade-offs the user must be made aware of.

## Dependencies

### Go Standard Library

- `crypto/aes`: AES-256-CBC implementation
- `crypto/cipher`: CBC mode
- `crypto/sha256`: Key derivation
- `math/big`: Diffie-Hellman arithmetic
- `net`: UDP networking

### Existing OpenRadar Packages (backend)

- `internal/photon`: Protocol18 deserializer and event/parameter parsing
- `internal/photon/events.go`: event 29 (NewCharacter) deserialization
- `internal/photon/eventcodes` (generated from `web/scripts/utils/EventCodes.js`): authoritative Albion event codes; the proxy MUST resolve codes here rather than hardcoding raw numbers
- `internal/capture`: network capture interface
- `internal/server`: WebSocket broadcasting

### Existing OpenRadar Frontend

- `web/scripts/core/EventRouter.js`: routes events on the real code in `params[252]`
- `web/scripts/handlers/PlayersHandler.js`: player detection, ignore list, alert gate, state
- `web/scripts/drawings/PlayersDrawing.js`: radar rendering and color coding

### External Dependencies

- `github.com/google/gopacket`: Packet capture and parsing (already used)
- DH Oakley 768-bit prime constants (hardcoded)
