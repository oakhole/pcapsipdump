# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

`pcapsipdump` is a C++ network utility for capturing and dumping SIP sessions alongside their associated RTP/RTCP/T.38 media streams to disk in standard libpcap format, saving one file per SIP call session (even with thousands of concurrent calls). It can capture live interface traffic or split existing bulk `.pcap` files.

## Build and Run Commands

### Compilation

Dependencies: `libpcap` (`libpcap-dev` / `libpcap-devel`), standard C++ runtime (`-lstdc++`), GNU make.

- Build binary: `make`
- Build with regular expression support for phone number filtering (`-n`): `make DEFS=-DUSE_REGEXP`
- Build debug binary (with `-ggdb`): `make pcapsipdump-debug`
- Clean build artifacts: `make clean`
- Install: `make install DESTDIR=<target_dir>`

### Execution & Testing

- Live capture: `./pcapsipdump -i <interface> -d <output_dir> [options]`
- Process offline pcap: `./pcapsipdump -r <input.pcap> -d <output_dir> [options]`

Key CLI Flags:
- `-f`: Do not fork / daemonize (keeps process attached to terminal)
- `-p`: Do not put interface into promiscuous mode
- `-P <port>` / `--port <port>`: SIP port to listen on (default: `5060`)
- `-R <filter>`: RTP filtering mode (`rtp+rtcp` [default], `rtp`, `rtpevent`, `t38`, `none`)
- `-U`: Packet-buffered pcap writing (slower, but partial files remain consistent)
- `-n <number>`: Record only calls to/from specified number (supports regex if built with `-DUSE_REGEXP`)
- `-t`: Record only calls containing T.38 payload in SDP
- `-v <level>`: Set verbosity level

## Architecture & Code Structure

- `pcapsipdump.cpp` / `pcapsipdump.h`:
  - Main entry point, command-line argument parsing, daemonization logic, and signal handling.
  - Libpcap packet loop processing UDP packets.
  - SIP message parsing: extracts Call-ID, From/To headers (caller/callee), CSeq, methods (INVITE, BYE, CANCEL), and SDP payloads (media IP/ports).
  - Matches RTP/RTCP streams to active calls via IP, port, and SSRC.
  - Generates hierarchically organized pcap dumps (`YYYY/MM/DD/HH/YYYYMMDD-HHMMSS-[caller]-[callee]-[call_id].pcap`) via `pcap_dump`.
- `calltable.cpp` / `calltable.h`:
  - `calltable` class managing in-memory call state (up to `calltable_max = 10240` active calls).
  - Tracks Call-ID, participant IP addresses/ports (`calltable_max_ip_per_call = 4`), SSRCs, timestamps, and pcap file descriptors.
  - `do_cleanup(currtime)`: Expires and closes pcap files for calls that received BYE or timed out (inactivity > 300s).
- `makefile-helpers/check_libpcap.c`:
  - Verification helper used by `Makefile` to detect `libpcap` presence.
- `debian/` & `redhat/`:
  - Packaging scripts, service init scripts, and RPM spec definitions.
