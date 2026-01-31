# PersonaPlex Latency Analysis Report

## Executive Summary

This document analyzes the feasibility of achieving sub-200ms latency for PersonaPlex deployment, comparing local and cloud compute options.

**Key Finding**: Sub-200ms latency is achievable only with local deployment on native Linux. Cloud compute adds unavoidable network latency that makes this target impractical.

---

## Current Plan Assessment

### Strengths

1. **Realistic latency expectations** - Correctly identifies <200ms is unachievable with WSLg
2. **Thorough pre-flight checks** - Verifies external resources before installation
3. **Proper Python version guidance** - Avoids Python 3.13 compatibility issues
4. **Thermal monitoring** - Important for sustained RTX 5090 operation
5. **Security considerations** - Documents lack of authentication on default server

### Gaps Identified

1. **No complete latency breakdown** - Only mentions "WSLg adds 30-50ms" without full component analysis
2. **No mention of model quantization** - INT8/FP8 could reduce VRAM usage and improve inference speed
3. **Missing WebRTC audio buffering impact** - Browser audio buffering adds latency beyond WSLg
4. **No comparison to cloud alternatives**

---

## Latency Budget Breakdown

For speech-to-speech AI, the latency stack consists of multiple components:

| Component | Local (WSL2) | Local (Native Linux) | Cloud (Optimized) |
|-----------|--------------|---------------------|-------------------|
| Audio capture + encode | 20-40ms | 10-20ms | 20-40ms |
| Network RTT | 0ms | 0ms | **15-50ms** |
| Speech encoding (Mimi) | 10-20ms | 10-20ms | 5-10ms |
| LLM inference (7B) | 80-150ms | 60-120ms | 30-60ms |
| Speech decoding | 10-20ms | 10-20ms | 5-10ms |
| Network return | 0ms | 0ms | **15-50ms** |
| Audio decode + playback | 30-50ms (WSLg) | 10-15ms | 20-30ms |
| **Total** | **150-280ms** | **100-195ms** | **110-250ms** |

### Component Analysis

#### Audio Capture and Encoding
- WebRTC/Opus encoding adds 20-40ms in browser environments
- Native applications with direct audio access can reduce this to 10-20ms
- Buffering is required for stable audio quality

#### Network Latency (Cloud Only)
- Best-case same-region RTT: 15-30ms
- Typical cross-region RTT: 40-100ms
- This is additive and unavoidable with cloud deployment

#### Model Inference
- 7B parameter model requires significant compute
- RTX 5090 local: 80-150ms (unoptimized)
- H100 cloud with TensorRT: 30-60ms
- Quantization can reduce by 30-50%

#### Audio Playback
- WSLg overhead: 30-50ms
- Native Linux ALSA/PulseAudio: 10-15ms
- Browser playback: 20-30ms

---

## Cloud Compute Evaluation

### Can Cloud Achieve Sub-200ms?

**Short answer: Extremely difficult, and not practical for most users.**

#### Requirements for Sub-200ms Cloud Deployment

1. **Edge location with <10ms network latency**
   - User must be physically close to datacenter
   - Requires edge deployment, not traditional cloud regions

2. **H100/H200 GPUs with TensorRT-LLM**
   - Inference must complete in <50ms
   - Requires INT8 quantization and speculative decoding

3. **Optimized audio pipeline**
   - Custom WebRTC with minimal buffering
   - Direct audio access on client (no browser)

4. **Low-latency streaming protocol**
   - WebTransport or custom UDP implementation
   - WebSocket adds overhead

### Cloud Provider Comparison

| Provider | GPU Options | Typical Network Latency | Realistic Total Latency | Notes |
|----------|-------------|------------------------|------------------------|-------|
| **NVIDIA DGX Cloud** | H100/H200 | Varies by region | 150-250ms | Best inference speed, network limited |
| **AWS (EC2 P5)** | H100 | 20-40ms | 180-280ms | Good availability, higher cost |
| **Lambda Labs** | A100/H100 | 15-30ms | 140-220ms | Good value, limited regions |
| **RunPod** | A100/H100 | 20-50ms | 160-260ms | Cost-effective, inconsistent network |
| **Google Cloud** | A100/H100 | 20-40ms | 170-260ms | Good infrastructure, complex pricing |
| **Azure** | A100/H100 | 20-40ms | 170-260ms | Enterprise features, higher cost |

