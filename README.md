FortiGate-VM KVM Home Lab
A practical, vendor-image-based guide for deploying FortiGate-VM on a generic Linux host with KVM/QEMU, libvirt, and virt-manager. It covers initial networking, web-GUI access, FortiCare evaluation activation, and an optional isolated LAN for safely testing firewall policies.
[!WARNING]

This repository is for a non-production learning environment. Do not connect an untested FortiGate configuration directly to a production network or expose its management interface to the public internet.
Contents
·	Lab topology
·	Requirements
·	Download the image
·	Install host dependencies
·	Prepare libvirt networking
·	Prepare the VM disk
·	Create the FortiGate VM
·	First boot and console setup
·	Open the web GUI
·	Activate the evaluation license
·	Optional: isolated LAN
·	Backups and snapshots
·	Troubleshooting
·	License notes
Lab topology
The initial build uses one NAT-connected interface for management and internet access. Add the isolated LAN only after confirming that initial management access and licensing work.
                         Internet
                            |
                            v
                      Linux host system
                            |
                            v
            libvirt default network (NAT)
                            |
                            v
              FortiGate-VM port1 (DHCP)
                   Management / simulated WAN
                            |
                            v
          FortiGate-VM port2 (10.10.10.1/24)
                            |
                            v
          libvirt fgt-lan (isolated virtual LAN)
                            |
                            v
       Test VM: Kali, Ubuntu, Windows, or another lab VM

Requirements
Component	Requirement
Host operating system	Any current 64-bit Linux distribution
CPU	Intel CPU with VT-x or AMD CPU with AMD-V/SVM enabled in UEFI/BIOS
Hypervisor	KVM/QEMU with libvirt
VM manager	virt-manager
Network services	dnsmasq and libvirt virtual networking
Fortinet account	Free FortiCare / Fortinet Support account
Initial VM allocation	1 vCPU and 2048 MiB RAM
Host resources	16 GB RAM and SSD storage recommended for a multi-VM lab

[!NOTE]

The permanent evaluation license is capacity-limited. Use 1 vCPU, no more than 2 GB RAM, and no more than three virtual NICs.
Download the image
1.	Sign in to the Fortinet Support portal.
2.	Navigate to the FortiGate VM image downloads.
3.	Download the new deployment KVM image for x86-64 systems:
4.	FGT_VM64_KVM-v<version>-FORTINET.out.kvm.zip

5.	Example:
6.	FGT_VM64_KVM-v7.6.7.M-build3704-FORTINET.out.kvm.zip

File selection
Image	Use case	Use for this guide?
FGT_VM64_KVM-...out.kvm.zip	Fresh FortiGate-VM deployment on standard x86-64 KVM/QEMU	Yes
FGT_VM64_KVM-...out	Firmware upgrade for an existing FortiGate-VM	No
FGT_ARM64_KVM-...	ARM64 KVM hosts	No, unless the host is ARM64
FFW_VM64_KVM-...	FortiFirewall-VM image family	No

After extracting the deployment archive, the file needed by KVM/QEMU is:
fortios.qcow2

Install host dependencies
Install QEMU/KVM, libvirt, virt-manager, and dnsmasq using the package manager for the host distribution.
Debian / Ubuntu / Linux Mint / Pop!_OS
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients virt-manager dnsmasq
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"

Fedora / RHEL / Rocky Linux / AlmaLinux / CentOS Stream
sudo dnf install @virtualization virt-manager dnsmasq
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"

Arch Linux and Arch-based distributions
sudo pacman -Syu
sudo pacman -S qemu-full libvirt virt-manager dnsmasq iptables-nft
sudo systemctl enable --now libvirtd
sudo usermod -aG libvirt,kvm "$USER"

Log out and back in, or reboot, after modifying group membership.
Verify that KVM is loaded:
lsmod | grep kvm

Expected output includes kvm and either kvm_amd or kvm_intel.
[!TIP]

If libvirtd is unavailable on a distribution using modular libvirt services, enable these sockets instead:
sudo systemctl enable --now virtqemud.socket virtnetworkd.socket

Prepare libvirt networking
The default libvirt network provides a private NAT subnet. It gives FortiGate outbound internet access for evaluation-license activation while keeping the VM isolated from the physical LAN.
Check the current virtual-network status:
sudo virsh net-list --all

If the default network is inactive, start it and make it persistent across reboots:
sudo virsh net-start default
sudo virsh net-autostart default

Confirm it is active:
sudo virsh net-list --all

Expected result:
 Name      State    Autostart   Persistent
------------------------------------------------
 default   active   yes         yes

[!IMPORTANT]

