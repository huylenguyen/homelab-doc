## **Description**
This repository contains the documentation for setting up my home server, with all the steps required to install Proxmox and set up different services on VMs/Containers.

For the remainder of this document, the home server machine will be referred to as 'host'.

## **Hardware**
- **Case**: Fractal Design Node 804
- **CPU**: Intel i5-12600k
- **Motherboard**: MSI MPG Z790i Edge Wifi
- **GPU**: Zotac AMP GeForce GTX 1060 6GB
- **RAM**: Corsair Vengeance Black 32GB 6000MHz DDR5
- **SSD**: 2x WD Blue SN570 500GB
- **HDD**: 1x Seagate EXOS 16TB
- **PSU**: be quiet! Pure Power 550W 80+ Gold

## **Operating System**
The OS is **Proxmox 7.3**. At the time of the build (03/2023), the official Proxmox installer did not work with my hardware configuration. In particular, I ran into errors:
- [Xorg crashes before showing EULA](https://forum.proxmox.com/threads/generic-solution-when-install-gets-framebuffer-mode-fails.111577/)
- [Bootloader does not install](https://forum.proxmox.com/threads/proxmox-7-0-failed-to-prepare-efi.95466/)

While the Xorg problem was solved by following the forum post, the bootloader problem persisted. In the end, I opted to [install Proxmox on top of Debian 11 Bullseye](https://pve.proxmox.com/wiki/Install_Proxmox_VE_on_Debian_11_Bullseye). The only special step is I extracted the host's IP address from my router configuration and set that as the static IP. 

However, note that installing Proxmox this way did not seem to have configured the initial logical volumes correctly - I only had a root and swap partition set up, and the data partition was missing from LVM. Not a big problem, since I have an additional SSD to run the VMs on.

## **NAS**
The steps to install **OpenMediaVault** as a VM are as follows:

1. First download the [OpenMediaVault](https://www.openmediavault.org/?page_id=77) ISO, either directly to the host ISO storage, or to a different computer and upload to the host via Proxmox GUI.
2. Install OpenMediaVault to a VM. The configurations I went with are 2 cores, 4095MB RAM. Before confirming the installation, make sure the VM does not directly start after installing.
3. There are two methods to pass HDDs directly to OpenMediaVault from Proxmox. If you have a PCIe controller for your drives, you'd simply pass this controller through. I do not (yet), so I want to pass through the individual drives.\
First do `lsblk -o +MODEL,SERIAL` and identify the folder name matching the model and serial number of the desired HDD in the previous step.\
Now run `ls /dev/disk/by-id` and identify the path to the desired drive, matching the model and serial to the previous step.\
Now run `qm set <VM_ID> -scsi<PORT_NUMBER> /dev/disk/by-id/<DRIVE-PATH>`, replacing the port number with the next available scsi port (to check which ports are in use, look at the VM summary), and the drive path is the folder name identified in the previous step.
4. Refresh the UI and now the new drives should show up as being passed to the VM.
5. Start the VM from Proxmox GUI and run through the installation. Once done, the VM will restart. Leave the VM window open and after the reboot you will be shown an IP address for the OMV GUI. Use this in a browser with default credentials U: admin, P: openmediavault. If the default credentials do not work, login as root in the VM CLI, and do `passwd admin` to change the password.
6. Go to Network -> Interfaces and edit the network device to use a static IP. Once you save and apply the changes, you will have to reconnect to OMV using the new IP.
7. Go to Storage -> Disks and wipe the desired storage disk(s). 
8. Go to Storage -> File Systems -> Create and select your newly wiped disk + EXT4. This may take a while if you have large HDDs. After this is done, you can mount the filesystem and save changes.
9. Go to Storage -> Shared Folders, create a new share and apply.
10. Go to Services -> SMB/CIFS -> Settings and enable sharing.
11. Go to Shares -> Create, select the filesystem you wish to share, then save and apply.
13. Finally, in Users -> Users create a new user to access the SMB share from. Save and apply settings. After this, you should be able to see the new filesystem on your network and access it with the new user credentials.

[**TODO**] In the OMV console, run `wget -O - https://github.com/OpenMediaVault-Plugin-Developers/packages/raw/master/install | bash` to install OMV-Extras - This lets us install `mergerfs` and `SnapRAID` later on when we have multiple drives. For now we only have one drive, so we'll skip this.