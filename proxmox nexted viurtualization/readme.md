# Proxmox CPU Setup for Nested Virtualization

Covers host preparation, web UI configuration, and CPU pinning for both AMD and Intel platforms.

---

## Part 1 — Host Preparation

Before creating the VM, confirm nested virtualization is enabled at the kernel level. Without this, the CPU flags you set in the VM have no effect.

---

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

---

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

---

## Part 2 — Web UI CPU Tab

When creating or editing a VM, go to the **CPU** tab.

### Common settings (both platforms)

| Field | Value |
|---|---|
| Sockets | `1` |
| Cores | Number of cores to assign (e.g. `8`) |
| NUMA | ❌ Disabled (unless you have multiple physical CPUs) |

---

### CPU Type — AMD

Set **Type** to `host`.

Using `host` is required on AMD for nested virtualization — it passes the physical SVM flag through to the guest. Generic types like `kvm64` or `x86-64-v2` do not expose SVM and nested virt will silently fail.

---

### CPU Type — Intel

Set **Type** to `host`.

Same reasoning as AMD — `host` passes the VMX flag through. If you need live migration between Intel hosts of different generations, use `max` instead, which exposes the most compatible set of features the host supports.

---

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

---

## Part 3 — Memory (required alongside CPU flags)

This is not a CPU setting, but it must be configured correctly for the CPU flags to be stable.

Go to the **Memory** tab and **disable ballooning** (set minimum to `0` or uncheck the balloon option entirely).

The balloon driver competes with Hyper-V for memory management control. With `+nested-virt` enabled and WSL2 or Hyper-V running inside the guest, a running balloon driver causes instability and memory pressure crashes.

---

## Part 4 — CPU Pinning (post-creation, shell only)

The web UI does not expose CPU pinning. This is done from the Proxmox host shell after the VM is created.

Pinning matters most on multi-CCD AMD processors (Ryzen 7000/9000, Threadripper) where vCPUs scattered across dies incur cross-CCD latency and share no L3 cache.

### Check your topology first

```bash
apt install hwloc -y
lstopo --of console
```

Look for `Die` blocks. Each die is a separate CCD with its own L3 cache. Note the physical CPU (`P#`) numbers within each die.

---

### AMD — Ryzen 9 7950X example

The 7950X has two CCDs:

| CCD | Physical cores | HT siblings |
|---|---|---|
| Die 0 | P#0–7 | P#16–23 |
| Die 1 | P#8–15 | P#24–31 |

Pin an 8-core VM to Die 0:

```bash
qm set <vmid> --cpuset 0-7,16-23
```

Pin to Die 1 instead (e.g. for a second VM):

```bash
qm set <vmid> --cpuset 8-15,24-31
```

This keeps all vCPUs within the same 32 MB L3 cache and eliminates cross-CCD latency.

---

### Intel — example (single-die)

Most desktop Intel CPUs (Core i-series) are single-die, so cross-die latency is not a concern. Pinning is still useful to prevent the host scheduler from moving vCPUs to efficiency cores (E-cores) on hybrid architectures (Alder Lake / Raptor Lake and newer).

List cores by type:

```bash
lstopo --of console | grep -E "Core|PU"
```

On a Raptor Lake system, P-cores appear first in the topology. Pin an 8-core VM to the first 8 P-cores and their HT siblings:

```bash
qm set <vmid> --cpuset 0-7,16-23
```

Adjust the range to match what `lstopo` shows for your specific CPU.

---

### Verify pinning was applied

```bash
grep cpuset /etc/pve/qemu-server/<vmid>.conf
```

---

## Part 5 — Verifying the guest sees nested virtualization

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

---

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
