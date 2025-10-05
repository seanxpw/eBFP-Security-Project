# eBFP-Security-Project

> CS202 Advanced Operating Systems — Security Project  
> Evaluating **SafeBPF** against **CVE-2017-16995** in the Linux eBPF subsystem.

All details are in the report: **[CS202_BPF_project_report.pdf](./CS202_BPF_project_report.pdf)**.

---

## Overview

- **Goal:** Evaluate whether **SafeBPF** can mitigate the eBPF privilege-escalation exploit **CVE-2017-16995** on **Linux v6.3.8**.
- **Approach:** QEMU-based lab on Debian 12; rebuild kernels; recreate CVE conditions by reverting the fix and by **bypassing the verifier** in controlled ways; compare **with/without SafeBPF**.
- **Findings (brief):**
  - Skipping the verifier often broke essential runtime setup (e.g., map pointer resolution), leading to stalls/faults — the verifier is critical for **correct execution**, not just security.   
  - **SafeBPF** intercepted faulty memory paths (e.g., page fault → controlled NULL deref), indicating **mitigation potential** in limited scenarios.
  - Full privilege escalation could **not** be reproduced on v6.3.8, but SafeBPF changed error-handling paths in ways consistent with defense-in-depth.

---

## Environment Setup

QEMU-based virtualized environment for x86_64 with a consistent Debian 12 rootfs and swappable custom kernels.

### 1) Sources
- **Linux Kernel v6.3.8** (x86_64) — kernel.org
- **Linux Kernel v4.4.110** — used to validate exploit behavior on an older baseline
- **Exploit PoC:** Kernel Exploit Factory — CVE-2017-16995
- **SafeBPF Patch:** UBC SafeBPF (targets 6.3.8)

> ⚠️ For research/education only. Do not deploy insecure configs on production systems.

### 2) Build Kernel (enable eBPF, optional hardening toggles)

Enable core eBPF options and build:
```bash
# from kernel source tree with a prepared .config
sed -i 's/# CONFIG_BPF_SYSCALL is not set/CONFIG_BPF_SYSCALL=y/' .config
sed -i 's/# CONFIG_BPF_JIT is not set/CONFIG_BPF_JIT=y/' .config

make oldconfig
make -j"$(nproc)"
```

(For testing exploit behavior) disable certain mitigations:
```bash
# only for controlled lab experiments
sed -i 's/^CONFIG_PAGE_TABLE_ISOLATION=.*/CONFIG_PAGE_TABLE_ISOLATION=n/' .config
sed -i 's/^CONFIG_RANDOMIZE_BASE=.*/CONFIG_RANDOMIZE_BASE=n/' .config
```

### 3) QEMU VM (Debian 12 cloud image)

- **Root FS:** Debian 12 (Bookworm) cloud image (`debian-12-nocloud-amd64.qcow2`)
- **Boot:** use `-kernel` to run your custom `bzImage`

```bash
qemu-system-x86_64 -enable-kvm \
  -m 8G \
  -smp 2 \
  -kernel ./bzImage \
  -append "console=ttyS0 root=/dev/sda1 rw earlyprintk=serial net.ifnames=0 kaslr no_hash_pointers idle=halt" \
  -drive file=./debian-12-nocloud-amd64.qcow2,format=qcow2 \
  -net user,host=10.0.2.10,hostfwd=tcp:127.0.0.1:10021-:22 \
  -net nic,model=e1000 \
  -cpu host \
  -nographic \
  -pidfile vm.pid
```

**Flag cheatsheet:** `-m 8G` (RAM), `-smp 2` (vCPUs), `-kernel bzImage` (custom kernel), `-drive` (Debian image), `-net ... hostfwd` (SSH: `localhost:10021` → guest `22`).

### 4) SafeBPF on 6.3.8

- Apply **SafeBPF** to Linux 6.3.8 and rebuild.
- Compare behavior across:
  - 6.3.8 (stock) vs. 6.3.8 + SafeBPF
  - Verifier “as-is” vs. intentionally-bypassed paths (for controlled experiments)

### 5) Summary Table

| Component       | Details                                             |
|----------------|------------------------------------------------------|
| Kernel         | Linux **6.3.8** & **4.4.110**                        |
| Exploit        | CVE-2017-16995 PoC (Kernel Exploit Factory)         |
| Virtualization | QEMU (fast kernel switching)                         |
| Filesystem     | Debian 12 (Bookworm) cloud image                     |
| Patching       | SafeBPF on 6.3.8 to observe defensive behavior       |

---

## Results (very short)

Verifier skipping breaks required runtime transforms (e.g., map replacement), often causing stalls/faults. **SafeBPF** altered fault handling and prevented crashes in some cases, suggesting partial mitigation, though full exploit reproduction on 6.3.8 was not achieved.

---

## Quick Reproduce

1. Build Linux **6.3.8** with **BPF_SYSCALL** and **BPF_JIT** enabled.
2. (Lab-only) Optionally disable **KPTI/KASLR** to simplify exploit reproducibility.
3. Boot the kernel in QEMU with the **Debian 12** cloud image.
4. Apply and build a **SafeBPF**-patched kernel, reboot QEMU, and compare runs.
5. Run the test programs/PoC only in a controlled environment and collect kernel logs.

---

## Resume-ready blurb

**Project:** *Evaluating SafeBPF’s Mitigation of CVE-2017-16995 on Linux Kernel v6.3.8*
**Objective:** Investigated if the SafeBPF security module could prevent a known privilege escalation exploit (CVE-2017-16995) within the Linux eBPF subsystem.
**Challenge & Approach:** As SafeBPF is only available on newer kernels (v6.3.8) where the CVE is patched, the core task was to re-create the vulnerability’s attack conditions in this modern environment using a QEMU VM.
**Execution:** Attempted several methods to simulate the exploit, primarily by modifying the kernel's C source code to bypass the eBPF verifier, a key component in the original attack vector.
**Findings & Conclusion:** The attempts revealed that bypassing the verifier often caused system stalls, uncovering its critical role in program execution, not just security. While a full exploit recreation was not achieved, in limited scenarios, SafeBPF was observed to alter the kernel's error handling, providing partial insight into its potential defensive capabilities.

---

## Acknowledgments

- CVE PoC: https://github.com/bsauce/kernel-exploit-factory/tree/main/CVE-2017-16995
- SafeBPF Patch: https://github.com/UBC-BPF/safebpf-patch

---

## Repository

GitHub: https://github.com/seanxpw/eBFP-Security-Project
