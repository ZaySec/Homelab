>[!hint] Redacted :)


>[!tldr] This is a `Opnsense` firewall setup for a new `Proxmox` ( 9.2.20 ) machine.
>Prerequisites: `2 Network Interface Cards (NICs) Required!`

---
# Guide 

1. `Download Opnsense`
	1. Download the ISO file from [Opnsense](https://opnsense.org/download/)
		- **Note:** `Image type` has to be `dvd`
	2. I had to download it and unzip the `bzip2` file manually. Then upload the `ISO` image
	
2. `Create a Virtual Machine (VM) in Proxmox`
	1. **General**
		1. Name: `Opnsense`
		2. VM ID: `100`
		3. Select these **Advanced Options**
			1. `Start at Boot` -> Enabled
			2. "`Start/Shutdown order:`" -> `1` 
				- -> This sets our opnsense vm to be the first vm that starts
			3. "`Startup delay:`" -> `15`
				- -> I like to have a delay so the system can start before we start more services
	2. **OS**
		1. `Storage:` `local`
			- Select the storage device where ISOs are stored 
		2. `ISO image:` *Opnsense image*
		3. `Type:` `Other`
	3. **System**
		1. Optional For PCI Passthrough:
			1. `Machine:` -> `q35`
			2. `BIOS:` -> `OVMF (UEFI)`
			3. `EFI-Storage` -> `local-lvm`
			4. `Pre-Enroll Keys:` -> Disable
			5. `Qemu Agent` -> Enable
	4. **Disks**
		1. `Bus/Device:` -> `SCSI`
		2. `Disk Size (GiB):` -> `16`
		3. `Discard` -> Enable
	5. **CPU**
		1. `Cores:` -> `4`
		2. `Extra CPU Flags:` -> "`aes`" -> On
	6. **Memory**
		1. `Memory (MiB):` -> `8192` 
		2. Disable `Balooning Device`
	7. **Network**
		1. `Bridge:` -> `vmbr0` (NIC)
		2. `Model:` -> `VirtIO (paravirtualized)`
	8. **Confirm** 
		1. `Start after created` -> Disable
	
3. `Add Second Network Device`
	1. Open `Opnsense` vm **Hardware** -> **Add** -> **Network Device**
		1. `Bridge:` -> `vmbr1`
		2. `Model:` -> `VirtIO (paravirtualized)`
	
4. `Finish Opnsense Installation`
	1. In the `Opnsense` VM 
		1. Click **Start** 
		2. Enter into **Console**
	2. Now watch the console display its info until `Login:` pops up
		1. Login:Password -> `installer`:`opnsense`
	3. Next it will ask general config settings
		1. I use american keyboard (default)
		2. Choose `UFS` over `ZFS` (default)
		3. Choose the `16GB` Storage -> Yes
	4. Make sure to change the password and write it down!
	5. Lastly, `Reboot`!!!
	
5. `Prepare Opnsense LAN Network`
	1. In the `Opnsense` VM 
		1. Click **Start** 
		2. Enter into **Console**
		3. Login to `Opnsense`
	2. Select `2` to **Set Interface IP Address**
		1. Select `1` to first configure **LAN (vtnet0)**
		2. "**Configure IPv4 address LAN Interface via DHCP**" -> `N`
			1. I will be setting it to `10.0.0.100` since the VMID is 100 and it feels more organized!
		3. For "Subnet Masks" -> `24`
		4. Press **Enter** for "For a LAN, press enter"
		5. "Configure IPv6 Address via..." not using IPv6 -> `N`
		6. "Configure IPv6 Address..." not using IPv6 -> `N`
		7. "Enter IPv6" -> `N`
		8. "Do you want to enable DHCP on LAN?" -> `Y`
			1. "Enter the start Range" -> `10.0.0.4` (I have 3 devices I want as the start)
			2. "Enter the end Range" -> `10.0.0.99` 
		9. The rest are optional for you
	
6. `Configure Proxmox for Opnsense`
	1. In your Proxmox node go to **System** -> **Network**  
		1. `IPv4/CIDR:` -> `10.0.0.1/24`
		2. `Gateway (IPv4)` -> `10.0.0.100`
	- YOU WILL LOSE ACCESS TO THE GUI !
	
7. `Connect Computer`
	1. Connect your computer to the ethernet port (`NIC`) which the router was previously using 
	2. Reboot your computer networking service 
	3. In Proxmox Physical Terminal: `reboot now`
	
8. `Opnsense Gui Configure`
	1. From your computer you can now access `Opnsense Gui`
		1. `http://10.0.0.100`
	2. Default Login is: `root`:`opnsense`
	3. The `System: Configuration: Wizard` will now start
		1. **General Information**
			1. `DNS` -> `1.1.1.1` (cloudflare for now)
		2. **Network `[WAN]`**
			1. `Type` -> `DHCP`
			2. `IP Address` -> Make sure its empty
			3. `Block RFC1918 Private Networks` -> Disable
			4. `Block bogon networks` -> Disable
		3. **Network `[LAN]`**
			1. We already setup earlier
		4. **Deployment Type**
			1. `Optimize for Multiwan` -> Disable

9. `Finishing Touches for Opnsense`
	1. **System -> Firmware -> Updates** 
		1. Then update!
	2. **Services -> Unbound -> DNS -> General**
		1. `Enable Unbound` -> Enable
		2. `Listen Port` -> `0`


---
# Sources

1. Blog Post: https://proxmoxr.com/blog/proxmox-opnsense-setup
2. YouTube Tutorial: https://www.youtube.com/watch?v=lg1bw1S5zCg