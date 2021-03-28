# teslak10-3d-enabled
My custom Nvidia Tesla K10 vBIOS to enable full 3d acceleration in CADs and games (DirectX, OpenGL and Vulkan). This solution is a great fit for someone looking to build a budget home server with full support for virtualization of remote workloads or gaming. Repository contains vBIOS for GPU#1 and GPU#2 as well as ready-to-go nvflash tool downloaded from TechPowerUp.

This solution is designed to work under open-source hypervisor Proxmox 6.3 with direct PCIe Passthrough to the Virtual Machine. 

Initial inspiration came from the user krutav from eevblog.com:
https://www.eevblog.com/forum/general-computing/hacking-nvidia-cards-into-their-professional-counterparts/msg3533212/#msg3533212

krutav's method allows to use method called "PCIe ID spoofing" that trick PCIe passthrough device to appear as another one what allows to install drivers and partially operate as a target GPU without doing a hardware mod on the vBIOS SPI. 

Project goal is to convert Tesla K10 (dual GK104 card) into a dual Quadro K5000 card, where each K5000 is passed to a different VM under Proxmox PCIe Passthrough.

## 0. GPU Differencies.
First Header | Second Header
------------ | -------------
Content from cell 1 | Content from cell 2
Content in the first column | Content in the second column
