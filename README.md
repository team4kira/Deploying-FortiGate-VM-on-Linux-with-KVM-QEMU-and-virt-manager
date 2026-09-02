Deploying FortiGate-VM on Linux with KVM/QEMU and virt-manager
This guide explains how to deploy an official FortiGate-VM KVM image on a Linux host using KVM/QEMU, libvirt, and virt-manager. It also covers connecting the VM to a NAT network, activating Fortinet's permanent evaluation license, and creating an optional isolated network for safe lab practice.
Scope: This is a non-production home-lab guide. Start with NAT and isolated virtual networks. Do not bridge the firewall directly to a production network until the topology, routing, and firewall policies are understood.
Table of Contents
	What You Need
	1. Download the Correct Image
	2. Install Virtualization Components
	3. Enable the Default NAT Network
	4. Extract the FortiGate Image
	5. Create the FortiGate VM
	6. Configure the VM Before First Boot
	7. First Console Login
	8. Configure port1 for NAT and GUI Access
	9. Access the FortiGate GUI
	10. Activate the Permanent Evaluation License
	11. Complete Initial Onboarding
	12. Create an Isolated LAN Lab Network
	13. Back Up and Snapshot
	Troubleshooting
What You Need
	A 64-bit Intel or AMD Linux host with hardware virtualization enabled:
o	Intel VT-x, or
o	AMD-V / SVM.
	A Linux distribution that supports KVM/QEMU, libvirt, and virt-manager.
	Internet access on the host.
	A free FortiCare / Fortinet Support account for the permanent evaluation license.
	At least 2 GB available RAM and a few GB of free disk space for the FortiGate VM itself.
	Recommended for larger labs: 16 GB or more host RAM and SSD storage.
1. Download the Correct Image
From the Fortinet Support portal, choose the new deployment package for the standard x86-64 KVM FortiGate image:
FGT_VM64_KVM-v<version>-FORTINET.out.kvm.zip

For example:
FGT_VM64_KVM-v7.6.7.M-build3704-FORTINET.out.kvm.zip

Do not select these for a normal x86-64 KVM deployment
File type	Why not use it?
FGT_VM64_KVM-...out	Firmware upgrade image for an already deployed FortiGate VM; not a fresh VM disk.
FGT_ARM64_KVM-...	Intended for ARM64 hosts, not ordinary Intel/AMD desktop systems.
FFW_VM64_KVM-...	FortiFirewall-VM product family, not FortiGate-VM.

The new-deployment archive contains fortios.qcow2, the bootable virtual disk used by KVM/QEMU.
2. Install Virtualization Components
Install these core components using the package manager for your Linux distribution:
	QEMU/KVM — virtualization engine and kernel acceleration
	libvirt — VM and virtual-network management service
	virt-manager — graphical virtual-machine manager
	dnsmasq — commonly used by libvirt for NAT-network DHCP and DNS services
Examples:
Debian, Ubuntu, Linux Mint, or Pop!_OS
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virt-manager dnsmasq

Fedora, RHEL, Rocky Linux, AlmaLinux, or CentOS Stream
sudo dnf install @virtualization virt-manager dnsmasq
sudo systemctl enable --now libvirtd

Arch Linux and Arch-based distributions
sudo pacman -Syu
sudo pacman -S qemu-full libvirt virt-manager dnsmasq iptables-nft

Enable and start libvirt if it was not started by the installation:
sudo systemctl enable --now libvirtd

Add the current user to the virtualization groups. Exact group names differ by distribution; common examples are:
sudo usermod -aG libvirt,kvm $USER

Log out and log back in, or reboot, before continuing.
Verify KVM kernel support:
lsmod | grep kvm

Expected output includes one of the following plus kvm:
kvm_amd

or:
kvm_intel

If modular libvirt services are used
Some current Linux distributions use modular libvirt services. If libvirtd is unavailable or virtual networking does not work, enable these sockets:
sudo systemctl enable --now virtqemud.socket virtnetworkd.socket

3. Enable the Default NAT Network
The libvirt default virtual network provides an isolated subnet with outbound NAT access through the Linux host. It is the best option for the first FortiGate interface because it permits internet access for license activation while keeping the lab separate from the physical LAN.
Check its state:
sudo virsh net-list --all

If default is inactive, start it and enable automatic startup:
sudo virsh net-start default
sudo virsh net-autostart default

Verify the result:
sudo virsh net-list --all

Expected result:
 Name      State    Autostart   Persistent
------------------------------------------------
 default   active   yes         yes

The network commonly uses the 192.168.122.0/24 range, but do not assume a specific IP address for the FortiGate VM. Confirm its actual address after configuration.
4. Extract the FortiGate Image
Create a folder for the VM image:
mkdir -p ~/VirtualMachines/FortiGate
cd ~/VirtualMachines/FortiGate

Extract the downloaded archive:
unzip FGT_VM64_KVM-v<version>-FORTINET.out.kvm.zip

