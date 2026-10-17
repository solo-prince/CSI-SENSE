# CSI SENSE: Wi-Fi Presence & Breathing Monitor

Detects people and breathing using only Wi-Fi signals. No camera, no wearable.

![Demo](docs/media/preview.gif)

▶ [Watch the full demo video](docs/media/demo.mp4)

![Hardware setup](docs/media/setup.jpg)

## How it works

An ESP32 transmits about 100 packets per second. An ESP32-C3 measures the Channel State Information (CSI) of each packet. A body in the path disturbs the signal: walking causes large changes, breathing causes slow, small ones.

## System architecture

```mermaid
flowchart LR
    TX["ESP32<br/>Transmitter"] -- "Wi-Fi packets 100/s" --> RX["ESP32-C3<br/>Receiver (CSI)"]
    RX -- "JSON over USB serial 115200" --> PC["Browser<br/>Web Serial API"]
    PC --> UI["Live dashboard<br/>status, chart, events"]
    PC --> CSV["CSV export"]
```

## Signal processing pipeline

```mermaid
flowchart TD
    A[Receive CSI packet] --> B[Compute amplitude per subcarrier]
    B --> C[Motion score = signal variation]
    C --> D{Calibrated?}
    D -- No --> E[Learn empty-room baseline ~30 s]
    E --> A
    D -- Yes --> F{Motion above adaptive threshold?}
    F -- Yes --> G[State: MOVING]
    F -- No --> H{Recently moving?}
    H -- Yes --> I[State: HOLD, up to 10 s]
    H -- No --> J[Goertzel breathing detector]
    J --> K{Confidence above 6?}
    K -- Yes --> L[State: STILL + breaths per minute]
    K -- No --> M[State: EMPTY]
```

## State machine

```mermaid
stateDiagram-v2
    [*] --> Calibrating
    Calibrating --> Empty: baseline learned
    Empty --> Moving: motion detected
    Moving --> Hold: motion stops
    Hold --> Moving: motion again
    Hold --> Still: breathing detected
    Hold --> Empty: timeout
    Still --> Moving: motion detected
    Still --> Empty: no breathing
```

## Data format

The receiver prints one JSON line per second:

| Field | Meaning |
|---|---|
| `node` | Receiver ID |
| `t` | Uptime (ms) |
| `pps` | Packets per second (healthy is about 100) |
| `rssi` | Signal strength (dBm) |
| `motion` / `thr` | Disturbance score / learned threshold |
| `present` | 1 if someone is detected |
| `state` | calibrating, empty, moving, hold, still |
| `bpm` / `q` | Breaths per minute / rhythm confidence |

## Hardware

- ESP32 (transmitter)
- ESP32-C3 (receiver)
- 2 USB cables
- 1 laptop with Chrome or Edge

## Quick start

```bash
git clone https://github.com/solo-prince/CSI-SENSE.git
cd CSI-SENSE
python3 -m http.server 8000
```

Open http://localhost:8000/web/ in Chrome, then click **Connect sensor** (or **Demo mode**).

## Usage

1. Flash the transmitter and receiver firmware (ESP-IDF).
2. Place the boards 2-3 m apart.
3. Stay out of the sensing area for 30 s while it calibrates.
4. Walk between the boards, then sit still for about 20 s to see the breathing rate.

## Limitations

One link only. Accuracy drops with several people or strong Wi-Fi interference. Position tracking needs 3 or more receivers. Not a medical device.

## Author

Pravin, B.Sc Electronics and Artificial Intelligence