Use Virtual network default: NAT for the first FortiGate NIC. Do not use a bridge to the physical LAN during the initial deployment.
Prepare the VM disk
Create a local working directory:
mkdir -p ~/VirtualMachines/FortiGate
cd ~/VirtualMachines/FortiGate

Extract the downloaded image:
unzip FGT_VM64_KVM-v<version>-FORTINET.out.kvm.zip

Confirm the deployment disk exists:
ls -lh fortios.qcow2

Optional: copy it to the default libvirt image path:
sudo mkdir -p /var/lib/libvirt/images/FortiGate
sudo cp fortios.qcow2 /var/lib/libvirt/images/FortiGate/fortios.qcow2

If the disk cannot be selected or opened in virt-manager, add the location as a libvirt storage pool instead of applying broad permission changes.
Create the FortiGate VM
1.	Open Virtual Machine Manager:
2.	virt-manager

3.	Confirm the connection is QEMU/KVM - localhost.
4.	Select Create a new virtual machine.
5.	Choose Import existing disk image.
6.	Browse to and select fortios.qcow2.
7.	Choose Generic Linux or the closest generic Linux version in the OS selector.
8.	Allocate:
9.	Memory: 2048 MiB
CPUs:   1

10.	Name the VM:
11.	FortiGate-VM

12.	Select Customize configuration before install.
13.	Click Finish.
Verify settings before boot
Device	Required setting
Virtual disk	fortios.qcow2, QCOW2 format, VirtIO bus where available
First NIC	Virtual network default: NAT
NIC model	VirtIO
Boot disk	FortiGate QCOW2 disk first in boot order
Firmware	Begin with BIOS/legacy firmware if UEFI causes boot problems
VM resources	1 vCPU and 2048 MiB RAM

Start the VM after verifying the configuration.
First boot and console setup
At the console prompt:
FortiGate-VM64-KVM login:

Use the default credentials:
Username: admin
Password: [leave blank and press Enter]

FortiOS prompts you to create a password immediately. Password characters are not displayed in the console; this is expected.
Configure port1
Configure port1 to request a DHCP address from the libvirt NAT network and permit local management access:
config system interface
    edit port1
        set mode dhcp
        set allowaccess ping https ssh
    next
end

Find the address assigned to port1:
diagnose ip address list

Check route availability:
get router info routing-table all

Test internet connectivity:
execute ping 1.1.1.1
execute ping fortinet.com

If the IP-address ping works but the DNS-name ping fails, set DNS servers:
config system dns
    set primary 1.1.1.1
    set secondary 8.8.8.8
end

Open the web GUI
On the Linux host, open a web browser and enter the port1 IP address found in the FortiGate console:
https://<port1-ip-address>

Example:
https://192.168.122.100

A certificate warning is expected because FortiGate initially uses a self-signed certificate.
1.	Open the browser's Advanced certificate details.
2.	Continue to the local IP address.
3.	Log in with username admin and the password created during the first console login.
Activate the evaluation license
A new VM can show VM is not licensed or license is invalid for current VM configuration. This is expected before evaluation activation.
1.	In the FortiGate GUI, open FortiGate VM License or select Activate License.
2.	Choose:
3.	Evaluation License

4.	Do not choose Upload License File unless a paid VM license has been purchased.
5.	Enter the FortiCare credentials used to sign in at support.fortinet.com.
6.	Submit the request.
7.	Allow FortiGate to apply the license and reboot.
8.	Wait for the console login: prompt to return, then refresh the browser and sign in again.
Evaluation limits
The permanent evaluation license does not expire, but it is restricted for lab use. Product limits can vary by release. Common limits include:
·	One evaluation VM per FortiCare account.
·	One vCPU.
·	Up to 2 GB RAM.
·	Up to three network interfaces.
·	Up to three firewall policies.
·	Up to three routes.
·	No FortiGuard subscription services or FortiCare support.
Initial onboarding choices
Prompt	Recommended learning-lab choice	Reason
Migration from an older FortiGate	Skip / No migration	A new lab has no older configuration to convert
Automatic patch upgrades	Disable initially	Prevents unexpected reboots or behavior changes while following labs
Dashboard template	Comprehensive	Shows more dashboards and FortiView monitors for exploration

[!TIP]

After the lab is stable, update FortiOS manually: export a configuration backup, create a snapshot, review the recommended upgrade path, then apply the update.
Optional: isolated LAN
After management access and licensing are confirmed, add an internal virtual network for test systems.
Create the lab network
In virt-manager:
1.	Open Edit -> Connection Details.
2.	Open Virtual Networks.
3.	Click +.
4.	Set the network name:
5.	fgt-lan

6.	Use the subnet:
7.	10.10.10.0/24

