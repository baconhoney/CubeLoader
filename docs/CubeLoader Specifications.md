# CubeLoader Specifications (draft 0.4)

CubeLoader is the host application for the LED Cube. It is the main control app for the Cube: it manages the GUI, files, playback, Programs and game controllers, and streams frames to the Cube through CubeDriver. It works with any Cube whose CubeLink profile CubeDriver knows, not only the current 16×16×16 one.

The network protocol and the Cube's own behavior are specified in `CubeProtocol Specifications.md` (the CubeLink Specifications). This document only gives an overview of them and does not define them.

## Table of contents
1. [Overview and scope](#1-overview-and-scope)
2. [Glossary](#2-glossary)
3. [Data conventions](#3-data-conventions)
4. [Cube working modes](#4-cube-working-modes)
5. [How the Cube communicates](#5-how-the-cube-communicates)
6. [CubeDriver](#6-cubedriver)
7. [File formats](#7-file-formats)
8. [Controller definitions](#8-controller-definitions)
9. [Architecture](#9-architecture)
10. [Usage modes](#10-usage-modes)
11. [Functions of the host app](#11-functions-of-the-host-app)
12. [Config](#12-config)
13. [Review items and open items](#13-review-items-and-open-items)



## 1. Overview and scope
- CubeLoader finds a Cube on the network, shows its state, and plays content on it: **animations** (pre-rendered files), **Programs** (Python code that generates frames, possibly interactively with game controllers) and **text**. It can run automatically (host autoplay) or be driven by a user (manual play).
- It supports any Cube up to the limits in chapter 3, as far as CubeDriver knows the Cube's profile. Nothing in CubeLoader may hard-code a size, channel arrangement or color depth: read them from the connected Cube.
- Two usage modes: **Full mode** (everything allowed) and **Restricted mode** (for visitors), see chapter 10.
- The GUI is touch-capable, scales to any screen, and works at the same time with touch, mouse and keyboard.
- Out of scope: 3D rendering of the Cube. A separate viewer app could stand in for a Cube by implementing CubeLink, so CubeLoader only needs a configurable Cube address (the local machine included).



## 2. Glossary
| Term | Meaning |
|---|---|
| **Animation** | A file holding a sequence of Frames and info on how to play them. |
| **Broadcast** | A UDP datagram sent to all devices on the local network at once. |
| **Channel Arrangement** | A string of Color Channel letters, e.g. `RGB`, stating which Color Channels a Cube or file has and in which order their values are stored for each Voxel. |
| **Color Channel** | One independently controlled color component of a Voxel, named by a letter (e.g. `R`, `G`, `B`). |
| **Color Depth** | The number of bits that represent the value of one Color Channel. |
| **Common Datagram** | A CubeLink datagram identified by ASCII text and valid for every Profile. |
| **Complete Frame** | A Frame of which all Frame Fragments have been received. |
| **Cube** | The LED cube device. |
| **Cube Address** | The IP address CubeLoader streams to (found by discovery or entered manually). |
| **Cube Autoplay** | The Cube playing Animations from its own SD card or USB flash drive, without a Host. |
| **Cube Parameters** | What CubeDriver reports about a Cube: its Profile name and whether a Host holds its Lock. |
| **CubeDriver** | The host-side Python module that implements the CubeLink protocol, and gives CubeLoader and Programs one API to use a Cube. |
| **CubeLink (Protocol)** | The network protocol formed by all UDP Datagrams exchanged between Cubes and Hosts or any other app. |
| **CubeLoader** | The reference and main application. |
| **Data Datagram** | A CubeLink datagram that carries Pixel Data. Its layout depends on the Profile. |
| **Datagram** | One UDP payload exchanged between a Cube and a host. |
| **Frame** | The colors of all Voxels at one moment. |
| **Frame Fragment** | One datagram carrying a slice of the Framedata of a Frame, plus the Frame Index. |
| **Frame Index** | A counter inside every Frame Fragment that tells which Frame the fragment belongs to. |
| **Frame-sync** | A datagram the Cube sends to its Lock Holder at the start of every refresh period. |
| **Framedata** | The binary data of one Frame: the values of all Color Channels of all Voxels, laid out as defined in the data conventions (see CubeLink specs). |
| **Full Mode** | The usage mode with all functions available, for the person operating the system. |
| **Game Controller** | A USB or Bluetooth input device with buttons and optional LEDs, connected to the Host. |
| **GUI** | The graphical user interface of CubeLoader. |
| **Host** | Any computer or program that sends data to a Cube. CubeLoader is the reference Host. |
| **Host Autoplay** | CubeLoader automatically cycling through animations and Programs. |
| **Idle Frame** | The built-in static picture a Cube shows when it has nothing else to display (not every Cube has one). |
| **Inactivity Timeout** | Time without user input after which CubeLoader clears the manual selection and returns to host autoplay. |
| **Keepalive** | Any datagram a host sends only to keep its lock. |
| **Lock** | An imaginary key provided to Hosts. If the Cube has an ethernet port, a single one exists, otherwise none. |
| **Lock Holder** | The Host that is currently holding the Lock and whose Data Datagrams are accepted by the Cube. |
| **Manual play** | Playing a user-selected animation or Program until it is cleared. |
| **Network mode** | The state of a Cube while a lock holder is streaming data to it. |
| **Partial frame** | A frame of which not all frame fragments have been received yet. |
| **Profile** | (CubeLink profile) The set of data datagram layouts (frame fragment, sparse update) for one class of Cube, identified by its profile name. |
| **Profile name** | `v<protocol version>-<sizeX>x<sizeY>x<sizeZ>-<channel arrangement><color depth>-<fps>`, read from the Cube's reply message. |
| **Program** | A python file that generates frames while it runs. Final name `[TBD]`. |
| **Program mode** | `passive` or `active`: how a `mixed` Program is run. |
| **Program type** | `static`, `passive`, `mixed` or `active` (7.3). |
| **Refresh period** | One display refresh of the Cube: 1 / fps. |
| **Restricted mode** | The usage mode for visitors: only a limited set of functions is available. |
| **Sparse update** | A datagram that changes the color of only some voxels of the displayed picture. |
| **Visitor** | A person using CubeLoader in Restricted mode. |
| **Voxel / LED** | One LED of the Cube, addressed by (x, y, z). The terms are used interchangeably. |



## 3. Data conventions

**Limits**
| Name | Value | Meaning |
|---|---|---|
| MAX_SIZE | 64 | maximum LEDs per axis (minimum 1) |
| MAX_CHANNELS | 8 | maximum color channels per voxel |
| COLOR_DEPTHS | 1, 2, 4, 8 | allowed color depths in bits |
| HOST_COLOR_VALUE | 0 .. 2^COLOR_DEPTHS-1 | host-side color value |

### 3.1 Numbers, text and bits
- CubeDriver builds and reads binary Datagrams; multi-byte numbers are big-endian. Nothing else in CubeLoader depends on byte order: animation files hold only ASCII text and single bytes, the config file is UTF-8 text.
- Packed values are stored most significant bit first.

### 3.2 Coordinates and voxel order
- Right-handed coordinate system. Origin (0,0,0) is the **bottom, front, left** corner as seen by a viewer standing in front of the Cube.
  - **X** increases to the right, **Y** increases away from the viewer (to the back), **Z** increases upward.
  - All Cubes, Datagrams, files and APIs use this convention.
- Voxels are always ordered by voxel index: x runs fastest, then y, then z.

### 3.3 Color channels
A color channel is named by one uppercase letter. A **channel arrangement** is a string of 1 to 8 letters, each letter at most once, in any combination. The order of the letters is the order in which the values are stored for each voxel.These are proposed letters only, new ones may get added as the need arises.

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

### 3.4 Framedata layout
- See [CubeLink Specifications.md](/docs/CubeLink%20Specifications.md), chapter 7.

### 3.5 Color conversion
Cubes and Hosts convert channels and depth to their own. The rules are defined in the CubeLink Specifications (Data conventions → Color conversion rules). In short: depth is scaled linearly, i.e. bit replication when scaling up; channels are matched by letter; `L` is derived from `R G B` (Rec. 709) and `R G B` from `L`; missing channels become 0, W and C can be added to RGB.



## 4. Cube working modes
Only what the host needs to know. Exact behavior and values: CubeLink Specifications (Cube behavior).
- **Network mode:** this Host holds the Cube's Lock and the Cube shows what the host sends. Another host holding the lock keeps this host out (the Cube reports "occupied" in discovery).
- **USB / SD autoplay:** the Cube plays animations from its own storage. This is what the Cube does when nobody holds its lock.
- **Idle:** the Cube shows its idle frame, if it has one.



## 5. How the Cube communicates
Overview only; every value and byte layout is in the CubeLink Specifications. CubeDriver does all of this (chapter 6).

- **Transport:** UDP datagrams on the CubeLink port (`CUBELINK_PORT`), in both directions. A datagram never exceeds one Ethernet frame (`MAX_UDP_PAYLOAD`); nothing is split, acknowledged or retransmitted. A lost datagram means a lost frame.
- **Finding a Cube:** the host broadcasts `HELLO`; each Cube answers with `WORLD...`.
- **Profile:** from the reply the host extracts the profile name and uses the profile of that name. Unknown version or profile → the Cube is refused.
- **Taking the Cube:** the first data datagram from a host makes it the lock holder. Other hosts are ignored until the lock expires.
- **Sending Frames:** data datagrams are either a Frame (several Frame Fragments, each with the same frame index) or a Sparse Update (a few voxels). Their layout depends on the profile. Frames must carry a frame index different from the previous frame's. Hosts should not send frames faster than the Cube's fps.
- **Keeping the lock:** any data datagram or `PING` from the lock holder restarts the Cube's lock timer; a host showing a static picture sends a keepalive at least every `HOST_KEEPALIVE`.
- **Latency and liveness:** `PING` is answered with `PONG`; round trip / 2 ≈ latency. Answers and `FRAMESYNC` datagrams tell the host the Cube is alive.
- **Pacing:** the Cube sends `FRAMESYNC` to its lock holder at the start of every refresh period; the host should use it to time its frames.

| Datagram | Direction | Identifier |
|---|---|---|
| Discovery request | host → Cube | `HELLO` |
| Discovery reply | Cube → host | `WORLD|…` |
| Ping | host → Cube | `PING` |
| Pong | Cube → host | `PONG` |
| Frame-sync | Cube → lock holder | `FRAMESYNC` |
| Frame fragment | host → Cube | data datagram, type `TYPE_FRAGMENT` |
| Sparse update | host → Cube | data datagram, type `TYPE_SPARSE` |



## 6. CubeDriver

### 6.1 What it is
CubeDriver is the Python module that talks to the Cube. CubeLoader and every Program use it as their only interface to the Cube; the rest of the code never builds datagrams. If the protocol changes, only CubeDriver changes.

### 6.2 What it does
- **Discovery:** broadcasts `HELLO` on all network interfaces, collects replies, builds profile names, marks unsupported Cubes.
- **Connection:** opens the connection to one Cube, then runs background tasks: keepalive, ping/latency measurement, frame-sync reception, loss detection and reconnecting.
- **Sending:** turns host frames and updates into datagrams of the Cube's profile: converts colors to the Cube's channel arrangement and depth, cuts frames into fragments, chooses between a full frame and sparse updates (whichever is cheaper), maintains the frame index, never sends faster than the Cube's fps, applies the brightness limit.
- **Pacing:** `wait_next_frame()` waits for the Cube's frame-sync if available, otherwise for a timer at the Cube's fps.
- **Controllers:** exposes keyboards and gamepads as controllers (chapter 8).

### 6.3 How it communicates with the Cube
UDP datagrams on the CubeLink port. The values common to all Cubes: a datagram is at most 1472 bytes and is never split; there are no acknowledgements. Everything else (datagram layouts, timeouts) is Cube-specific and comes from the CubeLink Specifications. CubeDriver gets the Cube's parameters from discovery and never assumes them.

### 6.4 Functions `[PROPOSED]`
Names and signatures are an outline. Colors are always RGB, 0–255 (3.6).

| Function | What it does |
|---|---|
| `discover(timeout)` | Finds Cubes; returns a list of Cube info (address, profile name, version, size, channels, depth, fps, occupied, supported). |
| `connect(address=None)` | Connects to the given Cube, or to the only one found; starts the background tasks; returns a Cube object. |
| `cube.size_x / size_y / size_z / fps` | Parameters of the Cube. Programs must use them instead of constants. |
| `cube.channels / cube.depth` | The Cube's channel arrangement and depth, for information (not needed to draw). |
| `cube.set(x, y, z, r, g, b)` | Sets one voxel in the pending frame. |
| `cube.get(x, y, z)` | Reads one voxel of the pending frame. |
| `cube.fill(r, g, b)` / `cube.clear()` | Fills the pending frame with one color / black. |
| `cube.show()` | Sends the pending frame (full frame or sparse update, as cheapest). |
| `cube.send_frame(array)` | Sends a complete host frame `(Z, Y, X, 3)`. |
| `cube.blackout()` | Sends an all-black frame and keeps the connection. |
| `cube.wait_next_frame()` | Waits for the next refresh period (frame-sync if available). Keeps frame pacing exact. |
| `cube.ping()` | Measures round-trip time. |
| `cube.has_lock / cube.connected / cube.rtt` | State of the connection. |
| `cube.set_brightness_limit(percent)` | Scales all outgoing colors (0–100 %). |
| `cube.disconnect()` | Stops sending; the Cube leaves network mode by itself. |
| `controllers` | Controller events and LEDs (chapter 8). |

Errors: `CubeLostError`, `UnsupportedCubeError`, `AnimationError`.



## 7. File formats

### 7.1 Animation file format
- Extension `.cubeanim`.
- **File names:** any legal name, preferred is ASCII only.
- **Structure:** always `header` + `\n` + `binary data`.

**Header** (one ASCII line, ended by a single line feed `\n` = 0x0A, no trailing `|`):
`<magic+version>|<sizeX>|<sizeY>|<sizeZ>|<channels>|<depth>|<frameCount>|<fps>`

| Field | Meaning | Allowed values |
|---|---|---|
| magic + version | `CUBE` + `v<versionNum>`, the file format version | `CUBEv1` |
| sizeX, sizeY, sizeZ | number of LEDs on that axis | integer, 1 .. 64 |
| channels | channel arrangement: the color channels and their order in the framedata | letters per section 3.3, at most 8, each once |
| depth | color depth: bits per color value | 1, 2, 4 or 8 |
| frameCount | number of frames in the file | integer ≥ 1 |
| fps | frames to send per second, spaced evenly | integer, 1 .. 60 |

Example: `CUBEv1|16|16|16|RGB|8|120|30\n` → 120 frames at 30 fps = 4 s.

**Data:** a binary dump of the framedata of all frames, in frame order, immediately after the `\n`. Each frame is laid out as in 3.4 (voxel order, channels in arrangement order, packed MSB-first, padded to a whole byte). The data length is exactly `frameCount × frameSize`. The example file has 120 × 12,288 = 1,474,560 data bytes. The data is binary: never search it for newlines.

**Validity:** a file is invalid if the magic or version is unknown, a header field is missing or out of range, or the data length does not match.

**Playing on a Cube:** the file's size must equal the Cube's size. Channels and depth are converted (3.5).

### 7.2 Loading files with other parameters (host only)
CubeLoader plays more animation files than a Cube can on its own. On load, CubeDriver adapts the file to the connected Cube:
- **Size:** a file that is smaller than the Cube on any axis is padded with black voxels so that the animation sits at the **front, center, bottom** of the Cube: X offset `floor((cubeSizeX − fileSizeX) / 2)`, Y offset 0, Z offset 0. A file that is larger on any axis is not playable (no downscaling).
- **Channels and depth:** converted with the rules of 3.5, in both directions.
- **fps:** Files with a framerate that is not an integer multiple of the Cube's fps are played back using frame-shifting, aligning each Frame to the nearest display refresh. Files with a higher framerate than the Cube's are played back slower, at the Cube's framerate.
- Files that cannot be adapted are shown as unavailable, with the reason.
- Conversions happen only in the host; the Cube plays only exact-size files.

### 7.3 Program file (`.cube.py`)
- Extension `.cube.py` `[TBD]`. The name without the extension can be shown in selectors. Read only by hosts, never by the Cube.
- Plain Python source that imports `CubeDriver`. It is trusted code (no sandbox, chapter 9).
- **Metadata** (module-level, readable without running the Program): `[PROPOSED]`
  ```python
  PROGRAM_TYPE   = "mixed"                    # required: static | passive | mixed | active
  PROGRAM_NAME   = "Snake"                    # optional, default: file name
  PROGRAM_INPUTS = {"controllers": (1, 1)}    # optional: inputs it needs (in active mode)
  ```
- **Entry point:** `def run(cube, ctx)`. `cube` is the CubeDriver Cube object (chapter 6). `ctx` offers `mode` (`"passive"` or `"active"`, for `mixed`), `should_stop()`, `sleep_until_next_frame()`, `controllers` and `time`.
- **Types:**
  | Type | Behavior | Host autoplay |
  |---|---|---|
  | `static` | generates one frame once; CubeDriver keeps it displayed | yes |
  | `passive` | generates frames continuously at any rate up to the Cube's fps, without external input | yes |
  | `mixed` | supports `passive` and `active` mode; the host passes the mode when starting it. Passive is used in autoplay and when inputs are disabled or unavailable | passive mode only |
  | `active` | needs continuous external input and generates frames from it | no |
  `[REVIEW 1]`
- A Program that needs inputs (`PROGRAM_INPUTS`) is not startable in active mode while they are missing (for example too few controllers), and is shown grayed out or hidden.
- Programs are mid-heavy on processing (math, trigonometry, FFT, simple physics). A wait-until-next-frame function keeps frame pacing exact.



## 8. Controller definitions
Not yet fully defined. Everything here is a guideline and can change.

- Up to 8 controllers, each with up to 10 buttons and optionally up to 10 LEDs. Wired (USB HID) and wireless (BLE HID) controllers can be mixed. Controllers connect to the host, never to the Cube.
- The host handles hot-plug, shows connected controllers and their player-slot assignment, and offers slot reassignment and input remapping. Keyboards, gamepads (Xbox, PlayStation, ...) and custom HID controllers should all work with games.

**Hybrid solution:** CubeDriver ships common backends (keyboard, standard gamepads) behind one event API: button press/release events with controller index, button and timestamp, and controller LED control. A Program may bypass them and open any device itself (hidapi, pygame, evdev, ...), so a new controller needs only a new Program.
- Caveats: keyboard input goes to the focused window (the GUI), so CubeLoader forwards keyboard events to the running Program through the CubeDriver API; exclusive keyboard capture is not possible on all systems. On Linux, raw device access needs a udev rule or group membership. Handling of extra Python packages used by Programs (one shared environment, or a requirements list per Program) is `[TBD]`.
- Target: input to visible frame within 20 ms (max 50) at 30 fps, scales with fps.



## 9. Architecture
A suggestion, not fixed. It describes what each part does and how they interact.

**GUI limits**
| Name | Value | Notes |
|---|---|---|
| <a id="p-min-resolution"></a>`MIN_RESOLUTION` | 640 × 480 | both orientations |
| <a id="p-max-resolution"></a>`MAX_RESOLUTION` | 8K (7680 × 4320) | both orientations |
| <a id="p-max-aspect"></a>`MAX_ASPECT` | 21:9 | landscape and portrait; wider must still work |
| <a id="p-touch-target"></a>`TOUCH_TARGET_MIN` | 10 mm (≈ 44 px) | |

### 9.1 Parts
```
 GUI  <----------->  Core  <------>  Playback engine ----> Program runner (separate process)
 (screen/web)        |   \                  |  \
                     |    \--> Config       |   \--> Animation player
                     |    \--> File scanner |
                     |    \--> Controller manager
                     v
                 Cube handler (CubeDriver connection) <----> Cube
```
- **GUI:** shows state and takes user input; no logic. Talks to the core through a defined interface.
- **Core:** owns all state and logic: usage mode and permissions, selection, playback control, config, file scanning, Cube connection state. Enforces permissions (chapter 10).
- **Cube handler:** the CubeDriver connection(s). Objects, not a global singleton, so several Cubes could be added later (not implemented).
- **Playback engine:** decides what plays (manual play, host autoplay, text, inactivity timeout) and runs the animation player or Program runner.
- **Animation player:** loads an animation and streams its frames at the file's fps.
- **Program runner:** runs Programs in a separate process so they can be stopped or killed without harming the app; it is for crash isolation, not security. Programs are trusted code.
- **Controller manager:** reads controllers through the CubeDriver backends and forwards events to the running Program.
- **File scanner:** lists animations and Programs, validates them, applies exclusions.
- **Config:** loads, validates and saves settings (chapter 12).

### 9.2 Guidelines
- Playback and streaming run independently of the GUI: a slow or frozen GUI must never cause frame glitches, and closing the GUI does not stop playback unless the app is quit.
- Core must not depend on the GUI toolkit. GUI technology proposal: PySide6. A web GUI served by the core is also possible; then it listens on localhost only, plus authenticated access for Full mode if needed.
- **GUI behavior:**
  - Auto-sizes to the available screen/window; works for resolutions between [`MIN_RESOLUTION`](#p-min-resolution) and [`MAX_RESOLUTION`](#p-max-resolution) and aspect ratios up to [`MAX_ASPECT`](#p-max-aspect) in both orientations. Beyond that it stays usable (content width capped, remaining area empty). No fixed pixel positions.
  - Modern and clean; every function reachable in few steps from a main screen. "Now playing", STOP and Cube-status indicators are always visible.
  - Touch and mouse+keyboard work at the same time. No function depends on hover, right-click or keyboard-only input. Touch targets at least [`TOUCH_TARGET_MIN`](#p-touch-target). An on-screen keyboard is available for text fields on touch-only systems.
  - Long lists scroll (touch drag and mouse wheel, visible scrollbar), load lazily when large, and can be searched, filtered and sorted.
  - Non-blocking feedback (toast/status line) for errors such as Cube lost, invalid file, Program crashed. Long operations never freeze the UI.
  - Shows only what the current usage mode allows.



## 10. Usage modes

### 10.1 Intended use
1. **Full mode:** the main control app, run on any machine. Everything is unlocked and freely usable.
2. **Restricted mode:** runs next to the Cube with a touch monitor and keyboard (exhibition). Visitors can interact but cannot change system behavior.

### 10.2 Permissions
| Function | Full | Restricted |
|---|---|---|
| Select and play animations/Programs (not excluded ones), manual play | yes | yes |
| Text display: text, color, speed, direction | yes | yes |
| STOP short press (blackout, stays connected) | yes | no |
| STOP long press (disconnect) | yes | no |
| Cube info panel | full | read-only |
| Settings (config, inactivity timeout, exclusions, Cube address, brightness limit, folders) | yes | no |
| Import/export/delete files, export to file | yes | no |
| Quit the app | yes | no |
| Switch mode | into Restricted by button or timeout | into Full by unlock |

### 10.3 Switching
- **Into Full mode:** a (secret) gesture or key combination opens a PIN prompt; the PIN is the UDP port number `[TBD]`.
- **Into Restricted mode:** a button in the Full-mode GUI. Optionally also automatically after 30 seconds of inactivity (this timeout can be disabled).
- The start mode is configurable (chapter 12).

### 10.4 Requirements
- Permissions are enforced in the **core**, not merely by hiding GUI elements (important if the GUI is a web page).
- When the inactivity timeout fires in Restricted mode, runtime settings (selection, text, text color and speed) are reset to their defaults, so every visitor starts clean.
- Text input is length-limited to 10000 or so characters.
- OS-level kiosk protection (blocking Alt+Tab etc.) is not the app's job.



## 11. Functions of the host app

### 11.1 Main functions
- **Manual play:** plays the item the user selected (animation or Program) until the user clears or stops it, or the inactivity timeout fires.
- **Host autoplay:** plays animations and Programs automatically from the configured folders and **all their subfolders** (symbolic links are not followed). Only Programs of type `static`, `passive`, and `mixed` (in passive mode) take part.
  - Each item runs for 30s. An animation loops as many whole times as it fits into the limit, but at least once: `n = max(floor(timeLimit / animationLength), 1)`. Programs run for the same time.
  - Order is random (default) or sequential. Items that are not playable are skipped.
- **Text display:** shows text typed into a text box (from an external or on-screen keyboard). Scroll direction, speed and color (static and dynamic) can be changed live. Font and rendering details `[TBD]`.

A Program is stopped on request within 1s (10s for testing), otherwise its process is killed. A Program that raises an error or hangs is stopped, the error is shown, and host autoplay (if running) moves on.

### 11.2 Other functions
- **Inactivity timeout** (toggleable in config): after 30s without user input (touch, mouse or keyboard), the manual selection is cleared and host autoplay starts. In Restricted mode runtime settings are reset too (10.4).
- **STOP button**:
  - short press (< 3s): sends an all-black frame and keeps the connection alive, so the Cube never leaves network mode.
  - long press: sends an all-black frame and disconnects from the Cube; all communication stops until the user connects again.
- **Brightness control:** an optional global factor (0–100 %) applied to all outgoing frames by CubeDriver. Config only.
- **Export to file:** records the output of a Program (or the text display) into an animation file. The user picks the duration; the frame count is `duration × fps`. A `mixed` Program is recorded in passive mode. File output shall be optimised to the smallest available size without data loss.
- **Exclusion:** files or folders can be excluded from host autoplay and/or from the selector list. Exclusions are stored in the config and apply to the folder and everything below it.
- **Cube info panel:** while open, it refreshes every 2 seconds and shows every Cube found with its Cube parameters, the measured round-trip time and approximate latency (round trip / 2), and whether this host or another host holds the lock.
- **Connecting to Cubes:** If another host holds the lock, the host says so and waits; it cannot take the Cube over. If the Cube does not answer for `[TBD]` seconds, the host shows a status indicator, keeps retrying and resumes playback when the Cube returns. A Cube with an unsupported version or profile is refused with a clear message.
- **Cube address:** discovered automatically, or entered manually (bypasses discovery; discovery still feeds the info panel). Only one Cube is controlled at a time (for now).
- Files that cannot be played (invalid or incompatible) are listed as unavailable, with the reason.



## 12. Config
- Every playback parameter is configurable in the app or in the config file. The file is human-readable (e.g. TOML or JSON) and stored in a fixed location (not user-changeable).
- Unknown or invalid values fall back to defaults with a warning; the app starts even with a damaged config file. Limits below are only those that would otherwise cause errors.
- **Who can change:** Full mode (GUI) and the config file. Restricted mode changes only runtime state (current selection, text, text color and speed), never the stored config. Default values that already appear in chapter 11 are not repeated here.

| Setting | Default | Changed by | Limits |
|---|---|---|---|
| `cube_address` | automatic | Full mode, file | `auto` or a valid IPv4 address (or resolvable host name) |
| `content_folders` | `[TBD]` | Full mode, file | existing readable folders; missing ones are skipped with a warning |
| `autoplay_switch_time` | 30s | Full mode, file | > 0 s |
| `autoplay_order` | random | Full mode, file | `random` or `sequential` |
| `inactivity_timeout` | 30s | Full mode, file | > 0 s when enabled |
| `exclusions` | none | Full mode, file | list of file/folder paths |
| `brightness_limit` | 100% | file only | 0–100 % |
| `startup_mode` | `[TBD]` | file only | `full` or `restricted` |
| `startup_item` | none | Full mode, file | name of an animation/Program, optional parameter |
| `full_mode_pin` | port number | file only | non-empty text |
| `full_mode_timeout` | 30s | Full mode, file | > 0 s when enabled |
| `restricted_permissions` | table in 10.2 | file only | known permission names |
| `text_defaults` | `[TBD]` | Full mode, file | color values within 0–255, speed > 0 |
| `gui_listen` (web GUI) | localhost | file only | valid address and port |
| `log_level` | info | Full mode, file | `debug`, `info`, `warning`, `error` |



## 13. Open items

**Open items**
- GUI technology; config file format; unlock gesture; `FULL_MODE_TIMEOUT`.
- Controller API, Program dependency handling, Program API details.
- Text display (font, rendering, color options).
- Gamma correction (currently none anywhere).
