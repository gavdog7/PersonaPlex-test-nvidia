# PersonaPlex Installation Progress

## Status: In Progress - Paused at Python Setup

**Last Updated:** January 30, 2026

---

## Completed Steps

### 1. Pre-Flight Verification ✅
- [x] GitHub repository accessible (NVIDIA/personaplex)
- [x] HuggingFace model page accessible (HTTP 200)
- [x] Disk space verified: 328 GB free (exceeds 50GB minimum)
- [x] WSL2 Ubuntu distro available

### 2. WSL2 Environment ✅
- [x] GPU visible in WSL2 (RTX 5090, 32GB VRAM)
- [x] System dependencies installed via apt
- [x] CUDA Toolkit 12.8 installed and working
  - `nvcc --version` shows: release 12.8, V12.8.93
- [x] Rust toolchain installed (rustc 1.93.0)

### 3. Python Environment 🔄 (In Progress)
- [x] pyenv installed
- [ ] **BLOCKED**: zshrc got corrupted with Windows paths
- [ ] Python 3.10.14 needs to be installed via pyenv

---

## Next Steps (Resume Here)

### Step 1: Fix zshrc and Install Python 3.10

Run this single command in WSL Ubuntu terminal:

```bash
cp ~/.zshrc ~/.zshrc.backup && echo -e 'export ZSH="$HOME/.oh-my-zsh"\nZSH_THEME="robbyrussell"\nplugins=(git)\n[ -f $ZSH/oh-my-zsh.sh ] && source $ZSH/oh-my-zsh.sh\nexport PATH=/usr/local/cuda-12.8/bin:$PATH\nexport LD_LIBRARY_PATH=/usr/local/cuda-12.8/lib64:$LD_LIBRARY_PATH\n[ -f "$HOME/.cargo/env" ] && source "$HOME/.cargo/env"\nexport PYENV_ROOT="$HOME/.pyenv"\n[[ -d $PYENV_ROOT/bin ]] && export PATH="$PYENV_ROOT/bin:$PATH"\neval "$(pyenv init -)"' > ~/.zshrc && source ~/.zshrc && pyenv install 3.10.14 && pyenv global 3.10.14 && python --version
```

Expected output: `Python 3.10.14`

### Step 2: Clone Repository and Create Virtual Environment

```bash
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/NVIDIA/personaplex.git
cd personaplex
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

### Step 3: Install PyTorch with CUDA 12.8 Support

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```

### Step 4: Install Moshi Package

```bash
pip install ./moshi -v
pip install accelerate python-dotenv
```

### Step 5: Configure HuggingFace Token

1. Get token from https://huggingface.co/settings/tokens
2. Accept model license at https://huggingface.co/nvidia/personaplex-7b-v1
3. Create .env file:
```bash
echo 'HF_TOKEN=your_token_here' > .env
chmod 600 .env
```

### Step 6: Verify Installation

```bash
python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'Device: {torch.cuda.get_device_name(0)}')"
```

### Step 7: Run Tests

See `docs/plan.md` for full testing plan including:
- Offline batch processing tests
- Web UI server tests
- Voice conversation tests

---

## Environment Summary

| Component | Status | Version/Notes |
|-----------|--------|---------------|
| GPU | ✅ | RTX 5090, 32GB VRAM |
| CUDA Toolkit | ✅ | 12.8 |
| Rust | ✅ | 1.93.0 |
| pyenv | ✅ | Installed |
| Python 3.10 | ❌ | Needs installation |
| PersonaPlex | ❌ | Not cloned yet |
| PyTorch | ❌ | Not installed yet |
| Moshi | ❌ | Not installed yet |

---

## Troubleshooting Notes

- The zshrc file got corrupted during setup due to path escaping issues between Windows and WSL
- Use the one-liner command above to fix it
- If pyenv commands fail, ensure `~/.pyenv/bin` is in PATH
