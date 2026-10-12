# adaptive-trigger-mod

DualSense adaptive trigger feedback for sim racing games that have no native support, starting with F1 2018. Written in Rust.

Research notes and build plan. Updated 2026-10-13.

## 1. Research findings

### F1 2018 has no adaptive trigger support anywhere

- **PS5 console:** F1 2018 is a PS4 title. In backwards-compatibility mode the DualSense only emulates DualShock 4 rumble. Adaptive triggers need the game itself to send trigger data, which only a few patched PS4 titles do (The Last of Us Part 2, God of War). F1 2018 never got such a patch.
- **PC:** F1 2018 sees the DualSense as a generic DirectInput pad. No trigger effects. Steam Input does not expose adaptive triggers at all.
- **Mods:** No mod exists for F1 2018, 2019 or 2020 on GitHub, Nexus, OverTake or the DSX game-mod list. The first F1 game with native support is F1 2021.

### Native support timeline (PC, from Steam and EA forum threads)

| Game | Adaptive triggers | Notes |
|------|------------------|-------|
| F1 2018 to F1 2020 | None | Generic pad only |
| F1 2021 | Yes, USB only | Reported "weak". Steam Input must be off |
| F1 22 | PS5: tied to tyre slip ratio. PC: added in a later patch | Missing at PC launch |
| F1 23 / F1 24 | Yes | Steam Input off. Haptics flaky |

### Closest existing tools

