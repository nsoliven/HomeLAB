
# Updated Proxmox GPU Passthrough Guide (Works for Windows 10 & 11)

## 1. Check Basic Requirements

1. **Hardware**  
   - CPU & Motherboard must support **VT-d/AMD-Vi** (IOMMU). Enable it in your BIOS/UEFI. (Often listed as “VT-d,” “AMD-Vi,” or “IOMMU.”)
   - Some motherboards have separate ACS or PCIe grouping settings that help create distinct IOMMU groups.  
   - If using **ZFS** as root, confirm you have an **EFI system partition** (ESP) for UEFI boot, or that you are indeed using GRUB in Legacy/BIOS.

2. **Proxmox Updates**  
   - Make sure Proxmox packages and kernels are up to date:
     ```bash
     apt update
     apt dist-upgrade
     ```
   - Reboot if a new kernel is installed.

3. **Identify Your Bootloader**  
   - If you’re on UEFI + ZFS, Proxmox often uses **systemd-boot** (pve-efiboot-tool).  
   - If you’re on UEFI but see `/EFI/proxmox/grubx64.efi` in `efibootmgr -v`, you’re using **GRUB** in UEFI mode.  
   - If running BIOS/Legacy, you’re definitely using GRUB.  
   - Confirm with:
     ```bash
     efibootmgr -v
     proxmox-boot-tool status
     cat /proc/cmdline
     ```
   - For **systemd-boot**, kernel parameters are set in `/etc/kernel/cmdline` and updated via `pve-efiboot-tool refresh`.  
   - For **GRUB** (UEFI or Legacy), kernel parameters are set in `/etc/default/grub` and updated via `update-grub`.

---

## 2. Enable IOMMU

Depending on your CPU vendor and bootloader:

### 2.1 If using **GRUB**

1. **Edit** `/etc/default/grub`:
   ```bash
   nano /etc/default/grub
   ```
2. In the line `GRUB_CMDLINE_LINUX_DEFAULT="quiet ..."`, add:
   - **Intel**: `intel_iommu=on iommu=pt`  
   - **AMD**: `amd_iommu=on iommu=pt`  
   > **Note**: Strictly speaking, modern AMD kernels often enable `amd_iommu=on` by default. But if passthrough fails, explicitly add it anyway.
3. **Optional Additional Params** (helpful for older Xeons or if GPU is your only card):
   - `pcie_acs_override=downstream,multifunction`  
   - `initcall_blacklist=sysfb_init` (or `nomodeset` / `video=efifb:off` to disable host usage of GPU).
   - Example:
     ```bash
     GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt nomodeset pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init"
     ```
4. **Update GRUB**:
   ```bash
   update-grub
   ```

### 2.2 If using **systemd-boot** (UEFI + ZFS)

1. **Edit** `/etc/kernel/cmdline`:
   ```bash
   nano /etc/kernel/cmdline
   ```
2. Make sure you have something like:
   - **Intel**:  
     ```
     root=ZFS=rpool/ROOT/pve-1 boot=zfs quiet intel_iommu=on iommu=pt
     ```
   - **AMD**:  
     ```
     root=ZFS=rpool/ROOT/pve-1 boot=zfs quiet amd_iommu=on iommu=pt
     ```
3. **Refresh** systemd-boot:
   ```bash
   pve-efiboot-tool refresh
   ```
4. **Optional**: Add advanced parameters if needed, e.g., `pcie_acs_override=downstream,multifunction`, etc.

### 2.3 Verify IOMMU

- **Reboot** and check:
  ```bash
  dmesg | grep -e DMAR -e IOMMU -e AMD-Vi
  ```
  - Look for lines like `IOMMU enabled` or `AMD-Vi: Interrupt remapping enabled`.  

---

## 3. Install / Load VFIO Modules

1. **Add VFIO Modules** to `/etc/modules` (if not already present):
   ```bash
   echo "vfio" >> /etc/modules
   echo "vfio_pci" >> /etc/modules
   echo "vfio_iommu_type1" >> /etc/modules
   # For Proxmox VE <=7.x, also:
   echo "vfio_virqfd" >> /etc/modules
   # (vfio_virqfd is removed in PVE 8 with kernel 6.2)
   ```
2. **Update initramfs**:
   ```bash
   update-initramfs -u -k all
   ```
   - On UEFI + ZFS, also run:
     ```bash
     proxmox-boot-tool refresh
     ```

---

## 4. Blacklist or Softdep Your GPU Driver

### 4.1 Blacklisting (Simple)

