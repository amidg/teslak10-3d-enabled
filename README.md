# teslak10-3d-enabled
My custom Nvidia Tesla K10 vBIOS to enable full 3d acceleration in CADs and games (DirectX, OpenGL and Vulkan). This solution is a great fit for someone looking to build a budget home server with full support for virtualization of remote workloads or gaming. Repository contains vBIOS for GPU#1 and GPU#2 as well as ready-to-go nvflash tool downloaded from TechPowerUp.

This solution is designed to work under open-source hypervisor Proxmox 6.3 with direct PCIe Passthrough to the Virtual Machine. 

Initial inspiration came from the user krutav from eevblog.com:
https://www.eevblog.com/forum/general-computing/hacking-nvidia-cards-into-their-professional-counterparts/msg3533212/#msg3533212

krutav's method allows to use method called "PCIe ID spoofing" that trick PCIe passthrough device to appear as another one what allows to install drivers and partially operate as a target GPU without doing a hardware mod on the vBIOS SPI. 

Project goal is to convert Tesla K10 (dual GK104 card) into a dual Quadro K5000 card, where each K5000 is passed to a different VM under Proxmox PCIe Passthrough.

## 0. GPU Differencies.

What do we know?
PCIe ID change using vBIOS SPI mod enables some of the features, but not all of them... 660Ti or some other cards converted to Grid K2 do not have all features of the Grid K2.
PCIe ID spoofing using purely Proxmox enables some of the features, but again, not all of them... Probably, KVM limitation, probably something else. 3D acceleration cannot be fully enabled by just a trick on the software side, thus vBIOS modification is required.

GPU Model | CUDA? | 3D Acceleration | NVENC | Video Out? | KVM?
------------ | ------------- | ------------- | ------------- | ------------- | -------------
Quadro K5000 | YES | YES | YES | YES | YES
Tesla K10 | YES | NO  | YES | NO | YES
Grid K2 | NO  | YES | NO  | NO | NO

Goal: bring 3D acceleration while keeping CUDA and NVENC working.

Note: EEVBlog.com and YouTube-channel CraftComputing have developed a hardware mod to convert Tesla K10 to Grid K2 because PCBs are 99.99% identical and require only a minor change to enable the 3D acceleration, however it works only under supported hypervisors with vGPU on Kepler cards: VMware or ESXI.