The archive should extract this file:
fortios.qcow2

Optional: store the disk in libvirt's image directory
sudo mkdir -p /var/lib/libvirt/images/FortiGate
sudo cp fortios.qcow2 /var/lib/libvirt/images/FortiGate/fortios.qcow2

Use the copied image in virt-manager. If virt-manager reports a permissions problem, use the storage-pool browser in virt-manager to add the image rather than applying arbitrary ownership or permission changes.
5. Create the FortiGate VM
1.	Open Virtual Machine Manager (virt-manager).
2.	Confirm that the connection is QEMU/KVM - localhost.
3.	Click Create a new virtual machine.
4.	Select Import existing disk image.
5.	Browse to and select fortios.qcow2.
6.	Select a generic Linux operating-system profile, such as Generic Linux 2022 or the nearest generic option.
7.	Set the VM resources:
o	Memory: 2048 MiB
o	CPUs: 1
8.	Set a descriptive VM name, for example:
FortiGate-VM

9.	Select Customize configuration before install.
10.	Click Finish.
The permanent evaluation license is intentionally resource-limited. Keep the VM at 1 vCPU and 2 GB RAM or less.
6. Configure the VM Before First Boot
In the customization window, verify the following settings before starting the VM.
Virtual disk
	Source: fortios.qcow2
	Disk format: qcow2
	Disk bus: VirtIO, where available
	Boot order: the FortiGate disk must be the first boot device
Network adapter
	Network source: Virtual network default: NAT
	Device model: VirtIO
	Link state: connected / active
Use only one network adapter initially. FortiGate recognizes it as port1.
Firmware
	Start with default BIOS/legacy firmware.
	If the VM does not boot, confirm that the virtual disk is first in boot order and try BIOS/legacy firmware instead of UEFI.
Click Apply if that button appears, then click Begin Installation or Run.
7. First Console Login
Wait for the console prompt:
FortiGate-VM64-KVM login:

Log in using:
Username: admin
Password: [leave blank and press Enter]

The initial admin password is blank. FortiOS then requires creation of a new administrator password.
Password characters are not displayed in the console. No dots or asterisks is normal.
When login succeeds, the CLI prompt appears similar to:
FortiGate-VM64-KVM #

8. Configure port1 for NAT and GUI Access
Configure the initial interface to request an address through DHCP from the libvirt NAT network and permit management access:
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
end

Check the assigned IP address:
diagnose ip address list

Look for the IPv4 address under port1. It may be in a range similar to 192.168.122.x.
Check the routing table:
get router info routing-table all

Test outbound network connectivity:
execute ping 1.1.1.1
execute ping fortinet.com

If IP connectivity works but name resolution does not, configure DNS:
config system dns
    set primary 1.1.1.1
    set secondary 8.8.8.8
end

9. Access the FortiGate GUI
On the Linux host, open a normal web browser. Navigate to the exact port1 address found in the console:
https://<fortigate-port1-ip>

Example:
https://192.168.122.100

A certificate warning is expected because the new FortiGate uses a self-signed administrative certificate.
1.	Open the browser's Advanced certificate details.
2.	Choose the option to continue to the local site.
3.	Log in as admin with the password created in the console.
Do not search for the address in a search engine; enter the full https:// URL in the browser address bar.
10. Activate the Permanent Evaluation License
A newly deployed VM may state that it is unlicensed or that the current configuration has no valid license. This is expected.
1.	In FortiOS, open the FortiGate VM License page or click Activate License.
2.	Select:
Evaluation License

3.	Do not select Upload License File unless a paid FortiGate-VM license has been purchased.
4.	Enter the same FortiCare / Fortinet Support credentials used for support.fortinet.com.
5.	Confirm the request and allow the VM to apply the license and reboot.
6.	Wait for the FortiGate console login prompt to return, typically a few minutes.
7.	Refresh the browser page and log in again.
Permanent evaluation limitations
The permanent evaluation license is intended for lab use. The exact product limits can change by FortiOS release; commonly documented restrictions include:
	One free evaluation VM per FortiCare account.
	One vCPU.
	Up to 2 GB RAM.
	Up to three network interfaces.
	Up to three firewall policies.
	Up to three routes.
	No FortiGuard subscription services or FortiCare support.
	Limited encryption capabilities, with exceptions for GUI management and FortiManager communication.
11. Complete Initial Onboarding
During onboarding:
	Migration prompt: choose No, Skip, or continue without migration when this is a new lab and no prior FortiGate configuration exists.
	Automatic patch upgrades: for a training lab, consider disabling automatic patch upgrades initially. Manual upgrades allow snapshots, configuration backups, and change review before the VM reboots or behavior changes.
	Dashboard layout: choose Comprehensive for learning. It makes more dashboard and FortiView monitoring pages visible. This choice affects only the GUI layout and can be changed later.