**If** you don’t want Proxmox to bind any driver (e.g., `nvidia`, `radeon`, `amdgpu`, `nouveau`, `i915`, etc.):

1. Create `/etc/modprobe.d/blacklist-gpu.conf`:
   ```bash
   nano /etc/modprobe.d/blacklist-gpu.conf
   ```
2. Add lines (for NVIDIA example):
   ```bash
   blacklist nouveau
   blacklist nvidia
   blacklist nvidiafb
   blacklist nvidia_drm
   ```
   - For AMD:
     ```bash
     blacklist radeon
     blacklist amdgpu
     ```
   - For Intel (only if you’re passing iGPU):
     ```bash
     blacklist i915
     ```
3. **Regenerate initramfs** and refresh (UEFI):
   ```bash
   update-initramfs -u -k all
   proxmox-boot-tool refresh  # If on systemd-boot
   ```

### 4.2 Softdep Method (Early Binding)

Rather than blacklisting, you can instruct the VFIO module to grab the device first:

1. Edit `/etc/modprobe.d/vfio.conf`:
   ```bash
   nano /etc/modprobe.d/vfio.conf
   ```
2. Specify the device IDs and **softdep** lines:

   ```bash
   options vfio-pci ids=10de:2782,10de:22bc  # Example NVIDIA GPU + Audio
   softdep nvidia pre: vfio-pci
   softdep nouveau pre: vfio-pci
   softdep nvidia_drm pre: vfio-pci
   softdep nvidiafb pre: vfio-pci
   # For AMD
   softdep radeon pre: vfio-pci
   softdep amdgpu pre: vfio-pci
   # For Intel
   softdep i915 pre: vfio-pci
   ```
3. Update initramfs & refresh if needed.

---

## 5. Bind the GPU to `vfio-pci`

1. **Find the PCI IDs** of your GPU & audio function:
   ```bash
   lspci -nn | grep -i vga
   lspci -nn | grep -i audio
   ```
   Example:
   - GPU: `10de:2782`
   - Audio: `10de:22bc`
2. **Add** to `/etc/modprobe.d/vfio-pci.conf` (if using the blacklisting approach):
   ```bash
   options vfio-pci ids=10de:2782,10de:22bc
   ```
   (You can skip this if you used the **softdep** approach above and already specified `ids=...`)

3. **Update initramfs** again:
   ```bash
   update-initramfs -u -k all
   proxmox-boot-tool refresh  # on systemd-boot
   ```
4. **Reboot**.

### 5.1 Checking the Binding

After reboot:

- `lspci -nnk | grep -E '2782|22bc' -A3` (for example)  
  - Should show `Kernel driver in use: vfio-pci`.  
- `lsmod | grep nvidia` should yield no output if blacklisted.  
- `lsmod | grep vfio` should show `vfio_pci`, `vfio_iommu_type1`, etc.

---

## 6. Create or Configure the Windows VM

1. **Create VM** in Proxmox GUI:
   - BIOS: **OVMF (UEFI)** is recommended for GPU passthrough.  
   - Machine type: **q35**.  
   - Disk: Use **virtio-scsi** if you want best performance (you’ll need VirtIO ISO for Windows).  
   - Set “Primary display” to something basic (like **Default**, or “VMware compatible display”) initially.  
   - Don’t start the VM yet.

2. **Add an EFI disk** (Proxmox will prompt you automatically if you pick OVMF).
3. **(Optional)** If using Windows 11, ensure the `TPM` and `Secure Boot` features are handled if you want a fully compliant Win11 install. (Proxmox 7.1+ supports adding a TPM emulated device.)

---

## 7. Pass the GPU to the VM

1. Go to VM → **Hardware** → **Add** → **PCI Device**.
2. Pick your GPU device from the drop-down. Also add the audio function if separate.  
3. Enable “**PCI-Express**” and “**All Functions**” if it’s a multi-function GPU.  
4. Optionally set “**Primary GPU**” if you want the VM to boot from that GPU’s output (i.e., you have a physical monitor attached).  
   > If you set “Primary GPU,” you may want to remove the “Display” hardware or set it to “none.” Otherwise, the guest might initialize the wrong adapter.  

5. If using **q35 + OVMF** and you have an older or special GPU, you might need a ROM file (`romfile=`). See more advanced guides if you get a Code 43 or no output on the monitor.  

---

## 8. Windows 10/11 Installation & Drivers

