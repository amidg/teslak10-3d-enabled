# teslak10-3d-enabled
My custom Nvidia Tesla K10 vBIOS to enable full 3d acceleration in CADs and games (DirectX, OpenGL and Vulkan). This solution is a great fit for someone looking to build a budget home server with full support for virtualization of remote workloads or gaming. Repository contains vBIOS for GPU#1 and GPU#2 as well as ready-to-go nvflash tool downloaded from TechPowerUp.

This solution is designed to work under open-source hypervisor Proxmox 6.3 with direct PCIe Passthrough to the Virtual Machine. 

Initial inspiration came from the user krutav from eevblog.com:
https://www.eevblog.com/forum/general-computing/hacking-nvidia-cards-into-their-professional-counterparts/msg3533212/#msg3533212

krutav's method allows to use method called "PCIe ID spoofing" that trick PCIe passthrough device to appear as another one what allows to install drivers and partially operate as a target GPU without doing a hardware mod on the vBIOS SPI. 

Project goal is to convert Tesla K10 (dual GK104 card) into a dual Quadro K5000 card, where each K5000 is passed to a different VM under Proxmox PCIe Passthrough.

## 0. GPU Differencies.

GPU Model | CUDA? | 3D Acceleration | NVENC | Video Out? | KVM?
------------ | ------------- | ------------- | ------------- | ------------- | -------------
Quadro K5000 | YES | YES | YES | YES | YES
Tesla K10 | YES | NO  | YES | NO | YES
Grid K2 | NO  | YES | NO  | NO | NO

Note: EEVBlog.com and YouTube-channel CraftComputing have developed a hardware mod to convert Tesla K10 to Grid K2 because PCBs are 99.99% identical and require only a minor change to enable the 3D acceleration, however it works only under supported hypervisors with vGPU on Kepler cards: VMware or ESXI.
https://www.youtube.com/watch?v=8Qm9IbSHkus&t=2s

Additionally, Grid K2 or any other vGPU cannot be used in the same server as other nvidia cards because it requires a host kernel module and only ONE nvidia driver can be installed at the same time, thus I cannot have my Plex Media Server with Quadro K2000 working at the same time as vGPU on the same machine. 

Goal: bring 3D acceleration while keeping CUDA and NVENC working under Proxmox PCIe Passthrough.

## 1. Problem Statement:
What do we know?
* PCIe ID change using vBIOS SPI mod enables some of the features, but not all of them... 660Ti or some other cards converted to Grid K2 do not have all features of the Grid K2.
* PCIe ID spoofing using purely Proxmox enables some of the features, but again, not all of them... Probably, KVM limitation, probably something else. 3D acceleration cannot be fully enabled by just a trick on the software side.

So... neither hardware mod nor software trick allow to enable 100% of the card functionality. 
Proof:
* GTX660ti/GTX680 -> Grid K2 via vBIOS SPI mod = no vGPU support.
* GTX680 -> Grid K2 SPI mod + Grid K2 vBIOS = vGPU works
* Tesla K10 -> vBIOS SPI + Grid K2 vBIOS = vGPU works (99.99% identical cards)
* Tesla K10 -> Quadro K5000 via softstrap = Vulkan 3D acceleration works, however no DirectX support.

Where does the problem come from?
If you do some basic googling in order to "sail the sever seas" and find a Kepler-based schematics (e.g. GTX770 that is also GK104 based), you will notice that schematic ROM page contains bunch of GPU die resistor straps that enable/disable certain features. (e.g. video out ports or memory model/amount), so hardware reconfiguration of the Tesla K10 to Quadro K5000 cannot be easily done... and, probably, it is not needed, because they share same amount of the ECC memory and most of the features excluding 3D acceleration. Additionally, they share same driver. 

My assumption that missing functionality comes from the vBIOS. 

## 2. How to enable 3D acceleration.
1. Download Quadro K5000 vBIOS from the TechPowerUp: https://www.techpowerup.com/vgabios/129867/nvidia-quadrok5000-4096-120817
2. Open it using Hex editor (e.g. HxD)
3. Find the following lines:
Subsystem ID:
```
00000450: E9 9E 2B 00 DE 10 65 09 FF FF FF 7F 00 00 00 00
```
PCIe Device ID:
```
00000590: 50 43 49 52 DE 10 BA 11 00 00 18 00 00 00 00 03
```
Line 00000450 < DE 10 65 09 > is responsible for the subsystem ID of the Quadro K5000. 
Line 00000590 < DE 10 BA 11 > is responsible for the PCIe Device ID of the Quadrdo K5000.

