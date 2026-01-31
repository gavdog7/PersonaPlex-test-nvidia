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
| **Python** | 3.13.6 | See Python compatibility note below |
| **WSL2** | Installed (Ubuntu distro available) | Currently stopped |
| **Docker** | 27.2.0 | Available |

### Disk Space Requirements

| Component | Size | Notes |
|-----------|------|-------|
| **PersonaPlex Model** | ~14 GB | 7B parameter model weights |
| **PyTorch + CUDA** | ~8 GB | cu130 wheels |
| **HuggingFace Cache** | ~5 GB | Tokenizers, configs |
| **Virtual Environment** | ~3 GB | Python packages |
| **Working Space** | ~5 GB | Logs, temp files, audio |
| **Total Minimum** | **~35 GB** | Recommend 50GB free |

Verify disk space before starting:
```bash
df -h ~
```

### Compatibility Analysis

#### GPU Compatibility (RTX 5090 - Blackwell)

The RTX 5090 uses NVIDIA's Blackwell architecture which has **experimental support** in PersonaPlex. Key considerations:

- **Special PyTorch Installation Required**: Must use cu130 index
  ```bash
  pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
  ```
- **CUDA 12.4+** required for Blackwell (system has CUDA 13.0)
- May require updates to rustymimi codec for full compatibility

#### Python Version Compatibility

**Warning**: Python 3.13 is relatively new and may have compatibility issues with some ML packages.

**Recommended approach**:
1. First attempt with system Python 3.13
2. If issues occur, install Python 3.10 or 3.11 via pyenv:
   ```bash
   # Install pyenv
   curl https://pyenv.run | bash

   # Install Python 3.10
   pyenv install 3.10.14
   pyenv local 3.10.14

   # Verify
   python --version  # Should show 3.10.14
   ```

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
   - **License**: NVIDIA Open Model License Agreement
   - **Additional**: CC-BY-4.0 for Moshi components
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
    libopusfile-dev \
    libportaudio2 \
    portaudio19-dev \
    libasound2-dev \
    libsndfile1-dev \
    build-essential \
    pkg-config \
    git \
    curl \
    wget \
    python3-pip \
    python3-venv \
    python3-dev \
    ffmpeg
```

### 4. CUDA Toolkit Installation (Required for rustymimi compilation)

**Important**: `nvidia-smi` works via driver passthrough, but the CUDA compiler (`nvcc`) requires explicit toolkit installation.

```bash
# Download and install CUDA keyring
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update

# Install CUDA toolkit (compiler and libraries)
sudo apt-get -y install cuda-toolkit-13-0

# Add to PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc

# Verify installation
nvcc --version
```

### 5. Rust Toolchain (for rustymimi codec)

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env

# Verify
rustc --version
cargo --version
```

### 6. Audio Configuration for WSL2

WSL2 audio passthrough requires WSLg (Windows Subsystem for Linux GUI). Verify and configure:

```bash
# Check if PulseAudio socket exists (provided by WSLg)
ls -la /mnt/wslg/PulseServer

# If missing, ensure WSLg is enabled in Windows
# Windows Settings > Apps > Optional Features > Windows Subsystem for Linux

# Test audio output
sudo apt install -y alsa-utils pulseaudio-utils
pactl info

# Test speakers (should produce sound)
speaker-test -t wav -c 2 -l 1

# List audio devices
pactl list short sinks
pactl list short sources
```

**Note on Audio Latency**: WSLg adds approximately 30-50ms of audio latency. For the lowest latency, consider native Windows deployment or a dedicated Linux machine.

---

## Installation Steps

### Phase 1: Environment Setup

#### Step 1.1: Start WSL2 Ubuntu

```powershell
wsl -d Ubuntu
```

#### Step 1.2: Verify NVIDIA GPU Access in WSL2

```bash
# Check GPU visibility
nvidia-smi

# Verify CUDA compiler
nvcc --version
```

Expected `nvidia-smi` output should show the RTX 5090 with CUDA 13.0.

#### Step 1.3: Check Disk Space

```bash
df -h ~ | awk 'NR==1 || /home/'
# Ensure at least 50GB free
```

#### Step 1.4: Create Python Virtual Environment

```bash
mkdir -p ~/personaplex
cd ~/personaplex

# Use system Python (try 3.13 first)
python3 -m venv venv
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip setuptools wheel
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
# Install with verbose output to catch compilation issues
pip install ./moshi -v
```

If rustymimi compilation fails, see Troubleshooting section.

#### Step 2.4: Install Additional Dependencies

```bash
# For CPU offloading support
pip install accelerate

# For .env file support
pip install python-dotenv
```

#### Step 2.5: Configure HuggingFace Token

**Option A**: Environment variable (current session only)
```bash
export HF_TOKEN=<YOUR_HUGGINGFACE_TOKEN>
```

