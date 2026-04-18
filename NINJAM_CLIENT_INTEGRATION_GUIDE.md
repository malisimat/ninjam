# NINJAM Client Integration Guide

> **Purpose:** A reference for extracting the minimal C/C++ code needed to connect to a NINJAM server as a client, without the server code and without a native GUI (suitable for integration into a project with its own rendering/UI pipeline, e.g. OpenGL).

---

## 1. Overview of the Architecture

NINJAM uses a clean separation between the *client engine* (`NJClient`) and any *GUI or audio backend*. This makes extraction straightforward. The protocol layer, network layer, audio mixing/encoding engine, and UI layers are distinct modules.

```
┌────────────────────────────────────────────────────┐
│  Your Application (OpenGL GUI, your audio backend) │
├────────────────────────────────────────────────────┤
│  NJClient  (njclient.h / njclient.cpp)             │  ← THE ENGINE TO KEEP
│    - Connect/Disconnect                            │
│    - Run() loop (drives net + mixing)              │
│    - AudioProc() (drives audio encode/decode)      │
│    - Chat callbacks                                │
│    - Remote user/channel state queries             │
├────────────────────────────────────────────────────┤
│  Net_Connection + Net_Message                      │  ← KEEP
│  (netmsg.h / netmsg.cpp)                          │
│  Message Parsers/Builders (mpb.h / mpb.cpp)        │  ← KEEP
├────────────────────────────────────────────────────┤
│  JNetLib TCP layer  (WDL/jnetlib/)                 │  ← KEEP (subset only)
├────────────────────────────────────────────────────┤
│  WDL utility headers (WDL/*.h, WDL/sha.cpp, etc.)  │  ← KEEP (subset)
├────────────────────────────────────────────────────┤
│  OGG Vorbis encode/decode  (libvorbis + libogg)    │  ← KEEP (system or bundled)
└────────────────────────────────────────────────────┘

NOT needed:
  ninjam/server/          – server (ninjamsrv.cpp, usercon.*)
  ninjam/guiclient/       – Win32 GUI client
  ninjam/winclient/       – Win32-specific client UI
  ninjam/cocoaclient/     – macOS Cocoa client
  ninjam/cursesclient/    – Curses text UI (but useful as a minimal reference!)
  ninjam/chanmix.*        – Win32 channel mixer dialog
  ninjam/audiostream_*.*  – Platform audio backends (replace with your own)
  ninjam/audioconfig.cpp  – Win32 audio device picker
  ninjam/njmisc.cpp       – Win32-only Jesusonic plugin integration
  WDL/lice/               – GUI image library
  WDL/swell/              – macOS/Linux Win32 API emulation
  WDL/wingui/             – Win32 GUI helpers
  WDL/jnetlib/httpget.*   – HTTP (not used by client)
  WDL/jnetlib/httpserv.*  – HTTP server
  WDL/jnetlib/listen.*    – TCP listen (server only)
  WDL/jnetlib/webserver.* – Web server
```

---

## 2. Files to Keep — Exact List

### 2a. Core Client Engine

| File | Lines | Purpose |
|------|-------|---------|
| `ninjam/njclient.h` | 289 | **Public API** of `NJClient` class — the entry point |
| `ninjam/njclient.cpp` | 3334 | **Full implementation**: auth, run loop, audio mixing, download/decode, encoding, session management |
| `ninjam/netmsg.h` | 151 | `Net_Message` and `Net_Connection` class declarations |
| `ninjam/netmsg.cpp` | 323 | Network message framing, send/receive loop, keepalive |
| `ninjam/mpb.h` | 293 | All NINJAM protocol message parser/builder class declarations |
| `ninjam/mpb.cpp` | 920 | Implementations of all message parsers/builders |

### 2b. WDL Network Layer (minimal subset)

| File | Purpose |
|------|---------|
| `WDL/jnetlib/jnetlib.h` | Top-level include |
| `WDL/jnetlib/netinc.h` | Platform socket includes |
| `WDL/jnetlib/asyncdns.h` + `asyncdns.cpp` | Async DNS resolution |
| `WDL/jnetlib/connection.h` + `connection.cpp` | TCP connection (JNL_Connection, JNL_IConnection) |
| `WDL/jnetlib/util.h` + `util.cpp` | Socket utility functions |

> **Do NOT include:** `httpget`, `httpserv`, `listen`, `webserver` — they are not used by the client.

