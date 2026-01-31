# PersonaPlex Local Deployment Plan

## Overview

This document outlines the plan to deploy and test NVIDIA PersonaPlex, a real-time full-duplex speech-to-speech conversational AI system, on the local development system.

**PersonaPlex** enables natural, low-latency spoken interactions with consistent persona control. It allows selection of diverse voices and role definition through text prompts.

---

## System Assessment

### Current Hardware

| Component | Specification | Status |
|-----------|---------------|--------|
| **GPU** | NVIDIA GeForce RTX 5090 | Blackwell Architecture (Experimental Support) |
| **VRAM** | 32 GB | Exceeds 24GB minimum, below 80GB recommended |
| **CPU** | AMD Ryzen (Family 26, ~4700 MHz) | Adequate |
| **RAM** | 63 GB | Excellent for CPU offloading if needed |
| **CUDA Version** | 13.0 | Supports Blackwell GPUs |
| **Driver** | 581.57 | Current |

### Current Software

| Component | Version | Status |
|-----------|---------|--------|
| **OS** | Windows 11 Pro (Build 26200) | Experimental support; WSL2 recommended |
| **Python** | 3.13.6 | Exceeds 3.10+ requirement |
| **WSL2** | Installed (Ubuntu distro available) | Currently stopped |
| **Docker** | 27.2.0 | Available |

### Compatibility Analysis

#### GPU Compatibility (RTX 5090 - Blackwell)

The RTX 5090 uses NVIDIA's Blackwell architecture which has **experimental support** in PersonaPlex. Key considerations:

- **Special PyTorch Installation Required**: Must use cu130 index
  ```bash
  pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
  ```
- **CUDA 12.4+** required for Blackwell (system has CUDA 13.0)
- May require updates to rustymimi codec for full compatibility

#### Memory Considerations

- **32GB VRAM** is above the 24GB minimum but below 80GB recommended
- CPU offloading available via `--cpu-offload` flag if memory issues occur
- 63GB system RAM provides excellent headroom for CPU offloading

#### Operating System

Windows 11 is listed as "experimental" for PersonaPlex. Three deployment paths are available:

1. **WSL2 + Ubuntu** (Recommended) - Best Linux compatibility
2. **Docker with NVIDIA Container Runtime** - Containerized, reproducible
3. **Native Windows** - Most experimental, potential issues

---

## Deployment Strategy

### Recommended Approach: WSL2 + Ubuntu

Given the experimental status of both Windows support and Blackwell GPUs, using WSL2 provides the most stable Linux environment while leveraging Windows GPU passthrough.

### Alternative Approach: Docker

Docker provides containerization and reproducibility but requires NVIDIA Container Toolkit configuration in WSL2.

---

## Pre-Installation Requirements

### 1. HuggingFace Account Setup

PersonaPlex model access requires HuggingFace authentication:

1. Create account at https://huggingface.co
2. Accept model license at https://huggingface.co/nvidia/personaplex-7b-v1
3. Generate access token at https://huggingface.co/settings/tokens
4. Store token securely for later use

### 2. WSL2 Configuration

Verify and configure WSL2 for GPU passthrough:

```powershell
# Check WSL status
wsl --list --verbose

# Start Ubuntu if stopped
wsl -d Ubuntu

# Update WSL to latest version
wsl --update
```

### 3. System Dependencies

Within WSL2 Ubuntu, install required system packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
    libopus-dev \
    build-essential \
    pkg-config \
    git \
    curl \
    python3-pip \
    python3-venv
```

### 4. Rust Toolchain (for rustymimi codec)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

---

## Installation Steps

### Phase 1: Environment Setup

#### Step 1.1: Start WSL2 Ubuntu

```powershell
wsl -d Ubuntu
```

#### Step 1.2: Verify NVIDIA GPU Access in WSL2

```bash
nvidia-smi
```

Expected output should show the RTX 5090 with CUDA 13.0.

#### Step 1.3: Create Python Virtual Environment

```bash
mkdir -p ~/personaplex
cd ~/personaplex
python3 -m venv venv
source venv/bin/activate
```

### Phase 2: PersonaPlex Installation

#### Step 2.1: Clone PersonaPlex Repository

```bash
git clone https://github.com/NVIDIA/personaplex.git
cd personaplex
```

#### Step 2.2: Install PyTorch with Blackwell Support

**Critical**: Use cu130 index for RTX 5090 compatibility:

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
```

#### Step 2.3: Install Moshi Package

```bash
pip install ./moshi
```

#### Step 2.4: Set HuggingFace Token

```bash
export HF_TOKEN=<YOUR_HUGGINGFACE_TOKEN>
```

Or create `.env` file:
```bash
echo "HF_TOKEN=<YOUR_HUGGINGFACE_TOKEN>" > .env
```

### Phase 3: Verification

#### Step 3.1: Verify PyTorch CUDA

```python
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'Device: {torch.cuda.get_device_name(0)}')"
```

Expected output:
```
CUDA available: True
Device: NVIDIA GeForce RTX 5090
```