**Option B**: Persistent .env file (recommended)
```bash
# Create .env file
cat > .env << 'EOF'
HF_TOKEN=<YOUR_HUGGINGFACE_TOKEN>
EOF

# Secure the file
chmod 600 .env
```

To load the .env file, either:
```bash
# Source manually before running
set -a && source .env && set +a

# Or use python-dotenv in Python scripts (automatic)
```

#### Step 2.6: Create Version Lock File

After successful installation, lock versions for reproducibility:

```bash
pip freeze > requirements.lock.txt
```

### Phase 3: Verification

#### Step 3.1: Verify PyTorch CUDA

```bash
python -c "
import torch
print(f'PyTorch version: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'CUDA version: {torch.version.cuda}')
print(f'Device: {torch.cuda.get_device_name(0)}')
print(f'Device count: {torch.cuda.device_count()}')
"
```

Expected output:
```
PyTorch version: 2.x.x+cu130
CUDA available: True
CUDA version: 13.0
Device: NVIDIA GeForce RTX 5090
Device count: 1
```

#### Step 3.2: Test Model Loading

```bash
python -c "from moshi import server; print('Moshi module loaded successfully')"
```

#### Step 3.3: Test Audio System

```bash
python -c "
import sounddevice as sd
print('Input devices:', sd.query_devices(kind='input'))
print('Output devices:', sd.query_devices(kind='output'))
"
```

---

## Network and Firewall Configuration

### WSL2 Network Access

WSL2 uses a virtual network adapter. For accessing the server from Windows:

```bash
# Get WSL2 IP address
hostname -I | awk '{print $1}'
```

**Windows localhost forwarding** (Windows 11 22H2+):
- `localhost:8998` from Windows should automatically forward to WSL2
- If not working, use the WSL2 IP directly

### Firewall Configuration

If the web UI is inaccessible:

**Windows Firewall** (PowerShell as Admin):
```powershell
# Allow inbound connections to port 8998
New-NetFirewallRule -DisplayName "PersonaPlex Server" -Direction Inbound -Protocol TCP -LocalPort 8998 -Action Allow
```

**WSL2 Firewall** (if ufw is enabled):
```bash
sudo ufw allow 8998/tcp
```

### SSL Certificate Handling

The server generates self-signed certificates. To avoid browser warnings:

**Option A**: Accept the warning (development only)
1. Navigate to `https://localhost:8998`
2. Click "Advanced" → "Proceed to localhost"

**Option B**: Persistent certificates (recommended for repeated use)
```bash
# Create persistent SSL directory
mkdir -p ~/.personaplex/ssl

# Use persistent directory
python -m moshi.server --ssl ~/.personaplex/ssl
```

---

## Testing Plan

### Test 1: Offline Batch Processing

Test the model in batch mode before attempting real-time inference.

#### Test 1a: Assistant Mode (Voice-only)

```bash
# Load environment
set -a && source .env && set +a

python -m moshi.offline \
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
python -m moshi.offline \
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
# Use persistent SSL directory
mkdir -p ~/.personaplex/ssl
python -m moshi.server --ssl ~/.personaplex/ssl
```

**Success Criteria:**
- [ ] Server starts without errors
- [ ] Prints URL (typically `https://localhost:8998`)
- [ ] Model loads successfully
- [ ] No CUDA out-of-memory errors

#### Test 2b: CPU Offload Mode (if memory issues occur)

```bash
python -m moshi.server --ssl ~/.personaplex/ssl --cpu-offload
```

**Success Criteria:**
- [ ] Server starts with reduced VRAM usage
- [ ] Inference completes (may be slower)

### Test 3: Web UI Interaction

Access the web interface and test real-time conversation.

#### Test 3a: Browser Access

1. Open browser to `https://localhost:8998`
2. Accept self-signed certificate warning
3. If localhost doesn't work, try WSL2 IP: `https://<WSL2_IP>:8998`

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

| Metric | Target | Stretch Goal | Notes |
|--------|--------|--------------|-------|
| **Audio Latency** | <500ms | <200ms | End-to-end response time; WSLg adds ~50ms |
| **VRAM Usage** | <28GB | <24GB | Leave headroom for stability |
| **First Response** | <5s | <3s | Initial model warmup |
| **Sustained Usage** | Stable | Stable | No memory leaks over 10+ turns |

**Note**: Actual latency depends on model complexity, GPU performance, and audio stack. The <200ms target is a stretch goal; <500ms is more realistic for initial deployment.

### Monitoring Commands

```bash
# Watch GPU usage during inference
watch -n 1 nvidia-smi

# Monitor memory usage continuously
nvidia-smi --query-gpu=memory.used,memory.total,utilization.gpu --format=csv -l 1

# Monitor system resources
htop
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
   python -m moshi.server --ssl ~/.personaplex/ssl --cpu-offload
   ```
