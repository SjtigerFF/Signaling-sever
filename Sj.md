# Web Audio API Microphone Noise Suppression Integration

Integrates real-time Web Audio API signal processing directly inside the `startSharing` workflow to filter out ambient room hum, keyboard clatter, and low-frequency HVAC rumble from the microphone stream before sending it to WebRTC peers.

### User Review & Critical Decisions

> [!IMPORTANT]
> The noise suppression processing chain is configured specifically for speech clarity while preserving original system and tab audio quality.
> 
> - **Audio Chain Architecture (Default)**: A 3-stage modular DSP pipeline consisting of a Biquad High-Pass filter (85 Hz cutoff to strip electric hum and desk vibration), a vocal notch/presence peaking band, a DynamicsCompressorNode (smoothing dynamic volume peaks), and a software Noise Gate (ducking silent intervals to eliminate hiss).
> - **Source Isolation**: Applied specifically to speech microphone audio tracks, keeping shared screen/system audio untouched and full-fidelity.
> - **Studio Controls**: Noise suppression toggle button and sensitivity threshold slider placed in both the floating stream control dock and the Settings modal, with a live Gate Activity indicator badge.

---

### 1. Overview & Core Concept

- **What It Does**: Enhances the presenter's voice during live screen sharing by routing microphone audio through an active Web Audio processing graph before attaching the resulting clean audio track to the WebRTC `RTCPeerConnection`.
- **Target Audience / Persona**: Presenters, educators, and gamers who share their screens in environments with background keyboard clicks, computer fan noise, or room reverb.
- **Key Value**: Crisp, broadcast-grade speech without requiring third-party virtual cables or paid desktop software.

---

### 2. User Experience & Visual Design

#### Key User Flows
1. **Starting Screen Share with Mic**: When the user clicks **"স্ক্রিন শেয়ার শুরু করুন" (Start Screen Share)** with microphone enabled, the app prompts for mic capture (or activates it via the dock), constructs the Web Audio pipeline, and injects the filtered speech track alongside the screen video.
2. **Monitoring Gate Status**: A quiet, glowing "Noise Filter Active" chip in the stream status overlay pulses emerald when speech is detected and dims amber when the gate closes to cut background silence.
3. **Adjusting Filter Sensitivity**: Presenters can click the audio filter button or open Settings to adjust suppression sensitivity (Soft, Medium, Aggressive) or bypass the DSP filter on the fly.

#### Visual Identity & Theme
- **Aesthetic Direction**: Sleek, professional broadcast studio console in dark slate (`#0d1117` and `#161b22`) with emerald (`#28a745`) and neon cyan (`#00ffcc`) telemetry accents.
- **Color Palette & Mood**:
  - Neutral Canvas: `#0d1117` (Deep studio dark)
  - Structural Surfaces: `#161b22` and `#21262d` with hairline borders (`#30363d`)
  - Accent Color: `#00ffcc` for active audio metrics and `#28a745` for live speech pass-through
- **Typography & Hierarchy**: Primary UI set in `Hind Siliguri` for clean Bengali typography with system fallback, and `JetBrains Mono` for decibel thresholds and audio frequency readouts.
- **Component Styling & Layout**: Integrated floating dock buttons with single-line labels and clear active/bypass states.

#### Interactive Feedback & Motion
- Real-time decibel level and gate state updates rendered on the existing audio canvas with sub-100ms response time.
- Smooth transitions when toggling noise suppression without interrupting the live WebRTC peer connection.

---

### 3. Key Product Decisions & Trade-Offs

- **Decision 1: Native Web Audio Graph vs. Heavy WASM RNNoise Model**
  - *Chosen Approach*: Native Web Audio API using `BiquadFilterNode` (High-Pass + Peaking) + `DynamicsCompressorNode` + `GainNode` noise gate driven by an `AnalyserNode`.
  - *Why*: Instant zero-latency startup, zero external binary bundle size, and 100% browser compatibility without WebAssembly memory overhead.
  - *Alternatives Considered*: Bundling a 1.5MB WASM neural network (RNNoise). Rejected due to network weight and audio buffer latency in live meetings.

- **Decision 2: Audio Track Replacement Strategy in WebRTC**
  - *Chosen Approach*: Use `MediaStreamAudioDestinationNode` to produce a new clean `MediaStreamTrack`, and swap it into the peer connection via `RTCRtpSender.replaceTrack()`.
  - *Why*: Allows switching noise suppression on/off or adjusting parameters dynamically without renegotiating the WebRTC connection or freezing the screen stream.

---

### 4. Technical Architecture & Data Strategy

#### Architecture & Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Presenter Client Application                        │
│                                                                             │
│  navigator.mediaDevices.getUserMedia({ audio: true })                       │
│                         │                                                   │
│                         ▼                                                   │
│            ┌─────────────────────────────┐                                  │
│            │  AudioContext (Web Audio)   │                                  │
│            └──────────────┬──────────────┘                                  │
│                           │                                                 │
│       ┌───────────────────┼───────────────────┐                             │
│       ▼                   ▼                   ▼                             │
│ ┌───────────────┐   ┌───────────────┐   ┌─────────────────┐                 │
│ │ High-Pass     │──▶│ Dynamics      │──▶│ Noise Gate      │                 │
│ │ Filter (85Hz) │   │ Compressor    │   │ (Gain + Analyser│                 │
│ └───────────────┘   └───────────────┘   └────────┬────────┘                 │
│                                                  │                          │
│                                                  ▼                          │
│                                      ┌───────────────────────┐              │
│                                      │ MediaStreamAudio-     │              │
│                                      │ DestinationNode       │              │
│                                      └───────────┬───────────┘              │
│                                                  │ Clean Mic Track          │
│                                                  ▼                          │
│  navigator.mediaDevices.getDisplayMedia() ──▶ PeerConnection                │
│     (Screen Video + System Audio Track)       (RTCRtpSender)                │
│                                                  │                          │
└──────────────────────────────────────────────────┼──────────────────────────┘
                                                   │ WebRTC SRTP
                                                   ▼
                                         Remote Viewer Video
```

#### Data Model & State
- `isNoiseSuppressionEnabled: boolean` (Default: `true`)
- `noiseGateThreshold: number` (Gate opening threshold in dB, default: `-48 dB`)
- `audioPipelineRef: MutableRefObject<AudioPipeline | null>`:
  - `audioContext: AudioContext`
  - `sourceNode: MediaStreamAudioSourceNode`
  - `highPassFilter: BiquadFilterNode`
  - `compressor: DynamicsCompressorNode`
  - `gateGainNode: GainNode`
  - `analyserNode: AnalyserNode`
  - `destinationNode: MediaStreamAudioDestinationNode`
  - `cleanStreamTrack: MediaStreamTrack`

#### Interactive Component & State Mapping
- `startSharing()`: Obtains display media, invokes `setupAudioNoiseSuppression()` with the presenter's mic track, and bundles both the video and processed audio tracks into `peerConnection.addTrack()`.
- Control Dock & Settings: Provides a quick toggle for "নয়েজ ফিল্টার" (Noise Suppression) and a sensitivity slider (`-60dB` to `-30dB`) with live feedback.
