---
title: "Real-Time ADS-B Aircraft Tracker with RTL-SDR"
date: 2026-10-01
draft: false
description: "Built a live aircraft surveillance system from a $30 software-defined radio — receiving 1090 MHz ADS-B transponder signals over Las Vegas and plotting callsign, altitude, speed, and position on a live map."
image: /images/projects/sdr-adsb.jpg
tags: ["RF", "Software Defined Radio", "ADS-B", "Python", "Signal Processing"]
---

## Overview

Every commercial and most private aircraft continuously broadcast their identity, altitude, speed, and GPS position over **ADS-B** (Automatic Dependent Surveillance–Broadcast) on **1090 MHz**. I built a receive chain from scratch that picks up those transmissions with a software-defined radio, decodes them, and displays live aircraft on an interactive map — all from a desk in Las Vegas.

**Hardware:** RTL-SDR Blog V4 (Rafael Micro R828D tuner, RTL2832U) + dipole antenna  
**Frequency:** 1090 MHz (Mode S / ADS-B)  
**Key result:** 1,175 ADS-B messages captured in 2 minutes; live tracking of airliners and private jets over the Las Vegas valley, including traffic near Nellis Air Force Base

---

## System Architecture

<svg viewBox="0 0 760 150" xmlns="http://www.w3.org/2000/svg" style="width:100%;max-width:760px;height:auto;font-family:sans-serif" role="img" aria-label="Signal chain: antenna to RTL-SDR dongle to USB to rtl_adsb and dump1090 decoders to Python map and live web map">
  <defs>
    <marker id="arr" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path d="M0,0 L10,5 L0,10 z" fill="#40dc8c"/></marker>
  </defs>
  <g fill="none" stroke="#40dc8c" stroke-width="2">
    <rect x="5" y="45" width="110" height="60" rx="8"/>
    <rect x="155" y="45" width="120" height="60" rx="8"/>
    <rect x="315" y="15" width="130" height="50" rx="8"/>
    <rect x="315" y="85" width="130" height="50" rx="8"/>
    <rect x="485" y="15" width="130" height="50" rx="8"/>
    <rect x="485" y="85" width="130" height="50" rx="8"/>
    <rect x="655" y="50" width="100" height="50" rx="8"/>
    <line x1="115" y1="75" x2="153" y2="75" marker-end="url(#arr)"/>
    <line x1="275" y1="65" x2="313" y2="42" marker-end="url(#arr)"/>
    <line x1="275" y1="85" x2="313" y2="108" marker-end="url(#arr)"/>
    <line x1="445" y1="40" x2="483" y2="40" marker-end="url(#arr)"/>
    <line x1="445" y1="110" x2="483" y2="110" marker-end="url(#arr)"/>
    <line x1="615" y1="40" x2="653" y2="68" marker-end="url(#arr)"/>
    <line x1="615" y1="110" x2="653" y2="84" marker-end="url(#arr)"/>
  </g>
  <g fill="currentColor" font-size="13" text-anchor="middle">
    <text x="60" y="72">Dipole</text><text x="60" y="89">antenna</text>
    <text x="215" y="72">RTL-SDR V4</text><text x="215" y="89">USB · WinUSB</text>
    <text x="380" y="37">rtl_adsb -V</text><text x="380" y="53" font-size="11">raw hex frames</text>
    <text x="380" y="107">dump1090</text><text x="380" y="123" font-size="11">Mode S decoder</text>
    <text x="550" y="37">planes.txt</text><text x="550" y="53" font-size="11">pandas + folium</text>
    <text x="550" y="107">--net :8080</text><text x="550" y="123" font-size="11">live web map</text>
    <text x="705" y="72">Aircraft</text><text x="705" y="89">on map</text>
  </g>
</svg>

The dongle samples the RF spectrum and hands raw I/Q data over USB. Two paths run off it: a **capture path** (`rtl_adsb` → text log → Python) for offline analysis, and a **live path** (`dump1090` → built-in web server) for real-time tracking.

---

## Phase 1: Hardware & Driver Bring-Up

The V4 uses a newer tuner (R828D) than older RTL-SDR dongles, so stock drivers don't work out of the box. Getting it recognized took three steps:

| Step | Tool | What it did |
|------|------|-------------|
| 1 | **Zadig v2.9** | Replaced the Windows default driver with **WinUSB** on Bulk-In Interface 0 |
| 2 | **RTL-SDR Blog V1.4.0 release** | Swapped `rtlsdr.dll` in SDR# for the V4-compatible build |
| 3 | **`rtl_test`** | Verified the hardware from the command line |

```text
> rtl_test
...
RTL-SDR Blog V4 Detected
Found Rafael Micro R828D tuner
```

**Lesson learned:** most "it doesn't work" problems with SDR hardware are driver problems, not hardware problems. Isolating the driver layer with a command-line test before opening any GUI saved a lot of guesswork.

---

## Phase 2: Live RF Spectrum

Before going after aircraft, I validated the whole receive chain on an easy target — **FM broadcast radio (99–107 MHz)** — in SDR#. The waterfall showed every Las Vegas station as a bright vertical band, confirming the antenna, tuner, and drivers were all working.