### 2c. WDL Utility Headers (all header-only unless noted)

| File | Notes |
|------|-------|
| `WDL/wdltypes.h` | Fundamental typedefs, macros (`WDL_FIXALIGN`, etc.) |
| `WDL/wdlstring.h` | `WDL_String` — dynamic string class |
| `WDL/wdlcstring.h` | Safe string functions (`lstrcpyn_safe`, `snprintf_append`) |
| `WDL/heapbuf.h` | `WDL_HeapBuf` — growable heap buffer |
| `WDL/queue.h` | `WDL_Queue` — byte queue for net buffers |
| `WDL/fastqueue.h` | Used by `WDL_Queue` internals |
| `WDL/ptrlist.h` | `WDL_PtrList<T>` — pointer list container |
| `WDL/mutex.h` | `WDL_Mutex`, `WDL_MutexLock` — cross-platform mutex |
| `WDL/sha.h` + `WDL/sha.cpp` | SHA-1 for authentication (compile `sha.cpp`) |
| `WDL/rng.h` + `WDL/rng.cpp` | Random number generator for GUIDs (compile `rng.cpp`) |
| `WDL/pcmfmtcvt.h` | PCM sample format conversion helpers (header-only) |
| `WDL/wavwrite.h` | Optional: WAV file writer for local recording |
| `WDL/win32_utf8.h` + `WDL/win32_utf8.c` | UTF-8 `fopen`/file wrappers. On non-Windows, `fopenUTF8` maps to standard `fopen`. |

### 2d. Audio Codec Dependency

NINJAM uses **OGG Vorbis** exclusively for audio encoding/decoding:

| Dependency | Notes |
|------------|-------|
| `libvorbis` + `libvorbisenc` | Encoding (transmitting audio) |
| `libogg` | Ogg bitstream framing |
| `WDL/vorbisencdec.h` | WDL wrapper — inline `VorbisEncoder` / `VorbisDecoder` classes. **Include this in njclient.cpp compilation.** |

The `WDL/vorbisencdec.h` file includes `vorbis/vorbisenc.h` and `vorbis/codec.h` from the system or bundled Vorbis SDK. On most Linux distributions: `sudo apt install libvorbis-dev libogg-dev`.

> **Note on `WDL/vorbisencdec.h`:** It also includes `WDL/assocarray.h` and `WDL/lice/lice.h` (for FLAC picture embedding metadata). These are only used in the `VorbisEncoder` when `VORBISENC_WANT_FULLCONFIG` is defined. For NINJAM client use you can add the stub: `#define WDL_VORBIS_INTERFACE_ONLY` before including, or see section 4 below for the cleaner approach.

### 2e. Optional: `ninjam/njmisc.h` + `ninjam/njmisc.cpp`

These provide dB/volume/pan utility functions (`DB2SLIDER`, `SLIDER2DB`, `VAL2DB`, `mkvolstr`, etc.) used by UIs. They are **not required by `NJClient`** itself—only by the UI. The Jesusonic plugin code inside `njmisc.cpp` is `#ifdef _WIN32`-gated. You can take just the utility functions you need, or include the whole file and ignore the Win32 plugin code.

---

## 3. Files to Exclude (What NOT to Copy)

### Server
```
ninjam/server/ninjamsrv.cpp
ninjam/server/usercon.cpp
ninjam/server/usercon.h
ninjam/server/Makefile
```

### GUI clients (all)
```
ninjam/guiclient/         – Win32 GUI (large, Windows-only)
ninjam/winclient/         – Win32 helper dialogs/integration
ninjam/cocoaclient/       – macOS Cocoa UI
ninjam/cursesclient/      – Curses TUI (good reference but not needed)
```

### Audio backends (replace with your own)
```
ninjam/audiostream.h          – interface (see Section 5)
ninjam/audiostream_alsa.cpp   – Linux ALSA backend
ninjam/audiostream_jack.cpp   – Linux JACK backend
ninjam/audiostream_mac.cpp    – macOS CoreAudio backend
ninjam/audiostream_win32.cpp  – Windows WaveOut/DirectSound/KS backend
ninjam/audioconfig.cpp        – Win32 audio device selection dialog
```

### Win32-only channel mixer UI
```
ninjam/chanmix.h
ninjam/chanmix.cpp
```