8.	Select an isolated network with no forwarding to the physical network.
9.	Disable libvirt DHCP if FortiGate will provide DHCP. Leave it enabled only for temporary testing if needed.
10.	Complete the wizard and start the network.
Add port2 to FortiGate
1.	Shut down the FortiGate VM gracefully.
2.	Open VM details in virt-manager.
3.	Select Add Hardware -> Network.
4.	Select virtual network fgt-lan.
5.	Select VirtIO as the device model.
6.	Apply changes and boot the VM.
The additional interface should appear as port2 in FortiOS.
Configure it:
config system interface
    edit port2
        set ip 10.10.10.1 255.255.255.0
        set allowaccess ping https ssh
    next
end

Attach a test VM to fgt-lan and configure it with values such as:
IP address: 10.10.10.10
Prefix:     /24
Gateway:    10.10.10.1
DNS:        1.1.1.1

[!NOTE]

To permit lab-client internet access, create an explicit FortiGate firewall policy from port2 to port1 with NAT enabled. Confirm interface addressing and default routing before creating the policy.
Backups and snapshots
Create a clean restore point after the following are working:
·	The permanent evaluation license is active.
·	port1 receives a NAT address.
·	The web GUI opens successfully.
·	The initial administrator password is stored securely.
Recommended actions
1.	In FortiOS, export a backup from System -> Settings -> Backup.
2.	Create a virt-manager snapshot/checkpoint if supported by the storage configuration.
3.	Alternatively, shut down the VM and make a safe copy of the QCOW2 disk.
4.	Record FortiOS version, virtual NIC mapping, assigned subnets, and resource allocation in your lab notes.
Troubleshooting
Issue	Likely cause	First action
default NAT network is inactive	Libvirt network is stopped	Run sudo virsh net-start default and sudo virsh net-autostart default
VM does not launch	KVM/libvirt service is down or hardware virtualization is disabled	Start the appropriate libvirt service; enable VT-x/AMD-V/SVM in firmware
VM disk cannot be selected	Image path is not in a usable libvirt storage pool	Add a storage pool or use /var/lib/libvirt/images/
port1 has no IP address	NIC is not attached to default: NAT, or port1 is not in DHCP mode	Verify the virt-manager NIC source; run the set mode dhcp configuration
GUI is unreachable	Wrong IP, no HTTPS administrative access, or NAT network stopped	Use diagnose ip address list; verify set allowaccess ping https ssh
FortiGate cannot reach FortiCare	NAT, default route, or DNS is missing	Test execute ping 1.1.1.1, then test a DNS name and configure DNS if needed
Trial activation fails	Account already has a trial VM or VM exceeds trial capacity	Confirm 1 vCPU, 2 GB RAM, up to 3 NICs; review FortiCare asset management
VM does not boot	Boot-order or firmware mismatch	Ensure the QCOW2 disk is first; try BIOS/legacy firmware
Certificate warning appears	Default self-signed administrative certificate	Verify the local VM IP, then continue through the browser's advanced warning page

Useful commands
Host commands
# Display defined and running VMs
virsh list --all

# Display all libvirt networks
sudo virsh net-list --all

# Start the default NAT network
sudo virsh net-start default

# Start the default NAT network automatically
sudo virsh net-autostart default

# Launch the graphical VM manager
virt-manager

FortiGate commands
# Display interface addressing
diagnose ip address list

# Display the routing table
get router info routing-table all

# Test IP connectivity
execute ping 1.1.1.1

# Test DNS resolution and connectivity
execute ping fortinet.com

# Display system and license-related status
get system status

Operating notes
·	The FortiGate VM must be powered on to provide its GUI, VPN, routing, or firewall functions.
·	The virt-manager program and console window may be closed after the VM is running; they are management tools, not the VM itself.
·	Shutting down, rebooting, suspending, or hibernating the Linux host stops the VM.
·	Keep the default NAT network active for the port1 management/WAN interface.
·	For a learning lab, start the VM manually until the configuration is stable. Enable VM autostart only if it is genuinely required.
License notes
·	Use only official Fortinet images and comply with Fortinet licensing terms.
·	The permanent evaluation license is a free lab/evaluation entitlement, not an unrestricted production license.
·	The specific limits and available features may change between FortiOS releases; confirm the license details in the FortiGate GUI and Fortinet documentation for the installed version.
·	Never commit configuration backups containing credentials, certificates, private keys, tokens, serial numbers, or IP information that should remain private.
Disclaimer
This project is an independent lab deployment guide and is not affiliated with, endorsed by, or supported by Fortinet. Fortinet, FortiGate, FortiOS, FortiCare, FortiGuard, and FortiConverter are trademarks of Fortinet, Inc.