1. **Attach Windows ISO** + **VirtIO Drivers ISO**.  
2. Boot the VM. In the Windows installer, load the **virtio** drivers for `SCSI` and `Network` from the VirtIO ISO.  
3. After Windows installs, open **Device Manager** to see if the GPU appears with an error.  
   - If you see Code 43 or an “Unable to start device” error, ensure you have the parameters `args: -cpu 'host,kvm=off,...'` or `hidden=1` in your VM config. Proxmox often sets `hidden=1` by default when you choose CPU type “host.”  
   - For older NVIDIA 10xx or 20xx cards, you may need a patched or extracted vBIOS ROM to avoid the error.  

4. **Install GPU drivers** (NVIDIA/AMD):
   - For NVIDIA, you may also need to pass in a custom `hv_vendor_id` to hide virtualization, e.g., `args: -cpu 'host,....,hv_vendor_id=MYKVMHACK'`.  

5. **Enable** RDP or VNC within Windows if you plan to remove the “Proxmox display” adapter.  
6. Reboot the VM. If you set your passed-through GPU as primary, you should see the Windows boot screen on the physical monitor connected to that GPU.

---

## 9. Troubleshooting Common Issues

1. **IOMMU Groups**: If your GPU is in the same group as other devices, you may need `pcie_acs_override=downstream,multifunction`. **Use with caution**—this can reduce security/isolation.  
2. **Code 43** (NVIDIA): Try setting `hidden=1` or advanced `args`. Some older GPUs may need a “patched” ROM.  
3. **No Video Output**: Check if the GPU has an **UEFI-compatible** ROM. If it only supports Legacy BIOS, you may need to switch to SeaBIOS for the VM.  
4. **GPU Reset Bugs** (Mostly AMD): Certain older AMD GPUs require the [vendor-reset](https://github.com/gnif/vendor-reset) module.  
5. **ZFS + UEFI** quirks: Always run:
   ```bash
   proxmox-boot-tool refresh
   ```
   after changes to `vfio.conf` or kernel parameters.  
6. **Interrupt Remapping** not enabled: Check your BIOS for “Interrupt Remapping” or “VT-d/AMD-Vi.”  
7. **Windows 11** requires TPM & Secure Boot if you want official support. Proxmox 7.1+ can emulate a vTPM. Alternatively, you can bypass Microsoft’s official checks.

---

## 10. Done! Enjoy Your Windows 10 or 11 GPU VM

- Once the VM is set up and your GPU driver (NVIDIA/AMD) is installed in the guest, you can use your VM for gaming, video editing, etc., with near-native performance.  
- Confirm “**Kernel driver in use: vfio-pci**” on the host so the host isn’t grabbing your GPU.  
- For Windows 11, the process is the same as Windows 10—just make sure you meet the Win11 “TPM + Secure Boot” demands (if you care about official compliance).

---

### Helpful Commands Recap

```bash
# Show current driver usage for PCI devices
lspci -nnk

# Show IOMMU grouping info via PVE API
pvesh get /nodes/<NODENAME>/hardware/pci --pci-class-blacklist ""

# Update Grub if using GRUB
update-grub

# Regenerate initramfs
update-initramfs -u -k all

# If on ZFS + systemd-boot
proxmox-boot-tool refresh

# Reboot
reboot
```

---

## Final Notes

- The **single biggest difference** for **ZFS + UEFI** is you’ll need `proxmox-boot-tool refresh` after you make changes that affect the kernel or initramfs.  
- **Windows 11** setup is nearly identical to Windows 10, aside from needing a **TPM** device and/or the registry tweak for bypassing hardware checks.  
- If the GPU is your **only** video card in the system, additional steps (like `nomodeset`, `video=efifb:off`, etc.) might be required so that Proxmox doesn’t grab it during boot.  
- For **older AMD** cards with the “reset bug,” see [vendor-reset](https://github.com/gnif/vendor-reset).  
- For **NVIDIA “Error 43”** issues, remember the `hidden=1` or patched vBIOS approach.  

Enjoy your **Windows 10/11** VM with full GPU passthrough on Proxmox! If you get stuck, consult:  
- [Proxmox Forum](https://forum.proxmox.com/)  
- [r/homelab](https://www.reddit.com/r/homelab/)  
- [r/VFIO](https://www.reddit.com/r/VFIO/)  

Sources used in this were:
- https://forum.proxmox.com/threads/pci-gpu-passthrough-on-proxmox-ve-8-installation-and-configuration.130218/
- https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_pci_passthrough
- https://gist.github.com/KasperSkytte/6a2d4e8c91b7117314bceec84c30016b