2. Close other GPU applications (check with `nvidia-smi`)
3. Reduce batch size if configurable
4. Restart WSL2 to clear GPU memory: `wsl --shutdown` then restart

### Issue: rustymimi Compilation Fails

**Symptoms:**
- Error during `pip install ./moshi`
- Rust/CUDA compilation errors
- `nvcc not found`

**Solutions:**
1. Verify CUDA toolkit installed:
   ```bash
   nvcc --version
   # If not found, install CUDA toolkit (see Pre-Installation section)
   ```
2. Ensure Rust toolchain is current:
   ```bash
   rustup update
   ```
3. Verify environment variables:
   ```bash
   echo $PATH | tr ':' '\n' | grep cuda
   echo $LD_LIBRARY_PATH
   ```
4. Try installing with verbose output:
   ```bash
   pip install ./moshi -v 2>&1 | tee moshi_install.log
   ```

### Issue: PyTorch CUDA Not Available

**Symptoms:**
- `torch.cuda.is_available()` returns `False`

**Solutions:**
1. Verify cu130 PyTorch installed:
   ```bash
   pip show torch | grep Version
   python -c "import torch; print(torch.__version__)"
   # Should show version ending in +cu130
   ```
2. Check NVIDIA driver in WSL2:
   ```bash
   nvidia-smi
   ```
3. Reinstall with correct index:
   ```bash
   pip uninstall torch torchvision torchaudio -y
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
   ```

### Issue: Web UI Not Accessible

**Symptoms:**
- Browser cannot connect to server
- SSL certificate errors
- Connection refused

**Solutions:**
1. Use `https://` not `http://`
2. Accept self-signed certificate in browser
3. Check server is running and listening:
   ```bash
   ss -tlnp | grep 8998
   ```
4. Get WSL2 IP and try that instead of localhost:
   ```bash
   ip addr show eth0 | grep inet
   # Use https://<IP>:8998
   ```
5. Check Windows firewall (see Network Configuration section)
6. Verify WSL2 port forwarding:
   ```powershell
   # In PowerShell
   netsh interface portproxy show v4tov4
   ```

### Issue: Model Download Fails

**Symptoms:**
- HuggingFace authentication errors
- Model not found
- 401 Unauthorized

**Solutions:**
1. Verify HF_TOKEN is set:
   ```bash
   echo $HF_TOKEN
   # Should not be empty
   ```
2. Test token validity:
   ```bash
   curl -H "Authorization: Bearer $HF_TOKEN" https://huggingface.co/api/whoami
   ```
3. Accept model license at https://huggingface.co/nvidia/personaplex-7b-v1
4. Generate new token if expired

### Issue: Audio Not Working

**Symptoms:**
- No microphone input detected
- No audio output
- `sounddevice` errors

**Solutions:**
1. Check browser microphone permissions (must grant access)
2. Verify WSLg audio:
   ```bash
   # Check PulseAudio connection
   pactl info

   # Test audio output
   speaker-test -t wav -c 2 -l 1

   # List available devices
   pactl list short sinks
   pactl list short sources
   ```
3. Restart WSLg:
   ```powershell
   # In PowerShell
   wsl --shutdown
   # Then restart WSL
   ```
4. Ensure Windows audio service is running
5. Check default audio devices in Windows Sound settings

### Issue: Python 3.13 Compatibility Errors

**Symptoms:**
- Import errors
- Package installation failures
- `ModuleNotFoundError` for compiled extensions

**Solutions:**
1. Install Python 3.10 via pyenv:
   ```bash
   curl https://pyenv.run | bash
   # Follow shell configuration instructions

   pyenv install 3.10.14
   cd ~/personaplex
   pyenv local 3.10.14

   # Recreate venv
   rm -rf venv
   python -m venv venv
   source venv/bin/activate

   # Reinstall packages
   pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
   pip install ./moshi
   ```

---

## Cleanup and Rollback

### Clean Restart Procedure

If installation fails and you need to start fresh:

```bash
# Deactivate virtual environment
deactivate

# Remove installation
cd ~
rm -rf ~/personaplex

# Clear pip cache
pip cache purge

# Clear HuggingFace cache (WARNING: removes all cached models)
rm -rf ~/.cache/huggingface

# Start fresh from Phase 1
```

### Partial Cleanup

To reinstall just the moshi package:

```bash
cd ~/personaplex/personaplex
pip uninstall moshi -y
pip install ./moshi -v
```

To reinstall PyTorch:

```bash
pip uninstall torch torchvision torchaudio -y
pip cache remove torch*
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu130
```

### WSL2 Reset

If WSL2 is in a bad state:

```powershell
# Shutdown WSL completely
wsl --shutdown

# Wait a few seconds, then restart
wsl -d Ubuntu
```

---

## Alternative Deployment: Docker

If WSL2 presents issues, Docker provides a containerized alternative.

### Docker Setup

#### Step 1: Install NVIDIA Container Toolkit

In WSL2 Ubuntu (using modern keyring method):

```bash
# Add NVIDIA package repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Configure Docker to use NVIDIA runtime
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# Verify
docker run --rm --gpus all nvidia/cuda:13.0-base-ubuntu22.04 nvidia-smi
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
2. **Compilation**: rustymimi may need source modifications for Blackwell-specific optimizations
3. **Performance**: Optimizations may not be fully tuned for Blackwell architecture

### Windows/WSL2 Considerations

1. **Audio Passthrough**: WSLg adds ~30-50ms latency; may not achieve <200ms target
2. **Network Access**: Port forwarding from WSL2 to Windows may require manual configuration
3. **File System**: Performance impact when accessing Windows files from WSL2 (use Linux filesystem)

### Memory Constraints (32GB VRAM)

1. **Full Model**: Works but near memory limits
2. **Concurrent Usage**: Close other GPU applications before running
3. **CPU Offload**: May be needed for stability during extended sessions

---

## Success Criteria Summary

### Minimum Viable Success

- [ ] Model loads without out-of-memory errors
- [ ] Offline batch processing produces valid audio output
- [ ] Server starts and web UI is accessible

### Full Success

- [ ] Real-time conversation works with <500ms latency
- [ ] Full-duplex (interruption) works correctly
- [ ] Multiple voice personas function correctly
- [ ] Sustained 30-minute conversation without crashes
- [ ] CPU offload mode works if needed

---

## Milestones

### Milestone 1: Environment Ready
- [ ] WSL2 configured with GPU access
- [ ] CUDA toolkit installed (`nvcc` works)
- [ ] Python environment created
- [ ] System dependencies installed
- [ ] Audio passthrough verified

### Milestone 2: PersonaPlex Installed
- [ ] Repository cloned
- [ ] PyTorch with Blackwell support installed
- [ ] Moshi package compiled and installed
- [ ] HuggingFace authentication configured
- [ ] Version lock file created

### Milestone 3: Offline Testing Complete
- [ ] Assistant mode works
- [ ] Service mode works
- [ ] Audio output quality verified

### Milestone 4: Real-time Testing Complete
- [ ] Web server running
- [ ] Browser interaction working
- [ ] Latency acceptable (<500ms)

### Milestone 5: Full Validation
- [ ] All voice personas tested
- [ ] Edge cases documented
- [ ] Performance benchmarks recorded
- [ ] Cleanup procedure verified

---

## Resources and References

- **PersonaPlex GitHub**: https://github.com/NVIDIA/personaplex
- **PersonaPlex Model**: https://huggingface.co/nvidia/personaplex-7b-v1
- **NVIDIA Research Page**: https://research.nvidia.com/labs/adlr/personaplex/
- **PersonaPlex Paper**: https://research.nvidia.com/labs/adlr/files/personaplex/personaplex_preprint.pdf
- **Moshi Base Architecture**: https://github.com/kyutai-labs/moshi
- **PyTorch CUDA Installation**: https://pytorch.org/get-started/locally/
- **CUDA Toolkit for WSL**: https://developer.nvidia.com/cuda-downloads?target_os=Linux&target_arch=x86_64&Distribution=WSL-Ubuntu

---

## Appendix A: Quick Start Commands

```bash
# Start WSL2
wsl -d Ubuntu

# Navigate to project
cd ~/personaplex/personaplex

# Activate environment
source ~/personaplex/venv/bin/activate

# Load environment variables
set -a && source .env && set +a

# Start server with persistent SSL
mkdir -p ~/.personaplex/ssl
python -m moshi.server --ssl ~/.personaplex/ssl
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
Python:           3.13.6 (3.10/3.11 recommended if issues)
WSL2:             Ubuntu available
Docker:           27.2.0
```

---

## Appendix C: Environment Variables Reference

| Variable | Required | Description |
|----------|----------|-------------|
| `HF_TOKEN` | Yes | HuggingFace API token for model access |
| `NO_TORCH_COMPILE` | No | Set to `1` to disable torch.compile (useful for debugging) |
| `CUDA_VISIBLE_DEVICES` | No | Specify which GPU to use (default: 0) |
| `PYTORCH_CUDA_ALLOC_CONF` | No | CUDA memory allocator settings |

---

*Document generated: January 30, 2026*
*Last updated: January 30, 2026 - Added disk space requirements, CUDA toolkit installation, audio configuration, network/firewall setup, cleanup procedures, version pinning, and expanded troubleshooting.*