#### Step 3.2: Test Model Loading

```bash
python -c "from moshi import server; print('Moshi module loaded successfully')"
```

---

## Testing Plan

### Test 1: Offline Batch Processing

Test the model in batch mode before attempting real-time inference.

#### Test 1a: Assistant Mode (Voice-only)

```bash
HF_TOKEN=$HF_TOKEN python -m moshi.offline \
    --voice-prompt "NATF2.pt" \
    --input-wav "assets/test/input_assistant.wav" \
    --seed 42424242 \
    --output-wav "output_assistant.wav" \
    --output-text "output_assistant.json"
```

**Success Criteria:**
- [ ] Command completes without errors
- [ ] `output_assistant.wav` is generated
- [ ] Audio file is playable and contains speech
- [ ] `output_assistant.json` contains transcript

#### Test 1b: Service Mode (Voice + Text Prompt)

```bash
HF_TOKEN=$HF_TOKEN python -m moshi.offline \
    --voice-prompt "NATM1.pt" \
    --text-prompt "$(cat assets/test/prompt_service.txt)" \
    --input-wav "assets/test/input_service.wav" \
    --seed 42424242 \
    --output-wav "output_service.wav" \
    --output-text "output_service.json"
```

**Success Criteria:**
- [ ] Command completes without errors
- [ ] Output reflects the persona defined in text prompt
- [ ] Voice matches selected voice prompt

### Test 2: Web UI Server

Start the web UI server for interactive testing.

#### Test 2a: Standard Launch

```bash
SSL_DIR=$(mktemp -d)
python -m moshi.server --ssl "$SSL_DIR"
```

**Success Criteria:**
- [ ] Server starts without errors
- [ ] Prints URL (typically `https://localhost:8998`)
- [ ] Model loads successfully
- [ ] No CUDA out-of-memory errors

#### Test 2b: CPU Offload Mode (if memory issues occur)

```bash
pip install accelerate
SSL_DIR=$(mktemp -d)
python -m moshi.server --ssl "$SSL_DIR" --cpu-offload
```

**Success Criteria:**
- [ ] Server starts with reduced VRAM usage
- [ ] Inference completes (may be slower)

### Test 3: Web UI Interaction

Access the web interface and test real-time conversation.

#### Test 3a: Browser Access

1. Open browser to `https://localhost:8998`
2. Accept self-signed certificate warning

**Success Criteria:**
- [ ] Web UI loads
- [ ] Microphone permission prompt appears
- [ ] Audio input/output controls visible

#### Test 3b: Voice Conversation

1. Grant microphone permission
2. Select voice persona
3. Start conversation
4. Speak and wait for response

**Success Criteria:**
- [ ] Audio input detected
- [ ] Response generated
- [ ] Response plays through speakers
- [ ] Latency feels conversational (<500ms)
- [ ] Full-duplex works (can interrupt)

### Test 4: Different Personas

Test various voice and role combinations.

#### Available Voice Prompts:

**Natural Voices:**
| Voice ID | Type |
|----------|------|
| NATF0-NATF3 | Female |
| NATM0-NATM3 | Male |

**Varied Voices:**
| Voice ID | Type |
|----------|------|
| VARF0-VARF4 | Female |
| VARM0-VARM4 | Male |

**Success Criteria:**
- [ ] Each voice produces distinct output
- [ ] Voice characteristics are consistent
- [ ] Role prompts modify behavior appropriately

---

## Performance Benchmarks

### Expected Performance Metrics

| Metric | Target | Notes |
|--------|--------|-------|
| **Audio Latency** | <200ms | End-to-end response time |
| **VRAM Usage** | <28GB | Leave headroom for stability |
| **First Response** | <3s | Initial model warmup |
| **Sustained Usage** | Stable | No memory leaks over 10+ turns |

### Monitoring Commands

```bash
# Watch GPU usage during inference
watch -n 1 nvidia-smi

# Monitor memory usage
nvidia-smi --query-gpu=memory.used,memory.total --format=csv -l 1
```

---

## Troubleshooting Guide

### Issue: CUDA Out of Memory

**Symptoms:**
- `RuntimeError: CUDA out of memory`
- Server crashes during model loading

**Solutions:**
1. Use CPU offloading:
   ```bash
   python -m moshi.server --ssl "$SSL_DIR" --cpu-offload
   ```
2. Close other GPU applications
3. Reduce batch size if configurable

### Issue: rustymimi Compilation Fails

**Symptoms:**
- Error during `pip install ./moshi`
- Rust/CUDA compilation errors

**Solutions:**
1. Ensure Rust toolchain installed:
   ```bash
   rustup update
   ```
2. Verify CUDA is in PATH:
   ```bash
   nvcc --version
   ```
3. Install CUDA toolkit in WSL2 if missing

### Issue: PyTorch CUDA Not Available

**Symptoms:**
- `torch.cuda.is_available()` returns `False`

**Solutions:**
1. Verify cu130 PyTorch installed:
   ```bash
   pip show torch
   ```