### Heavy WDL subsystems not needed
```
WDL/lice/          – LICE image/GUI rendering library
WDL/swell/         – macOS/Linux Win32 emulation layer
WDL/wingui/        – Win32 GUI helpers
WDL/convoengine.*  – Convolution engine (DSP, not needed)
WDL/resample.*     – Resampler (not needed unless doing srate conversion in NJClient)
WDL/lameencdec.*   – MP3 encoder (not used by NINJAM client)
WDL/filebrowse.*   – File browser dialogs
WDL/jnetlib/httpget.*
WDL/jnetlib/httpserv.*
WDL/jnetlib/listen.*
WDL/jnetlib/webserver.*
WDL/jnetlib/irc_util.h
```

---

## 4. Dependency Handling — Avoiding `lice.h`

`WDL/vorbisencdec.h` pulls in `WDL/lice/lice.h` for an external bitmap loader used in FLAC picture metadata embedding. To avoid this dependency without modifying the WDL source:

**Option A (cleanest):** When including `vorbisencdec.h` in your build, define `WDL_VORBIS_INTERFACE_ONLY`. This is already done in `njclient.cpp` when built for REANINJAM (`#define WDL_VORBIS_INTERFACE_ONLY`), but for a standalone client you can add it to the compilation of `njclient.cpp`:

```cpp
// At the top of njclient.cpp, before the vorbis include, or as a -D flag:
#define WDL_VORBIS_INTERFACE_ONLY
```

Wait — if you do this, the concrete `VorbisEncoder`/`VorbisDecoder` classes won't be defined and `CreateNJEncoder`/`CreateNJDecoder` macros (in njclient.cpp) will fail. The correct approach is:

**Option B (recommended):** In the `njclient.cpp` build unit, ensure `VORBISENC_WANT_FULLCONFIG` is **not** defined (it is off by default). The LICE dependency is only triggered when `VORBISENC_WANT_FULLCONFIG` is defined. With the default build settings, `_LICE_LoadImage` is referenced but only used conditionally — you just need to provide a stub:

```cpp
// In one of your .cpp files (not a header):
#include "WDL/lice/lice.h"
LICE_IBitmap* (*_LICE_LoadImage)(const char*, LICE_IBitmap*, bool) = nullptr;
```

Or **Option C (simplest for full isolation):** Strip the LICE-dependent block from `vorbisencdec.h` lines ~369–385 (the `HasScheme("FLACPIC", ...)` block) and remove the `lice.h` include and `_LICE_LoadImage` extern. This is a small, low-risk edit.

---

## 5. Integrating into Your Project — Step by Step

### 5.1 Provide Your Audio Backend

`NJClient` does not call any platform audio API directly. You need to call `NJClient::AudioProc(...)` from your audio thread. The signature is:

```cpp
void NJClient::AudioProc(
    float **inbuf,   // array of nch input buffers (from microphone/instrument)
    int innch,       // number of input channels
    float **outbuf,  // array of nch output buffers (you fill the mix here)
    int outnch,      // number of output channels
    int len,         // number of samples per buffer
    int srate,       // sample rate in Hz (e.g. 44100, 48000)
    bool justmonitor = false,  // if true, only monitor local input (no network)
    bool isPlaying = true,
    bool isSeek = false,
    double cursessionpos = -1.0
);
```

Call this from your audio callback. Example stub:

```cpp
// Your audio callback (whatever form your audio system uses):
void myAudioCallback(float** inputs, float** outputs, int nframes, int srate)
{
    g_njclient->AudioProc(inputs, 2, outputs, 2, nframes, srate);
}
```

### 5.2 Run the Client Loop

Call `NJClient::Run()` regularly from your main/update thread — at least every 50ms. It returns nonzero when sleeping is OK:

```cpp
// In your update thread or main loop timer:
while (!g_njclient->Run());  // call repeatedly until it says "sleep ok"
```

Or in a background thread:

```cpp
void networkThread()
{
    while (running) {
        WDL_MutexLock lock(&myMutex);
        if (!g_njclient->Run())
            continue;  // Run wants to be called again immediately
        // sleep briefly
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}
```

### 5.3 Connect and Configure

