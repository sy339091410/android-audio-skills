# 🎵 Android Audio Skills

> Teach AI to understand the Android audio subsystem like an audio architect — from App to HAL, from Java to DSP.

A curated knowledge base of Android audio subsystem skills, purpose-built for AI coding assistants. Each skill is verified against AOSP source code, providing complete call chains, architecture diagrams, and debugging commands to help you quickly diagnose audio initialization failures, glitches, and routing anomalies.

---

## 📦 Skills

### 🏗️ [android-audioflinger-architecture](./android-audioflinger-architecture/SKILL.md)

**AudioFlinger Architecture** — The "heart" of the audio system

- Four-layer audio stack: App → Framework → Native → HAL
- Four PlaybackThread types: MixerThread / DirectOutputThread / OffloadThread / DuplicatingThread
- Dual mixing strategies: conventional Mixer vs FastMixer (SCHED_FIFO + Lock-free)
- Complete data flow: playback path & recording path & Ring Buffer mechanism
- Latency tiers: ~5ms ultra-low → >100ms power-saving
- Debug quick reference: dumpsys / logcat / Perfetto one-liners

### 🔗 [android-audio-call-chain](./android-audio-call-chain/SKILL.md)

**AudioRecord Call Chain** — Every step from `new AudioRecord()` to HAL

- 10+ layer complete call sequence: Java → JNI → Native → AudioFlinger → AudioPolicy → HAL
- Error code -22 (BAD_VALUE) generation and propagation path
- Engine device selection logic: Bluetooth → Wired → USB → Built-in Mic priority
- QCOM HAL loading mechanism: `dlopen` → `adev_open` → `open_input_stream`
- Power state impact: how WAIT_FOR_VHAL causes recording initialization failures

### 🎧 [android-audiotrack-playback](./android-audiotrack-playback/SKILL.md)

**AudioTrack Playback Flow Deep-dive** — From `new AudioTrack()` to the speaker cone

- Five-stage pipeline: create → shared memory allocation → write → start → mix & output
- Shared memory truth: `TrackBase` concatenates control block + PCM ring buffer into one block (who allocates in MODE_STREAM vs MODE_STATIC)
- ClientProxy / ServerProxy dual proxies + futex sync, plus the exact underrun detection predicate
- Real cause of start latency: the `FS_FILLING → FS_FILLED` startThreshold mute gate
- Copy ledger: 7 memory blocks, 6 transfers — what "zero-copy" really covers
- Android 15 changes: system-invalidated direct/offload tracks, targetSdk 35 focus restriction, bit-perfect in main path
- Debug quick reference: dumpsys field-by-field decoding, 4 common issue triage flows, Perfetto period jitter

### 🚗 [android-car-audio-focus](./android-car-audio-focus/SKILL.md)

**Android 15 Car Audio Focus Management** — The "traffic controller" for multi-source audio

- Four focus gain types + four loss notification types with car-specific response strategies
- MediaFocusControl arbitration: FocusStack LIFO stack management
- Delayed focus mechanism: media requests queued during calls, auto-restored on hangup
- Android 15 enhancements: system-level auto-duck (-12dBFS), targetSdk 35+ background restrictions
- Car-specific: multi-zone audio, CarAudioService coordination, safety alert top priority
- Debug quick reference: dumpsys / logcat for focus conflict diagnosis

### 🧭 [android-car-audio-policy](./android-car-audio-policy/SKILL.md)

**Android 15 Car AudioPolicyService Strategy Management** — The "decision brain" of audio routing

- Strategy-execution separation: APS (decision) ↔ AudioFlinger (execution)
- Three-tier configuration: topology XML + policy rules XML + car-specific config
- Complete routing decision chain: AudioAttributes → Strategy → DevicePriority → FinalDevice
- Device management: connect/disconnect notifications, failover mechanism, BT A2DP special handling
- ForceUse override routing + four audio mode transitions
- Android 15 features: AIDL HAL migration, volume group routing binding, CAP Engine
- Car-specific: multi-zone policy routing, CarAudioService coordination, car_audio_configuration.xml

---

## ✨ Why This Matters

| Feature | Description |
|---------|-------------|
| 📖 **Source-level Precision** | Every reference includes file paths and line numbers, verifiable directly |
| 🔍 **Problem-driven** | Built from real debugging scenarios, not textbook-style enumeration |
| 🧩 **AI-native** | Purpose-built structured knowledge for AI coding assistants, trigger-on-demand |
| 🎯 **Battle-tested** | Each skill includes debugging commands, common issue diagnosis, and troubleshooting flows |

---

## 🚧 Roadmap

- [x] AudioTrack playback flow deep-dive
- [x] AudioPolicyService routing strategy analysis
- [ ] AAudio MMAP low-latency path
- [ ] Bluetooth audio (A2DP / LE Audio) call chain
- [x] AudioFocus management mechanism
- [ ] Car audio scenarios (CarAudioService) deep-dive

> 💡 PRs welcome! Suggest topics via Issues.

---

## 📂 Usage

Place these skill directories under your project's `.trae/skills/`:

```
your-project/
└── .trae/
    └── skills/
        ├── android-audio-call-chain/
        │   └── SKILL.md
        ├── android-audiotrack-playback/
        │   └── SKILL.md
        ├── android-audioflinger-architecture/
        │   └── SKILL.md
        ├── android-car-audio-focus/
        │   └── SKILL.md
        └── android-car-audio-policy/
            └── SKILL.md
```

The AI assistant will automatically invoke the corresponding skill when analyzing audio-related issues.

---

## 📚 Source Code Foundation

Based on **Android 15 (API Level 35)** AOSP source analysis, key paths:

```
# === AudioFlinger Architecture (Native Execution Layer) ===
frameworks/av/services/audioflinger/     # AudioFlinger service (mixing, threads, tracks)
frameworks/av/media/libaudioclient/      # Native client library (AudioTrack/AudioRecord)
frameworks/av/media/audioserver/         # audioserver entry point

# === AudioPolicyService Strategy (Native Decision Layer) ===
frameworks/av/services/audiopolicy/      # Audio policy service (APM, Engine, routing)
frameworks/av/services/audiopolicy/enginedefault/src/  # Default Engine implementation

# === Audio Focus & System Services (Java Layer) ===
frameworks/base/services/core/java/com/android/server/audio/  # AudioService, MediaFocusControl
frameworks/base/media/java/android/media/                     # AudioManager client API

# === Car Audio Services ===
packages/services/Car/service/src/com/android/car/audio/      # CarAudioService (multi-zone, car focus)
packages/services/Car/service/src/com/android/car/power/      # CarPowerManagementService (power state)

# === HAL Interfaces & Implementations ===
hardware/interfaces/audio/               # Audio HAL interface definitions (AIDL)
frameworks/av/media/libaudiohal/         # HAL client wrappers (StreamHalAidl / 3 FMQs)
hardware/qcom/audio/hal/                 # QCOM HAL implementation

# === Driver & Kernel (tail of the playback chain) ===
external/tinyalsa/                       # tinyalsa (pcm_write → ioctl WRITEI)
sound/core/pcm_lib.c                     # ALSA core (copy_from_user → dma_area)

# === Policy Configuration Files ===
/vendor/etc/audio_policy_configuration.xml        # Audio topology config (mixPort/devicePort/route)
/system/etc/audio_policy_engine_configuration.xml # Engine policy rules (ProductStrategy)
/vendor/etc/car_audio_configuration.xml           # Car audio config (zones/volume groups/CAP Engine)
```

---

<p align="center">
  <b>Made with ❤️ for Android audio developers</b>
</p>