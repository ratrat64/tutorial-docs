# Contents

- [Proxmox CPU Setup for Nested Virtualization](#proxmox-cpu-setup-for-nested-virtualization)
    - [Part 1 — Host Preparation](#part-1--host-preparation)
        - [AMD (Ryzen / EPYC)](#amd-ryzen--epyc)
        - [Intel (Core / Xeon)](#intel-core--xeon)
    - [Part 2 — Web UI CPU Tab](#part-2--web-ui-cpu-tab)
        - [Common Settings](#common-settings-both-platforms)
        - [CPU Type — AMD](#cpu-type--amd)
        - [CPU Type — Intel](#cpu-type--intel)
        - [Extra CPU Flags](#extra-cpu-flags)
    - [Part 3 — Memory](#part-3--memory-required-alongside-cpu-flags)
    - [Part 4 — Verifying the Guest Sees Nested Virtualization](#part-4--verifying-the-guest-sees-nested-virtualization)
        - [From Windows (inside the VM)](#from-windows-inside-the-vm)
        - [From the Proxmox Host](#from-the-proxmox-host-while-vm-is-running)
    - [Quick Reference](#quick-reference)
- [Create Scheduled Job for Rename Script](#create-scheduled-job-for-rename-script)

# Proxmox CPU Setup for Nested Virtualization

Covers host preparation, web UI configuration, and CPU pinning for both AMD and Intel platforms.



## Part 1 — Host Preparation

Before creating the VM, confirm nested virtualization is enabled at the kernel level. Without this, the CPU flags you set in the VM have no effect.



### AMD (Ryzen / EPYC)

Check current state:

```bash
cat /sys/module/kvm_amd/parameters/nested
```

If the output is not `1`:

```bash
echo "options kvm_amd nested=1" > /etc/modprobe.d/kvm-amd.conf
modprobe -r kvm_amd && modprobe kvm_amd
```

Verify again — must return `1` before continuing.



### Intel (Core / Xeon)

Check current state:

```bash
cat /sys/module/kvm_intel/parameters/nested
```

If the output is not `1`:

```bash
echo "options kvm_intel nested=1" > /etc/modprobe.d/kvm-intel.conf
modprobe -r kvm_intel && modprobe kvm_intel
```

Verify again — must return `1` before continuing.

> **Note:** Some Intel systems also benefit from enabling unrestricted guest mode:
> `echo "options kvm_intel nested=1 unrestricted_guest=1" > /etc/modprobe.d/kvm-intel.conf`



## Part 2 — Web UI CPU Tab

When creating or editing a VM, go to the **CPU** tab.

### Common settings (both platforms)

| Field | Value |
|---|---|
| Sockets | `1` |
| Cores | Number of cores to assign (e.g. `8`) |
| NUMA | ❌ Disabled (unless you have multiple physical CPUs) |



### CPU Type — AMD

Set **Type** to `host`.

Using `host` is required on AMD for nested virtualization — it passes the physical SVM flag through to the guest. Generic types like `kvm64` or `x86-64-v2` do not expose SVM and nested virt will silently fail.



### CPU Type — Intel

Set **Type** to `host`.

Same reasoning as AMD — `host` passes the VMX flag through. If you need live migration between Intel hosts of different generations, use `max` instead, which exposes the most compatible set of features the host supports.



### Extra CPU Flags

Click **Extra CPU Flags** at the bottom of the CPU tab. These are additional CPUID bits to expose to the guest.

#### AMD flags

| Flag | Setting | Purpose |
|---|---|---|
| `nested-virt` | `+` on | Exposes SVM to the guest — required for WSL2 / Hyper-V inside Windows |
| `ibpb` | `+` on | Spectre v2 mitigation (branch predictor barrier) |
| `virt-ssbd` | `+` on | Spectre v4 mitigation via virtual SSBD MSR |
| `amd-ssbd` | `+` on | Spectre v4 mitigation via hardware SSBD — more efficient than `virt-ssbd` alone |
| `pdpe1gb` | `+` on | Enables 1 GB huge pages — reduces TLB pressure for large memory workloads |
| `aes` | `+` on | Exposes AES-NI to the guest |

Resulting config line:

```
cpu: host,flags=+nested-virt;+ibpb;+virt-ssbd;+amd-ssbd;+pdpe1gb;+aes
```

#### Intel flags

| Flag | Setting | Purpose |
|---|---|---|
| `nested-virt` | `+` on | Exposes VMX to the guest — required for WSL2 / Hyper-V inside Windows |
| `ibpb` | `+` on | Spectre v2 mitigation |
| `ibrs` | `+` on | Spectre v2 mitigation — use instead of `amd-ssbd` on Intel |
| `ssbd` | `+` on | Spectre v4 mitigation via SSBD MSR |
| `pdpe1gb` | `+` on | 1 GB huge pages (supported on most Intel Xeon and recent Core generations) |
| `aes` | `+` on | AES-NI |
| `hv-evmcs` | `+` on | Enlightened VMCS — reduces Hyper-V/WSL2 VM-exit overhead on Intel; **AMD does not support this** |

Resulting config line:

```
cpu: host,flags=+nested-virt;+ibpb;+ibrs;+ssbd;+pdpe1gb;+aes;+hv-evmcs
```

> `hv-evmcs` is an Intel-only optimisation. It reduces the overhead of nested hypervisor VM-exits, which noticeably improves WSL2 and Hyper-V performance inside the guest. There is no AMD equivalent.


## Part 3 — Memory (required alongside CPU flags)

This is not a CPU setting, but it must be configured correctly for the CPU flags to be stable.

Go to the **Memory** tab and **disable ballooning** (set minimum to `0` or uncheck the balloon option entirely).

The balloon driver competes with Hyper-V for memory management control. With `+nested-virt` enabled and WSL2 or Hyper-V running inside the guest, a running balloon driver causes instability and memory pressure crashes.


## Part 4 — Verifying the guest sees nested virtualization

### From Windows (inside the VM)

Open PowerShell as administrator:

```powershell
# Check Hyper-V is present
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V

# Or just try WSL2
wsl --install
```

If WSL2 installs without a virtualisation error, nested virt is working.

### From the Proxmox host (while VM is running)

Confirm Hyper-V enlightenments are being passed through:

```bash
ps aux | grep qemu | grep -o 'hv_[a-z_]*'
```

Expected output includes: `hv_relaxed`, `hv_vapic`, `hv_spinlocks`, `hv_time`.  
On Intel with `hv-evmcs`: also `hv_evmcs`.



## Quick Reference

| Setting | AMD | Intel |
|---|---|---|
| Kernel module | `kvm_amd` | `kvm_intel` |
| Nested param file | `/sys/module/kvm_amd/parameters/nested` | `/sys/module/kvm_intel/parameters/nested` |
| CPU type in UI | `host` | `host` (or `max` for migration) |
| Nested virt flag | `+nested-virt` | `+nested-virt` |
| Spectre v4 flags | `+virt-ssbd;+amd-ssbd` | `+ssbd;+ibrs` |
| Intel-only flag | — | `+hv-evmcs` |
| Multi-CCD pinning | Required on Ryzen 7000+ / Threadripper | Rarely needed; useful on hybrid (P/E-core) CPUs |

# Create scheduled job for **rename script**

Run this in PowerShell as Administrator on your template, before converting it to a template in Proxmox:

```Powershell
powershell$action = New-ScheduledTaskAction -Execute "PowerShell.exe" `
    -Argument "-ExecutionPolicy Bypass -File C:\Scripts\FirstBootRename.ps1"

$trigger = New-ScheduledTaskTrigger -AtStartup

$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -RunOnlyIfNetworkAvailable:$false

$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask `
    -TaskName "FirstBootRename" `
    -Action $action `
    -Trigger $trigger `
    -Settings $settings `
    -Principal $principal `
    -Description "Renames the VM on first boot, then deletes itself"
```