This "known-good signal first" step is the RF equivalent of a hello-world: if you can't see strong FM stations, you won't see 1090 MHz bursts either.

---

## Phase 3: Capturing ADS-B Messages

ADS-B frames are short (56 or 112 bits), pulse-position-modulated bursts at 1 Mbit/s. `rtl_adsb` demodulates them and prints each frame as hex:

```bash
rtl_adsb -V > planes.txt
```

In **2 minutes** this captured **1,175 ADS-B messages** from Las Vegas airspace. Each line looks like:

```text
*8DAD36A858...;
 └┬┘└──┬──┘
  │    └── ICAO 24-bit aircraft address (AD36A8 = SWA4282)
  └─────── Downlink Format 17 (ADS-B extended squitter)
```

### Decoding the frame header

| Bits | Field | Example | Meaning |
|------|-------|---------|---------|
| 1–5 | DF (Downlink Format) | `10001` = 17 | ADS-B extended squitter |
| 6–8 | CA (Capability) | `101` | Transponder level |
| 9–32 | ICAO address | `AD36A8` | Unique airframe ID |
| 33–88 | ME (message) | varies | Position, velocity, or ID |
| 89–112 | Parity | CRC-24 | Error detection |

---

## Phase 4: Python Processing & Mapping

I wrote `map.py` to parse the captured log and build an interactive map with **pandas** and **folium** (Leaflet under the hood). The core idea: pull the 24-bit ICAO address out of each DF17 frame with a regex, aggregate messages per aircraft, and render the result on a map centered on Las Vegas.

```python
import re
import pandas as pd
import folium

LAS_VEGAS = (36.1699, -115.1398)

# Each DF17 frame starts with "*8D" followed by the 6-hex-digit ICAO address
pattern = re.compile(r"\*8D([0-9A-F]{6})", re.IGNORECASE)

with open("planes.txt") as f:
    icaos = [m.group(1).upper() for line in f if (m := pattern.search(line))]

df = pd.Series(icaos).value_counts().rename_axis("icao").reset_index(name="messages")
print(f"{len(icaos)} frames from {len(df)} unique aircraft")

m = folium.Map(location=LAS_VEGAS, zoom_start=9, tiles="CartoDB dark_matter")
folium.Marker(LAS_VEGAS, tooltip="Receiver").add_to(m)
m.save("flights.html")
```

The output is a standalone `flights.html` that opens in any browser.

---

## Phase 5: Live Tracking with dump1090

For real-time tracking, `dump1090` does the full Mode S decode — including **CPR-encoded positions**, velocity, and callsigns — and serves its own web map:

```bash
dump1090 --interactive --net
# terminal table + live map at http://localhost:8080
```

### Aircraft tracked live (October 1, 2026)

| Callsign | ICAO | Altitude | Speed | Aircraft |
|----------|------|----------|-------|----------|
| **SWA4282** | `AD36A8` | 38,275 ft | 498 kt | Southwest Airlines, cruising over Las Vegas |
| **N40EP** | `A4AABA` | 15,725 ft | 188.8 kt | Cessna Citation II, operating near Nellis AFB |
| **VIV113** | — | 3,425 ft | 173 kt | Low-altitude traffic |

The live map showed each aircraft's position, heading, altitude, speed, and GPS coordinates, with Nellis Air Force Base visible on the map near the private-jet traffic.

---

## Software Stack

| Tool | Version | Purpose |
|------|---------|---------|
| SDR# (SDRSharp) | 1.0.0.1921 | Live spectrum & waterfall |
| RTL-SDR Blog drivers | V1.4.0 | V4-compatible hardware support |
| Zadig | 2.9.788 | WinUSB driver install |
| rtl_adsb | — | Raw ADS-B frame capture |
| dump1090 | Windows build | Full Mode S decode + web map |
| Python | 3.14 | Data processing |
| pandas / folium | — | Parsing & map visualization |
| .NET Runtime | 9.0 | Required by SDR# |

---

## Skills Demonstrated

- ✅ **RF fundamentals:** spectrum analysis, tuning, antenna setup, receive-chain validation
- ✅ **Digital communications:** Mode S frame structure, pulse-position modulation, CRC-24 parity
- ✅ **Driver & USB debugging:** WinUSB installation, DLL replacement, CLI hardware verification
- ✅ **Python data pipeline:** regex parsing, pandas aggregation, geospatial mapping with folium
- ✅ **Real-time systems:** network-streamed decoder output and live visualization
- ✅ **Systematic bring-up:** validating each layer (driver → known signal → target signal) before moving on

---

## What's Next

- **DSP from first principles:** use PySDR to capture raw I/Q samples and write my own PPM demodulator in Python instead of relying on `rtl_adsb`
- **Position decoding:** implement CPR (Compact Position Reporting) decoding myself to plot true aircraft tracks from the captured log
- **Better antenna:** build a tuned 1090 MHz collinear antenna and measure the range improvement
- **Coverage logging:** record message rate vs. distance to characterize receiver performance
