# Requirements Document: MITM Proxy Player Positions

## Introduction

This feature adds real-time player position tracking to OpenRadar by implementing a Photon MITM (Man-In-The-Middle) proxy. Currently, OpenRadar operates as a passive capture tool that can detect player spawns and identities but cannot display live player positions because they are encrypted with a two-layer encryption scheme: Photon AES-256-CBC at the transport layer and Albion XOR encryption at the application layer.

The MITM proxy will intercept the Diffie-Hellman key exchange between the Albion Online client and server, derive the session key, decrypt Photon events, extract the XorCode from the KeySync event, and decrypt player positions from Event 3 (Move) and Event 29 (NewCharacter) (subject to change across Albion patches; resolve symbolically).

## Glossary

- **MITM_Proxy**: The Man-In-The-Middle proxy component that intercepts and decrypts Photon traffic between the Albion client and server
- **Photon**: The UDP-based protocol used by Albion Online for game communication on port 5056
- **Session_Key**: The AES-256 encryption key derived from the Diffie-Hellman shared secret using SHA256
- **XorCode**: An 8-byte value transmitted in the KeySync event, used to decrypt player positions. Note: Albion shifts Photon event code numbers across patches, so the implementation MUST resolve event codes symbolically (via the generated eventcodes / EventCodes.js) rather than relying on a fixed number.
- **KeySync**: The Photon event that carries the 8-byte XorCode (currently event code 600, subject to change across Albion patches). The implementation MUST resolve the KeySync code symbolically via the generated eventcodes / EventCodes.js rather than hardcoding a number.
- **DH_Exchange**: Diffie-Hellman key exchange using Oakley 768-bit MODP group with generator 22
- **AES_Decryptor**: Component responsible for AES-256-CBC decryption of Photon events
- **Position_Decryptor**: Component responsible for XOR decryption of player coordinates using XorCode
- **Capture_Manager**: Existing OpenRadar component that manages packet capture
- **Player_Record**: Data structure containing player id, nickname, guild, alliance, position, equipment, and flag
- **Fallback_Mode**: Operating mode where the proxy is disabled and OpenRadar reverts to passive capture

## Requirements

### Requirement 1: Transparent UDP Proxy Core

**User Story:** As a player, I want the MITM proxy to intercept Photon traffic transparently, so that I can see live player positions without modifying my game client configuration.

#### Acceptance Criteria

1. WHEN the MITM_Proxy starts, THE MITM_Proxy SHALL create a transparent UDP proxy listening on localhost port 5056
2. WHEN the MITM_Proxy receives a UDP packet from the Albion client, THE MITM_Proxy SHALL forward the packet to the Albion server (live.albiononline.com:5056)
3. WHEN the MITM_Proxy receives a UDP packet from the Albion server, THE MITM_Proxy SHALL forward the packet to the Albion client
4. THE MITM_Proxy SHALL preserve packet ordering and timing characteristics within 10ms latency overhead
5. IF the MITM_Proxy fails to start on port 5056, THEN THE MITM_Proxy SHALL log an error and fall back to Fallback_Mode

### Requirement 2: Diffie-Hellman Key Exchange Interception

**User Story:** As a developer, I want the MITM proxy to intercept the Diffie-Hellman key exchange, so that I can derive the session key for decryption.

#### Acceptance Criteria

1. WHEN the Albion client initiates a Photon connection, THE MITM_Proxy SHALL detect the DH_Exchange initiation packet
2. WHEN the MITM_Proxy detects a DH_Exchange initiation, THE MITM_Proxy SHALL intercept and respond with a crafted DH_Exchange response using Oakley 768-bit MODP group with generator 22
3. THE MITM_Proxy SHALL compute the shared secret from the intercepted DH_Exchange parameters
4. WHEN the shared secret is computed, THE MITM_Proxy SHALL derive the Session_Key using SHA256 hash of the shared secret
5. THE MITM_Proxy SHALL store the Session_Key for the duration of the game session
6. IF the DH_Exchange fails or times out within 30 seconds, THEN THE MITM_Proxy SHALL log the failure and continue in Fallback_Mode

### Requirement 3: AES-256-CBC Event Decryption

**User Story:** As a developer, I want the MITM proxy to decrypt Photon events, so that I can access the encrypted game data including the XorCode.

#### Acceptance Criteria

1. WHEN the MITM_Proxy receives an encrypted Photon event from the server, THE MITM_Proxy SHALL decrypt the event using AES-256-CBC with the Session_Key and 16 null bytes as IV
2. WHEN the MITM_Proxy decrypts an event successfully, THE MITM_Proxy SHALL parse the event code and parameters
3. IF AES decryption fails due to an invalid Session_Key, THEN THE MITM_Proxy SHALL log the failure and attempt Session_Key re-derivation
4. IF AES decryption fails due to corrupted packet data, THEN THE MITM_Proxy SHALL skip the event and continue processing
5. THE AES_Decryptor SHALL process events at a minimum rate of 100 events per second

### Requirement 4: XorCode Extraction from the KeySync Event

**User Story:** As a developer, I want the MITM proxy to extract the XorCode from the KeySync event, so that I can decrypt player positions.

#### Acceptance Criteria

1. WHEN the MITM_Proxy detects the KeySync event, THE MITM_Proxy SHALL extract the 8-byte XorCode from the event parameters
2. WHEN the XorCode is extracted, THE MITM_Proxy SHALL store the XorCode for the current game session
3. WHEN the MITM_Proxy receives a new KeySync event, THE MITM_Proxy SHALL update the stored XorCode with the new value
4. IF the KeySync event parameters do not contain a valid 8-byte XorCode, THEN THE MITM_Proxy SHALL log a warning and retain the previous XorCode
5. THE MITM_Proxy SHALL make the XorCode available to the Position_Decryptor within 1ms of extraction