### The Fundamental Network Problem

Network latency is additive and governed by physics. Even with perfect optimization:

```
Minimum overhead = Audio capture (20ms) + Audio playback (20ms) = 40ms fixed
Available for network + inference = 200ms - 40ms = 160ms

With 40ms RTT: Inference budget = 120ms (achievable)
With 60ms RTT: Inference budget = 100ms (tight)
With 80ms RTT: Inference budget = 80ms (very difficult)
```

For sub-200ms with cloud, you need:
- RTT < 40ms **AND** inference < 80ms

This is only achievable with edge deployment or same-city colocated GPU servers.

---

## Recommendations

### Option A: Native Linux (Best for Sub-200ms)

Eliminate WSLg overhead entirely with direct Linux deployment.

**Implementation:**
- Dual-boot Ubuntu or use dedicated Linux machine
- Direct ALSA/PulseAudio access (10-15ms audio overhead)
- Same RTX 5090 hardware

**Expected latency: 100-180ms**

**Pros:**
- Lowest achievable latency
- No additional cost
- Full control over audio pipeline

**Cons:**
- Requires system reconfiguration
- Less convenient than WSL2

### Option B: Quantized Model on Local Hardware

Reduce inference time through model optimization.

**Implementation:**
- INT8 quantization (if PersonaPlex supports it)
- TensorRT-LLM optimization
- Flash Attention 3 (native to Blackwell architecture)

**Expected improvement: 30-50% faster inference**

**Pros:**
- Works with existing setup
- Significant speed improvement

**Cons:**
- May require custom model conversion
- Potential slight quality degradation

### Option C: Optimized WSL2 (Current Plan, Enhanced)

Improve the current WSL2 setup with targeted optimizations.

**Implementation:**
- Use native Windows audio when possible
- Optimize WebRTC settings
- Reduce buffering where safe

**Expected latency: 150-250ms**

**Pros:**
- Minimal changes to current plan
- Maintains Windows workflow

**Cons:**
- Cannot achieve sub-200ms reliably
- WSLg overhead is fundamental

### Option D: Hybrid Edge-Cloud (Future Consideration)

Use local GPU for audio processing, cloud for heavy inference.

**Implementation:**
- Local: Audio capture, encoding, playback
- Cloud: LLM inference only
- Requires custom pipeline development

**Expected latency: Variable (depends on implementation)**

**Pros:**
- Could leverage powerful cloud GPUs
- Offloads compute from local machine

**Cons:**
- Complex implementation
- Still subject to network latency
- Not practical without significant development

---

## Latency Target Feasibility Summary

| Target | WSL2 Setup | Native Linux | Cloud Compute |
|--------|------------|--------------|---------------|
| <500ms | Yes | Yes | Yes |
| <300ms | Unlikely | Yes | Possible |
| <200ms | No | Possible | No (network physics) |
| <150ms | No | Difficult | No |
| <100ms | No | No | No |

---

## Conclusion

For PersonaPlex deployment with sub-200ms latency goals:

1. **Local deployment is superior to cloud** for latency-critical applications
2. **Native Linux eliminates 30-50ms WSLg overhead** - the single largest optimization available
3. **Cloud compute is not viable for sub-200ms** due to unavoidable network latency
4. **The RTX 5090 is capable hardware** - the bottleneck is software/OS overhead, not GPU performance

### Recommended Path

| Priority | Action | Expected Improvement |
|----------|--------|---------------------|
| 1 | Switch to native Linux | -30-50ms |
| 2 | Apply TensorRT optimization | -20-40ms |
| 3 | Optimize audio pipeline | -10-20ms |
| 4 | Consider INT8 quantization | -15-30ms |

With all optimizations applied on native Linux, sub-200ms latency is achievable on the current RTX 5090 hardware.

---

## References

- [PersonaPlex GitHub Repository](https://github.com/NVIDIA/personaplex)
- [NVIDIA TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)
- [WebRTC Audio Processing](https://webrtc.org/)
- [PulseAudio Latency Control](https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/Developer/Clients/LatencyControl/)

---

*Document created: January 30, 2026*
*Related: [PersonaPlex Deployment Plan](./plan.md)*