12. Create an Isolated LAN Lab Network
After the VM is licensed and the GUI is accessible, create a separate internal network for test VMs. This provides a safe topology without connecting the practice environment directly to the physical LAN.
Create fgt-lan in virt-manager
1.	Open virt-manager.
2.	Select Edit -> Connection Details.
3.	Open Virtual Networks.
4.	Click + to create a new virtual network.
5.	Name it:
fgt-lan

6.	Use an address range such as:
10.10.10.0/24

7.	Choose an isolated network with no forwarding to a physical network.
8.	Disable libvirt DHCP if FortiGate will provide DHCP later. Alternatively, leave it enabled temporarily for basic connectivity testing.
9.	Finish the wizard and start the network.
Add a second FortiGate NIC
1.	Shut down the FortiGate VM gracefully.
2.	Open its VM details in virt-manager.
3.	Click Add Hardware -> Network.
4.	Select virtual network fgt-lan.
5.	Set model to VirtIO.
6.	Apply the change and boot the VM.
FortiGate should see the new adapter as port2.
Configure port2
In the FortiGate CLI:
config system interface
    edit port2
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping https ssh
    next
end

Attach a Kali, Ubuntu, or Windows test VM to fgt-lan. Configure a static address such as:
IP address: 10.10.10.10
Prefix:     /24
Gateway:    10.10.10.1
DNS:        1.1.1.1

A basic safe lab design is:
Internet
   |
Linux host
   |
libvirt default NAT network
   |
FortiGate port1 (DHCP, management/WAN)
   |
FortiGate port2 (10.10.10.1/24)
   |
libvirt isolated fgt-lan
   |
Test VM (Kali, Ubuntu, Windows, etc.)

To provide port2 clients outbound connectivity through port1, later create an explicit FortiGate firewall policy from port2 to port1 with NAT enabled. Create this policy only after confirming the interfaces and routing are correct.
13. Back Up and Snapshot
After confirming that the license is active, port1 has connectivity, and GUI access works:
1.	Export a FortiGate configuration backup from System -> Settings -> Backup.
2.	In virt-manager, create a snapshot/checkpoint if the storage configuration supports it; alternatively, shut down the VM and make a safe copy of the qcow2 disk.
3.	Record the FortiOS version, VM resource allocation, interface mapping, and lab subnet plan in repository documentation.
A clean post-install checkpoint makes it easy to restore the lab after policy, VPN, routing, IPS, or Security Fabric experiments.
Troubleshooting
Problem	Likely cause	Recommended first action
default NAT network is inactive	The libvirt virtual network is stopped	Run sudo virsh net-start default and sudo virsh net-autostart default
VM will not start	KVM/libvirt service inactive or virtualization disabled in firmware	Start libvirt services and enable VT-x/AMD-V/SVM in BIOS/UEFI
fortios.qcow2 cannot be selected	Storage-path or permission restriction	Use a libvirt storage pool or place the image in /var/lib/libvirt/images/
FortiGate gets no IP on port1	NIC not connected to default: NAT, or interface not configured for DHCP	Confirm NIC source in virt-manager and run set mode dhcp on port1
Browser cannot open GUI	Wrong IP address or HTTPS administrative access is not allowed	Confirm address with diagnose ip address list; ensure set allowaccess ping https ssh
VM cannot reach FortiCare	NAT/DNS/default route issue	Check default network state; test execute ping 1.1.1.1; configure DNS if necessary
Evaluation license fails	Trial already registered to the FortiCare account or VM exceeds limits	Confirm 1 vCPU, 2 GB RAM, no more than 3 NICs; check FortiCare asset management
VM does not boot under UEFI	Firmware or boot-order issue	Switch to BIOS/legacy boot and ensure fortios.qcow2 is first in boot order
Browser shows certificate warning	FortiGate uses a self-signed certificate initially	Verify the local IP and continue through the browser's advanced warning option

Operating Notes
	The FortiGate VM must be running to access its GUI or process traffic.
	The virt-manager application and console window do not need to remain open after the VM is running.
	Shutting down, suspending, or rebooting the Linux host stops the VM unless an advanced host-level configuration is used.
	Keep the default NAT network active for the initial management/WAN interface.
	For a home lab, start the VM manually while learning; configure VM autostart only after the setup is stable.
Useful Commands
Host: check VM state
virsh list --all

Host: start the default NAT network
sudo virsh net-start default

Host: make the NAT network start automatically
sudo virsh net-autostart default

FortiGate: inspect interfaces
diagnose ip address list

FortiGate: inspect routes
get router info routing-table all

FortiGate: test network connectivity
execute ping 1.1.1.1
execute ping fortinet.com


Disclaimer
This repository documents a non-production FortiGate-VM home-lab deployment. Follow Fortinet licensing terms, use official Fortinet images, protect administrative credentials, and avoid connecting untested firewall configurations directly to production networks.