### Requirement 5: Player Position Decryption

**User Story:** As a player, I want the MITM proxy to decrypt player coordinates, so that I can see live player positions on the radar.

#### Acceptance Criteria

1. WHEN the MITM_Proxy detects Event 3 (Move) with encrypted coordinates, THE Position_Decryptor SHALL decrypt the X coordinate using XorCode bytes 0-3
2. WHEN the MITM_Proxy detects Event 3 (Move) with encrypted coordinates, THE Position_Decryptor SHALL decrypt the Y coordinate using XorCode bytes 4-7
3. WHEN the MITM_Proxy detects Event 29 (NewCharacter) with spawn position, THE Position_Decryptor SHALL decrypt the spawn coordinates using the XorCode
4. THE Position_Decryptor SHALL apply XOR decryption: `decrypted_byte = encrypted_byte XOR XorCode_byte`
5. THE Position_Decryptor SHALL convert the decrypted 4-byte arrays to 32-bit floating point coordinates (little-endian)
6. IF the XorCode is not available, THEN THE Position_Decryptor SHALL skip position decryption and mark the position as encrypted

### Requirement 6: OpenRadar Integration

**User Story:** As a player, I want the decrypted player positions to appear on the OpenRadar display, so that I can track other players in real-time.

#### Acceptance Criteria

1. WHEN the MITM_Proxy decrypts a player position, THE MITM_Proxy SHALL update the Player_Record with the new posX and posY values
2. WHEN a Player_Record is updated, THE MITM_Proxy SHALL forward the update to the Capture_Manager
3. THE Capture_Manager SHALL process decrypted player positions identically to other radar entities
4. THE PlayersDrawing component SHALL render decrypted player positions with the same visual styling as spawn positions
5. WHEN the MITM_Proxy decrypts a position update, THE position SHALL appear on the radar within 50ms

### Requirement 7: Fallback to Passive Mode

**User Story:** As a player, I want OpenRadar to continue working even if the MITM proxy fails, so that I can still use other radar features.

#### Acceptance Criteria

1. WHEN the MITM_Proxy encounters a fatal error, THE MITM_Proxy SHALL switch to Fallback_Mode
2. WHILE in Fallback_Mode, THE MITM_Proxy SHALL disable player position decryption
3. WHILE in Fallback_Mode, THE Capture_Manager SHALL continue to detect player spawns and identities using passive capture
4. WHEN the user disables the MITM_Proxy in settings, THE MITM_Proxy SHALL gracefully shut down and switch to Fallback_Mode
5. THE UI SHALL display a status indicator showing whether MITM mode or Fallback_Mode is active

### Requirement 8: Error Handling and Recovery

**User Story:** As a player, I want the MITM proxy to recover from errors automatically, so that I experience minimal disruption during gameplay.

#### Acceptance Criteria

1. IF the Session_Key becomes invalid, THE MITM_Proxy SHALL attempt to re-intercept the DH_Exchange on the next connection attempt
2. IF the XorCode is not received within 60 seconds of session start, THE MITM_Proxy SHALL log a warning and continue in Fallback_Mode for player positions
3. WHEN the game client disconnects, THE MITM_Proxy SHALL clear the Session_Key and XorCode
4. WHEN the game client reconnects, THE MITM_Proxy SHALL reinitialize the DH_Exchange interception
5. IF the MITM_Proxy crashes, THE MITM_Proxy SHALL restart automatically within 5 seconds
6. THE MITM_Proxy SHALL log all errors with timestamps and error codes for debugging

### Requirement 9: Performance Requirements

**User Story:** As a player, I want the MITM proxy to have minimal impact on game performance, so that my gameplay experience is not affected.

#### Acceptance Criteria

1. THE MITM_Proxy SHALL add no more than 10ms latency to packet round-trip time
2. THE MITM_Proxy SHALL use no more than 100MB of RAM during operation
3. THE MITM_Proxy SHALL use no more than 10% CPU on a quad-core 2.5GHz processor during peak event processing
4. THE MITM_Proxy SHALL handle a minimum of 500 events per second without packet loss
5. THE MITM_Proxy SHALL not cause packet reordering that affects game behavior

### Requirement 10: Security and Detection Mitigation

**User Story:** As a player, I want the MITM proxy to minimize detection risk, so that my account remains safe while using the radar.

#### Acceptance Criteria

1. THE MITM_Proxy SHALL operate only on localhost (127.0.0.1) and not expose services to external networks
2. THE MITM_Proxy SHALL not modify packet contents except for decryption purposes
3. THE MITM_Proxy SHALL not inject packets into the game traffic stream
4. THE MITM_Proxy SHALL close connections cleanly when the game client disconnects
5. THE MITM_Proxy SHALL NOT log or store decrypted player data to disk in plain text format

### Requirement 11: Configuration and User Control

**User Story:** As a player, I want to control the MITM proxy through the OpenRadar settings, so that I can enable or disable it as needed.

#### Acceptance Criteria

1. THE Settings_UI SHALL provide a toggle to enable or disable the MITM_Proxy
2. WHEN the user toggles MITM_Proxy off, THE MITM_Proxy SHALL stop intercepting traffic and switch to Fallback_Mode
3. WHEN the user toggles MITM_Proxy on, THE MITM_Proxy SHALL initialize and begin intercepting traffic
4. THE Settings_UI SHALL display the current MITM status: Active, Inactive, Error, or Fallback
5. THE Settings_UI SHALL display the last Session_Key derivation timestamp when MITM is active