Replace the line above with IDs of the Tesla K10 in according to the table below. IMPORTANT: values in Hex file are switched, so instead of 10DE you get DE10 and instead of 11BA you get BA11.

GPU Model | PCIe ID | Subsystem ID
------------ | ------------- | -------------
Quadro K5000 | 10DE 11BA | 10DE 0965
K10 GPU #1 (video side) | 10DE 118F | 10DE 0970
K10 GPU #2 (power side) | 10DE 118F | 10DE 097F

As a result you should get the following lines for the GPU #1:
Subsystem ID:
```
00000450: E9 9E 2B 00 DE 10 70 09 FF FF FF 7F 00 00 00 00
```
PCIe Device ID:
```
00000590: 50 43 49 52 DE 10 8F 11 00 00 18 00 00 00 00 03
```

GPU #2:
Subsystem ID:
```
00000450: E9 9E 2B 00 DE 10 7F 09 FF FF FF 7F 00 00 00 00
```
PCIe Device ID:
```
00000590: 50 43 49 52 DE 10 8F 11 00 00 18 00 00 00 00 03
```

Save your vBIOS and note what vBIOS is GPU #1 and what vBIOS is GPU #2.

4. Open KeplerBiosTweaker:
https://www.techpowerup.com/download/kepler-bios-tweaker/

Open your vBIOS #1 and save it again in order to verify the checksum. Do the same for vBIOS #2.

vBIOS without fixed checksum will brick your card!

5. Using nvflash with mismatched board IDs flash vBIOS #1 to GPU #1 (index=0) and vBIOS #2 to GPU #2 (index=1) using the follwing command:
```
nvflash.exe --index=0 -6 newBIOS1.rom
```
nvflash.exe must be run through Windows CMD using Admin rights. Follow nvflash command prompt and then reboot your PC. 

6. Install your new Tesla K10 into your Proxmox server.
Follow this guide to enable VFIO-PCI driver on both Tesla K10 GPUs:
https://www.reddit.com/r/homelab/comments/b5xpua/the_ultimate_beginners_guide_to_gpu_passthrough/

7. Attach your new GPU under a Windows/Linux VM using the following command in VMID.conf of the Proxmox VM.
VMID.conf is located in the path below, where VMID is a corresponding number starting with 100:
```
/etc/pve/qemu-server/vmid.conf
```

Open xxx.conf using your favorite terminal text editor (nano or vi) and enter the following:
```
args: -cpu 'host,hv_time,kvm=off' -device 'vfio-pci,host=0X:00.0,id=hostpci0.0,bus=ich9-pcie-port-1,addr=0x0.0,multifunction=on,romfile=quadrok5000.rom,x-pci-vendor-id=0x10de,x-pci-device-id=0x11BA,x-pci-sub-vendor-id=0x10de,x-pci-sub-device-id=0x0965'
```

Note #1: replace 0X:00.0 with your device number in PCI tree. Run lspci -vvv command to find out.
Note #2: quadrok5000.rom is a real k5000 vbios with no modification. Place it to your Proxomox host directory below:
```
/usr/share/kvm
```

8. Proceed to your VM (Windows or Linux) and install propriatory nvidia driver using either default .exe (Windows) / .run (Linux) or Quadro Experience (Windows only). You must install Quadro K5000 driver from Nvidia's website or through your Linux distro dependencies. Website is preffered.

## 3. Experiment Results
Tesla K10 can be identified by the drivers in Quadro Experience or games/benchmarks. E.g. Vulkan in Doom (2016) will indicate it as Tesla K10. MSI afterburner also see it as Tesla K10, however for whatever reason Windows itself identifies device as Quadro K5000. My assumption that games/benchmarks refer to the drivers and they ignor PCIe ID spoofing on the Windows side, thus installation of Quadro K5000 drivers becomes possible on Tesla K10.

However, none of this matters, because K5000 vBIOS on Tesla K10 tells the card to enable 3D acceleration, so card's name does not matter anymore. You can now flex that your Doom runs on the card with no 3D acceleration. Testing results are the following:

Test | Works?
------------ | -------------
DirectX (< 12) | yes 
Vulkan | yes
OpenGL | yes
NVENC | yes (incl. Parsec)
CUDA | yes (cap. 3.5)
CAD | yes (tested SW2020)
Render | yes (tested Blender, SW2020)