```cpp
NJClient *client = new NJClient();

// Set a working directory for temporary OGG files
client->SetWorkDir("/tmp/ninjam_work/");

// Register callbacks
client->LicenseAgreementCallback = [](void* user, const char* license) -> int {
    // Return 1 to accept, 0 to decline
    printf("License: %s\n", license);
    return 1;  // auto-accept
};
client->LicenseAgreement_User = nullptr;

client->ChatMessage_Callback = [](void* user, NJClient* inst, const char** parms, int nparms) {
    if (parms[0]) printf("[Chat] %s: %s\n", parms[1] ? parms[1] : "server", parms[2] ? parms[2] : "");
};
client->ChatMessage_User = nullptr;

// Configure local channel
client->SetLocalChannelInfo(0, "My Channel",
    /*setsrcch=*/true,  /*srcch=*/0,
    /*setbitrate=*/true, /*bitrate=*/64,
    /*setbcast=*/true,  /*broadcast=*/true);

// Connect
client->Connect("yourserver.example.com:2049", "username", "password");
```

### 5.4 Monitor Connection Status

```cpp
int status = client->GetStatus();
switch (status) {
    case NJClient::NJC_STATUS_PRECONNECT:   // connecting / authenticating
    case NJClient::NJC_STATUS_OK:           // connected and in a room
    case NJClient::NJC_STATUS_CANTCONNECT:  // TCP connection failed
    case NJClient::NJC_STATUS_INVALIDAUTH:  // auth rejected
    case NJClient::NJC_STATUS_DISCONNECTED: // disconnected after being connected
}
```

### 5.5 Read Remote User/Channel Info (for your UI)

```cpp
// Call HasUserInfoChanged() each frame — returns 1 if changed since last call
if (client->HasUserInfoChanged()) {
    int nusers = client->GetNumUsers();
    for (int u = 0; u < nusers; u++) {
        float vol, pan; bool mute;
        const char *name = client->GetUserState(u, &vol, &pan, &mute);
        // Enumerate channels for this user
        for (int i = 0; ; i++) {
            int cidx = client->EnumUserChannels(u, i);
            if (cidx < 0) break;
            bool sub; float cvol, cpan; bool cmute, csolo;
            const char *chname = client->GetUserChannelState(u, cidx, &sub, &cvol, &cpan, &cmute, &csolo);
        }
    }
    // BPM/BPI
    float bpm = client->GetActualBPM();
    int bpi   = client->GetBPI();
}
```

---

## 6. Protocol Summary (for Reference)

NINJAM uses a custom binary TCP protocol on port **2049** (default).

### Message Format
```
[1 byte type] [4 bytes little-endian payload length] [payload bytes...]
```

### Handshake Sequence
```
Server → Client:  0x00  AUTH_CHALLENGE     (8-byte challenge, server caps, proto version, license text)
Client → Server:  0x80  AUTH_USER          (SHA1(SHA1(user:pass) + challenge), username, client caps)
Server → Client:  0x01  AUTH_REPLY         (flag: success/fail, max channels, effective username)
Server → Client:  0x02  CONFIG_CHANGE      (BPM, BPI — audio sync info, enables audio)
Server → Client:  0x03  USERINFO_CHANGE    (user/channel add/remove/update)
```

### Ongoing Messages
| Type | Direction | Purpose |
|------|-----------|---------|
| `0x02` | S→C | BPM/BPI change |
| `0x03` | S→C | User/channel info change |
| `0x04` | S→C | `DOWNLOAD_INTERVAL_BEGIN` — announces an OGG block to download |
| `0x05` | S→C | `DOWNLOAD_INTERVAL_WRITE` — chunk of OGG audio data |
| `0x81` | C→S | `SET_USERMASK` — subscribe/unsubscribe to remote channels |
| `0x82` | C→S | `SET_CHANNEL_INFO` — announce your channels |
| `0x83` | C→S | `UPLOAD_INTERVAL_BEGIN` — announce you are uploading an OGG block |
| `0x84` | C→S | `UPLOAD_INTERVAL_WRITE` — chunk of your OGG audio |
| `0xC0` | Both | `CHAT_MESSAGE` — text chat (MSG, PRIVMSG, TOPIC, SESSION) |
| `0xfd` | Both | KEEPALIVE — no payload |

### Authentication Detail
```
hash1 = SHA1(username + ":" + password)
hash2 = SHA1(hash1 + challenge[8 bytes])
Send hash2 as passhash in AUTH_USER message.
```
See: `ninjam/njclient.cpp` lines ~1032–1079, `WDL/sha.h`, `WDL/sha.cpp`.

---

## 7. Recommended Minimal File Set for Extraction

