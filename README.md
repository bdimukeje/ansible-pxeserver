# ansible-pxeserver
Build Preboot Environment with Ansible

Objective

The goal of this project is to configure a PXE/Kickstart server using Ansible playbooks on a RHEL host.
 This server enables automated installations of a RHEL7 using Kickstart files against targeted devices via their MAC addresses. 
 This implementation leverages Podman to run the HTTP server hosting Kickstart files.

We are using Rhel 9.5 virtual machine as a pxe server for automated installation of rhel 7 on a pxe client

Folder Structure

rhel-iso-links/      # Directory link to download RHEL binary dvd iso
inventory            # Ansible inventory file
setup-pxe.yml        # Main Ansible playbook
Readme.md            # This file

Rhel9-9.5-x86_64-dvd # Example ISO file


BIOS booting - X86_64

Core Service Setup: DHCP, TFTP, Podman HTTP Server

Features:

DHCP Server: Configures DHCP to provide IP addresses dynamically to PXE clients.

TFTP Server: Hosts PXE boot files.

Podman HTTP Server: Hosts Kickstart files in a Podman container for consistency and portability.

Dynamic Network Interface Selection: Automatically identifies the appropriate interface for DHCP and HTTP configurations.


Prerequisites

Before running the playbooks, ensure the following is configured on the pxe-server.
(My process was Installing RHEL-9 using the dvd.iso using virtualbox on my PC with NAT for adapter 1 and host-only for adapter 2 both enabled):  
 VBOX host-only adapters ips typically fall in 192.168.56.1 so my solution would work for creating multiple pxe servers static ips close to this range
 
 
My initial start was mounting the rhel 9 disc iso in the rhel-isos folder on vbox and running with NAT(NAT) and host-only adapter (adapter 2) then starrting the playbook
and doing the inital disc setup to harddrive.

To use the playbooks I did some initial steps to configure the pxe-server in the root terminal for sshd

if not installed:

SSH Configuration:
sudo yum install -y openssh-server
sudo systemctl enable sshd
sudo systemctl start sshd

Firewall Rules:
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=tftp --permanent
sudo firewall-cmd --add-port=69/udp --permanent
sudo firewall-cmd --reload

optional configure your static ip to work with the ip range:
ip addr
find your host-only interface eg enp0s8 and use 192.168.56.1
nmcli con add type ethernet con-name host-only ifname enp0s8 ipv4.addresses <static_ip>/24 ipv4.gateway <gateway_ip> ipv4.method manual ipv6.method ignore

In the inventory file update ansible_host to add your ip

if you do not do this step or have a static ip withing range then you would need to update the setup-pxe.yml file defined variables  with the appropriate subnet and range
subnet: "192.168.56.0/24"
dhcp_range_start: "192.168.56.100"
dhcp_range_end: "192.168.56.200"
#domain_name: "pxe.local"  # Add this line to define domain_name
#dns_server: "8.8.8.8"  # Google Public DNS or a local DNS server IP

the code should dynamically identify the correct ip for the host-only network on the pxeserver based on the subnet

WSL setup to run ansible:
	To ensure you are able to ssh from wsl to the server:
	Edit /etc/ssh/sshd_config in the pxe_server:
	verify in the file these lines are set to yes or add it if necessary:
			PermitRootLogin yes
			PasswordAuthentication yes

	Restart the SSH service:
		sudo systemctl restart sshd

attempt to login with your password using command: 
	root@<pxe_server_ip>

setup ssh on wsl
	ssh-keygen -t rsa -b 4096
	ssh-copy-id root@<pxe_server_ip>
	you can press enter twice when setting this up if preferred
	
	inventory.yml is currently configured to use sshprivatekey
	
	

Before proceeding to run the playbook ensure the rhel 7 dvd iso is mounted  to /mnt from your root directory:
In vbox it can be done by adding the rhel-server-7.0-x86_64-dvd.iso to the controller ide in the storage when the machine is off.
	when you turn it back on run mount /dev/sr0 /mnt   (this process assumes rhel 9 or the linux vm is booted first on hard disk the priority boot order on virtualbox should have hard disk first ( can be changed in: vbox>settings>system>motherboard)
	
	

	
Ansible Playbooks

1. setup-pxe.yml

This playbook sets up the PXE server. Key tasks include:

Configuring and starting DHCP, TFTP, and HTTP services.

Dynamically retrieving the PXE server's host-only adapter IP.

Using Podman to create a containerized HTTP server.
	
	
	
To run the playbook:

	Navigate to the folder in WSL: 
	this can be done with cd /mnt to your windows directory
	eg: cd /mnt/c/Users/.../ansible-pxe-setup
	
	run command:

	ansible-playbook -i inventory.yml setup-pxe.yml	
		

This will configure the pxe_server.

To make a pxe client on vbox:
	PXE clients on the same network will be discoverable
	
	
Steps to create a pxe client on vbox:
	Open VirtualBox and create a new VM.
	this should match the pxe server 
	Select your desired OS type and version (e.g., Linux Subtyoe Redhat Redhat64 bit) 	this should match the pxe server 
	Set memory and storage (default)
	Configure the Network Adapter:
	Go to Settings > Network.
	Set Adapter 1 to Host-Only Adapter.
	Ensure Cable connected is checked.
	Select the Host-Only Network (e.g., vboxnet0). 
	By default the host-only for the pxeclient using vbox will be in the same subnet as the pxe-server 
	Enable PXE Boot:

	Go to Settings > System > Motherboard.
	In the Boot Order section, make sure Network is the first option.
	Enable PXE Boot ROM:



	Power on the VM. It should now attempt to boot from the PXE server using the Host-Only Network connection.