# claw

Windows 11 Pro + Hyper-V + built-in tools (GPU  Sharing)


Enable Hyper-V

Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All

⸻

The architecture we’re aiming for (so you know why each step matters)

Goal state:
	•	Host = your real Windows 11 install (keeps full GPU control)
	•	VM = Windows 11 “OpenClaw Sandbox”
	•	GPU = shared with the VM via Hyper-V GPU Partitioning (GPU-P)
→ Host keeps using the GPU; the VM gets a real slice of it for CUDA/DirectML.

This is different from passthrough (which would steal the GPU from your host). You do not want passthrough for your use case.

⸻

Phase 1 — Prep the host (takes ~10–15 min)

1) Confirm you can actually use GPU-P

Open PowerShell as Administrator and run:

Get-VMHostPartitionableGpu

You want to see your GPU listed with Partitionable = True.

If it says False, stop — your driver or GPU config is blocking GPU-P (common fix: update NVIDIA drivers, or remove workstation/creator drivers that disable partitioning).

⸻

2) Turn on the right Hyper-V features (if not already on)

In PowerShell (Admin):

Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
Enable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -All

Reboot.

⸻

3) Create a clean virtual switch (NAT)

Open Hyper-V Manager → Virtual Switch Manager → New → Internal
Name it:

HyperV-NAT

Then in PowerShell (Admin):

New-VMSwitch -SwitchName "HyperV-NAT" -SwitchType Internal
New-NetIPAddress -IPAddress 172.16.100.1 -PrefixLength 24 -InterfaceAlias "vEthernet (HyperV-NAT)"
New-NetNat -Name "HyperV-NAT-Network" -InternalIPInterfaceAddressPrefix "172.16.100.0/24"

Why this matters:
Your VM gets internet without seeing your LAN (cameras, Home Assistant, NAS, router, etc.).

⸻

Phase 2 — Create the Windows 11 VM (correctly)

4) New VM (Gen 2)

In Hyper-V Manager:
	•	New → Virtual Machine
	•	Generation 2
	•	RAM: give it a real amount (I’d start with 16GB if you can spare it)
	•	Network: HyperV-NAT
	•	Disk: 80–120GB (you’ll be installing models later)
	•	Install Windows 11 Pro normally

When you land on the desktop:
	•	Install Windows Updates
	•	Install current NVIDIA drivers (same branch as host) inside the VM
(yes — you need drivers in both host and VM)

⸻

Phase 3 — Turn on GPU sharing (the fun part)

5) Add the partitionable GPU to the VM

Shut down the VM.

In PowerShell (Admin) on the host, replace OpenClawVM with your VM name:

Add-VMGpuPartitionAdapter -VMName "OpenClawVM"
Set-VMGpuPartitionAdapter -VMName "OpenClawVM" -MinPartitionVRAM 1GB -MaxPartitionVRAM 8GB -OptimalPartitionVRAM 4GB
Set-VMGpuPartitionAdapter -VMName "OpenClawVM" -MinPartitionEncode 1 -MaxPartitionEncode 8 -OptimalPartitionEncode 4
Set-VMGpuPartitionAdapter -VMName "OpenClawVM" -MinPartitionDecode 1 -MaxPartitionDecode 8 -OptimalPartitionDecode 4
Set-VMGpuPartitionAdapter -VMName "OpenClawVM" -MinPartitionCompute 1 -MaxPartitionCompute 8 -OptimalPartitionCompute 4

Start the VM.

⸻

6) Verify GPU inside the VM

Inside the VM:
	•	Open Device Manager → Display Adapters

You should see something like:

“Microsoft Basic Display Adapter” → then flips to → “NVIDIA GPU Partition”

Then test CUDA (once you install Python later):

nvidia-smi

If this works, GPU sharing is live.

⸻

Phase 4 — Make this a real sandbox (important)

Before you install OpenClaw or models, take a clean checkpoint:

In Hyper-V Manager → Right-click VM → Checkpoint → “Base + GPU Working”

From here on:
	•	Treat this VM as disposable
	•	Revert often instead of “cleaning up”
	•	Don’t sign into your personal Microsoft account inside the VM
	•	Use sandbox API keys only

⸻

Phase 5 — What this enables for OpenClaw + open models

With this setup you can now:

Inside the VM:
	•	Run:
	•	OpenClaw
	•	Ollama / vLLM / text-generation-webui / llama.cpp (CUDA builds)
	•	Local models (Llama, Mistral, Qwen, etc.)
	•	Use real GPU acceleration
	•	Snapshot before risky experiments
	•	Revert in seconds if something goes sideways

⸻