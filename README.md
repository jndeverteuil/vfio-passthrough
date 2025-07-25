# VFIO Passthrough Setup on Pop!_OS 22.04

This documentation guides you through setting up VFIO (Virtual Function I/O) passthrough on Pop!\_OS 22.04. VFIO allows you to dedicate a physical PCI device, such as a graphics card, directly to a virtual machine, providing near-native performance for your guest OS. This is particularly useful for gaming or demanding applications within a Windows 11 VM.

**Please read all instructions carefully before proceeding.**<span style="white-space: pre-wrap;"> VFIO setup can be complex, and incorrect steps may lead to system instability or graphical issues. This guide assumes a fresh installation and aims to be thorough for users less experienced with VFIO.</span>

## References:

- [https://mathiashueber.com/passthrough-windows-11-vm-ubuntu-22-04/](https://mathiashueber.com/passthrough-windows-11-vm-ubuntu-22-04/)
- [https://gist.github.com/k-amin07/47cb06e4598e0c81f2b42904c6909329](https://gist.github.com/k-amin07/47cb06e4598e0c81f2b42904c6909329)
- [https://mathiashueber.com/virtual-machine-audio-setup-get-pulse-audio-working/](https://mathiashueber.com/virtual-machine-audio-setup-get-pulse-audio-working/)
- [https://wiki.archlinux.org/title/PCI\_passthrough\_via\_OVMF#Passing\_audio\_from\_virtual\_machine\_to\_host\_via\_JACK\_and\_PipeWire](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_JACK_and_PipeWire)
- [https://www.heiko-sieger.info/blacklisting-graphics-driver/#Using\_the\_driver\_override\_feature](https://www.heiko-sieger.info/blacklisting-graphics-driver/#Using_the_driver_override_feature)
- [https://github.com/mr2527/pop\_OS-win10-KVM-setup](https://github.com/mr2527/pop_OS-win10-KVM-setup)
- [https://www.reddit.com/r/VFIO/comments/wx855g/working\_intel\_alder\_lake\_setup\_on\_a\_12700k/](https://www.reddit.com/r/VFIO/comments/wx855g/working_intel_alder_lake_setup_on_a_12700k/)
- [https://forums.developer.nvidia.com/t/cant-rebind-gpu-with-driverctl-if-system-booted-with-gpu-attached-to-nvidia-driver/191350](https://forums.developer.nvidia.com/t/cant-rebind-gpu-with-driverctl-if-system-booted-with-gpu-attached-to-nvidia-driver/191350)
- [https://forum.level1techs.com/t/solved-rtx-3090-gpu-passthrough-just-displays-a-black-screen-with-qemu/175538](https://forum.level1techs.com/t/solved-rtx-3090-gpu-passthrough-just-displays-a-black-screen-with-qemu/175538)
- [https://looking-glass.io/wiki/Using\_JACK\_and\_PipeWire](https://looking-glass.io/wiki/Using_JACK_and_PipeWire)

## 1. Install the Required Software

Before you begin, you need to install the necessary virtualization software packages.

- **QEMU/KVM (`<strong class="editor-theme-bold editor-theme-code">qemu-kvm</strong>`, `<strong class="editor-theme-bold editor-theme-code">qemu-utils</strong>`):**<span style="white-space: pre-wrap;"> These are the core components of the Kernel-based Virtual Machine (KVM) hypervisor and QEMU emulator, which provide the virtualization capabilities.</span>
- **Libvirt (`<strong class="editor-theme-bold editor-theme-code">libvirt-daemon-system</strong>`, `<strong class="editor-theme-bold editor-theme-code">libvirt-clients</strong>`):**<span style="white-space: pre-wrap;"> Libvirt is a powerful open-source API, daemon, and management tool for managing virtualization platforms like KVM. </span>`<span class="editor-theme-code">libvirt-daemon-system</span>`<span style="white-space: pre-wrap;"> provides the background service, while </span>`<span class="editor-theme-code">libvirt-clients</span>`<span style="white-space: pre-wrap;"> provides command-line tools to interact with it.</span>
- **Bridge Utilities (`<strong class="editor-theme-bold editor-theme-code">bridge-utils</strong>`):**<span style="white-space: pre-wrap;"> This package provides tools to create and manage network bridges. A network bridge is essential for allowing your virtual machine to access your physical network, often appearing as a separate device on your local network.</span>
- **Virt-Manager (`<strong class="editor-theme-bold editor-theme-code">virt-manager</strong>`):**<span style="white-space: pre-wrap;"> This is a graphical user interface (GUI) application that simplifies the creation, management, and monitoring of virtual machines through Libvirt. It's highly recommended for ease of use.</span>
- **OVMF (`<strong class="editor-theme-bold editor-theme-code">ovmf</strong>`):**<span style="white-space: pre-wrap;"> Open Virtual Machine Firmware provides UEFI firmware for your virtual machines. Windows 11 specifically requires UEFI for installation, making this package crucial for your VM.</span>

To install these packages, open a terminal and run the following command:

```
sudo apt install\
    qemu-kvm\
    qemu-utils\
    libvirt-daemon-system\
    libvirt-clients\
    bridge-utils\
    virt-manager\
    ovmf
```

## 2. Enabling IOMMU Feature

IOMMU (Input/Output Memory Management Unit) is a crucial hardware feature that allows the operating system to assign specific hardware devices directly to virtual machines. Without IOMMU, VFIO passthrough is not possible because the VM cannot directly access the hardware.

### For AMD CPUs:

1. Open the GRUB (Grand Unified Bootloader) configuration file for editing. GRUB is the bootloader that loads your operating system.  
    ```
    sudo nano /etc/default/grub
    ```
2. <span style="white-space: pre-wrap;">Locate the line that starts with </span>`<span class="editor-theme-code">GRUB_CMDLINE_LINUX_DEFAULT</span>`. This line specifies default kernel parameters.
3. <span style="white-space: pre-wrap;">Modify the line to include </span>`<span class="editor-theme-code">amd_iommu=on</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">iommu=pt</span>`.
    - `<span class="editor-theme-code">amd_iommu=on</span>`: This parameter enables the IOMMU feature on AMD processors.
    - `<span class="editor-theme-code">iommu=pt</span>`: This enables "passthrough" mode for IOMMU, which is generally recommended for VFIO as it allows the kernel to optimize IOMMU usage for device passthrough.
    
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt"
    ```

### For Intel CPUs:

1. Open the GRUB configuration file for editing:  
    ```
    sudo nano /etc/default/grub
    ```
2. <span style="white-space: pre-wrap;">Locate the line that starts with </span>`<span class="editor-theme-code">GRUB_CMDLINE_LINUX_DEFAULT</span>`.
3. <span style="white-space: pre-wrap;">Modify the line to include </span>`<span class="editor-theme-code">intel_iommu=on</span>`.  
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on"
    ```

### Apply and Verify GRUB Changes:

1. <span style="white-space: pre-wrap;">Save the changes to the GRUB configuration file. In </span>`<span class="editor-theme-code">nano</span>`<span style="white-space: pre-wrap;">, press </span>`<span class="editor-theme-code">CTRL+x</span>`<span style="white-space: pre-wrap;">, then </span>`<span class="editor-theme-code">y</span>`<span style="white-space: pre-wrap;"> to confirm saving, and then </span>`<span class="editor-theme-code">Enter</span>`<span style="white-space: pre-wrap;"> to confirm the filename.</span>
2. Update GRUB to apply the new configuration. This command rebuilds the GRUB boot menu with your new kernel parameters.  
    ```
    sudo update-grub
    ```
3. **Reboot your system**<span style="white-space: pre-wrap;"> for the GRUB changes to take effect. The IOMMU feature is enabled at boot time.</span>  
    ```
    sudo reboot
    ```
4. After rebooting, verify that IOMMU is enabled by checking the kernel messages.
    - For AMD CPUs, run:  
        ```
        sudo dmesg | grep AMD-Vi
        ```
        
          
        <span style="white-space: pre-wrap;">You should see output indicating that </span>`<span class="editor-theme-code">AMD-Vi</span>`<span style="white-space: pre-wrap;"> (AMD IOMMU) is initialized, similar to </span>`<span class="editor-theme-code">[ 0.000000] AMD-Vi: AMD IOMMUv2 functionality enabled</span>`.
    - For Intel CPUs, run:  
        ```
        sudo dmesg | grep DMAR
        ```
        
          
        <span style="white-space: pre-wrap;">You should see output related to </span>`<span class="editor-theme-code">DMAR</span>`<span style="white-space: pre-wrap;"> (Intel IOMMU), indicating it's enabled, e.g., </span>`<span class="editor-theme-code">[ 0.000000] DMAR: IOMMU enabled</span>`.

## 3. Identification of the Guest GPU and its IOMMU Group

This is a critical step to identify your dedicated graphics card and ensure it's in a proper IOMMU group for passthrough. IOMMU groups are sets of devices that are electrically connected in such a way that they must be passed through together to a VM, or not at all. Ideally, your GPU and its associated audio device should be in their own isolated IOMMU group.

If your motherboard does not support proper IOMMU grouping (meaning devices you want to pass through are grouped with other essential system devices), you might encounter issues. In such cases, advanced solutions like patching your kernel with the ACS override patch might be necessary, but this is an advanced topic and beyond the scope of this basic setup guide.

To determine your devices and their IOMMU grouping, you can use the following bash script:

```
#!/bin/bash
# This script lists all PCI devices organized by their IOMMU group.
# The '999' is an upper limit for IOMMU groups; adjust if you have more.
shopt -s nullglob
for d in /sys/kernel/iommu_groups/{0..999}/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

Run this script in your terminal. The output will list all PCI devices grouped by their IOMMU group. Look for your dedicated GPU (e.g., NVIDIA RTX 3090) and its associated HDMI audio device. These two components should ideally be in the same IOMMU group.

For example, for an RTX 3090, you might see output similar to this (your device IDs and IOMMU group numbers will differ):

```
IOMMU Group 18 0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204] (rev a1)
IOMMU Group 18 0000:01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio [10de:1aef] (rev a1)
```

<span style="white-space: pre-wrap;">In this example, the NVIDIA RTX 3090 (PCI ID </span>`<span class="editor-theme-code">10de:2204</span>`<span style="white-space: pre-wrap;">) and its audio device (PCI ID </span>`<span class="editor-theme-code">10de:1aef</span>`<span style="white-space: pre-wrap;">) are both in </span>`<span class="editor-theme-code">IOMMU Group 18</span>`<span style="white-space: pre-wrap;">. </span>**Make note of the PCI IDs for your graphics card and its audio device.**<span style="white-space: pre-wrap;"> These are the numbers in the format </span>`<span class="editor-theme-code">XXXX:XXXX</span>`<span style="white-space: pre-wrap;"> (e.g., </span>`<span class="editor-theme-code">10de:2204</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">10de:1aef</span>`). You will need these in the next step.

## 5. Reboot the System

**ATTENTION:**<span style="white-space: pre-wrap;"> After the following reboot, the isolated GPU will be ignored by the host OS. This means you will need to use your other GPU (if you have one) or your integrated graphics for your host OS display. If you only have one GPU and are passing it through, your host system will have no graphical output after this step, and you'll need to rely on SSH or a TTY (text-only console) for interaction until the VM is running.</span>

Reboot your system to apply the kernel parameter changes:

```
sudo reboot
```

## 6. Verify the Isolation

<span style="white-space: pre-wrap;">After rebooting, you need to verify that your GPU and its associated audio device are successfully bound to the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver. This confirms that they are ready for passthrough.</span>

1. Open a terminal and run the following command. This command lists all PCI devices in a verbose (detailed) and numeric format.  
    ```
    lspci -nnv
    ```
2. <span style="white-space: pre-wrap;">Scroll through the extensive output or use </span>`<span class="editor-theme-code">grep</span>`<span style="white-space: pre-wrap;"> to filter for your GPU and its audio part using their PCI IDs (e.g., </span>`<span class="editor-theme-code">lspci -nnv | grep -E '10de:2204|10de:1aef'</span>`).
3. <span style="white-space: pre-wrap;">For both your GPU and its audio device, you should find a line that says "Kernel driver in use: </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;">". This confirms that the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver has successfully claimed the devices and they are no longer being used by the host's display drivers.</span>  
    **Example (partial output for an RTX 3090 after successful isolation):**  
    ```
    0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: NVIDIA Corporation Device [10de:147e]
        Flags: bus master, fast devsel, latency 0, IRQ 10
        Memory at f6000000 (32-bit, non-prefetchable) [size=16M]
        Memory at e0000000 (64-bit, prefetchable) [size=256M]
        Memory at f0000000 (64-bit, prefetchable) [size=32M]
        Capabilities: [60] Power Management version 3
        ...
        Kernel driver in use: vfio-pci
        Kernel modules: nouveau, nvidia_drm, nvidia
    
    0000:01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio [10de:1aef] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:147e]
        Flags: bus master, fast devsel, latency 0, IRQ 11
        Memory at f7080000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        ...
        Kernel driver in use: vfio-pci
        Kernel modules: snd_hda_intel
    ```

<span style="white-space: pre-wrap;">If the </span>`<span class="editor-theme-code">Kernel driver in use</span>`<span style="white-space: pre-wrap;"> is </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> for both your GPU and its audio device, you have successfully isolated them for passthrough!</span>

## <span style="white-space: pre-wrap;">7. Using </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> for Dynamic Driver Management (RTX 3090 Example)</span>

<span style="white-space: pre-wrap;">While the previous steps ensure your GPU is bound to </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> at boot, there might be scenarios where you want to temporarily use your RTX 3090 with your host Pop!\_OS (e.g., for native Linux gaming, GPU-accelerated tasks on the host, or troubleshooting). </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> is a utility that allows you to dynamically switch the driver bound to a PCI device without requiring a full system reboot.</span>

**Important Considerations Before Using `<strong class="editor-theme-bold editor-theme-code">driverctl</strong>`:**

- **Graphical Session Must Be Stopped:**<span style="white-space: pre-wrap;"> You </span>**must stop your graphical display manager (GDM for Pop!\_OS)**<span style="white-space: pre-wrap;"> before attempting to switch drivers for your primary GPU. Attempting to switch a GPU's driver while it's actively in use by a running graphical session will lead to system instability, a frozen screen, or a crash.</span>
- **PCI Address:**<span style="white-space: pre-wrap;"> Ensure you use the correct PCI address for your RTX 3090 (e.g., </span>`<span class="editor-theme-code">0000:01:00.0</span>`<span style="white-space: pre-wrap;"> from your </span>`<span class="editor-theme-code">lspci</span>`<span style="white-space: pre-wrap;"> output). This is the full address, not just the vendor/device ID.</span>

**Disabling persistence mode with `<strong class="editor-theme-bold editor-theme-code">nvidia-smi -pm 0</strong>` allows the `<strong class="editor-theme-bold editor-theme-code">nvidia</strong>` kernel module to be unloaded, and driverctl to work as expected**

### <span style="white-space: pre-wrap;">Add Aliases to your </span>`<span class="editor-theme-code">~/.bashrc</span>`

<span style="white-space: pre-wrap;">To make switching drivers easier and more convenient, you can add aliases to your </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file. This allows you to execute a sequence of commands with a single, memorable command.</span>

1. <span style="white-space: pre-wrap;">Open your </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file for editing. This file contains shell configuration and aliases for your user.</span>  
    ```
    nano ~/.bashrc
    ```
2. <span style="white-space: pre-wrap;">Add the following lines to the end of the file. </span>**Carefully replace `<strong class="editor-theme-bold editor-theme-code">0000:01:00.0</strong>` with the actual PCI address of your RTX 3090**<span style="white-space: pre-wrap;"> that you identified in Section 3.</span>  
    ```
    # Alias to set VFIO driver for RTX 3090, preparing it for VM passthrough
    alias set-vfio='\
        sudo systemctl stop gdm &&\
        sudo driverctl --nosave set-override 0000:01:00.0 vfio-pci &&\
        sudo systemctl start gdm'
    
    # Alias to unset VFIO driver and set NVIDIA driver for RTX 3090,
    # allowing the host OS to use it
    alias unset-vfio='\
        sudo systemctl stop gdm &&\
        sudo driverctl --nosave set-override 0000:01:00.0 nvidia &&\
        sudo systemctl start gdm'
    ```
    
      
    Let's break down these commands:
    - `<span class="editor-theme-code">sudo systemctl stop gdm</span>`: This command stops the GNOME Display Manager, which is responsible for your graphical login screen and desktop environment. This is the crucial step to release the GPU from the host's control.
    - `<span class="editor-theme-code">sudo driverctl --nosave set-override 0000:01:00.0 vfio-pci</span>`<span style="white-space: pre-wrap;">: This is the core </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> command. It instructs </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to bind the PCI device at the specified address (</span>`<span class="editor-theme-code">0000:01:00.0</span>`<span style="white-space: pre-wrap;">) to the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver. The </span>`<span class="editor-theme-code">--nosave</span>`<span style="white-space: pre-wrap;"> flag ensures that this change is temporary and will not persist across reboots; the kernel parameters set in Section 4 will still take precedence on next boot.</span>
    - `<span class="editor-theme-code">sudo driverctl --nosave set-override 0000:01:00.0 nvidia</span>`<span style="white-space: pre-wrap;">: This command does the opposite, binding the PCI device back to the </span>`<span class="editor-theme-code">nvidia</span>`<span style="white-space: pre-wrap;"> driver (assuming you have the NVIDIA proprietary drivers installed).</span>
    - `<span class="editor-theme-code">sudo systemctl start gdm</span>`: After the driver change, this command restarts the GNOME Display Manager, bringing your graphical session back up, now using the newly bound driver.
3. <span style="white-space: pre-wrap;">Save and close the </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file (</span>`<span class="editor-theme-code">CTRL+x</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">y</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">Enter</span>`).
4. Apply the changes to your current shell session. This makes the newly defined aliases available immediately without needing to log out and back in.  
    ```
    source ~/.bashrc
    ```

### How to Use the Aliases:

- **To prepare your RTX 3090 for VM passthrough (bind to `<strong class="editor-theme-bold editor-theme-code">vfio-pci</strong>`):**<span style="white-space: pre-wrap;"> When you want to use your GPU with your Windows 11 VM, you need to ensure it's bound to </span>`<span class="editor-theme-code">vfio-pci</span>`.
    1. <span style="white-space: pre-wrap;">Switch to a TTY (text-only console) by pressing </span>`<span class="editor-theme-code">CTRL+ALT+F3</span>`<span style="white-space: pre-wrap;"> (or </span>`<span class="editor-theme-code">F4</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">F5</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">F6</span>`<span style="white-space: pre-wrap;"> -- one of these will usually work).</span>
    2. Log in with your username and password.
    3. Run the alias command:  
        ```
        set-vfio
        ```
    
    After running this command, your graphical session will briefly stop and restart. Your RTX 3090 should now be available for your VM. You can then switch back to your graphical session (`<span class="editor-theme-code">CTRL+ALT+F1</span>`<span style="white-space: pre-wrap;"> or </span>`<span class="editor-theme-code">F2</span>`) and start your VM.
- **To use your RTX 3090 with your host Pop!\_OS (bind to `<strong class="editor-theme-bold editor-theme-code">nvidia</strong>` driver):**<span style="white-space: pre-wrap;"> When you're done with your VM and want to use your GPU for your Linux desktop or native Linux games, you can switch it back.</span>
    1. Ensure your VM is shut down or suspended.
    2. <span style="white-space: pre-wrap;">Switch to a TTY (text-only console) by pressing </span>`<span class="editor-theme-code">CTRL+ALT+F3</span>`.
    3. Log in.
    4. Run the alias command:  
        ```
        unset-vfio
        ```
    
    After running this command, your graphical session will briefly stop and restart. Your RTX 3090 should now be used by your Pop!\_OS system.

<p class="callout warning">**Important:**<span style="white-space: pre-wrap;"> Always ensure you are in a TTY (text-only console) when running these </span>`<span class="editor-theme-code">set-vfio</span>`<span style="white-space: pre-wrap;"> or </span>`<span class="editor-theme-code">unset-vfio</span>`<span style="white-space: pre-wrap;"> commands to avoid issues with your graphical environment. You can switch back to your graphical session using </span>`<span class="editor-theme-code">CTRL+ALT+F1</span>`<span style="white-space: pre-wrap;"> (or </span>`<span class="editor-theme-code">F2</span>`, depending on your setup).My VFIO Passthrough Setup on Pop!\_OS 22.04</p>

This document outlines the steps I took to set up VFIO (Virtual Function I/O) passthrough on my Pop!\_OS 22.04 PC. VFIO allows me to dedicate a physical PCI device, specifically my NVIDIA RTX 3090 graphics card, directly to my Windows 11 virtual machine. This gives me near-native performance in my VM, which is fantastic for gaming and demanding applications.

I'm writing this documentation as if it's for a fresh install, and I'll try to be as thorough as possible, keeping in mind that someone less experienced with VFIO might be following along.

**Please read all instructions carefully before proceeding.**<span style="white-space: pre-wrap;"> VFIO setup can be complex, and I've learned that incorrect steps can lead to system instability or graphical issues.</span>

## References:

- [PCI Passthrough on Ubuntu 20.04 Virtual Machine](https://mathiashueber.com/pci-passthrough-ubuntu-2004-virtual-machine/ "null")
- [VFIO Passthrough Gist](https://gist.github.com/k-amin07/47cb06e4598e0c81f2b42904c6909329 "null")
- [Define a Network Bridge using Ubuntu's / Linux Mint's Network Manager Application](https://www.heiko-sieger.info/define-a-network-bridge-using-ubuntus-linux-mints-network-manager-application/ "null")

## Prerequisites

Before I even start with the software, I need to make sure my system meets a few hardware and BIOS/UEFI configuration requirements:

- **Virtualization Support Enabled in BIOS/UEFI:**
    - Since I have an Intel CPU, I need to go into my motherboard's BIOS/UEFI settings and look for "Intel VT-d" or "Intel Virtualization Technology for Directed I/O" and make sure it's enabled. If I had an AMD CPU, I'd look for "AMD-V" or "SVM Mode." These settings are absolutely crucial for IOMMU functionality, which is the foundation of VFIO passthrough. I'll consult my motherboard's manual if I can't find them.
- **Dedicated GPU for Passthrough:**<span style="white-space: pre-wrap;"> I'm using my NVIDIA RTX 3090 as the dedicated GPU for my virtual machine. It's important that this GPU can ideally be in its own IOMMU group, which I'll verify later.</span>
- **Integrated Graphics (iGPU) for Host:**<span style="white-space: pre-wrap;"> My Pop!\_OS system needs a display output. Since I'm passing through my RTX 3090, it won't provide a display for the host. Luckily, my Intel CPU has integrated graphics (iGPU), which I've enabled in my BIOS/UEFI to drive my Pop!\_OS display. This is a common and convenient setup.</span>
- **Pop!\_OS 22.04 Installation:**<span style="white-space: pre-wrap;"> This guide is specifically for Pop!\_OS 22.04, which is what I'm running. While some concepts might apply to other Linux distributions, command syntax and system configurations might differ.</span>

## 1. Installing the Required Software

First, I need to install all the necessary virtualization software packages.

- **QEMU/KVM (`<strong class="editor-theme-bold editor-theme-code">qemu-kvm</strong>`, `<strong class="editor-theme-bold editor-theme-code">qemu-utils</strong>`):**<span style="white-space: pre-wrap;"> These are the core components of the Kernel-based Virtual Machine (KVM) hypervisor and QEMU emulator. They provide the fundamental virtualization capabilities.</span>
- **Libvirt (`<strong class="editor-theme-bold editor-theme-code">libvirt-daemon-system</strong>`, `<strong class="editor-theme-bold editor-theme-code">libvirt-clients</strong>`):**<span style="white-space: pre-wrap;"> Libvirt is a powerful open-source API, daemon, and management tool for managing virtualization platforms like KVM. </span>`<span class="editor-theme-code">libvirt-daemon-system</span>`<span style="white-space: pre-wrap;"> provides the background service, while </span>`<span class="editor-theme-code">libvirt-clients</span>`<span style="white-space: pre-wrap;"> provides command-line tools to interact with it.</span>
- **Bridge Utilities (`<strong class="editor-theme-bold editor-theme-code">bridge-utils</strong>`):**<span style="white-space: pre-wrap;"> This package provides tools to create and manage network bridges. I'll need a network bridge to allow my virtual machine to access my physical network, making it appear as a separate device on my local network.</span>
- **Virt-Manager (`<strong class="editor-theme-bold editor-theme-code">virt-manager</strong>`):**<span style="white-space: pre-wrap;"> This is a graphical user interface (GUI) application that simplifies the creation, management, and monitoring of virtual machines through Libvirt. I highly recommend it for ease of use.</span>
- **OVMF (`<strong class="editor-theme-bold editor-theme-code">ovmf</strong>`):**<span style="white-space: pre-wrap;"> Open Virtual Machine Firmware provides UEFI firmware for virtual machines. Windows 11 specifically requires UEFI for installation, so this package is crucial for my VM.</span>

To install these packages, I'll open a terminal and run the following command:

```
sudo apt install\
    qemu-kvm\
    qemu-utils\
    libvirt-daemon-system\
    libvirt-clients\
    bridge-utils\
    virt-manager\
    ovmf
```

## 2. Enabling IOMMU Feature

IOMMU (Input/Output Memory Management Unit) is a crucial hardware feature that allows the operating system to assign specific hardware devices directly to virtual machines. Without IOMMU, VFIO passthrough isn't possible because the VM can't directly access the hardware.

### For Intel CPUs (My Setup):

<span style="white-space: pre-wrap;">Since I have an Intel CPU, I need to enable IOMMU by adding the </span>`<span class="editor-theme-code">intel_iommu=on</span>`<span style="white-space: pre-wrap;"> kernel parameter. I have two main ways to do this on Pop!\_OS: using </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> (my preferred method) or directly editing the GRUB configuration if I've switched to it.</span>

#### <span style="white-space: pre-wrap;">Using </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> (Pop!\_OS Default):</span>

1. <span style="white-space: pre-wrap;">I'll use </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> to add the </span>`<span class="editor-theme-code">intel_iommu=on</span>`<span style="white-space: pre-wrap;"> kernel parameter. This tells my kernel to enable the IOMMU feature on my Intel processor.</span>  
    ```
    sudo kernelstub -a "intel_iommu=on"
    ```

#### Using GRUB (If I've switched to GRUB as my bootloader):

1. I would open the GRUB configuration file for editing:  
    ```
    sudo nano /etc/default/grub
    ```
2. <span style="white-space: pre-wrap;">I would locate the line that starts with </span>`<span class="editor-theme-code">GRUB_CMDLINE_LINUX_DEFAULT</span>`. This line specifies default kernel parameters.
3. <span style="white-space: pre-wrap;">I would modify the line to include </span>`<span class="editor-theme-code">intel_iommu=on</span>`.  
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on"
    ```

### For AMD CPUs (If I had one):

<span style="white-space: pre-wrap;">If I had an AMD CPU, I would need to enable IOMMU by adding </span>`<span class="editor-theme-code">amd_iommu=on</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">iommu=pt</span>`<span style="white-space: pre-wrap;"> kernel parameters.</span>

- `<span class="editor-theme-code">amd_iommu=on</span>`: This parameter enables the IOMMU feature on AMD processors.
- `<span class="editor-theme-code">iommu=pt</span>`: This enables "passthrough" mode for IOMMU, which is generally recommended for VFIO as it allows the kernel to optimize IOMMU usage for device passthrough.

#### <span style="white-space: pre-wrap;">Using </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> (Pop!\_OS Default):</span>

1. <span style="white-space: pre-wrap;">I would use </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> to add both </span>`<span class="editor-theme-code">amd_iommu=on</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">iommu=pt</span>`<span style="white-space: pre-wrap;"> kernel parameters.</span>  
    ```
    sudo kernelstub -a "amd_iommu=on iommu=pt"
    ```

#### Using GRUB (If I've switched to GRUB as my bootloader):

1. I would open the GRUB configuration file for editing:  
    ```
    sudo nano /etc/default/grub
    ```
2. <span style="white-space: pre-wrap;">I would locate the line that starts with </span>`<span class="editor-theme-code">GRUB_CMDLINE_LINUX_DEFAULT</span>`.
3. <span style="white-space: pre-wrap;">I would modify the line to include </span>`<span class="editor-theme-code">amd_iommu=on</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">iommu=pt</span>`.  
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="amd_iommu=on iommu=pt"
    ```

### Applying and Verifying Changes:

1. <span style="white-space: pre-wrap;">If I used </span>`<span class="editor-theme-code">kernelstub</span>`, the changes are automatically applied to my boot configuration. If I edited a GRUB file, I would need to update GRUB:  
    ```
    sudo update-grub
    ```
2. **I'll reboot my system**<span style="white-space: pre-wrap;"> for the changes to take effect. The IOMMU feature is enabled at boot time.</span>  
    ```
    sudo reboot
    ```
3. After rebooting, I'll verify that IOMMU is enabled by checking the kernel messages.
    - Since I have an Intel CPU, I'll run:  
        ```
        sudo dmesg | grep DMAR
        ```
        
          
        <span style="white-space: pre-wrap;">I should see output related to </span>`<span class="editor-theme-code">DMAR</span>`<span style="white-space: pre-wrap;"> (Intel IOMMU), indicating it's enabled, e.g., </span>`<span class="editor-theme-code">[ 0.000000] DMAR: IOMMU enabled</span>`.
    - If I had an AMD CPU, I'd run:  
        ```
        sudo dmesg | grep AMD-Vi
        ```
        
          
        <span style="white-space: pre-wrap;">I would expect output indicating that </span>`<span class="editor-theme-code">AMD-Vi</span>`<span style="white-space: pre-wrap;"> (AMD IOMMU) is initialized, similar to </span>`<span class="editor-theme-code">[ 0.000000] AMD-Vi: AMD IOMMUv2 functionality enabled</span>`.

## 3. Identification of My Guest GPU and its IOMMU Group

This is a critical step for me to identify my dedicated graphics card (the RTX 3090) and ensure it's in a proper IOMMU group for passthrough. IOMMU groups are sets of devices that are electrically connected in such a way that they must be passed through together to a VM, or not at all. Ideally, my GPU and its associated audio device should be in their own isolated IOMMU group.

If my motherboard didn't support proper IOMMU grouping (meaning devices I want to pass through were grouped with other essential system devices), I might encounter issues. In such cases, advanced solutions like patching my kernel with the ACS override patch might be necessary, but that's an advanced topic and beyond the scope of this basic setup guide.

To determine my devices and their IOMMU grouping, I'll use the following bash script:

```
#!/bin/bash
# This script lists all PCI devices organized by their IOMMU group.
# The '999' is an upper limit for IOMMU groups; I'll adjust if I have more.
shopt -s nullglob
for d in /sys/kernel/iommu_groups/{0..999}/devices/*; do
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

I'll run this script in my terminal. The output will list all PCI devices grouped by their IOMMU group. I'll look for my dedicated GPU (NVIDIA RTX 3090) and its associated HDMI audio device. These two components should ideally be in the same IOMMU group.

Based on my system's output, here's what I see for my NVIDIA RTX 3090:

```
IOMMU Group 17 0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204] (rev a1)
IOMMU Group 17 0000:01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio Controller [10de:1aef] (rev a1)
```

<span style="white-space: pre-wrap;">In this output, my NVIDIA RTX 3090 (PCI ID </span>`<span class="editor-theme-code">10de:2204</span>`<span style="white-space: pre-wrap;">) and its audio device (PCI ID </span>`<span class="editor-theme-code">10de:1aef</span>`<span style="white-space: pre-wrap;">) are both in </span>`<span class="editor-theme-code">IOMMU Group 17</span>`<span style="white-space: pre-wrap;">. </span>**I'll make note of the PCI IDs for my graphics card and its audio device.**<span style="white-space: pre-wrap;"> These are the numbers in the format </span>`<span class="editor-theme-code">XXXX:XXXX</span>`<span style="white-space: pre-wrap;"> (e.g., </span>`<span class="editor-theme-code">10de:2204</span>`<span style="white-space: pre-wrap;"> and </span>`<span class="editor-theme-code">10de:1aef</span>`). I'll need these in the next step.

## <span style="white-space: pre-wrap;">4. Preparing the Guest GPU for VFIO (Ensuring </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> Module Readiness)</span>

**Important Note:**<span style="white-space: pre-wrap;"> While I'll be using </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> for dynamic GPU switching, the steps in this section are still absolutely necessary. They ensure that the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> kernel module is loaded and that essential kernel parameters (like IOMMU enablement and framebuffer handling) are set at boot. This groundwork allows </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to successfully bind your GPU to </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> when you execute your aliases.</span>

<span style="white-space: pre-wrap;">This step ensures that the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> kernel module </span>**can**<span style="white-space: pre-wrap;"> claim my NVIDIA RTX 3090 and its audio device, making them available for VFIO passthrough. While I'll be using </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to actively switch the GPU's driver for isolation, setting these kernel parameters at boot is a necessary prerequisite to allow the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver to even recognize and bind to these devices.</span>

### <span style="white-space: pre-wrap;">4.1. Loading the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> Module at Boot</span>

<span style="white-space: pre-wrap;">To ensure the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> module is always loaded when my system starts, I'll add it to the </span>`<span class="editor-theme-code">modules-load.d</span>`<span style="white-space: pre-wrap;"> configuration. This makes the module available for </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to bind to my GPU when I need it.</span>

1. <span style="white-space: pre-wrap;">I'll create a new configuration file for </span>`<span class="editor-theme-code">modules-load.d</span>`:  
    ```
    sudo nano /etc/modules-load.d/vfio-pci.conf
    ```
2. Inside this file, I'll add the following line:  
    ```
    vfio-pci
    ```
3. I'll save and close the file (`<span class="editor-theme-code">CTRL+x</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">y</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">Enter</span>`).

### 4.2. Setting Essential Kernel Parameters

<span style="white-space: pre-wrap;">These kernel parameters are still crucial for IOMMU functionality and to prevent the host OS from initializing my dedicated GPU. I'll add these using either </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> or GRUB, depending on my bootloader.</span>

- `<span class="editor-theme-code">kvm.ignore_msrs=1</span>`: This parameter is often necessary for Windows 10 versions higher than 1803 (and Windows 11) to prevent Blue Screens of Death (BSODs) related to Model-Specific Register (MSR) access within the KVM hypervisor.
- `<span class="editor-theme-code">video=efifb:off,vesafb:off</span>`: These parameters disable the kernel's framebuffer drivers for EFI and VESA. This is crucial because my RTX 3090 is in my system's primary graphics slot, and these framebuffer drivers can sometimes interfere with GPU passthrough by trying to initialize the card.

#### <span style="white-space: pre-wrap;">Using </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> (Pop!\_OS Default):</span>

1. <span style="white-space: pre-wrap;">I'll run the </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> command to append these kernel parameters to my current boot entry.</span>  
    ```
    sudo kernelstub -a\
        'intel_iommu=on\
        iommu=pt\
        kvm.ignore_msrs=1\
        video=efifb:off,vesafb:off'
    ```

#### Using GRUB (If I've switched to GRUB as my bootloader):

1. I would edit the GRUB configuration file:  
    ```
    sudo nano /etc/default/grub
    ```
2. <span style="white-space: pre-wrap;">I would update the </span>`<span class="editor-theme-code">GRUB_CMDLINE_LINUX_DEFAULT</span>`<span style="white-space: pre-wrap;"> line to include the necessary parameters.</span>  
    ```
    GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt kvm.ignore_msrs=1 video=efifb:off,vesafb:off"
    ```
3. I would save and close the file (`<span class="editor-theme-code">CTRL+x</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">y</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">Enter</span>`).
4. I would update GRUB:  
    ```
    sudo update-grub
    ```

## 5. Rebooting the System

<span style="white-space: pre-wrap;">I'll reboot my system to apply the kernel parameter changes and ensure the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> module is loaded:</span>

```
sudo reboot
```

## <span style="white-space: pre-wrap;">6. Verifying the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> Driver Binding</span>

<span style="white-space: pre-wrap;">After rebooting, I need to verify that my system is ready for </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to manage my GPU. At this point, my GPU might still be bound to the </span>`<span class="editor-theme-code">nvidia</span>`<span style="white-space: pre-wrap;"> driver (if it was before the reboot) or no driver, but the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> module should be loaded. The actual binding to </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> will happen when I run my </span>`<span class="editor-theme-code">set-vfio</span>`<span style="white-space: pre-wrap;"> alias.</span>

1. I'll open a terminal and run the following command. This command lists all PCI devices in a verbose (detailed) and numeric format.  
    ```
    lspci -nnv
    ```
2. <span style="white-space: pre-wrap;">I'll scroll through the extensive output or use </span>`<span class="editor-theme-code">grep</span>`<span style="white-space: pre-wrap;"> to filter for my GPU and its audio part using their PCI IDs (e.g., </span>`<span class="editor-theme-code">lspci -nnv | grep -E '10de:2204|10de:1aef'</span>`).
3. <span style="white-space: pre-wrap;">For both my GPU and its audio device, I should find a line that indicates a driver is in use (likely </span>`<span class="editor-theme-code">nvidia</span>`<span style="white-space: pre-wrap;"> if it's currently active on the host, or potentially no driver). The important thing here is that the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> module is loaded and the IOMMU groups are correct. The actual </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> binding will be confirmed after using </span>`<span class="editor-theme-code">driverctl</span>`.  
    **Example (partial output for an RTX 3090, before `<strong class="editor-theme-bold editor-theme-code">set-vfio</strong>` is run):**  
    ```
    0000:01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA102 [GeForce RTX 3090] [10de:2204] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: NVIDIA Corporation Device [10de:147e]
        Flags: bus master, fast devsel, latency 0, IRQ 10
        Memory at f6000000 (32-bit, non-prefetchable) [size=16M]
        Memory at e0000000 (64-bit, prefetchable) [size=256M]
        Memory at f0000000 (64-bit, prefetchable) [size=32M]
        Capabilities: [60] Power Management version 3
        ...
        Kernel driver in use: nvidia  # <--- This might still be 'nvidia' or no driver
        Kernel modules: nouveau, nvidia_drm, nvidia
    
    0000:01:00.1 Audio device [0403]: NVIDIA Corporation GA102 High Definition Audio [10de:1aef] (rev a1)
        Subsystem: NVIDIA Corporation Device [10de:147e]
        Flags: bus master, fast devsel, latency 0, IRQ 11
        Memory at f7080000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [60] Power Management version 3
        ...
        Kernel driver in use: snd_hda_intel # <--- This might still be 'snd_hda_intel'
        Kernel modules: snd_hda_intel
    ```

## <span style="white-space: pre-wrap;">7. Dynamic GPU Isolation with </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> (My Preferred Method)</span>

<span style="white-space: pre-wrap;">Now that my GPU is configured to be bound by the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver at boot (as verified in the previous step), I can use </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to dynamically switch its driver between </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> (for VM passthrough) and </span>`<span class="editor-theme-code">nvidia</span>`<span style="white-space: pre-wrap;"> (for host usage). This is my preferred method because it avoids needing a full system reboot every time I want to switch which OS uses my RTX 3090.</span>

**Important Considerations Before Using `<strong class="editor-theme-bold editor-theme-code">driverctl</strong>`:**

- **Graphical Session Must Be Stopped:**<span style="white-space: pre-wrap;"> I </span>**must stop my graphical display manager (GDM for Pop!\_OS)**<span style="white-space: pre-wrap;"> before attempting to switch drivers for my GPU. Attempting to switch a GPU's driver while it's actively in use by a running graphical session will lead to system instability, a frozen screen, or a crash.</span>
- **PCI Address:**<span style="white-space: pre-wrap;"> I need to ensure I use the correct PCI address for my RTX 3090 (e.g., </span>`<span class="editor-theme-code">0000:01:00.0</span>`<span style="white-space: pre-wrap;"> from my </span>`<span class="editor-theme-code">lspci</span>`<span style="white-space: pre-wrap;"> output). This is the full address, not just the vendor/device ID.</span>

### <span style="white-space: pre-wrap;">Adding Aliases to my </span>`<span class="editor-theme-code">~/.bashrc</span>`

<span style="white-space: pre-wrap;">To make switching drivers easier and more convenient, I've added aliases to my </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file. This allows me to execute a sequence of commands with a single, memorable command.</span>

1. <span style="white-space: pre-wrap;">I'll open my </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file for editing. This file contains shell configuration and aliases for my user.</span>  
    ```
    nano ~/.bashrc
    ```
2. <span style="white-space: pre-wrap;">I'll add the following lines to the end of the file. I'll </span>**carefully replace `<strong class="editor-theme-bold editor-theme-code">0000:01:00.0</strong>` with the actual PCI address of my RTX 3090**<span style="white-space: pre-wrap;"> that I identified in Section 3.</span>  
    ```
    # Alias to set VFIO driver for RTX 3090, preparing it for VM passthrough
    alias set-vfio='\
        sudo systemctl stop gdm &&\
        sudo driverctl --nosave set-override 0000:01:00.0 vfio-pci &&\
        sudo systemctl start gdm'
    
    # Alias to unset VFIO driver and set NVIDIA driver for RTX 3090,
    # allowing the host OS to use it
    alias unset-vfio='sudo driverctl --nosave set-override 0000:01:00.0 nvidia'
    ```
    
      
    Let's break down these commands:
    - `<span class="editor-theme-code">sudo systemctl stop gdm</span>`: This command stops the GNOME Display Manager, which is responsible for my graphical login screen and desktop environment. This is the crucial step to release the GPU from the host's control.
    - `<span class="editor-theme-code">sudo driverctl --nosave set-override 0000:01:00.0 vfio-pci</span>`<span style="white-space: pre-wrap;">: This is the core </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> command. It instructs </span>`<span class="editor-theme-code">driverctl</span>`<span style="white-space: pre-wrap;"> to bind the PCI device at the specified address (</span>`<span class="editor-theme-code">0000:01:00.0</span>`<span style="white-space: pre-wrap;">) to the </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> driver. The </span>`<span class="editor-theme-code">--nosave</span>`<span style="white-space: pre-wrap;"> flag ensures that this change is temporary and will not persist across reboots; the kernel parameters set in Section 4 will still take precedence on next boot.</span>
    - `<span class="editor-theme-code">sudo driverctl --nosave set-override 0000:01:00.0 nvidia</span>`<span style="white-space: pre-wrap;">: This command does the opposite, binding the PCI device back to the </span>`<span class="editor-theme-code">nvidia</span>`<span style="white-space: pre-wrap;"> driver (assuming I have the NVIDIA proprietary drivers installed).</span>
    - `<span class="editor-theme-code">sudo systemctl start gdm</span>`: After the driver change, this command restarts the GNOME Display Manager, bringing my graphical session back up, now using the newly bound driver.
3. <span style="white-space: pre-wrap;">I'll save and close the </span>`<span class="editor-theme-code">~/.bashrc</span>`<span style="white-space: pre-wrap;"> file (</span>`<span class="editor-theme-code">CTRL+x</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">y</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">Enter</span>`).
4. I'll apply the changes to my current shell session. This makes the newly defined aliases available immediately without needing to log out and back in.  
    ```
    source ~/.bashrc
    ```

### How to Use the Aliases:

- **To prepare my RTX 3090 for VM passthrough (bind to `<strong class="editor-theme-bold editor-theme-code">vfio-pci</strong>`):**<span style="white-space: pre-wrap;"> When I want to use my GPU with my Windows 11 VM, I need to ensure it's bound to </span>`<span class="editor-theme-code">vfio-pci</span>`.
    1. <span style="white-space: pre-wrap;">I'll switch to a TTY (text-only console) by pressing </span>`<span class="editor-theme-code">CTRL+ALT+F3</span>`<span style="white-space: pre-wrap;"> (or </span>`<span class="editor-theme-code">F4</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">F5</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">F6</span>`<span style="white-space: pre-wrap;"> -- one of these will usually work).</span>
    2. I'll log in with my username and password.
    3. I'll run the alias command:  
        ```
        set-vfio
        ```
    
    After running this command, my graphical session will briefly stop and restart. My RTX 3090 should now be available for my VM. I can then switch back to my graphical session (`<span class="editor-theme-code">CTRL+ALT+F1</span>`<span style="white-space: pre-wrap;"> or </span>`<span class="editor-theme-code">F2</span>`) and start my VM.
- **To use my RTX 3090 with my host Pop!\_OS (bind to `<strong class="editor-theme-bold editor-theme-code">nvidia</strong>` driver):**<span style="white-space: pre-wrap;"> When I'm done with my VM and want to use my GPU for my Linux desktop or native Linux games, I can switch it back.</span>
    1. I'll ensure my VM is shut down or suspended.
    2. <span style="white-space: pre-wrap;">I'll switch to a TTY (text-only console) by pressing </span>`<span class="editor-theme-code">CTRL+ALT+F3</span>`.
    3. I'll log in.
    4. I'll run the alias command:  
        ```
        unset-vfio
        ```
    
    After running this command, my graphical session will briefly stop and restart. My RTX 3090 should now be used by my Pop!\_OS system.

**Important:**<span style="white-space: pre-wrap;"> I'll always ensure I'm in a TTY (text-only console) when running these </span>`<span class="editor-theme-code">set-vfio</span>`<span style="white-space: pre-wrap;"> or </span>`<span class="editor-theme-code">unset-vfio</span>`<span style="white-space: pre-wrap;"> commands to avoid issues with my graphical environment. I can switch back to my graphical session using </span>`<span class="editor-theme-code">CTRL+ALT+F1</span>`<span style="white-space: pre-wrap;"> (or </span>`<span class="editor-theme-code">F2</span>`, depending on my setup).

## 8. Next Steps: Creating the Virtual Machine in Virt-Manager

<span style="white-space: pre-wrap;">You can now start </span>[Virt-Manager](https://virt-manager.org/), you will be presented with this screen:

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt1.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt1.png)

<span style="white-space: pre-wrap;">Click on the screen with a yellow light icon or navigate to </span>`<span class="editor-theme-code">File > New Virtual Machine</span>`<span style="white-space: pre-wrap;">. You will be presented with this screen. Choose </span>`<span class="editor-theme-code">Local install media (ISO image or CDROM)</span>`<span style="white-space: pre-wrap;"> and select </span>`<span class="editor-theme-code">Forward</span>`. You will then see:

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt2.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt2.png)

<span style="white-space: pre-wrap;">Remember those ISOs we installed earlier? We are going to need them now. I store them on my </span>`<span class="editor-theme-code">Desktop/</span>`<span style="white-space: pre-wrap;">. Store them wherever you want and navigate to that installation location by selecting the </span>`<span class="editor-theme-code">browse</span>`<span style="white-space: pre-wrap;"> button. Choose the Win10 ISO. The </span>`<span class="editor-theme-code">Choose the operating system you are installing:</span>`<span style="white-space: pre-wrap;"> section should now autocomplete. Keep the button checked and continue </span>`<span class="editor-theme-code">Forward</span>`. You will be presented with:

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt3.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt3.png)

<span style="white-space: pre-wrap;">You will now configure your Memory (RAM) and CPU settings. In my case, I will designate 16GB of ram for now (16384) and since I have a 12c/24t CPU, I will pass in all 12 cores. Proceed </span>`<span class="editor-theme-code">Forward</span>`<span style="white-space: pre-wrap;"> and be met with:</span>

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt4.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt4.png)

<span style="white-space: pre-wrap;">In this case, you will be creating a custom storage for the Windows install. Select </span>`<span class="editor-theme-code">Enable storage for this virtual machine</span>`<span style="white-space: pre-wrap;"> and then </span>`<span class="editor-theme-code">Select or create custom storage</span>`<span style="white-space: pre-wrap;"> and then navigate to the Storage Volume menu and create a storage volume with any size above 50GB. In this case, you want to create a storage volume with any name you would like and with a format of </span>`<span class="editor-theme-code">qcow2</span>`<span style="white-space: pre-wrap;"> the rest doesn't matter. It can be stored wherever you like. Once created select it and proceed with </span>`<span class="editor-theme-code">Choose Volume</span>`<span style="white-space: pre-wrap;"> and lastly go </span>`<span class="editor-theme-code">Forward</span>`.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt6.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt6.png)

<span style="white-space: pre-wrap;">Lastly, select </span>`<span class="editor-theme-code">Customize configuration before install</span>`<span style="white-space: pre-wrap;"> and then </span>`<span class="editor-theme-code">Finish</span>`.

<span style="white-space: pre-wrap;">A new window will now appear called </span>`<span class="editor-theme-code">$VM_NAME on QEMU/KVM</span>`<span style="white-space: pre-wrap;">. This will allow you to configure more advanced options. You can alter these options through GUI or libvirt XML settings. Please ensure that while on the </span>`<span class="editor-theme-code">Overview</span>`<span style="white-space: pre-wrap;"> page under </span>`<span class="editor-theme-code">Firmware</span>`<span style="white-space: pre-wrap;"> you select </span>`<span class="editor-theme-code">UEFI x86_64: /usr/share/OVMF_CODE_4M.fd</span>`<span style="white-space: pre-wrap;"> and none of the other options.</span>

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt7.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt7.png)

<span style="white-space: pre-wrap;">Next up we will move to the </span>`<span class="editor-theme-code">CPUs</span>`<span style="white-space: pre-wrap;"> tab. Here we are going to change under </span>`<span class="editor-theme-code">Configuration</span>`<span style="white-space: pre-wrap;"> to make sure </span>`<span class="editor-theme-code">Copy host cpu configuration</span>`<span style="white-space: pre-wrap;"> is </span>**NOT**<span style="white-space: pre-wrap;"> checked, change </span>`<span class="editor-theme-code">Model</span>`<span style="white-space: pre-wrap;"> to </span>`<span class="editor-theme-code">host-passthrough</span>`<span style="white-space: pre-wrap;">, and select </span>`<span class="editor-theme-code">Enable available CPU security flaw mitigations</span>`. This is to prevent Spectre/Meltdown vulnerabilities. Don't bother with Topology yet.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt8.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt8.png)

<span style="white-space: pre-wrap;">Next up you can remove a few options from the side. </span>***Remove***<span style="white-space: pre-wrap;"> </span>`<span class="editor-theme-code">Tablet</span>`<span style="white-space: pre-wrap;">, </span>`<span class="editor-theme-code">Channel Spice</span>`<span style="white-space: pre-wrap;">, and </span>`<span class="editor-theme-code">Console</span>`.

<span style="white-space: pre-wrap;">Select </span>`<span class="editor-theme-code">Sata Disk 1</span>`<span style="white-space: pre-wrap;"> &gt; </span>`<span class="editor-theme-code">Advanced options disk bus</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">VirtIO</span>`

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt9.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt9.png)

<span style="white-space: pre-wrap;">Navigate to </span>`<span class="editor-theme-code">NIC</span>`<span style="white-space: pre-wrap;"> and change the Device model to </span>`<span class="editor-theme-code">virtio</span>`

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt10.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt10.png)

`<span class="editor-theme-code">Add Hardware</span>`<span style="white-space: pre-wrap;">, add </span>`<span class="editor-theme-code">Channel Device</span>`<span style="white-space: pre-wrap;">, keep the default name, and choose </span>`<span class="editor-theme-code">Device type - Unix Socket</span>`.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt11.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt11.png)

`<span class="editor-theme-code">Add Hardware</span>`<span style="white-space: pre-wrap;">, Storage, Browse, Browse Local, choose </span>`<span class="editor-theme-code">virtio-win-0.1.185.iso</span>`.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt12.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt12.png)

`<span class="editor-theme-code">Add Hardware</span>`<span style="white-space: pre-wrap;">, PCI host device, (in my case I added), </span>`<span class="editor-theme-code">PCI host device, 0b:00.0 NVIDIA Corporation Device</span>`<span style="white-space: pre-wrap;">, you will need to find your appropriate grouping with your </span>`<span class="editor-theme-code">VFIO-pci driver</span>`. This is my GPU visuals

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt13.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt13.png)

`<span class="editor-theme-code">Add Hardware</span>`<span style="white-space: pre-wrap;">, PCI host device, (in my case I added), </span>`<span class="editor-theme-code">PCI host device, 0b:00.1 NVIDIA Corporation Device</span>`<span style="white-space: pre-wrap;">, you will need to find your appropriate grouping with your </span>`<span class="editor-theme-code">VFIO-pci driver</span>`. This is my GPU audio.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt14.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt14.png)

Now if you want to pass in any USB Host Devices feel free to add whatever ones you want. I did this and changed it later to pass in the entire PCI device.

<span style="white-space: pre-wrap;">Now we are going to have to get our hands dirty with editing the XML file. Go to </span>`<span class="editor-theme-code">Virtual Machine Manager</span>`<span style="white-space: pre-wrap;">, select </span>`<span class="editor-theme-code">edit</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">preferences</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">General</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">Enable XML editing</span>`<span style="white-space: pre-wrap;"> and now you can navigate to the </span>`<span class="editor-theme-code">$VM_NAME on QEMU/KVM</span>`<span style="white-space: pre-wrap;"> window and then select </span>`<span class="editor-theme-code">Overview</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">XML</span>`.

[![](https://github.com/mr2527/pop_OS-win10-KVM-setup/raw/main/Photos/virt15.png)](https://github.com/mr2527/pop_OS-win10-KVM-setup/blob/main/Photos/virt15.png)

#### Nvidia GPU Error 43

<span style="white-space: pre-wrap;">If you are passing in an NVIDIA GPU to the VM you may run into </span>[Error 43](https://passthroughpo.st/apply-error-43-workaround/)<span style="white-space: pre-wrap;">. This is because NVIDIA doesn't enable virtualization on their GeForce cards. The workaround is to have the hypervisor hide its existence. From here navigate to the </span>`<span class="editor-theme-code"><hyperv></span>`<span style="white-space: pre-wrap;"> section and add this.</span>

```
<hyperv>
<features>
    ...
    <hyperv>
        <relaxed state="on"/>
        <vapic state="on"/>
        <spinlocks state="on" retries="8191"/>
        <vendor_id state="on" value="kvm hyperv"/>
    </hyperv>
    ...
</features>
```

<span style="white-space: pre-wrap;">Next directly under the </span>`<span class="editor-theme-code"></hyperv></span>`<span style="white-space: pre-wrap;"> line add</span>

```
<features>
    ...
    <hyperv>
        ...
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    ...
</features>
```

<span style="white-space: pre-wrap;">If QEMU 4.0 is being used with a q35 chipset you will need to add this to the end of </span>`<span class="editor-theme-code"><features></span>`.

```
<features>
    ...
    <ioapic driver="kvm"/>
</features>
```

Error 43 should no longer occur.

## Adding a VBIOS to Your Guest (Windows 10) VM

Since we completed the Windows install we can do some more setup by sending the proper VBIOS for your guest. You don't have to do this but it's highly suggested to get the best performance.

<span style="white-space: pre-wrap;">To find the RTX 3090 VBIOS on Linux, use the </span>`<span class="editor-theme-code">nvidia-smi</span>`<span style="white-space: pre-wrap;"> utility, which is part of the </span>`<span class="editor-theme-code">nvidia-utils</span>`<span style="white-space: pre-wrap;"> package. You can run this command in any console or remotely and parse the output programmatically. The specific command to use is </span>`<span class="editor-theme-code">nvidia-smi -q | grep BIOS</span>`

```bash
jndeverteuil@pop-os:/etc/firmware$ nvidia-smi -q | grep BIOS
    VBIOS Version                         : 94.02.42.00.A9
```

<span style="white-space: pre-wrap;"> </span>***IF YOU NEED TO GET YOUR VBIOS***<span style="white-space: pre-wrap;"> navigate to -&gt; </span>`<span class="editor-theme-code">https://www.techpowerup.com/vgabios/</span>`<span style="white-space: pre-wrap;"> and find your model and correct VBIOS. Once you have this you can download it to a location you will remember, and then:</span>

```
$ sudo mkdir /etc/firmware
$ sudo cp Asus.RTX3090.24576.210308.rom /etc/firmware/
$ sudo chown root:root /etc/firmware/Asus.RTX3090.24576.210308.rom
$ sudo chmod 744 /etc/firmware/Asus.RTX3090.24576.210308.rom
```

I use nano so:

```
$ sudo nano /etc/apparmor.d/abstractions/libvirt-qemu
```

append this to the very end and the SPACES ARE IMPORTANT!!!

```
/etc/firmware/* r,
```

<span style="white-space: pre-wrap;">Once written and you're ready to quit (on Vim) shift + : then type </span>`<span class="editor-theme-code">:wq!</span>`<span style="white-space: pre-wrap;">, this will write and quit. If you write </span>`<span class="editor-theme-code">:Wq!</span>`<span style="white-space: pre-wrap;"> it will not work.</span>

run:

```
$ sudo systemctl restart apparmor.service
```

Now navigate back to your virt-manager and find the PCI device that you added that your GPU is under. Click XML and edit the XML to include this:

```
<hostdev mode="subsystem" type="pci" managed="yes">
  <source>
    <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
  </source>
  <rom bar="on" file="/etc/firmware/Asus.RTX3090.24576.210308.rom"/>
  <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
</hostdev>
```

<span style="white-space: pre-wrap;">The important bit is the </span>`<span class="editor-theme-code"><rom bar="on" file="/etc/firmware/Asus.RTX3090.24576.210308.rom"/></span>`<span style="white-space: pre-wrap;"> This will be different depending on your IOMMU grouping, your graphics card, and your VBIOS so please keep an eye open and add the appropriate content.</span>

## Further Optimization

Begin editing the XML for some changes that will boost performance!

<span style="white-space: pre-wrap;">Navigate to </span>`<span class="editor-theme-code">Overview</span>`<span style="white-space: pre-wrap;"> -&gt; </span>`<span class="editor-theme-code">XML</span>`<span style="white-space: pre-wrap;"> we got some dirty work to do.</span>

Change the very first line to be:

```
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
```

<span style="white-space: pre-wrap;">Do not apply yet. Go all the way to the bottom of the XML and under the </span>`<span class="editor-theme-code"></devices></span>`<span style="white-space: pre-wrap;"> section add:</span>

```
...
  </devices>
  <qemu:commandline>
    <qemu:arg value="-cpu"/>
    <qemu:arg value="host,hv_time,kvm=off,hv_vendor_id=null"/>
  </qemu:commandline>
</domain>
```

Now apply. These new insertions should stay. If you did it incorrectly they will disappear after applying. Double-check to make sure that it sticks. Proceed with the instructions in the next section.

### CPU Pinning

Now we have to learn how to do some CPU pinning ONLY if you have a multithreaded capable CPU. VMs are incapable of distinguishing between physical and logical cores (threads). Virt-manager can see that 24 vCPUs exist and are available but the host has two cores mapped to a single physical core on the physical CPU die. If you want a terminal view of the cores run the command:

```
$ lscpu -e
```

This will provide output that looks like this (for me):

```
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ   MINMHZ      MHZ
  0    0      0    0 0:0:0:0          yes 4900.0000 800.0000  800.155
  1    0      0    0 0:0:0:0          yes 4900.0000 800.0000  800.000
  2    0      0    1 4:4:1:0          yes 4900.0000 800.0000  800.000
  3    0      0    1 4:4:1:0          yes 4900.0000 800.0000  800.000
  4    0      0    2 8:8:2:0          yes 4900.0000 800.0000  800.029
  5    0      0    2 8:8:2:0          yes 4900.0000 800.0000  800.000
  6    0      0    3 12:12:3:0        yes 4900.0000 800.0000  799.941
  7    0      0    3 12:12:3:0        yes 4900.0000 800.0000  800.000
  8    0      0    4 16:16:4:0        yes 5000.0000 800.0000  800.000
  9    0      0    4 16:16:4:0        yes 5000.0000 800.0000  800.422
 10    0      0    5 20:20:5:0        yes 5000.0000 800.0000  799.987
 11    0      0    5 20:20:5:0        yes 5000.0000 800.0000  800.000
 12    0      0    6 24:24:6:0        yes 4900.0000 800.0000  800.044
 13    0      0    6 24:24:6:0        yes 4900.0000 800.0000  800.000
 14    0      0    7 28:28:7:0        yes 4900.0000 800.0000  800.111
 15    0      0    7 28:28:7:0        yes 4900.0000 800.0000  800.000
 16    0      0    8 36:36:9:0        yes 3800.0000 800.0000  800.004
 17    0      0    9 37:37:9:0        yes 3800.0000 800.0000  800.000
 18    0      0   10 38:38:9:0        yes 3800.0000 800.0000  800.000
 19    0      0   11 39:39:9:0        yes 3800.0000 800.0000 1436.923
```

"A matching core id (I.e. "CORE" Column) means that the associated threads (i.e. "CPU" column) run on the same physical core.

<span style="white-space: pre-wrap;">We will now return to editing the XML configuration for the VM. Under the </span>`<span class="editor-theme-code"><currentMemory></span>`<span style="white-space: pre-wrap;"> section add the following:</span>

```
...
<currentMemory unit="KiB">16777216</currentMemory>
<vcpu placement='static'>12</vcpu>
<iothreads>1</iothreads>
<cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <vcpupin vcpu='2' cpuset='2'/>
    <vcpupin vcpu='3' cpuset='3'/>
    <vcpupin vcpu='4' cpuset='4'/>
    <vcpupin vcpu='5' cpuset='5'/>
    <vcpupin vcpu='6' cpuset='6'/>
    <vcpupin vcpu='7' cpuset='7'/>
    <vcpupin vcpu='8' cpuset='8'/>
    <vcpupin vcpu='9' cpuset='9'/>
    <vcpupin vcpu='10' cpuset='10'/>
    <vcpupin vcpu='11' cpuset='11'/>
    <emulatorpin cpuset='16-19'/>
    <iothreadpin iothread='1' cpuset='16-19'/>
</cputune>
```

***REMEMBER THAT YOUR PINNING IS NOT GUARANTEED TO BE ANYTHING LIKE MINE!***<span style="white-space: pre-wrap;"> Please figure out your own pinning and apply the changes.</span>

<span style="white-space: pre-wrap;">Now we will go down to the end of </span>`<span class="editor-theme-code"></features></span>`<span style="white-space: pre-wrap;"> and edit </span>`<span class="editor-theme-code"><cpu></span>`<span style="white-space: pre-wrap;"> by adding the following:</span>

```
...
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="6" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
...
```

This is based on the topology of your CPU and will vary. This is how I set it up for my Intel 12700k.

### Getting Audio on Host

<span style="white-space: pre-wrap;">You can configure libvirt to run QEMU virtual machines under your user by adding the following line to </span>`<span class="editor-theme-code">/etc/libvirt/qemu.conf</span>`:

```
user = "example"
```

#### **Using pulseaudio**

Edit the config of your VM, the audio part shall read:

- disabling the software mixer included in QEMU. This should, in theory, be more efficient and allow for lower latencies since mixing will then take place on your host only
- removes crackling

```
<audio id="1" type="pulseaudio" serverName="/run/user/1000/pulse/native">
      <input mixingEngine="no"/>
      <output mixingEngine="no"/>
    </audio>
```

Edit the permission so qemu has access to pulseaudio

```
sudo usermod -aG audio libvirt-qemu
```

Restart qemu service

```
sudo systemctl restart virtqemud
```

After this you will have sound from your VM via pulseaudio

#### Using Pipewire

N/A without qemu 8.1 which is not available on Pop\_OS 22.04

#### Using Pipewire/Jack

<span style="white-space: pre-wrap;">Scroll down until you find </span>`<span class="editor-theme-code"><audio id="1" type="spice"/></span>`<span style="white-space: pre-wrap;"> line. Replace it with these codes </span>**(change win11 to win10 or anything, doesn't matter)**:

```
<audio id="1" type="jack">
      <input clientName="vm-win11" connectPorts="Built-in Audio Analog Stereo"/>
      <output clientName="vm-win11" connectPorts="Built-in Audio Analog Stereo"/>
    </audio>
```

<span style="white-space: pre-wrap;">under </span>`<span class="editor-theme-code"></devices></span>`, add these:

```
<qemu:commandline>
    <qemu:env name="PIPEWIRE_RUNTIME_DIR" value="/run/user/1000"/>
    <qemu:env name="PIPEWIRE_LATENCY" value="512/48000"/>
  </qemu:commandline>
```

### Installing VirtIO Drivers in Windows 11

After installing Windows 11 in my VM, I'll need to install VirtIO drivers for optimal performance. These drivers provide paravirtualized devices that offer significantly better performance than emulated hardware.

1. **Download VirtIO Drivers:**<span style="white-space: pre-wrap;"> I can usually find the latest VirtIO drivers (often called </span>`<span class="editor-theme-code">virtio-win.iso</span>`) from the Fedora KVM project or similar sources. I'll download this ISO.
2. **Attach VirtIO ISO to VM:**<span style="white-space: pre-wrap;"> In Virt-Manager, with my VM selected, I'll go to "Add Hardware" &gt; "Storage" &gt; "CDROM device". I'll browse to the downloaded </span>`<span class="editor-theme-code">virtio-win.iso</span>`<span style="white-space: pre-wrap;"> and attach it.</span>
3. **Install Drivers:**<span style="white-space: pre-wrap;"> I'll boot into my Windows 11 VM. The </span>`<span class="editor-theme-code">virtio-win.iso</span>`<span style="white-space: pre-wrap;"> will appear as a CD drive. I'll browse its contents and install the necessary drivers (e.g., for storage, network, ballooning). I might need to manually update drivers in Device Manager for devices that show warnings.</span>

## Troubleshooting Common Issues

- **VM won't boot / Black screen in VM:**
    - I'll verify IOMMU is enabled (`<span class="editor-theme-code">sudo dmesg | grep IOMMU</span>`).
    - <span style="white-space: pre-wrap;">I'll ensure my GPU and audio device are bound to </span>`<span class="editor-theme-code">vfio-pci</span>`<span style="white-space: pre-wrap;"> (</span>`<span class="editor-theme-code">lspci -nnv</span>`).
    - <span style="white-space: pre-wrap;">I'll double-check </span>`<span class="editor-theme-code">video=efifb:off,vesafb:off</span>`<span style="white-space: pre-wrap;"> kernel parameters are correctly set, especially since my passthrough GPU is in the primary slot.</span>
    - I'll ensure OVMF is selected as the firmware in Virt-Manager.
    - I'll check if I have a second display connected to my passthrough GPU. The VM's output will appear there.
- **Windows BSODs (Blue Screen of Death) in VM:**
    - <span style="white-space: pre-wrap;">Most commonly, this is due to a missing </span>`<span class="editor-theme-code">kvm.ignore_msrs=1</span>`<span style="white-space: pre-wrap;"> kernel parameter. I'll ensure it's correctly added in my </span>`<span class="editor-theme-code">kernelstub</span>`<span style="white-space: pre-wrap;"> configuration.</span>
- **Poor performance in VM (network, disk):**
    - I'll ensure I have installed the VirtIO drivers inside my Windows VM for storage and network devices.
- **No network connectivity in VM:**
    - I'll verify my network bridge setup on the host.
    - I'll ensure the VM's network adapter is configured to use the correct bridge device and that VirtIO is selected as the device model.
- **Something is hanging onto nvidia drivers**```
    sudo lsof /dev/nvidia*
    ```

## Final XML Config

```xml
<domain xmlns:qemu="http://libvirt.org/schemas/domain/qemu/1.0" type="kvm">
  <name>windows11</name>
  <uuid>22729c84-cb79-4daf-8a60-982d3a367200</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://microsoft.com/win/11"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">33554432</memory>
  <currentMemory unit="KiB">33554432</currentMemory>
  <vcpu placement="static">12</vcpu>
  <iothreads>1</iothreads>
  <cputune>
    <vcpupin vcpu="0" cpuset="0"/>
    <vcpupin vcpu="1" cpuset="1"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="3"/>
    <vcpupin vcpu="4" cpuset="4"/>
    <vcpupin vcpu="5" cpuset="5"/>
    <vcpupin vcpu="6" cpuset="6"/>
    <vcpupin vcpu="7" cpuset="7"/>
    <vcpupin vcpu="8" cpuset="8"/>
    <vcpupin vcpu="9" cpuset="9"/>
    <vcpupin vcpu="10" cpuset="10"/>
    <vcpupin vcpu="11" cpuset="11"/>
    <emulatorpin cpuset="16-19"/>
    <iothreadpin iothread="1" cpuset="16-19"/>
  </cputune>
  <os>
    <type arch="x86_64" machine="pc-q35-6.2">hvm</type>
    <loader readonly="yes" type="pflash">/usr/share/OVMF/OVMF_CODE_4M.fd</loader>
    <nvram>/var/lib/libvirt/qemu/nvram/windows11_VARS.fd</nvram>
  </os>
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vendor_id state="on" value="kvm hyperv"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
    <ioapic driver="kvm"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="6" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
  <clock offset="localtime">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="block" device="disk">
      <driver name="qemu" type="raw" cache="none" io="native" discard="unmap"/>
      <source dev="/dev/disk/by-id/nvme-WD_BLACK_SN850X_1000GB_23436D805053"/>
      <target dev="sda" bus="sata"/>
      <boot order="1"/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="9" port="0x18"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="10" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="10" port="0x19"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x1"/>
    </controller>
    <controller type="pci" index="11" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="11" port="0x1a"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x2"/>
    </controller>
    <controller type="pci" index="12" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="12" port="0x1b"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x3"/>
    </controller>
    <controller type="pci" index="13" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="13" port="0x1c"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x4"/>
    </controller>
    <controller type="pci" index="14" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="14" port="0x1d"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x5"/>
    </controller>
    <controller type="pci" index="15" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="15" port="0x1e"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x6"/>
    </controller>
    <controller type="pci" index="16" model="pcie-to-pci-bridge">
      <model name="pcie-pci-bridge"/>
      <address type="pci" domain="0x0000" bus="0x08" slot="0x00" function="0x0"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <interface type="bridge">
      <mac address="52:54:00:2c:c3:54"/>
      <source bridge="virbr0"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="1"/>
    </channel>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <tpm model="tpm-crb">
      <backend type="emulator" version="2.0"/>
    </tpm>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
      <image compression="off"/>
    </graphics>
    <sound model="ich9">
      <codec type="micro"/>
      <audio id="1"/>
      <address type="pci" domain="0x0000" bus="0x0b" slot="0x00" function="0x0"/>
    </sound>
    <audio id="1" type="pulseaudio" serverName="/run/user/1000/pulse/native">
      <input mixingEngine="no"/>
      <output mixingEngine="no"/>
    </audio>
    <video>
      <model type="virtio" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
      </source>
      <rom bar="on" file="/etc/firmware/Asus.RTX3090.24576.210308.rom"/>
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </hostdev>
    <hostdev mode="subsystem" type="pci" managed="yes">
      <source>
        <address domain="0x0000" bus="0x01" slot="0x00" function="0x1"/>
      </source>
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </hostdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="2"/>
    </redirdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="3"/>
    </redirdev>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </memballoon>
  </devices>
  <qemu:commandline>
    <qemu:arg value="-cpu"/>
    <qemu:arg value="host,hv_time,kvm=off,hv_vendor_id=null"/>
  </qemu:commandline>
</domain>
```