# Fedora Setup Guide (Generic)

author: Sean Pomeroy
last modified: 2026-09-20

## Table of Contents

- [Fedora Setup Guide (Generic)](#fedora-setup-guide-generic)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Notes and Tips](#notes-and-tips)
  - [1. Fedora! Install, Setup, and Updates](#1-fedora-install-setup-and-updates)
  - [2. GPU Drivers and tools, AMD/NVidia](#2-gpu-drivers-and-tools-amdnvidia)
    - [Install Vulkan](#install-vulkan)
    - [GPU Monitoring](#gpu-monitoring)
    - [AMD GPU Stack (ROCm + Gaming + AI) - Untested](#amd-gpu-stack-rocm--gaming--ai---untested)
    - [NVIDIA GPU Stack (CUDA + Gaming + AI)](#nvidia-gpu-stack-cuda--gaming--ai)
  - [3. Network and Connection Setup](#3-network-and-connection-setup)
    - [Fedora Cockpit setup - WIP](#fedora-cockpit-setup---wip)
    - [Tailscale](#tailscale)
    - [Remote Desktop Connection](#remote-desktop-connection)
    - [Mount shares](#mount-shares)
  - [4. Apps/etc Setup](#4-appsetc-setup)
    - [VS CODE](#vs-code)
    - [Docker Rootless Setup ( a few confirmations)](#docker-rootless-setup--a-few-confirmations)
  - [5. LLM Tools](#5-llm-tools)
    - [LM Studio - My Main tool](#lm-studio---my-main-tool)
    - [Vllm](#vllm)
    - [AMD Lemonade](#amd-lemonade)
    - [AMD llama.cpp Deployment - Untested](#amd-llamacpp-deployment---untested)
    - [NVidia llama.cpp Deployment - Untested](#nvidia-llamacpp-deployment---untested)
    - [OpenHands Open source AI focused development tool](#openhands-open-source-ai-focused-development-tool)
    - [Pi and other tools](#pi-and-other-tools)
  - [Other information](#other-information)

## Introduction

This guide is a generic version of my notes for deploying Fedora 44 and various LLM tools and other apps. It is set up for both Nvidia or AMD GPUs, not Intel or Apple devices.

You should consider your hardware and and goals and take notes alongside this document for your custom setup. You can customize a lot of load options like llama.cpp as it is based on llama.cpp. llama.cpp has more options and customizations if you want to dive into the technical side.

I mostly use LM Studio and it's Link tool + my own taislcale account to run and connect to a the server at home. Like other tools, it provides an OpenAI compatible REST API endpoint. I then have VS Code (and a few other coding/etc tools) connecting directly to the REST API or using an extension to handle requests. You can use Copilot Chat but it ca be finicky and may not always work as expected, and it will can default to subscription credits.

This is not a BECU supported guide. Use at your own risk (of learning Fedora and how to set up and use local LLM tools!). I built this in my own time on my own hardware.

## Notes and Tips

- Commands are bash unless otherwise specified.
- Some apps are already installed or even set up. Only add what is required.
- Review each step manually on deployment to confirm setup process and requirements, ex FW notes.
- This was developed for both Nvidia and amd GPUs, but most testing is on an AMD Strxi Haloe 395+ 128gb system
- Most of this can be translated directly to other Fedora like distros. Other distros may reuqire their own driver installation and setup steps.
- This guide assumes a certain level of familiarity with Linux command line and system administration.

## 1. Fedora! Install, Setup, and Updates

- Quick setup to update:

  `sudo dnf upgrade -y`

- Firmware updates (BIOS):
  - This is specific to Framework AMD Devices and Linux.
  - This is for reference, not for every install.

```bash
sudo fwupdmgr refresh --force
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

## 2. GPU Drivers and tools, AMD/NVidia

### Install Vulkan

- Regardless of hardware, install Vulkan:

  `sudo dnf install -y  vulkan-tools`
  
  for nvidia add: `libva-nvidia-driver` May not exist

### GPU Monitoring

Btop is open to any hardware and solid.

- Install nvtop and/or btop:

    `sudo dnf install -y  btop`

- Run:

    `# nvtop`

    `btop -l`

- Vulkan directly:

    `vulkaninfo | grep NVIDIA`

Each GPU manufacturer amnd driver setup has it's on TUI/top style monitoring tools like nvidia-smi and radeontop. Check what is current for your hardware and find the proper install steps for your OS.

### AMD GPU Stack (ROCm + Gaming + AI) - Untested

- Enable required kernel modules

   `sudo modprobe amdgpu`

- Install core ROCm components (240 ish components, a few minutes to install)

```bash
sudo tee /etc/yum.repos.d/amdrocm-stable.repo <<EOF
[amdrocm-stable]
name=ROCm 10.0.0
baseurl=https://stable.repo.amd.com/rocm/core/packages/rhel10/x86_64
enabled=1
gpgcheck=1
gpgkey=https://stable.repo.amd.com/rocm/gpg/packages.gpg
EOF

sudo dnf clean all

# Install amdrocm10.0
sudo dnf install -y  amdrocm10.0

amd-smi version

# Install HIP Rntime (CUDA like)
sudo dnf install -y  amdrocm-runtime

# Install OpenCL runtime
sudo dnf install -y  rocm-opencl rocm-opencl-devel

# Validate ROCm installation

rocminfo
clinfo

# Monitoring:
# Install radeontop and btop:
sudo dnf install -y  radeontop btop
  
# Run:
# radeontop
```

### NVIDIA GPU Stack (CUDA + Gaming + AI)

Install and setup drivers for a modern Nvidia GPU:

```bash
# Add driver/etc repo:

sudo dnf install -y https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm 
sudo dnf install -y https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm

# Install drivers

sudo dnf install -y  xorg-x11-drv-nvidia-cuda xorg-x11-drv-nvidia-cuda-libs akmod-nvidia

# CUDA setup:

wget https://developer.download.nvidia.com/compute/cuda/13.3.1/local_installers/cuda-repo-fedora44-13-3-local-13.3.1_610.43.02-1.x86_64.rpm
sudo rpm -i cuda-repo-fedora44-13-3-local-13.3.1_610.43.02-1.x86_64.rpm
sudo dnf clean all
sudo dnf install -y cuda-toolkit-13-3

# Confirm CUDA: May need additional setup checks
nvcc --version

# Reboot and Validate:
nvidia-smi
```

## 3. Network and Connection Setup

### Fedora Cockpit setup - WIP

Already installed and will prompt to set up when ready

- Access Cockpit:

    `http://localhost:9090` or `http://your-fedora-ip:9090`

### Tailscale

```bash
# Install tailscale (curl):
curl -fsSL https://tailscale.com/install.sh | sh

# Start the client:
sudo tailscale up
```

Connect from browser when asked

### Remote Desktop Connection

- Setup a remote user within Fedora settings. Ensure debices on your network can find the server by name.

### Mount shares

- Create mount points and chmod

    ```bash
    sudo mkdir -p /mnt/nas_sharename # repeat for each share
    ```

- Set up credentials file like:

  - Create credential file:

    `sudo nano /etc/nas-creds.csv`

  - Add info, text
  
    ```text
    username=username
    password=PW
    domain=WORKGROUP # Windows share, no domain, example
    ```

  - Lock down file:

    `sudo chmod 600 /etc/nas-creds.csv`

- Set up/etc/fstab with share info:

    `sudo nano /etc/fstab`

  - Ex, text:

    ```text
    #
    # /etc/fstab Working
    # Created by anaconda on Mon Aug 31 04:18:29 2026
    #
    # Accessible filesystems, by reference, are maintained under '/dev/disk/'.
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info.
    #
    # After editing this file, run 'systemctl daemon-reload' to update systemd
    # units generated from this file.
    #

    # UUID content here

    # NAS server shares ADD THIS SECTION
    //NAS.local/SHARENAME  /mnt/nas_SHARENAME  cifs  credentials=/etc/nas-creds.csv,uid=1000,gid=1000,iocharset=utf8,vers=3.0,_netdev,x-systemd.automount,x-systemd.requires=network-online.target  0  0
    ```

## 4. Apps/etc Setup

### VS CODE

  ```bash
  # Enable Microsoft repo
  sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
  sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
  
  # Install VS Code
  sudo dnf -y install code
  ```

I have Copilot Chat pointed to my LLM server, but that changes based on the LLm tool and can break when the extension or vscode updates. I have the following st up currently again with a focus on LM Studio.

- Continue- Open source AI codiung agent (Continue.continue)
  - Open source standalone chat tool. Usually works well but is model specific.
  - Connects directly to rest API url of any tool.
- LM Studio Code (cgaspard.lmstudio-code)
  - Connects directly to your local Lm Studio instance.
  - LM Studio can then Link to a remote server or point directly to ti by url/rest api.
- LM Studio Copilot Chat Provider (NullSetIndustries.lmstudio-byok-chat-provider)
  - Designed to add LM Studio to Copilot Chat, not a standalone chat tool.
- Github Copilot
  - Same as what we use internally.
  - This can point to local models but is unreliable, sometimes still burning tokens. I disable it when I can
- WSL
- Docker

### Docker Rootless Setup ( a few confirmations)

  ```bash
  # Prep, add repo:
  sudo dnf config-manager addrepo --from-repofile https://download.docker.com/linux/fedora/docker-ce.repo

  # Install Docker:
  sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

  # Enable rootless:
  dockerd-rootless-setuptool.sh install

  # Restart docker
  sudo systemctl restart docker

  # Validate:
  docker info
  ```

- OBS Studio (flatpak!):

    `sudo flatpak install -y obs-studio`

- Wine:

    `sudo dnf install -y wine winetricks`

## 5. LLM Tools

### LM Studio - My Main tool

LM Studio may not be cutting edge or the most efficient tool but it i stable and allows connecting from other devioc using a few methods, including it's own Link program. You can connect to aM Studio on a remote system using an Open I like REST API. This allows you otopoint vs code extensions, other coding tools, apps, and more. LM Studio usses GGUF models in most cases and you can host them on a network share. If you are constantly unloading/loading models then local is preferred.

If you are using multiple GPUS, a mix of models or mix of AMD/Nvidia/Intel, vLLM and LLama.cpp or other core CLI llm tools will work better or allow models to run properly at all. LM Studio is great for setting GPU offload to max (run everything iun VRMA/on the GPU, default is CPU), and overflow into system RAM if you're one a typical PC.

- Install LM Studio:

    `curl -fsSL https://lmstudio.ai/install.sh | bash`

- Point to NAS share for models (requires networking section)

text edit .lmstudio settings.json. Update the downloadsfolder value. Will require restarting the server.
ex: /mnt/nas_software/LLM/Models

- unblock lms from SELinux (may not be required):
    `sudo semanage fcontext -a -t bin_t "/home/fedora/.lmstudio/bin(/.*)?"`
    `sudo restorecon -Rv /home/fedora/.lmstudio/bin`

- Configure server mode using systemctl and service file, text:

    `sudo nano /etc/systemd/system/lms.service`

Note that the below example loads 3 models at launch. These have no TTL and will stay in memeoy until manaully dropped, or triggered to drop by a tool asking for a lodel that can't fit in free RAM. This setup is designed for an AMD Strixc Halo 128gb but will use the default or primary GPU.

```text
[Unit]
Description=LM Studio Server
After=network.target

[Service]
Type=simple
User=fedora
WorkingDirectory=/home/fedora
Environment="HOME=/home/fedora"

# Start the background daemon first
ExecStart=/home/fedora/.lmstudio/bin/lms server start --port 1234 --bind 0.0.0.0

# Ensure the server is listening, then load all models safely inside one block
ExecStartPost=/usr/bin/bash -c ' \
  sleep 5; \
  echo "Loading Model 1..."; /home/fedora/.lmstudio/bin/lms load google/gemma-4-e4b --yes --gpu max --context-length 120000 || true; \
  echo "Loading Model 2..."; /home/fedora/.lmstudio/bin/lms load gemma-4-26b-a4b-it --yes --gpu max --context-length 160000 || true; \
  echo "Loading Model 3..."; /home/fedora/.lmstudio/bin/lms load qwen/qwen3.8-27b --yes --gpu max --context-length 160000 || true'

# Clean shutdown
ExecStop=/home/fedora/.lmstudio/bin/lms daemon down

# Disable systemd startup timeouts so massive models don't get interrupted mid-load
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

- Add Firewall port info

    ```bash
    sudo semanage port -a -t http_port_t -p tcp 8001
    sudo firewall-cmd --add-port=8001/tcp --permanent
    sudo firewall-cmd --reload
    ```

- Restart service:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable lms.service
    sudo systemctl start lms.service
    ```

### Vllm

- Install vLLM:

```bash
docker pull rocm/vllm:rocm10.0.0_ubuntu24.04_py3.14_pytorch_2.12.0_vllm_0.27.0
docker run -it --rm \
   --device /dev/kfd \
   --device /dev/dri \
   --network=host \
   --ipc=host \
   --group-add=video \
   --cap-add=SYS_PTRACE \
   --security-opt seccomp=unconfined \
   -v <path/to/your/models>:/app/models \
   -e HF_HOME="/app/models" \
   rocm/vllm:rocm10.0.0_ubuntu24.04_py3.14_pytorch_2.12.0_vllm_0.27.0 \
   bash
 ```

- Run as service (not set up):

    `sudo nano /etc/systemd/system/vllm.service`

- Example ExecStart:

    `ExecStart=/usr/bin/python3 -m vllm.entrypoints.openai.api_server --model /path/to/model --port 8002`

- Reload:

    `sudo systemctl daemon-reload`

- Enable:

    `sudo systemctl enable vllm`

- Start:

    `sudo systemctl start vllm`

### AMD Lemonade

Easy to set upobut I have not used it mtha much. Developed in partnership with AMD engineers to push ADM hardware as far as possible, specifically the Strix Halo boxes. Direct competitor to Nvidia backed tools and CUDA.

- Install ROCm dependencies (See Drivers section)
  
- Lemonade: `https://lemonade-server.ai/docs/guide/install/fedora/`

  - Download from:  `https://github.com/lemonade-sdk/lemonade/releases`

  - Run install:
   `sudo dnf install -y ./lemonade-server-11.9.0-fc44.x86_64.rpm`
- Validate Lemonade installation

     `lemonade --help`

- Run a test model

     `lemonade run --model llama-3.1-8b --device hip`

### AMD llama.cpp Deployment - Untested

- Build llama.cpp for AMD

    `git clone https://github.com/ggerganov/llama.cpp`
    `cd llama.cpp`
    `make hip`
    `./main -m model.gguf --gpu-layers 20`

- AMD Performance Tuning TBD

### NVidia llama.cpp Deployment - Untested

- Build llama.cpp for NVIDIA/cuda:

    ```bash
    git clone https://github.com/ggerganov/llama.cpp
    cd llama.cpp
    make cuda
    ```

- NVIDIA Performance Tuning TBD

### OpenHands Open source AI focused development tool

- Install:

    `sudo dnf install -y python3 python3-pip python3-devel git`
    `pip install openhands`

- Server (leave open):

    `openhands server`

- UI:
    `openhands ui`

### Pi and other tools

There are many other tools for Working with or using local models.

## Other information

This was developed outside BECU on my own time and hardware. This is a personal project document and should not be considered official guidance from BECU for the selection of OS, hardware, software, model, or any other technical decisions.
