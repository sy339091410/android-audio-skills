# Android Audio Skills

A collection of AI-assisted skills for analyzing and debugging the Android audio subsystem, covering AudioFlinger architecture, AudioRecord call chains, and audio HAL interactions.

## Skills

### [android-audioflinger-architecture](./android-audioflinger-architecture/SKILL.md)

Documents the Android AudioFlinger architecture: four-layer audio stack, PlaybackThread types (MixerThread, DirectOutputThread, OffloadThread, DuplicatingThread), Mixer/FastMixer mechanism, audio data flow, latency tiers, and debugging quick reference.

### [android-audio-call-chain](./android-audio-call-chain/SKILL.md)

Documents the complete Android AudioRecord call chain from Java to HAL, including error code -22 (BAD_VALUE) path, Engine device selection logic, QCOM HAL loading mechanism, and power state effects (WAIT_FOR_VHAL).

## Source

Skills are based on Android 15 (API Level 35) AOSP source code analysis.

## Usage

These skills are designed to be used with AI coding assistants that support skill-based workflows. Place them in the `.trae/skills/` directory of your project.
