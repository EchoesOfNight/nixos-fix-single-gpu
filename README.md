# nixos-fix-single-gpu
this is a quickfix for nixos using a single gpu to passthrough based off this tutorial by blandmanstudios   
https://youtu.be/eTWf5D092VY?si=1O2EXiYISSR45V4N

Step 1# - Follow this tutorial by blandmanstudios   
Single GPU Passthrough Tutorial - KVM/VFIO
https://youtu.be/eTWf5D092VY?si=1O2EXiYISSR45V4N

Step 2# - NixOS modifications   
a. install packages
```
kvm 
libvirt
qemu
virt-manager # (optional for GUI)
```
b. enable libvirt  
```
virtualisation.libvirtd = {
  enable = true;
  onBoot = "ignore";
  onShutdown = "shutdown";
  qemuOvmf = true;
  qemuRunAsRoot = true; #option has no impact 
  #quickfix run hooks manually
  #do not uncomment, doesn't work 
  #hooks.qemu = {   
  #win10 = "/etc/nixos/";
   #};
};
``` 

c. enable qemu  
```
 #Enables VM connection
  programs.dconf.profiles.user.databases = [
    { lockAll = true;

      settings = {
        "org/virt-manager/virt-manager/connections" = {
          autoconnect = [ "qemu:///system" ];
          uris = [ "qemu:///system" ];
        };
      };
    }
  ];
```
d. enable iommu on boot (note: also turn on intel/amd iommu in BIOS advanced cpu settings)  
use `lspci -nnk` to find vifo-pci.ids  
use `"intel_iommu = on"` for intel CPUs 
```
    # Sets the kernel parameters, equivalent to editing /etc/sysconfig/grub 
   boot.kernelParams = [
    "amd_iommu=on"
    "iommu=pt"
    #Do not use these example vfio-pci ids, they are user-specific 
    "vfio-pci.ids=1a2b:3c4d,5e6f:7g8h"
      ];

```
-after rebuiliding use `journalctl | grep -i iommu` to check for iommu, output should look like...
```
Oct 14 15:02:33 nixos kernel: pci 0000:00:01.0: Adding to iommu group 0
Oct 14 15:02:33 nixos kernel: pci 0000:00:01.1: Adding to iommu group 1
Oct 14 15:02:33 nixos kernel: pci 0000:00:01.2: Adding to iommu group 2
Oct 14 15:02:33 nixos kernel: pci 0000:00:02.0: Adding to iommu group 3
Oct 14 15:02:33 nixos kernel: pci 0000:00:02.1: Adding to iommu group 4
etc...
```

e. WARNING: other essential configurations  
```
 services.openssh.enable = true; #enables sshd for nixos
 networking.firewall.allowedTCPPorts = [ 5900 5901 ];  # Required to open VNC ports to insall drivers
```
run in console to start network by default
```
sudo virsh net-autostart default
```

-make sure to add card as a pcie device to virt-manager, before continuing 

Step #3 - NixOS Hooks  
a. It is difficult to enable hooks, since NixOS has immutable root directories, ie /var/ or /etc/. Therefore hooks in /var/lib/libvirt/qemu/qemu.d/hooks/win10 or /etc/libvirt/qemu.d/hooks/win10 will not work (to my understanding).   
Because of this, ssh is used to manually run hooks. However, this can be done easily through termux, as NixOS' sshd server is always running. Note that these hooks are modified from https://github.com/joeknock90/Single-GPU-Passthrough   
Make sure to `sudo chmod x+` all scripts.   
~/run.sh
```
#!/run/current-system/sw/bin/bash
set -x 

# To fully remove and prevent hanging run script twice with & 
sudo bash ./start.sh &

# Wait 10 seconds
sleep 10
sudo bash ./start.sh

# Wait 5 seconds, replace win10 with your VM name
sleep 5
sudo virsh start win10
```
If sudo virsh start win10 cannot find the VM, run `sudo virsh define /var/lib/libvirt/qemu/win10.xml` or whatever is your xml path        
~/start.sh  
```              
#!/run/current-system/sw/bin/bash
set -x

# Stop display manager
systemctl stop display-manager.service
killall gdm-x-session 
sudo rmmod nvidia_drm 
sudo rmmod nvidia_uvm 
sudo rmmod nvidia_modeset
sudo rmmod nvidia 

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Unbind EFI-Framebuffer (ignore error if device doesn't exist)
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind >

# Avoid a race condition by waiting 2 seconds
sleep 2

# Unbind the GPU from display driver with a timeout or in the background
virsh nodedev-detach pci_0000_01_00_0 
virsh nodedev-detach pci_0000_01_00_1 

# Load VFIO Kernel Module  
modprobe vfio-pci
```
Find the correct PCI IDS of your GPU with `lspci -nnk`   
~/end.sh
```
#!/run/current-system/sw/bin/bash

# note VM must be shutdown before this is ran

# Step 1: Unload vfio-pci to release the GPU from passthrough
sudo modprobe -r vfio-pci 

# Step 2: Re-Bind GPU to Nvidia Driver using virsh
virsh nodedev-reattach pci_0000_01_00_1
virsh nodedev-reattach pci_0000_01_00_0

# Step 3: Reload Nvidia modules
sudo modprobe -r nvidia_drm
sudo modprobe -r nvidia_modeset
sudo modprobe -r nvidia_uvm
sudo modprobe -r nvidia

sudo modprobe nvidia
sudo modprobe nvidia_modeset
sudo modprobe nvidia_uvm
sudo modprobe nvidia_drm

# Step 4: Rebind VT consoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
# Uncomment this line if you have more VT consoles to bind
#echo 1 > /sys/class/vtconsole/vtcon1/bind
```
b. To run, find IP address using `ip addr` and start an SSHD daemon `sudo systemctl start sshd` . Then connect on another machine on linux or termux (android)/PuTTy (windows), using the command `ssh <username>@<ipAddress> `   
Then use the `sudo bash ~/run.sh` script to start the machine, and after turning off the VM, run `sudo bash ~/end.sh` to enter the host environment again.    

To install on your first time you must use a VNC client, like pkgs.tigervnc or androidVNC and then install the correct nvidia drivers https://www.nvidia.com/en-us/drivers/

Issues:   
Hooks are not automatic and need to be cleaned up   
Second monitor will not work (display breaks)   





















