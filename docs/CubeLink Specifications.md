# CubeLink Protocol Specifications (v0.4.0)

This document defines how a Host talks to an LED Cube: the **CubeLink Protocol** (all UDP Datagrams between Cubes and Hosts), the **CubeLink Profiles** (per-Cube-class Datagram layouts), and the behavior of a Cube as seen from outside. It is written for software developers; no hardware knowledge beyond "an LED Cube that shows Frames" is needed.

The Host application is specified in [CubeLoader Specifications.md](/docs/CubeLoader%20Specifications.md).

## Table of contents
1. [Overview and scope](#1-overview-and-scope)
2. [Glossary](#2-glossary)
3. [Data conventions](#3-data-conventions)
4. [Cube behavior](#4-cube-behavior)
5. [CubeLink protocol](#5-cubelink-protocol)
6. [Common datagrams](#6-common-datagrams)
7. [CubeLink profiles](#7-cubelink-profiles)



## 1. Overview and scope
- The Cube is a standalone network device. Any Host that speaks CubeLink can show Frames on it. Without a Host it plays Animations from its own SD card or USB flash drive, or shows an Idle Frame.
- This document covers: Datagram formats, protocol versioning, Profiles, color/data conventions, and what a Cube does with each of its connections. It also covers the Host-side obligations that CubeDriver fulfils (section 5.7). The CubeDriver API itself is in the CubeLoader Specifications.
- Cubes differ in size, Color Channels, Color Depth and refresh rate. Everything that depends on those lives in a **Profile** (chapter 6).
- Not covered: the internal workings of Cube hardware and firmware, and the Host application.



## 2. Glossary
| Term | Meaning |
|---|---|
| **Animation** | A file holding a sequence of Frames and info on how to play them. |
| **Broadcast** | A UDP Datagram sent to all devices on the local network at once. |
| **Channel Arrangement** | A string of Color Channel letters, e.g. `RGB`, stating which Color Channels a Cube or file has and in which order their values are stored for each Voxel. |
| **Color Channel** | One independently controlled color component of a Voxel, named by a letter (e.g. `R`, `G`, `B`). |
| **Color Depth** | The number of bits that represent the value of one Color Channel. |
| **Control Datagram** | A Datagram identified by ASCII text and valid for every Profile. |
| **Complete Frame** | A Frame of which all Frame Fragments have been received. |
| **Cube** | The LED Cube device. |
| **Cube Autoplay** | The Cube playing Animations from its own SD card or USB flash drive, without a Host. |
| **CubeDriver** | The Host-side Python module that implements the CubeLink Protocol and Animation file reading/writing, and gives CubeLoader and Programs one API to use a Cube. |
| **CubeLink** | The network protocol ("CubeLink Protocol") formed by all UDP Datagrams exchanged between Cubes and Hosts or any other app. |
| **CubeLoader** | The example Host app that uses the CubeDriver module to interface with Cubes. |
| **Data Datagram** | A Datagram that carries pixel data (Frame Fragment, Sparse Update). Its layout depends on the Profile. |
| **Datagram** | One UDP payload exchanged between a Cube and a Host. Datagrams are never split between multiple UDP packets. |
| **Frame** | The colors of all Voxels at one moment. |
| **Frame Fragment** | One Datagram carrying a slice of a Frame's Framedata and some identification data. |
| **Frame Index** | A counter inside every Frame Fragment that tells which Frame the fragment belongs to. |
| **Frame-sync** | A Datagram the Cube sends to its Lock Holder at the start of every Refresh Period. |
| **Framedata** | The binary data of one Frame: the values of all Color Channels of all Voxels, laid out as defined in the data conventions. |
| **Host** | Any computer or program that sends data to a Cube. CubeLoader is the reference Host. |
| **Idle Frame** | The built-in static picture a Cube shows when it has nothing else to display (Cubes may not have one). |
| **Keepalive** | Any Datagram a Host sends only to keep its Lock. |
| **Link-local Address** | An automatically assigned `169.254.x.x` IPv4 address, used when no DHCP server answers. |
| **Lock** | An imaginary key provided to Hosts. If the Cube has an ethernet port, a single one exists, otherwise none. |
| **Lock Holder** | The Host that is currently holding the Lock and whose Data Datagrams are accepted by the Cube. |
| **Network Mode** | The state of a Cube while a Lock Holder is streaming data to it. |
| **Partial Frame** | A Frame of which not all Frame Fragments have been received yet. |
| **(CubeLink) Profile** | The set of Data Datagram layouts (Frame Fragment, Sparse Update) for one class of Cube, identified by its Profile Name. |
| **Profile Name** | `v<protocol version>-<sizeX>x<sizeY>x<sizeZ>-<channel arrangement><color depth>-<fps>`, built from a Cube's discovery reply. Example: `v1-16x16x16-RGB8-60`. |
| **Refresh Period** | The time between 2 display refreshes. `t = 1 / fps` |
| **Sparse Update** | A Datagram that changes the color of only some Voxels of the displayed picture. |
| **Voxel / LED** | One LED of the Cube, addressed by (x, y, z). The terms are used interchangeably. |



## 3. Data conventions

**Limits**
| Name | Value | Meaning |
|---|---|---|
| <a id="p-max-size"></a>`MAX_SIZE` | 64 | maximum number of LEDs per axis |
| <a id="p-max-channels"></a>`MAX_COLOR_CHANNELS` | 8 | maximum Color Channels per Voxel |
| <a id="p-color-depths"></a>`COLOR_DEPTHS` | 1, 2, 4, 8 | allowed Color Depths in bits |

### 3.1 Numbers, text and bits
- Multi-byte numbers in binary Datagrams are **big-endian** (most significant byte first), as drawn in the bit diagrams.
- Text Datagrams are ASCII. Fields are separated by `|`; numbers inside text are decimal ASCII.
- Bits inside a byte are numbered from the most significant bit (leftmost in diagrams). Unused/reserved bits (`-`) are sent as 0 and ignored on receive.

### 3.2 Coordinates and voxel order
- Right-handed coordinate system. Origin (0,0,0) is the **bottom, front, left** corner as seen by a viewer standing in front of the Cube.
  - **X** increases to the right, **Y** increases away from the viewer (to the back), **Z** increases upward.
  - All Cubes, Datagrams, files and APIs use this convention.
- Voxels are always ordered by Voxel Index: x runs fastest, then y, then z.

### 3.3 Color channels
A Color Channel is named by one uppercase letter. A **Channel Arrangement** is a string of 1 to [`MAX_COLOR_CHANNELS`](#p-max-channels) letters, each letter at most once, in any combination. The order of the letters is the order in which the values are stored for each Voxel. These are proposed letters only, new ones may get added as the need arises.

| Letter | Channel | Meaning |
|---|---|---|
| `L` | Luminance | Single-color (monochrome) LED |
| `R` | Red | |
| `G` | Green | |
| `B` | Blue | |
| `W` | White | Unspecified or neutral/warm-tinted white LED |
| `C` | Cool white | Cool-tinted white LED |
| `A` | Amber | |
| `U` | Ultraviolet | |

Examples: `L`, `RGB`, `GRB` (same channels, different order).

### 3.4 Color conversion rules
Used by a Cube when it plays an Animation file whose channels or depth differ from its own, and by Hosts for the same purpose. Intermediate results use full precision and are rounded once at the end.
- **Depth:** `v_target = round(v_source × (2^targetDepth − 1) / (2^sourceDepth − 1))`. When scaling up this equals bit replication.
- **Channels** are matched by letter. For each channel of the target:
  - present in the source → copied.
  - `L` missing, source has `R`, `G`, `B` → `L = 0.2126·R + 0.7152·G + 0.0722·B` (Rec. 709).
  - `R`, `G`, `B` all missing, source has `L` → `R = G = B = L`.
  - anything else → *To Be Determined*.
- Source channels the target does not have are dropped. `W` and `C` of the source is added to R, G and B with a multiplier when target doesnt have `W` and `C`.

### 3.5 Gamma correction
Currently not applied anywhere (neither by the Cube nor by CubeDriver). Values are displayed as given.

### 3.6 Size conversion rules
- Size is never converted by a Cube, sizes must match exactly.
- Size may be converted by Hosts.



## 4. Cube behavior
No values appear in this chapter; they are in the parameter table of chapter 5.

### 4.1 Connections
A Cube has at least one of the following and accepts data on every one it has:
- **Ethernet:** accepts CubeLink Datagrams (chapter 5). It also sends information back: discovery replies, pongs and Frame-sync Datagrams. Ethernet is the only connection a Host can use.
- **SD card slot:** FAT32 card. The Cube plays the Animation files in its root folder (Cube Autoplay).
- **USB slot:** FAT32 flash drive. Same as the SD card slot.

There is no mode switch; the Cube chooses its mode automatically.

### 4.2 Autoplay Modes
Cubes change the mode they are in based on the following conditions, with highest priority being first:
| **Priority** | **Mode** | **Condition** |
| :-: | - | - |
|1 | **Network Mode** | A Host is streaming. |
|2 | **USB Autoplay** | A USB drive with at least one playable Animation is present. |
|3 | **SD Autoplay** | An SD card with at least one playable Animation is present. |
|4 | **Idle** | *None* |

The Cube re-evaluates the conditions immediately when a Host acquires or looses the Lock, or a USD drive or SD card is (dis)connected.

### 4.3 Cube autoplay
- Only Animation files in the root folder are used, all (sub)folder and their contents are ignored.
- A file is playable if its header is valid, its size (X, Y, Z) equals the Cube's size exactly and the data length is consistent. Channels and Color Depth are converted to the Cube's own (3.5).
- Playback order is undetermined, should be considered random.
- Each file plays `n = max(floor(AUTOPLAY_TIME / animationLength), 1)` times (see [`AUTOPLAY_TIME`](#p-autoplay-time)), with `animationLength = frameCount / fps` seconds.
- Files with a framerate that is not an integer multiple of the Cube's fps are played back using frame-shifting, aligning each Frame to the nearest display refresh. Files with a higher framerate than the Cube's are played back slower, at the Cube's framerate.
- Resuming is not guaranteed, playback order may change at any time.

### 4.4 Lock
- **Acquire:** while unlocked, the first valid Data Datagram (Frame Fragment or Sparse Update) from any IP address makes that address the Lock Holder. Nothing else (discovery, ping) acquires the Lock.
- **While locked:** Data Datagrams from other addresses are dropped silently.
- **Refresh:** any valid Data Datagram or `PING` from the Lock Holder restarts the Lock timer.
- **Release:** after [`LOCK_TIMEOUT`](#p-lock-timeout) without a *refreshing* Datagram. There is no explicit release Datagram.

### 4.5 Frame handling (network mode)
The Cube has memory for two Frames: **R** (received) and **D** (displayed).
- **Frame Fragments** are collected into R. When a fragment arrives, its Frame Index is compared with the Frame Index R currently holds:
  - **equal:** the fragment's data is stored at its place in R (overwriting the same fragment if it was received earlier).
  - **different (or R empty):** all of R is dropped; R now holds the new Frame Index and this fragment.
- When R holds all fragments of its Frame (Complete Frame), it is copied into D, which is what is displayed. Copying happens as soon as the display allows; until then R keeps the Complete Frame and new Frame Fragments are ignored. After the copy, R's Frame Index is reset and any new Frame Fragment will overwite R.
- **Sparse Updates** are applied to D directly and never touch R.
- Hosts should not send Frames faster than the Cube's frames per second.
- Latency from the last Datagram received to a visible change is at most [`MAX_LATENCY`](#p-max-latency).

### 4.6 Network handling
- The Cube gets its address by DHCP; if no server answers within [`DHCP_TIMEOUT`](#p-dhcp-timeout) it picks a random Link-local Address (checked for conflicts per RFC 3927). While in Idle mode, DHCP is rechecked every [`DHCP_RECHECK_TIME`](#p-dhcp-recheck-time).
- There is no fixed address; Hosts find Cubes by Broadcast discovery.
- All Datagrams in both directions use [`CUBELINK_PORT`](#p-cubelink-port). Replies always go to the sender's address.



## 5. CubeLink protocol

**Parameters**
| Name | Value | Notes |
|---|---|---|
| <a id="p-cubelink-port"></a>`CUBELINK_PORT` | `19523` | Source and destination port for all Datagrams |
| <a id="p-cubelink-version"></a>`CUBELINK_VERSION` | `1` | Current protocol version |
| <a id="p-max-udp-payload"></a>`MAX_UDP_PAYLOAD` | 1472 B | 1500 MTU - 20 IP - 8 UDP; no Datagram is larger |
| <a id="p-type-fragment"></a>`TYPE_FRAGMENT` | `0b1000` | Datagram type: Frame Fragment |
| <a id="p-type-sparse"></a>`TYPE_SPARSE` | `0b1001` | Datagram type: Sparse Update |
| (reserved types) | `0b1010` … `0b1111` | |
| <a id="p-lock-timeout"></a>`LOCK_TIMEOUT` | 30 s | Temporary for testing |
| <a id="p-host-keepalive"></a>`HOST_KEEPALIVE` | ≲ 15 s | Longest silence of a Host that wants to keep its Lock. |
| <a id="p-max-latency"></a>`MAX_LATENCY` | 3 Refresh Periods | Time between last Frame Fragment reception and display updating to new Frame. |
| <a id="p-autoplay-time"></a>`AUTOPLAY_TIME` | 30 s | Target time per file in Cube Autoplay |
| <a id="p-dhcp-timeout"></a>`DHCP_TIMEOUT` | `[TBD]` | Time before Link-local Address fallback |
| <a id="p-dhcp-recheck-time"></a>`DHCP_RECHECK_TIME` | `[TBD]` | Time between DHCP retries in Idle mode. |

### 5.1 General
- Every Datagram is one UDP payload, never larger than [`MAX_UDP_PAYLOAD`](#p-max-udp-payload), so it always fits in one Ethernet frame. Datagrams are never split, acknowledged or retransmitted at the protocol level.
- Malformed Datagrams (wrong length, unknown type, out-of-range values) are dropped silently.
- Control Datagrams (common datagrams) are text (ASCII, first byte < 0x80). Data Datagrams are binary and always start with a fragment type identifier nibble (first byte ≥ 0x80).

### 5.2 Versioning
- The protocol version is a single integer ([`CUBELINK_VERSION`](#p-cubelink-version)).
- **Every version change is breaking.** A Host implements the versions it knows and refuses a Cube that reports any other version, with a clear message. A Cube never has to understand a Host of another version.
- The start of the discovery reply, `WORLD|v<version>|`, never changes, so any Host can read the version of any Cube.
- Profile Names contain the version, so Profiles belong to exactly one version. Adding a new Profile does not change the version; changing anything existing in a breaking way does.

### 5.3 Datagram list
| Datagram | Direction | Kind | Purpose |
|---|---|---|---|
| Frame Fragment | Host → Cube | data | Carries one slice of a Frame's Framedata; all fragments of a Frame make one Frame. |
| Sparse Update | Host → Cube | data | Sets the color of a few Voxels without sending a whole Frame. |
| Discovery request (`HELLO`) | Host → Cube | common | Asks every Cube on the network to identify itself. |
| Discovery reply (`WORLD`) | Cube → Host | common | Describes the Cube: version, occupied flag, size, channels, Color Depth, fps, etc. |
| Ping (`PING`) | Host → Cube | common | Asks for an immediate answer, to measure latency and keep the Lock. |
| Pong (`PONG`) | Cube → Host | common | The answer to a ping. |
| Frame-sync (`FRAMESYNC`) | Cube → Lock Holder | common | Marks the start of every Refresh Period; lets the Host pace its Frames. |

### 5.4 Receiving rules
A received Datagram is classified as:
- First byte < 0x80 → text:
    - Exactly `HELLO` → discovery request.
    - Exactly `PING` → ping.
    - Otherwise dropped.
- First byte ≥ 0x80 → Data Datagram. Its type is the high nibble:
    - [`TYPE_FRAGMENT`](#p-type-fragment) → Frame Fragment.
    - [`TYPE_SPARSE`](#p-type-sparse) → Sparse Update.
    - Otherwise dropped.
- Otherwise dropped.

Length and contents are checked against the Cube's Profile (chapter 6). Hosts receive `WORLD…`, `PONG` and `FRAMESYNC`.

### 5.5 Host-side obligations (CubeDriver)
1. Discover Cubes with `HELLO`, read `WORLD`, check the version, build the Profile Name, refuse unknown versions and Profiles.
2. Use [`CUBELINK_PORT`](#p-cubelink-port) as source and destination port and never send Datagrams larger than [`MAX_UDP_PAYLOAD`](#p-max-udp-payload).
3. Send at most as many Frames per second as the Cube's fps; give each Frame a Frame Index that differs from the previous Frame's.
4. Keep the Lock alive with a Keepalive at least every [`HOST_KEEPALIVE`](#p-host-keepalive) while it wants to keep the picture.
5. Prefer Frame-sync for pacing when available. Measure latency with ping.
6. Choose between Frame and Sparse Updates by cost (chapter 6).


	
## 6. Common datagrams

### `HELLO`: Discovery request (host → Cube)
The 5 ASCII bytes `HELLO` (`48 45 4C 4C 4F`). Used to request all Cubes to identify themself and to request Cube info.
- Sent to the network broadcast address (or unicast to a known Cube). Hosts should broadcast on every active network interface.
- Never changes the Lock.

### `WORLD`: Discovery reply (Cube → host)
The 5 ASCII bytes `WORLD` (`57 4F 52 4C 44`) followed by 2 fields separated by `|` (`7C`), in ASCII. Used by Cubes to send their profile to Hosts.
- Sent to the requester's address.
- Hosts must refuse all Cubes with unknown-to-the-host Profiles.
- Never changes the Lock.
- Profiles are defined in chapter 7.

`WORLD|<occupied>|<profile_string>`

| Field | Meaning | Allowed values | Example |
|---|---|---|---|
| `WORLD` | identifier | N/A | `WORLD` |
| occupied | if a Host holds the Lock then `1`, else `0` | `0`, `1` | `0` |
| profile_string | The Profile identifier string. | any profile string | `v1-16x16x16-RGB8-60` |

Example: The reply for the example values above is `WORLD|0|v1-16x16x16-RGB8-60`.

### `PING`: Refresh + latency measurement (host → Cube)
The 4 ASCII bytes `PING` (`50 49 4E 47`). Used to reset the Lock Timer and to measure latency.
- Nothing else is in this datagram.
- Always answered.
- A ping from the Lock Holder restarts the Lock Timer; a ping from any other address changes nothing.

### `PONG`: Latency measurement (Cube → host)
The 4 ASCII bytes `PONG` (`50 4F 4E 47`). Used to measure latency.
- Nothing else is in this datagram.
- The host can measure the round-trip time itself; one-way latency ≈ round trip / 2.

### Frame-sync
The 9 ASCII bytes `FRAMESYNC` (`46 52 41 4D 45 53 59 4E 43`). Used to sync up Hosts with the Cubes refreshing rate.
- Sent unconditionally to the Lock Holder at the start of every refresh period while locked.
- Hosts are encouraged to use it to pace their Frames (for example one frame per frame-sync).



## 7. CubeLink profiles
A **Profile** defines the Data Datagrams for one class of Cube, described by its size, Channel Arrangement, Color Depth and fps. All Profiles share the common Datagrams of chapter 6.
- A Cube implements exactly one Profile, the one named by its discovery reply. A Host implements a set of Profiles and refuses Cubes whose Profile it does not know.
- Defined Profiles only change with version change. New Cubes get new Profiles, which may copy the layouts of existing ones.
- Datagram types are the same for all Profiles (chapter 5); the layout behind a type is per Profile.

### `v1-16x16x16-RGB8-60`
Meaning:
- Version: v1
- Size: 16 x 16 x 16 Voxels / LEDs (Cuboid)
- Color Channels: Red, Green, Blue (in this order)
- Color Depth: 8 bits per channel, 0-255.
- Refresh rate: 60 frames/second.

**Parameters**
| Name | Value | Notes |
|---|---|---|
| Frame size | 12,288 bytes | Number of bytes per Frame. |
| Frame Fragment count | 12 | Number of Frame Fragment Datagrams to complete 1 Frame. |
| Frame Fragment datagram size | 1028 bytes | Total size, header + data |
| Sparse Update datagram max size | 1402 bytes | Total size, header + 280 entries |
| Frame Fragment Frame Index size | 24 bits | Frame Index 0 … 16,777,215 |

#### Frame fragment
A Frame is sent as 12 fragments of exactly 1024 bytes.

```
byte  0     : 1000 xxxx                                type 1000, x = fragment number (0 .. 11)
bytes 1-3   : yyyy yyyy | yyyy yyyy | yyyy yyyy        y = frame index (24 bit)
bytes 4-1027: zzzz zzzz | ... (exactly 1024 bytes)     z = framedata
```
- The 12288 bytes of Framedata are cut into consecutive 1024-byte pieces; piece *k* goes into fragment number *k*. Voxel boundaries do **not** align with fragment boundaries.
- All fragments of a Frame carry the same Frame Index. A Frame's index must differ from the previous Frame's; normally it increments by 1 and wraps after 16,777,215. Only "different from the previous Frame" is required.
- Fragments may be sent back-to-back and in any order; the Cube buffers incoming Datagrams. A lost fragment loses the Frame; the Host just sends the next Frame. Reassembly rules: section 4.5.

#### Sparse update
Changes up to 280 Voxels of the displayed picture.

```
byte 0:    1001 --ll     type 1001, l = high 2 bits of the voxel count
byte 1:    llll llll                l = low 8 bits  (10 bit in total, 1 .. 280)
byte 5i+2: ---- zzzz     z coordinate (0..15), i = 0..(l-1)
byte 5i+3: yyyy xxxx     y and x coordinates (0..15)
byte 5i+4: rrrr rrrr     red channel value
byte 5i+5: gggg gggg     green channel value
byte 5i+6: bbbb bbbb     blue channel value
```
- The 12 coordinate bits `zzzz yyyy xxxx` form the Voxel Index `x + 16y + 256z`.
- The Datagram length must equal `SPARSE_HEADER + SPARSE_ENTRY · l` (at most [`SPARSE_MAX_DATAGRAM`](#p-p1-sparse-max-size)).
- Each Datagram is applied as a whole. If several Datagrams change the same Voxel, do not rely on their order.
- Example: set Voxel (x=3, y=5, z=7) to R=255, G=128, B=0 → `90 01 07 53 FF 80 00`.
- Guidance: a Frame costs [`FRAME_SIZE`](#p-p1-frame-size) bytes, a Sparse Update [`SPARSE_ENTRY`](#p-p1-sparse-entry) bytes per Voxel (break-even ≈ 2,400 Voxels; practically use Sparse Updates for ≲ 560 Voxels, 2 Datagrams).


### More profiles will be added here when a new Cube is made.
