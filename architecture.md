# Architecture

## Target hardware (submission)
- **Linux side — Qualcomm Dragonwing QRB2210:** camera capture + AI inference.
  A quantized (INT8) hand/object-proximity model, sourced and optimized via
  Qualcomm AI Hub, runs through the QNN runtime on the GPU/Hexagon DSP.
- **Real-time side — STM32U585 MCU:** deterministic cutoff logic. Receives a
  danger flag over the UNO Q's built-in Linux↔MCU bridge and trips a relay
  within single-digit milliseconds, independent of whatever the Linux side
  is doing. Includes a watchdog: if it stops hearing from the Linux side, it
  trips anyway (fail-safe).

## This prototype (Raspberry Pi, no Arduino/STM32 available)
Both roles run on one Pi, but as **separate threads with a narrow interface**
— `RealtimeCore.signal(danger: bool)` — specifically so the safety thread
(`src/safety/realtime_core.py`) is a drop-in replacement target for real
STM32 firmware later, without touching the vision or dashboard code.

```
 Camera / simulated frames
          │
          ▼
   Detector (OpenCV)   →  danger flag  →  RealtimeCore (own thread)
          │                                        │
          ▼                                        ▼
   MJPEG stream                              relay trip / GPIO
          │                                        │
          └──────────────► Flask dashboard ◄───────┘
                     (video feed + live status + event log)
```

## Why the split matters
A single Linux-side "if danger: cut power" check is at the mercy of Python's
GIL, OS scheduling, and whatever else is running. Isolating the cutoff
decision in its own tight loop — with a watchdog that fails safe if the
vision side stalls — is what makes this a *safety* system rather than a
monitoring dashboard with an alarm bolted on. That's true on the Pi
prototype, and it's the same reason the real submission needs the STM32
core rather than doing everything in Linux.

## Swapping in real hardware later
- Replace `CameraSource` with the QRB2210 camera pipeline.
- Replace `Detector._analyze_frame` with the quantized Qualcomm AI Hub model
  running via QNN.
- Replace `RealtimeCore` with STM32U585 firmware that exposes the same
  `signal(danger)` / watchdog contract over the Linux↔MCU bridge.
- Dashboard code (Flask app + templates) is hardware-agnostic and can be
  reused as-is, pointed at the real device's status endpoint.
