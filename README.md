# Overview
This guide covers the basic Eve-ng emulation environment setup along with basic BGP routing lab. These instructions cover virtualbox configuration, though other methods/systems will work.

# Preparation
First, download and install the Eve-ng Windows [Client Integration Pack](https://www.eve-ng.net/index.php/download/).
Then, download the Eve-ng [Community Edition ISO](https://www.eve-ng.net/index.php/download/)

# Virtualbox Configuration
Eve-ng requires nested virtualization in order to run a simulated network. This option is available under:
* Settings->System->Processor->Enable Nested VT-X

If you are using a Windows platform, this option may not be directly selectable, but can be configured from the command line:
*C:\>cd "Program Files\Oracle\VirtualBox"
*C:\Program Files\Oracle\VirtualBox>VBoxManage modifyvm [VMName] --nested-hw-virt on

#VBox Eve-NG Configuration
*Create a new Vbox Machine: Tools->New
*Assign maximum amount of memory you can. Recommend at least 8GB.
*Create a Virtual Hard Disk and select 50GB, recommend VHD for portability.
Once VM is initialized, go to settings and adjust the following:
*System->Motherboard->uncheck floppy
*Processor-> Minimum of 2, but more the better
*Processor-> Ensure checks for "Enable PAE/NX and Nested VT-x are checked. If not, check them (see above on Windows nested virtualization).
*Storage->click CD graphic and select Eve-NG ISO
*Audio->Disable, not needed
*Network->Select Bridged adapter and enable Promiscuous mode. Only one interface is needed for Eve, but can add more if desired.
*USB->Disable, not needed.

Then proceed to boot your VM and go through the Eve-ng installation/setup process.

#EVE-ng Network Setup with Host
Eve-ng creates 10 bridged interfaces by default associated with the 10 cloud interfaces available within the Eve-ng environment (eg. pnet1=Cloud 1 Interface). However, they are not configured. This section provides the steps necessary to setup an internal lab IP space and bridge to the host PC.
After Eve-ng is installed, you need to setup networking. You can follow [this guide](https://www.itnetworkeng.org/connect-nodes-inside-eve-ng-with-the-internet/) (with graphical aids) or just follow commands below.
*Determine what IP block you want to use for your lab gateway (can be anything). I am choosing 192.168.2.0/24 for this task.
*Assign the ip to your bridged interface: $ip address add 192.168.2.1/24 dev pnet1
*Make configuration survivable between reboots by modifying the /etc/network/interfaces file and configuring the pnet1 settings to the following:
	iface eth1 inet manual
	auto pnet1
	iface pnet1 inet static
	address 192.168.2.1
	netmask 255.255.255.0
	bridge_ports eth1
	bridge_stp off
*Enable ip forwarding in the kernel to allow communication between guest and host:
	echo 1 > /proc/sys/net/ipv4/ip_forward
*Make forwarding survivable between reboots by configuring /etc/sysctl.conf and removing the comment '#' before net.ipv4.ip_forward=1. 
*Configure address translation between pnet1 subnet and VM (note pnet0 is used as that is the dynamically configured 'Management' interface. You are just telling the VM to translate this new block into the auto-configured Eve-ng management interface here. Do not change to pnet1): $iptables -t nat -A POSTROUTING -o pnet0 -s 192.168.2.0/24 -j MASQUERADE
*Make iptables rule survive restarts by installing iptables persistent module: apt-get install iptables
*Save current iptables configuration via netfilter-persistent:
	netfilter-persistent save
	netfilter-persistent reload

#QEMU Device Setup for FRR install
In this section we create a base FRR device in Eve-NG which can then be used to simulate many routing devices. If you want to install other devices (firewall, proxy, etc.) you can follow this same approach. FRR will run on a base ubuntu server image. All of this setup will be done in the Eve-NG machine. Follow [this guide](https://www.2stacks.net/blog/getting-started-with-frr-on-eveng/) for included graphical aids or the commands below:
*Download ubuntu server: root@eve-ng:# wget http://releases.ubuntu.com/20.04.3/ubuntu-20.04.3-live-server-amd64.iso
*Eve relies on QEMU (Quick Emulation) to launch virtualized devices, therefore we need to implement a device via QEMU:
*Move to the Eve-ng qemu directory and then create a new directory for our FRR device: cd /opt/unetlab/addons/qemu/ followed by: mkdir linux-frr-8.1 && cd ./linux-frr-8.1
*Move our linux server iso to the cdrom device so we can boot: cp /root/ubuntu-20.04.3-live-server-amd64.iso cdrom.iso
*Create a new hard disk for our FRR device: /opt/qemu/bin/qemu-img create -f qcow2 virtioa.qcow2 16G

#Create our new device in Eve-NG
Log in to your Eve-ng webportal interface (user: admin pass:whatever you set on install) (There should have been an IP displayed when booting, you will use that. Mine was 192.168.160.217.) We will now create a new device which will be our FRR router:
*create a new lab by clicking on the file icon and give it a name
*in the environment click the '+' and add a new node.
*Scroll through templates until you find 'Linux' and then select the 'linux-frr-8.1' image we created. Leave rest of default configs for now. Click 'save'.
*Click '+' to add a new object again, this time select 'Network' and select your Cloud 1 Interface (the one we configured earlier as pnet1).
*Connect the linux box to the cloud network by selecting the little plug and dragging the connection. We will configure the interface during the initial configuration in a bit.
*right click on your linux device and select 'start'. This will boot the linux system with the server.iso we placed in the cdrom of the QEMU image. If you installed the windows integration pack, you can now click on the linux device to bring up the VNC connection. It will take a minute to boot the install, so give a few minutes.
*Go through the ubuntu server install using defaults until you get to network setup. Select 'static' and then give your system an IP in the subnet you created earlier for pnet1. Use the pnet1 IP (192.168.2.1) as your gateway.
*Continue with defaults until you get to 'Filesystem setup' Select 'Manual' and then select the /dev/vda 16Gb disk we created in QEMU. Under this disk, select 'Add Partion' and select the default max value. Click 'Create'
*Continue through menu to 'Profile Setup'. Create a profile for your frr system (do not select FRR as the user as it will break the configuration later). 
*Select 'Install OpenSSH server' and then 'Done'. Let rest of install complete. Do NOT select 'Cancel update and reboot'. Wait for installation to complete.
*Once install is complete, go to your eve-ng host and remove the cdrom.iso prior to rebooting FRR install: cd /opt/unetlab/addons/eqmu/linux-frr-8.1, followed by: rm -rf cdrom.iso. Now you can reboot your Ubuntu Server.

#Ubuntu server upgrades and FRR install
Update packages and prep linux server prior to installing FRR. You can optionally upgrade the kernel, but check on potential FRR conflicts first. Then upgrade your system:
*(Optional) sudo apt-get install --install-recommends linux-generic-hwe-20.04
*sudo apt-get update
*sudo apt-get dist-upgrade
*sudo reboot

#Install FRR from Debian Repository
*$curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
*$FRR="frr-stable"
*$echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list
*$sudo apt update && sudo apt install frr frr-pythontools

#Configure Base FRR settings
The following are baseline configs that will apply to every FRR device you create in Eve-ng. 
*Allow access to FRR vtysh configuration utility: $sudo usermod -a -G frr,frrvty <your Linux Server Username>
*Enable routing daemons (note, only select the ones you plan to use to limit resource usage. You can edit this later if need be. Here I am just booting the BGP daemon). To launch a daemon, go to '/etc/frr/daemons' file and then change the state from 'no' to 'yes' for the daemon you want to enable.
*restart FRR: $sudo systemctl restart frr.service
*verify daemons have started: $ps -ef | grep frr  (you should see three: zebra, staticd, and bgp at this point)
*Enable kernel forwarding by editing the '/etc/sysctl.conf' file and uncommenting the 'net.ipv4.ip_forward=1' line (do same for ipv6 if you plan to use).
*Note: in earlier versions there was a bug where this kernel edit wouldn't hold. See website tutorial if you notice, but I didn't have this issue and ignored the workaround suggested.

#Initial Router Configuration
*enter your router terminal: $vtysh (you should enter a command prompt similar to Router#)
*create device password: 
	Router# configure terminal
	Router(config)# password <something>
	Router(config)# enable password <something>
	Router(config)# exit
	Router# write memory
	Router# exit

#finalize setup/final tasks
*(optional) Allow telnet to router if you want to use a standard config tool like putty. Note: run this in your FRR (ubuntu server) instance, NOT ON THE EVE-NG Server:
	sudo sed -i 's/GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty"/' /etc/default/grub
	sudo update-grub
*commit all these setup changes to finalize our QEMU image for repeat deployments:
	*shutdown the FRR (ubuntu server) node: $sudo shutdown -h now
	*On Eve-ng server, commit our changes to our image and modify permissions for repeat deployments:
		$/opt/qemu/bin/qemu-img commit virtioa.qcow2
		$/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
		
#Device deployments in FRRVER
We can now use our base FRR image we just created to launch multiple instances within eve-ng (eg. each device corresponding to a router, switch, etc.) To launch a device:
*select '+' to add new node, select 'Linux' and then select our recently created image 'linux-frr-8.1'
*select how many interface we need the device to have (we will do 3 for now). Click ok. You should now have two "Linux" nodes in your eve-ng environment that correspond to FRR routers (you can rename them).
*drag a connection between your two routers and select which interface your want to use for connection. 
*add two VPCs using the same add node process. Connect these to your routers on a different interface.
*power on all systems and let them boot (may take a bit). At this point, everything is on, but nothing should have connectivity because we have not configured anything yet. We will need to create two subnets for our local 'Lan' network, and one for our router-router connection. You can select anything, but I followed the image below: