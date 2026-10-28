# CCSDS 766.1-B Digital Motion Imagery (DMI) Engineering Specification

## 1\. Executive Summary & Normative Baseline

- **Document Identifier**: CCSDS 766.1-B-1 / CCSDS 766.1-B-3 (Blue Book, Recommended Standard)  
- **Title**: Digital Motion Imagery  
- **Consultative Committee for Space Data Systems (CCSDS)**: Spacecraft Onboard Interface Services (SOIS) & Space Internetworking Services (SIS)  
- **Implementation**: DMIForge Pro (v1.0.0)

This engineering specification establishes the end-to-end transport, video coding, audio compression, metadata synchronization, and network encapsulation requirements for space-to-ground and inter-spacecraft motion imagery streams. It provides flight avionics software engineers, payload leads, and ground station teams with a reference standard for high-assurance space video delivery.

---

## 2\. MPEG-2 Transport Stream (ISO/IEC 13818-1) Framing (§3.2)

Spacecraft digital motion imagery utilizes fixed-length MPEG-2 Transport Stream packets to achieve robust synchronization across noisy RF links:

### 2.1 Packet Geometry & Header Structure

- **Packet Length**: Strictly 188 octets.  
- **Sync Byte**: The first octet shall always be `0x47` (ASCII 'G').  
- **Header Fields**:  
  - `Transport Error Indicator (TEI)` (1 bit): Set by RF demodulator upon uncorrectable RS/LDPC frame error.  
  - `Payload Unit Start Indicator (PUSI)` (1 bit): Set to '1' on packets initiating a Program Specific Information (PSI) section or Packetized Elementary Stream (PES) header.  
  - `Transport Priority` (1 bit): Indicates priority multiplexing.  
  - `Packet Identifier (PID)` (13 bits): Uniquely identifies elementary stream or signaling table.  
  - `Transport Scrambling Control` (2 bits): '00' for unencrypted streams; encryption handled at SDLS or IPsec layer.  
  - `Adaptation Field Control` (2 bits): '01' (Payload only), '10' (Adaptation field only), '11' (Adaptation field followed by payload).  
  - `Continuity Counter` (4 bits): Increments 0 to 15 per PID, wrapping periodically to detect missing packets.

### 2.2 Standard Aerospace PID Allocation

| PID (Hex) | Decimal | Stream Type / Table | Description |
| :---- | :---- | :---- | :---- |
| `0x0000` | 0 | Program Association Table (PAT) | Maps Program Number 1 to PMT PID |
| `0x1000` | 4096 | Program Map Table (PMT) | Identifies Video, Audio, and Metadata PIDs |
| `0x1011` | 4113 | Video Elementary Stream (AVC) | H.264 High Profile Level 4.0/4.1 (1080p/720p) |
| `0x1012` | 4114 | Audio Elementary Stream (AAC) | MPEG-4 AAC-LC (48 kHz, Stereo) |
| `0x1013` | 4115 | Private Stream 1 (KLV Metadata) | SMPTE RP 210 / MISB ST 0601 Ephemeris Tags |
| `0x1FFF` | 8191 | Null Packets | CBR rate stuffing / link smoothing |

---

## 3\. Video Compression & Bounded Closed-GOP (§3.4)

- **Video Standard**: ITU-T H.264 / ISO/IEC 14496-10 Advanced Video Coding (AVC).  
- **Profile / Level**: High Profile (Level 4.0, 4.1, or 4.2).  
- **Group of Pictures (GOP) Constraint (§3.4.2)**:  
  - **Maximum GOP Length**: Bounded strictly to \$N \\le 30\$ frames (at 30 fps, maximum 1.0 second duration).  
  - **Closed-GOP Structure**: Every GOP shall start with an Instantaneous Decoder Refresh (IDR) frame. Inter-picture prediction across GOP boundaries is strictly prohibited.  
  - **Rationale**: Mitigates transmission errors over space links. When an RF dropout corrupts telemetry frames, the ground decoder recovers clean synchronization within at most 1 second without propagating macroblock artifacts.

---

## 4\. Audio & Telemetry Metadata Synchronization (§3.5 & §3.6)

### 4.1 Audio Compression (§3.5)

- Standardized to MPEG-4 AAC-LC (ISO/IEC 14496-3) or uncompressed Linear PCM (SMPTE 302M).  
- Aerospace standard audio sampling frequency: **48.0 kHz**.  
- Synchronization: Audio PES packets contain 33-bit Presentation Time Stamps (PTS) synchronous with the video time base.

### 4.2 Synchronous KLV / Orbital Telemetry Metadata (§3.6)

- **Standard**: SMPTE RP 210 / MISB ST 0601 / ST 0107\.  
- **Encapsulation**: Formatted into Private Stream 1 (`stream_id = 0xBD`) and mapped to PID `0x1013`.  
- **Content**: Microsecond UTC time tags, spacecraft ephemeris (\$X, Y, Z\$ and Lat/Lon/Alt), velocity vectors, camera gimbal pointing angles (azimuth and elevation), and camera optical zoom factors.  
- **Frame-Accurate Coupling**: KLV metadata packets share the exact PTS of corresponding video frames to guarantee sub-millisecond telemetry correlation.

---

## 5\. Network Encapsulation & Space Data Link Mapping (§3.6.1 & §4.2)

### 5.1 7-TS Packet UDP/RTP Encapsulation (RFC 2250 / RFC 3550\)

- **Payload Ratio**: Exactly seven (7) 188-byte TS packets are bundled into one UDP/RTP datagram: \$\$\\text{Payload} \= 7 \\times 188 \= 1,316 \\text{ octets}\$\$  
- **RTP Header**: 12 octets with Payload Type \= 33 (`MP2T`).  
- **Total Datagram**: \$12 \+ 1,316 \= 1,328 \\text{ octets}\$.  
- **MTU Safety**: Fits comfortably within standard 1500-byte Ethernet, Wi-Fi, and space internetworking MTU limits without IP fragmentation.

### 5.2 CCSDS Space Data Link Protocol Integration (§4.2)

- Video streams map to dedicated Virtual Channels (VCs) on AOS Space Data Link Protocols (CCSDS 732.0-B-4) or USLP (CCSDS 732.1-B-2).  
- Ensures high-priority video channels do not contend with critical telecommand or spacecraft safety housekeeping streams.