| Tool | Language | What it does | Gap |
|------|----------|--------------|-----|
| [DSX](https://store.steampowered.com/app/1812620/DSX/) | – | Paid Steam app. Owns the pad, static per-game trigger profiles, UDP listener on 6969 for mods | No telemetry by itself |
| [Paliverse/DualSenseX](https://github.com/Paliverse/DualSenseX) | C# | GitHub v1.4.9, abandoned. Has the UDP protocol example zip | Mod authors report UDP mods need the Steam build |
| [RacingSense](https://github.com/Micah-Zheng/RacingSense) | C# | Forza + DiRT UDP to triggers. Three output modes: direct HID, virtual Xbox 360 (ViGEm), DSX link | DiRT parser uses Codemasters "extradata" format, not the F1 2018 format |
| [dualsense-command](https://github.com/shiftedx/dualsense-command) | Rust | Forza UDP + Assetto Corsa Rally shared memory, direct controller writes, per-game profiles | No F1 |
| [dsz](https://github.com/MaximumFreak/dsz) | Rust | Triggers, Bluetooth haptics, virtual Xbox pad | No telemetry |
| [LumaBridge](https://github.com/EmilB04/LumaBridge) | C++ | Parses F1 2018 to F1 25 car telemetry packet | Drives RGB lighting only. Parser offsets reusable |
| [DualSenseXLink](https://www.overtake.gg/downloads/dualsensexlink-dualsense-adaptive-trigger-and-led-effects.48675/) | – | Assetto Corsa app to DSX | AC only, v0.1, reportedly broken with Steam DSX |
| [Forza-Horizon-DualSense-Python](https://github.com/HamzaYslmn/Forza-Horizon-DualSense-Python) | Python | Small UDP to DSX example | Forza only |

### Telemetry parsers to crib from

- [f1-telemetry-client](https://github.com/racehub-io/f1-telemetry-client) (TypeScript, F1 2018 to F1 23 packet structs)
- [pytelemetry](https://github.com/alessandrobessi/pytelemetry) (Python, F1 2018)
- [simetry](https://crates.io/crates/simetry) (Rust, iRacing / AC / ACC / rFactor 2 / DiRT Rally 2.0 readers)
- [F1 2018 UDP spec](https://forums.codemasters.com/discussion/136948/f1-2018-udp-specification) (official)

## 2. Architecture

Three layers. Same pattern every Forza / BeamNG trigger mod uses.

```
game telemetry  -->  CarState  -->  TriggerEffect  -->  controller
 (UDP / shm)        (normalised)     (mapper)          (DSX or HID)
```

### 2.1 Telemetry sources

One trait, one adapter per game family.

```rust
pub struct CarState {
    pub throttle: f32,        // 0..1
    pub brake: f32,           // 0..1
    pub rpm: f32,
    pub max_rpm: f32,
    pub speed_kmh: f32,
    pub gear: i8,
    pub wheel_slip: [f32; 4], // RL, RR, FL, FR (Codemasters order)
    pub abs_active: bool,
    pub rev_limiter: bool,
    pub on_track: bool,
}

pub trait TelemetrySource {
    fn poll(&mut self) -> Option<CarState>;
}
```

| Family | Transport | Notes |
|--------|-----------|-------|
| F1 2018 to F1 25 | UDP 20777 | Header `packetFormat` (2018, 2019, ...) selects the struct layout. Packet id 6 = car telemetry (throttle, brake, RPM, rev lights). Packet id 0 = motion (per-wheel slip floats). One adapter, per-year layouts |
| Codemasters extradata | UDP 20777 | DiRT Rally 1 / 2.0, DiRT 4, GRID, F1 2015 to 2017. Fixed float offsets: speed 28, slip 100..112, throttle 116, brake 120, rpm 148, max rpm 252 |
| Forza Horizon / Motorsport | UDP 5300 | Sled and Dash layouts, fixed offsets |
| BeamNG | UDP OutGauge | Fixed layout |
| Assetto Corsa, ACC, rFactor 2, iRacing | Windows shared memory | Use the `simetry` crate |

### 2.2 Effect mapper

Pure function `fn map(state: &CarState, profile: &Profile) -> (TriggerEffect, TriggerEffect)`. Easy to unit test.

Core rules:

- **L2 (brake):** resistance scales with brake pressure.
- **L2 lockup:** front slip over threshold, or ABS flag, switches to pulse.
- **R2 (throttle) wheelspin:** rear slip under throttle switches to vibrate.
- **R2 rev limiter:** pulse.
- Per-game tuning (thresholds, strengths) lives in a TOML profile.

### 2.3 Output backends

**DSX bridge (build first).** UDP JSON to `127.0.0.1:6969`. Protocol from the Paliverse example:

```json
{"instructions":[{"type":1,"parameters":[0, 1, 13, 0, 8]}]}
```

- `type`: 1 = TriggerUpdate, 2 = RGBUpdate, 3 = PlayerLED, 4 = TriggerThreshold
- `parameters`: `[controllerIndex, trigger (1 = Left, 2 = Right), mode, ...modeParams]`
- Modes: Normal 0, GameCube 1, VerySoft 2, Soft 3, Hard 4, VeryHard 5, Hardest 6, Rigid 7, VibrateTrigger 8, Choppy 9, Medium 10, VibrateTriggerPulse 11, CustomTriggerValue 12, Resistance 13, Bow 14, Galloping 15, SemiAutomaticGun 16, AutomaticGun 17, Machine 18
- Port can be overridden by DSX via `C:\Temp\DualSenseX\DualSenseX_PortNumber.txt`
- Requires the paid Steam DSX. DSX owns the pad, so game input, Xbox emulation and Steam Input conflicts are handled for you.
- Crates: `std::net::UdpSocket`, `serde`, `serde_json`. About 80 lines.

**Direct HID (later, removes the DSX dependency).**

- Crate `hidapi`. USB output report 0x02, Bluetooth output report 0x31 with CRC32 trailer.
- Trigger effect bytes: mode byte plus params at fixed offsets for R2 and L2.
- Crate `dualsense-rs` (v0.6) wraps reports. Verify it exposes trigger effects before relying on it.
- Cost: device ownership. Steam Input or the game grabbing the pad breaks writes. Standard fix is `vigem-client` (virtual Xbox 360 pad) plus HidHide to hide the real pad. Roughly doubles the work.

## 3. Requirements

- **Platform:** Windows primary. Shared-memory sims and DSX are Windows only. UDP games plus direct HID also work on Linux (udev rule for hidraw, Proton games).
- **Controller:** DualSense or DualSense Edge on USB for first tests. Bluetooth needs the CRC path and support varies per game.
- **Rust:** stable toolchain. Crates: `hidapi`, `serde`, `serde_json`, `toml`, `simetry`, optional `vigem-client`, `dualsense-rs`.
- **F1 2018 in-game settings:** Telemetry Settings, UDP Telemetry On, Broadcast Off, IP `127.0.0.1`, Port `20777`, Send Rate 60 Hz, UDP Format 2018. Disable Steam Input for the game.
- **Latency:** under 20 ms from packet to trigger write. Games send at 60 Hz, so a 60 Hz poll loop on a plain thread is enough. No async needed.
- **Testing without driving:** record one pcap of F1 packets, replay it in unit tests. The mapper is a pure function.

## 4. Build order

1. Workspace with three crates: `telemetry`, `effects`, `output`.
2. F1 2018 UDP adapter (format 2018 only) to mapper to DSX. Verify on a real pad.
3. Add F1 2019 to F1 25 layouts behind the `packetFormat` switch.
4. Codemasters extradata and Forza adapters (fixed offsets, cheap).
5. Shared-memory sims through `simetry`.
6. Direct HID backend only if the DSX dependency bothers users.

Before step 1: read the `dualsense-command` source. If its adapter trait is clean, the laziest path is adding an F1 UDP adapter there instead of building a new output layer.

## 5. Open questions

- Does `dualsense-rs` expose trigger effect bytes, or must they be hand-rolled from the reverse-engineered report spec?
- DSX Steam v3 may have added JSON fields since the 2021 example zip. Confirm against a live DSX install.
- Bluetooth trigger writes on the direct HID path are fiddly. Budget extra time.

## Sources

- [PCGamingWiki: Controller DualSense](https://www.pcgamingwiki.com/wiki/Controller:DualSense)
- [F1 2021 Steam thread](https://steamcommunity.com/app/1134570/discussions/0/5190945662879066208)
- [F1 22 Steam thread](https://steamcommunity.com/app/1692250/discussions/0/3409804177168890117/)
- [F1 23 Steam thread](https://steamcommunity.com/app/2108330/discussions/0/3812907397168286119/)
- [F1 24 Steam thread](https://steamcommunity.com/app/2488620/discussions/0/4331979056057832923/)
- [EA forum: DualSense features exclusive to PS5?](https://forums.ea.com/discussions/f1-games-franchise-discussion-en/re-are-the-ps5-dualsense-controller-features-100-exclusive-to-it/8432523)
- [GameSpot: F1 22 DualSense](https://gamespot.com/articles/ps5-dualsense-for-f1-22-should-add-to-the-realism-of-the-drive/1100-6504744/)
- [DSX game mods list](https://dualsensex.com/game-mods/)
- [BeamNG DSX mod](https://www.beamng.com/resources/dsx-dualsense-adaptive-triggers.28759/)
- [Racelogic: F1 telemetry settings](https://support-centre.racelogic.co.uk/motorsport/software/vbox-sim/ug/04-vbox-sim-game-requirements/)
- [VGC: PS4 games with DualSense features on PS5](https://www.videogameschronicle.com/news/last-of-us-part-2-and-god-of-war-on-ps5-support-new-dualsense-features)
- [Destructoid: DualSense haptics in BC mode](https://destructoid.com/?p=267240)