2. Check NVIDIA driver in WSL2:
   ```bash
   nvidia-smi
   ```
3. Reinstall with correct index:
   ```bash
   pip uninstall torch torchvision torchaudio
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
   ```

### Issue: Web UI Not Accessible

**Symptoms:**
- Browser cannot connect to server
- SSL certificate errors

**Solutions:**
1. Use `https://` not `http://`
2. Accept self-signed certificate in browser
3. Check firewall settings
4. Try accessing from Windows browser using WSL2 IP

### Issue: Model Download Fails

**Symptoms:**
- HuggingFace authentication errors
- Model not found

**Solutions:**
1. Verify HF_TOKEN is set:
   ```bash
   echo $HF_TOKEN
   ```
2. Accept model license at https://huggingface.co/nvidia/personaplex-7b-v1
3. Generate new token if expired

### Issue: Audio Not Working

**Symptoms:**
- No microphone input detected
- No audio output

**Solutions:**
1. Check browser microphone permissions
2. Verify PulseAudio/ALSA in WSL2:
   ```bash
   sudo apt install pulseaudio
   ```
3. For Windows audio passthrough, may need WSLg

---

## Alternative Deployment: Docker

If WSL2 presents issues, Docker provides a containerized alternative.

### Docker Setup

#### Step 1: Install NVIDIA Container Toolkit

In WSL2 Ubuntu:

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo systemctl restart docker
```

#### Step 2: Build PersonaPlex Image

```bash
cd ~/personaplex/personaplex
docker build -t personaplex:latest .
```

#### Step 3: Run Container

```bash
docker run --gpus all -p 8998:8998 \
    -e HF_TOKEN=$HF_TOKEN \
    -e NO_TORCH_COMPILE=1 \
    -v ~/.cache:/root/.cache \
    personaplex:latest
```

### Docker Compose

```bash
docker-compose up
```

---

## Known Limitations

### RTX 5090 (Blackwell) Experimental Status

1. **PyTorch Support**: May encounter edge cases with cu130
2. **Compilation**: rustymimi may need source modifications
3. **Performance**: Optimizations may not be tuned for Blackwell

### Windows/WSL2 Considerations

1. **Audio Passthrough**: May require WSLg for seamless audio
2. **Network Access**: Port forwarding from WSL2 to Windows
3. **File System**: Performance impact when accessing Windows files

### Memory Constraints (32GB VRAM)

1. **Full Model**: May work but near memory limits
2. **Concurrent Usage**: Close other GPU applications
3. **CPU Offload**: May be needed for stability

---

## Success Criteria Summary

### Minimum Viable Success

- [ ] Model loads without out-of-memory errors
- [ ] Offline batch processing produces valid audio output
- [ ] Server starts and web UI is accessible

### Full Success

- [ ] Real-time conversation works with <300ms latency
- [ ] Full-duplex (interruption) works correctly
- [ ] Multiple voice personas function correctly
- [ ] Sustained 30-minute conversation without crashes
- [ ] CPU offload mode works if needed

---

## Timeline and Milestones

### Milestone 1: Environment Ready
- WSL2 configured with GPU access
- Python environment created
- Dependencies installed

### Milestone 2: PersonaPlex Installed
- Repository cloned
- PyTorch with Blackwell support installed
- Moshi package installed
- HuggingFace authentication configured

### Milestone 3: Offline Testing Complete
- Assistant mode works
- Service mode works
- Audio output quality verified

### Milestone 4: Real-time Testing Complete
- Web server running
- Browser interaction working
- Latency acceptable

### Milestone 5: Full Validation
- All voice personas tested
- Edge cases documented
- Performance benchmarks recorded

---

## Resources and References

- **PersonaPlex GitHub**: https://github.com/NVIDIA/personaplex
- **PersonaPlex Model**: https://huggingface.co/nvidia/personaplex-7b-v1
- **NVIDIA Research Page**: https://research.nvidia.com/labs/adlr/personaplex/
- **Moshi Base Architecture**: https://github.com/kyutai-labs/moshi
- **PyTorch CUDA Installation**: https://pytorch.org/get-started/locally/

---

## Appendix A: Quick Start Commands

```bash
# Start WSL2
wsl -d Ubuntu

# Navigate to project
cd ~/personaplex/personaplex

# Activate environment
source ~/personaplex/venv/bin/activate

# Set token
export HF_TOKEN=<your_token>

# Start server
SSL_DIR=$(mktemp -d)
python -m moshi.server --ssl "$SSL_DIR"
```

---

## Appendix B: System Specifications Reference

```
Host Name:        GAVDOG-PC
GPU:              NVIDIA GeForce RTX 5090
VRAM:             32 GB
CUDA Version:     13.0
Driver:           581.57
CPU:              AMD Ryzen (~4700 MHz)
RAM:              63 GB
OS:               Windows 11 Pro (Build 26200)
Python:           3.13.6
WSL2:             Ubuntu available
Docker:           27.2.0
```

---

*Document generated: January 30, 2026*
