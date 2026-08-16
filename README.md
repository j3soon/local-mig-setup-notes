# Local NVIDIA MIG Setup for RTX PRO Blackwell

Unofficial minimal instructions for enabling and managing NVIDIA Multi-Instance GPU (MIG) on a local RTX PRO Blackwell system.

The examples use one NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition installed in an Ubuntu 24.04 host. The basic MIG lifecycle below was tested on that system.

## Prerequisites

- A [supported MIG GPU and system](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/supported-gpus.html).
- A supported Linux distribution.
- NVIDIA data center driver R575 (`575.51.03`) or newer for RTX PRO 6000/5000 Blackwell. The latest supported driver is recommended.
- Root access for changing MIG mode and, with the default capability permissions, managing GPU and compute instances.
- For RTX PRO 6000 Workstation/Max-Q and RTX PRO 5000 Blackwell GPUs that ship in graphics personality, a supported VBIOS and the GPU display personality set to `compute`.

Check the installed GPU, driver, VBIOS, and MIG state:

```sh
nvidia-smi
nvidia-smi --query-gpu=index,name,pci.bus_id,vbios_version,mig.mode.current,mig.mode.pending --format=csv
```

Minimum VBIOS versions for RTX PRO Blackwell are:

| GPU | Minimum VBIOS |
| --- | --- |
| RTX PRO 6000 Blackwell Workstation Edition | `98.02.55.00.00` |
| RTX PRO 6000 Blackwell Max-Q Workstation Edition | `98.02.6A.00.00` |
| RTX PRO 5000 Blackwell | `98.02.73.00.00` |

Contact the system vendor or reseller if a VBIOS update is required.

## Set the Display Personality to Compute

This section is required for supported RTX PRO Blackwell GPUs that are currently in graphics personality. Skip it if MIG mode is already reported as `Disabled` or `Enabled` rather than `N/A`.

```sh
nvidia-smi -q | grep -i -A 2 "MIG Mode"
```

A GPU in the default graphics personality typically reports:

```text
MIG Mode
    Current : N/A
    Pending : N/A
```

A GPU in compute personality can expose MIG and reports a real MIG state:

```text
MIG Mode
    Current : Disabled
    Pending : Disabled
```

> [!WARNING]
> Display Mode Selector changes the GPU firmware personality and PCI BAR layout. NVIDIA requires a system qualified for the requested mode and warns that an incompatible system can leave the GPU and host unusable. Compute personality also disables the physical display outputs. Establish remote access and confirm system support before continuing.

I have reproduced a `PCI BAR assignment ... is invalid` failure on an unqualified motherboard. Recovery required moving the GPU to a qualified system. Do not treat this operation as an ordinary driver setting.

