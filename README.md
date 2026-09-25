# Reflex Guard

An AI vision safety interlock for unguarded power tools and machinery —
built for the Snapdragon® AI Lab Build & Present Challenge, targeting the
Arduino UNO Q (Qualcomm Dragonwing QRB2210 + STM32U585).

A camera watches the blade/cutting zone. An on-device model detects a hand
entering the danger zone. A real-time safety layer cuts power within
single-digit milliseconds — independent of the AI/Linux side, and fail-safe
if that side ever stalls.

This repo is a **working prototype on a Raspberry Pi** (no Arduino/STM32
required to run it) that mirrors the same architecture the real submission
uses on Snapdragon hardware. See [`docs/architecture.md`](docs/architecture.md)
for how the two map to each other.

## What's in here
- `src/vision/detector.py` — OpenCV-based hand/object-proximity detector
  (stands in for the quantized Qualcomm AI Hub model)
- `src/safety/realtime_core.py` — isolated safety-cutoff thread with a
  fail-safe watchdog (stands in for the STM32 real-time core)
- `src/dashboard/` — Flask dashboard: live feed, current state, event log,
  reset control
- `src/config.py` — all tunables (danger zone, thresholds, camera vs.
  simulate mode)

## Quick start

```bash
git clone <your-repo-url>
cd reflex-guard
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m src.dashboard.app
```

Open `http://<device-ip>:5000` in a browser.

### No camera yet?
`src/config.py` defaults to `MODE = "simulate"`, which generates a synthetic
"hand" sweeping in and out of the danger zone — so the whole pipeline
(detection → safety core → dashboard) runs and demos end-to-end with zero
hardware. Once you have a USB webcam or Pi camera:

```bash
REFLEX_MODE=camera python -m src.dashboard.app
```

### Optional: real relay control on a Pi
Set `USE_GPIO = True` in `src/config.py` and wire a relay module to
`RELAY_PIN` (BCM numbering). Falls back to logging-only if `RPi.GPIO` isn't
installed, so the rest of the code runs unchanged on a laptop too.

## How the pieces talk to each other
```
camera/simulated frames → Detector → danger flag → RealtimeCore (own thread, watchdog)
                                                        │
                                                        ▼
                                          relay trip (or logged event)
                                                        │
                              Flask dashboard ◄─────────┘
                         (live feed, state, latency, event log)
```

Full write-up in [`docs/architecture.md`](docs/architecture.md).

## Roadmap
- [ ] Swap the OpenCV detector for a Qualcomm AI Hub quantized hand-detection
      model running via QNN, benchmarked on the actual QRB2210
- [ ] Port `RealtimeCore`'s contract to real STM32U585 firmware
- [ ] Add audio (blade-sound signature) as a second detection modality
- [ ] Fleet dashboard aggregating near-miss events across a workshop

## License
MIT
