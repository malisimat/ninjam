# libninjam — NINJAM Client Static Library

A minimal, buildable extraction of the [NINJAM](https://www.cockos.com/ninjam/) client engine (`NJClient`) as a static library. All GUI code, audio backends, and server code have been removed. Only the network protocol, audio encode/decode pipeline, and session management remain.

The public API surface is `ninjam/njclient.h`. Everything else in this repository is an implementation detail.

---

## Repository Layout

```
ninjam/
  njclient.h        ← Public API (the only header you consume)
  njclient.cpp      ← Full engine implementation
  netmsg.h/.cpp     ← Network message framing
  mpb.h/.cpp        ← NINJAM protocol message parsers/builders
WDL/
  vorbisencdec.h    ← OGG Vorbis encoder/decoder wrapper
  jnetlib/          ← Minimal TCP networking (asyncdns, connection, util)
  *.h / sha.cpp     ← WDL support headers and crypto
CMakeLists.txt      ← Build system
```

---

## Dependencies

| Dependency | Purpose | Install (Ubuntu/Debian) |
|---|---|---|
| **libvorbis** + **libvorbisenc** | OGG Vorbis audio codec | `sudo apt install libvorbis-dev` |
| **libogg** | OGG bitstream framing | `sudo apt install libogg-dev` |
| **pthreads** | Thread-safe async DNS (Linux/macOS) | bundled with OS |

macOS (Homebrew): `brew install libvorbis libogg`

Windows: obtain the Vorbis SDK from https://xiph.org/downloads/ and add the include/lib paths to your build system.

---

## Building

### CMake (Linux / macOS / Windows)

```sh
mkdir build && cd build
cmake ..
cmake --build .
```

This produces `libnjclient.a` (Linux/macOS) or `njclient.lib` (Windows).

To install headers and the archive:

```sh
cmake --install . --prefix /usr/local
```

### Makefile (Linux / macOS)

```makefile
CXX     = g++
CC      = gcc
AR      = ar
CFLAGS  = -O2 -I. $(shell pkg-config --cflags ogg vorbis vorbisenc)
CXXFLAGS = $(CFLAGS) -std=c++11

SRCS = ninjam/njclient.cpp ninjam/netmsg.cpp ninjam/mpb.cpp \
       WDL/sha.cpp WDL/rng.cpp \
       WDL/jnetlib/connection.cpp WDL/jnetlib/asyncdns.cpp WDL/jnetlib/util.cpp

SRCS_C = WDL/win32_utf8.c

OBJS = $(SRCS:.cpp=.o) $(SRCS_C:.c=.o)

libnjclient.a: $(OBJS)
	$(AR) rcs $@ $^

%.o: %.cpp
	$(CXX) $(CXXFLAGS) -c $< -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) libnjclient.a
```

---

## Integrating into Your Project

### 1. Include the header

```cpp
#include "njclient.h"
```

### 2. Connect to a server

```cpp
NJClient client;

// Called once when user accepts the license agreement:
client.LicenseAgreementCallback = [](void *, const char *) { return 1; };

// Receive chat messages:
client.ChatMessage_Callback = [](void *, NJClient *, const char **parms, int n) {
    // parms[0] = "MSG", parms[1] = username, parms[2] = text
};

client.Connect("hostname.example.com:2049", "username", "password");
```

### 3. Drive the Run loop (separate thread or timer, ≤50 ms cadence)

```cpp
// In your network/UI thread:
while (!client.Run()); // Run() returns 0 when it wants to be called again immediately
```

### 4. Drive audio from your audio thread

```cpp
// Called from your audio callback (e.g. JACK, PortAudio, CoreAudio):
// inbuf[ch] and outbuf[ch] are non-interleaved float arrays of `len` samples.
client.AudioProc(inbuf, innch, outbuf, outnch, len, sampleRate);
```

`AudioProc` handles all encoding, decoding, mixing, and metering internally.
It is safe to call from a dedicated audio thread concurrently with `Run()`.

### 5. Set up a local channel for transmit

```cpp
// Register channel 0 as a mono input from hardware input channel 0:
client.SetLocalChannelInfo(
    /*ch*/       0,
    /*name*/     "my channel",
    /*setsrcch*/ true,  /*srcch*/ 0,       // input channel index
    /*setbr*/    true,  /*br*/   64,        // kbps
    /*setbcast*/ true,  /*bcast*/true       // broadcasting on
);
```

### 6. Query remote user state (for a custom UI)

```cpp
int nusers = client.GetNumUsers();
for (int u = 0; u < nusers; u++) {
    float vol, pan;
    bool mute;
    const char *name = client.GetUserState(u, &vol, &pan, &mute);

    for (int ci = 0; (ci = client.EnumUserChannels(u, ci)) >= 0; ) {
        bool sub;
        const char *chname = client.GetUserChannelState(u, ci, &sub);
    }
}
```

---

## Notes

### Thread Safety

- **`AudioProc()`** must be called from your **audio thread only**. No mutex is required by the caller.
- **`Run()`** must be called from a **single dedicated thread** (not the audio thread).
- All other `Get*`/`Set*` methods on remote channel state should be called with `client.m_remotechannel_rd_mutex` held if called outside the `Run()` thread.

### Working directory

`NJClient` writes temporary `.OGGv` segment files during a session. Call `SetWorkDir()` before `Connect()`:

```cpp
client.SetWorkDir("/tmp/ninjam-work");
```

Set `config_savelocalaudio = -1` to delete segment files as soon as they are decoded (saves disk space).

### Transmit support

Transmit (encoding and uploading local audio) is compiled in by default. To build a receive-only client, define `NJCLIENT_NO_XMIT_SUPPORT` in your compiler flags.

### Default port

NINJAM servers listen on TCP port **2049** by default. Pass it as part of the hostname string: `"hostname:2049"`.

### Sample rate

`NJClient` is sample-rate agnostic. Pass your actual hardware sample rate to `AudioProc()`. Common values: 44100, 48000.

### License

This code is licensed under the **GNU General Public License v2 or later** (see `LICENSE`). The WDL headers and JNetLib are provided under a permissive Zlib/WDL license. If you link this library into a proprietary project, the GPL terms apply to the linked result.