Download the latest [NVIDIA Display Mode Selector Tool](https://developer.nvidia.com/displaymodeselector). Version `1.76.0` was used here:

```sh
unzip NVIDIA-Display-Mode-Selector-Tool-1.76.0-May26.zip
cd NVIDIA-Display-Mode-Selector-Tool-1.76.0-May26/linux/x64
chmod +x displaymodeselector

sudo ./displaymodeselector --list
sudo ./displaymodeselector --listgpumodes
```

If the GPU is used by Xorg, a CUDA workload, a monitoring agent, or the NVIDIA persistence service, stop those clients and unload the driver first. Keep an SSH session open because this stops the graphical desktop:

```sh
sudo systemctl isolate multi-user.target
sudo systemctl stop nvidia-persistenced.service
sudo systemctl stop nvidia-fabricmanager.service 2>/dev/null || true
sudo systemctl stop nvidia-dcgm.service 2>/dev/null || true

sudo fuser -v /dev/nvidia*
sudo modprobe -r nvidia_uvm nvidia_drm nvidia_modeset nvidia
lsmod | grep '^nvidia'
```

The last two commands should show no NVIDIA clients or loaded modules. Re-run the read-only checks, then switch the GPU only after verifying the selected adapter:

```sh
sudo ./displaymodeselector --listgpumodes
sudo ./displaymodeselector --gpumode compute
```

Confirm the update when prompted and reboot the Ubuntu host. To return to the default personality on a qualified system, use `--gpumode graphics` instead.

Do not patch the utility or attempt to reproduce the firmware update with `setpci` or `nvflash`.

## Basic MIG Operations

The following commands were tested with driver `580.173.02`, CUDA `13.0`, and VBIOS `98.02.6A.00.03` on an RTX PRO 6000 Blackwell Max-Q Workstation Edition.

Stop GPU clients before enabling MIG or creating instances. On desktop Ubuntu, Xorg may keep the parent GPU open and make `-cgi` fail with `In use by another client`, even when it uses only a few MiB. From an SSH session, stop the graphical target and verify that no process holds the device nodes:

```sh
sudo systemctl isolate multi-user.target
sudo systemctl stop nvidia-persistenced.service
sudo fuser -v /dev/nvidia*
```

`fuser` should produce no output. Do not unload the NVIDIA kernel modules for MIG operations: on Blackwell, unloading the driver disables MIG mode. Enable MIG on GPU 0:

```sh
sudo nvidia-smi -i 0 -mig 1
nvidia-smi --query-gpu=index,name,mig.mode.current,mig.mode.pending --format=csv
```

List the GPU instance profiles supported by the installed GPU and driver:

```sh
sudo nvidia-smi mig -i 0 -lgip
```

The tested 96 GB GPU exposes these base compute profiles, plus media-engine and `+gfx` variants:

| Profile | Maximum instances | Reported memory per instance |
| --- | ---: | ---: |
| `1g.24gb` | 4 | 23.62 GiB |
| `2g.48gb` | 2 | 47.38 GiB |
| `4g.96gb` | 1 | 95.00 GiB |

Always use `-lgip` as the source of truth for the current GPU and driver.

Create one `1g.24gb` GPU instance (GI) and its default compute instance (CI):

```sh
sudo nvidia-smi mig -i 0 -cgi 1g.24gb -C
```

List the resulting instances and their MIG UUIDs:

```sh
sudo nvidia-smi mig -i 0 -lgi
sudo nvidia-smi mig -i 0 -lci
nvidia-smi -L
```

Select a MIG device for a CUDA workload by its UUID:

```sh
CUDA_VISIBLE_DEVICES=MIG-<UUID> <your-cuda-command>
```

Destroy all compute instances before destroying their parent GPU instances:

```sh
sudo nvidia-smi mig -i 0 -dci
sudo nvidia-smi mig -i 0 -dgi
```

Disable MIG mode when it is no longer needed:

```sh
sudo nvidia-smi -i 0 -mig 0
```

### Persistence on Blackwell

On Hopper and newer GPUs, including RTX PRO Blackwell, MIG mode remains enabled only while the NVIDIA kernel driver stays loaded. Unloading/reloading the driver or rebooting disables MIG mode. GI and CI configurations are also not persistent and must be recreated after a GPU or system reset.

For a persistent desired layout, automate it at startup with [NVIDIA MIG Partition Editor (`mig-parted`)](https://github.com/NVIDIA/mig-parted) or an equivalent service.

## Troubleshooting

### GPU is in use

Find clients holding NVIDIA device nodes:

```sh
sudo fuser -v /dev/nvidia*
nvidia-smi
```

Stop the listed workloads or services before changing MIG mode or display personality. On a desktop system, `systemctl isolate multi-user.target` stops the graphical session; restore it afterward with:

```sh
sudo systemctl isolate graphical.target
```

### Insufficient permissions

MIG configuration operations require root by default:

```sh
sudo nvidia-smi mig -lgi
sudo nvidia-smi mig -lci
```

Permissions can be delegated through the NVIDIA MIG capability device nodes, but that is outside the scope of this minimal setup.

## References

- [NVIDIA Multi-Instance GPU User Guide](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/)
  - [Getting Started with MIG](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/getting-started-with-mig.html)
  - [Supported MIG Profiles](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/supported-mig-profiles.html)
  - [Device Nodes and Capabilities](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/device-nodes-and-capabilities.html)
- [NVIDIA Display Mode Selector Tool](https://developer.nvidia.com/displaymodeselector)
- [NVIDIA MIG Partition Editor](https://github.com/NVIDIA/mig-parted)