```
ninjam/
    njclient.h
    njclient.cpp
    netmsg.h
    netmsg.cpp
    mpb.h
    mpb.cpp

WDL/
    wdltypes.h
    wdlstring.h
    wdlcstring.h
    heapbuf.h
    queue.h
    fastqueue.h
    ptrlist.h
    mutex.h
    sha.h
    sha.cpp
    rng.h
    rng.cpp
    pcmfmtcvt.h
    wavwrite.h         (optional — only if you want local WAV recording)
    win32_utf8.h
    win32_utf8.c
    vorbisencdec.h     (include from njclient.cpp build unit — needs vorbis/ogg)
    jnetlib/
        jnetlib.h
        netinc.h
        asyncdns.h
        asyncdns.cpp
        connection.h
        connection.cpp
        util.h
        util.cpp

External:
    libvorbis + libvorbisenc + libogg  (system packages or bundled SDK)
```

**Total NINJAM-specific C/C++ lines: ~4,577** (njclient: 3334, mpb: 920, netmsg: 323)

---

## 8. Minimal Build Example (Linux/CMake)

```cmake
cmake_minimum_required(VERSION 3.14)
project(ninjam_client_lib)

find_package(PkgConfig REQUIRED)
pkg_check_modules(VORBIS REQUIRED vorbis vorbisenc)
pkg_check_modules(OGG REQUIRED ogg)

add_library(ninjam_client STATIC
    ninjam/njclient.cpp
    ninjam/netmsg.cpp
    ninjam/mpb.cpp
    WDL/sha.cpp
    WDL/rng.cpp
    WDL/jnetlib/asyncdns.cpp
    WDL/jnetlib/connection.cpp
    WDL/jnetlib/util.cpp
    # Add a stub for _LICE_LoadImage if not stripping it from vorbisencdec.h:
    # your_lice_stub.cpp
)

target_include_directories(ninjam_client PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${VORBIS_INCLUDE_DIRS}
    ${OGG_INCLUDE_DIRS}
)

target_link_libraries(ninjam_client PUBLIC
    ${VORBIS_LIBRARIES}
    ${OGG_LIBRARIES}
    pthread
)
```

Equivalent Makefile (Linux, matching `cursesclient/Makefile` structure):
```makefile
OBJS = ninjam/njclient.o ninjam/netmsg.o ninjam/mpb.o \
       WDL/sha.o WDL/rng.o \
       WDL/jnetlib/asyncdns.o WDL/jnetlib/connection.o WDL/jnetlib/util.o

CXXFLAGS = -O2 -I. -std=c++11
LFLAGS = -lvorbis -lvorbisenc -logg -lpthread

libninjam.a: $(OBJS)
    ar rcs $@ $(OBJS)
```

---

## 9. Key Integration Notes

1. **Thread safety:** `NJClient::Run()` and any code reading remote channel state must share a mutex. `AudioProc()` uses its own internal locking and can be called from a separate audio thread safely.

2. **Working directory:** Call `client->SetWorkDir(path)` with a writable directory. NJClient writes temporary `.OGGv` files there for incoming audio. Set `config_savelocalaudio = -1` to delete them ASAP if disk space is a concern.

3. **No `#define NJCLIENT_NO_XMIT_SUPPORT`:** Leave this undefined unless you are writing a receive-only client (like njcast). With it undefined, local channel upload support is compiled in.

4. **`njmisc.cpp` is optional:** The `NJClient` class does not depend on it. `njmisc.h`/`njmisc.cpp` only add dB UI helpers and Win32 Jesusonic plugin support. Skip both if you don't need dB conversion utilities.

5. **`audiostream.h` is not needed:** This is just an interface used by the bundled audio backends. You do not include it; you simply call `NJClient::AudioProc()` directly from your audio engine.

6. **`chanmix.h`/`chanmix.cpp`:** Win32 only, dialog-based channel mixer. Not needed. If you want channel mixing, use `NJClient::ChannelMixer` callback instead.

7. **Port:** Default NINJAM port is **2049**, encoded as `NJ_PORT` in `njclient.cpp`. The host string passed to `Connect()` can include a port override: `"host:port"`.

8. **Protocol version:** Client announces `PROTO_VER_CUR = 0x00020000`. The server must be at least `PROTO_VER_MIN = 0x00020000`. Standard NINJAM servers (Cockos, open-source) all speak this version.
