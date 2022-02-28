# Overview
This guide covers the basic Eve-ng emulation environment setup along with basic BGP routing lab. These instructions cover virtualbox configuration, though other methods/systems will work. There are two options: Easy (unverified) and Full Setup (step by step clean build). Reference the FRR Manual as needed (helpful for router commands/verification):  [FRR Documentation](http://docs.frrouting.org/en/latest/setup.html#basic-setup)

# Easy Setup
* Download virtualbox disc image of pre-configured system and run. Everything should be setup per the final test lab (See bottom of this file).
* Likely will need to configure network adapters still, but need to check on that.

# Clean Install Setup

## Preparation
First, download and install the Eve-ng Windows [Client Integration Pack](https://www.eve-ng.net/index.php/download/).
Then, download the Eve-ng [Community Edition ISO](https://www.eve-ng.net/index.php/download/)

## Virtualbox Configuration
Eve-ng requires nested virtualization in order to run a simulated network. This option is available under:
* Settings->System->Processor->Enable Nested VT-X

If you are using a Windows platform, this option may not be directly selectable, but can be configured from the Windows command line:
* `C:\>cd "Program Files\Oracle\VirtualBox"`
* `C:\Program Files\Oracle\VirtualBox>VBoxManage modifyvm [VMName] --nested-hw-virt on`

## VBox Eve-NG Configuration
* Create a new Vbox Machine: Tools->New
* Assign maximum amount of memory you can. Recommend at least 8GB.
* Create a Virtual Hard Disk and select 50GB, recommend VHD for portability.

Once VM is initialized, go to settings and adjust the following:
* System->Motherboard->uncheck floppy
* Processor-> Minimum of 2, but more the better
* Processor-> Ensure checks for "Enable PAE/NX and Nested VT-x are checked. If not, check them (see above on Windows nested virtualization).
* Storage->click CD graphic and select Eve-NG ISO
* Audio->Disable, not needed
* Network->Select Bridged adapter and enable Promiscuous mode. Only one interface is needed for Eve, but can add more if desired.
* USB->Disable, not needed.

Then proceed to boot your VM and go through the Eve-ng installation/setup process.

## EVE-ng Network Setup with Host
Eve-ng creates 10 bridged interfaces by default associated with the 10 cloud interfaces available within the Eve-ng environment (eg. pnet1=Cloud 1 Interface). However, they are not configured. This section provides the steps necessary to setup an internal lab IP space and bridge to the host PC.
After Eve-ng is installed, you need to setup networking. You can follow [this guide](https://www.itnetworkeng.org/connect-nodes-inside-eve-ng-with-the-internet/) (with graphical aids) or just follow commands below.
* Determine what IP block you want to use for your lab gateway (can be anything). I am choosing `192.168.2.0/24` and the `pnet1` interface for this task.
* Assign an ip to your bridged interface: `$ip address add 192.168.2.1/24 dev pnet1`
* Make configuration survivable between reboots by modifying the `/etc/network/interfaces` file and configuring the pnet1 settings to the following:
```
iface eth1 inet manual
auto pnet1
iface pnet1 inet static
address 192.168.2.1
netmask 255.255.255.0
bridge_ports eth1
bridge_stp off
```

* Enable ip forwarding in the kernel to allow communication between guest and host:
	`$echo 1 > /proc/sys/net/ipv4/ip_forward `
* Make forwarding survivable between reboots by configuring `/etc/sysctl.conf` and removing the comment `#` before `net.ipv4.ip_forward=1`. 
* Configure address translation between `pnet1` subnet and VM (note `pnet0` is used as that is the dynamically configured 'Management' interface configured during Eve-NG setup as the default VM interface. You are just telling the VM to translate this new block into the auto-configured Eve-ng management interface here. Do not change to `pnet1` as it will result in no connectivity): `$iptables -t nat -A POSTROUTING -o pnet0 -s 192.168.2.0/24 -j MASQUERADE`
* Make iptables rule survive restarts by installing iptables persistent module: `apt-get install iptables`
* Save current iptables configuration via netfilter-persistent:
```
$netfilter-persistent save
$netfilter-persistent reload
```

## QEMU Device Setup for FRR install
In this section we create a base FRR device in Eve-NG which can then be used to simulate many routing devices. If you want to install other devices (firewall, proxy, etc.) you can follow this same approach. FRR will run on a base ubuntu server image. All of this setup will be done in the Eve-NG machine. Follow [this guide](https://www.2stacks.net/blog/getting-started-with-frr-on-eveng/) for included graphical aids or the commands below:
* Download ubuntu server: `$wget http://releases.ubuntu.com/20.04.3/ubuntu-20.04.3-live-server-amd64.iso`
* Eve relies on QEMU (Quick Emulation) to launch virtualized devices, therefore we need to implement a device via QEMU:
* Move to the Eve-ng qemu directory and then create a new directory for our FRR device: 
```
$cd /opt/unetlab/addons/qemu/
$mkdir linux-frr-8.1 && cd ./linux-frr-8.1
``` 
* Move our linux server iso to the cdrom device so we can boot: `$cp /root/ubuntu-20.04.3-live-server-amd64.iso cdrom.iso`
* Create a new hard disk for our FRR device: `/opt/qemu/bin/qemu-img create -f qcow2 virtioa.qcow2 16G` (Note: if this is a new device, you would name the image `virtiob.qcow2`, etc. per QEMU naming conventions)

## Create our new device in Eve-NG
Log in to your Eve-ng webportal interface (`user: admin pass:whatever you set on install`) (There should have been an IP displayed when booting, you will use that. Mine was `192.168.160.217`.) We will now create a new device which will be our FRR router:
* Create a new lab by clicking on the `File` icon and give it a name
* In the lab environment click the `+` and add a new node.
* Scroll through templates until you find `Linux` and then select the `linux-frr-8.1` image we created. Leave rest of default configs for now. Click `save`.
* Click `+` to add a new object again, this time select `Network` and select your `Cloud 1` interface (the one we configured earlier as `pnet1`).
* Connect the linux box to the cloud network by selecting the little plug and dragging the connection. We will configure the interface during the initial configuration in a bit.
* Right click on your linux device and select `start`. This will boot the linux system with the server.iso we placed in the cdrom of the QEMU image. If you installed the Windows integration pack, you can now click on the linux device to bring up the VNC connection. It will take a minute to boot the install, so give a few minutes. If not, notice the IP in the bottom left of Eve-ng when you hover over the device. You would have to SSH or whatever to that in order to connect if you didn't use the integration package.
* Go through the ubuntu server install using defaults until you get to network setup. Select `static` and then give your system an IP in the subnet you created earlier for `pnet1`. Use the `pnet1` IP (`192.168.2.1`) as your gateway.
* Continue with defaults until you get to `Filesystem setup` Select `Manual` and then select the `/dev/vda` 16Gb disk we created in QEMU. Under this disk, select `Add Partion` and select the default max value. Click `Create`
* Continue through menu to `Profile Setup`. Create a profile for your frr system (do not select `FRR` as the user as it will break the configuration later). 
* Select `Install OpenSSH server` and then `Done`. Let rest of install complete. Do NOT select `Cancel update and reboot`. Wait for installation to complete.
* Once install is complete, go to your Eve-ng host and remove the `cdrom.iso` prior to rebooting FRR install: 
```
$cd /opt/unetlab/addons/eqmu/linux-frr-8.1
$rm -rf cdrom.iso
```
* Now you can reboot your Ubuntu Server.

## Ubuntu server upgrades and FRR install
Update packages and prep linux server prior to installing FRR. You can optionally upgrade the kernel, but check on potential FRR conflicts first. Then upgrade your system:
* (Optional) `sudo apt-get install --install-recommends linux-generic-hwe-20.04`
```
sudo apt-get update
sudo apt-get dist-upgrade
sudo reboot 
``` 

Once rebooted, then proceed to install FRR from Debian Repository:
```
$curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add - 
$FRR="frr-stable"
$echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list
$sudo apt update && sudo apt install frr frr-pythontools ` 
```

## Configure Base FRR settings
The following are baseline configs that will apply to every FRR device you create in Eve-ng. 
* Allow access to FRR vtysh configuration utility: `$sudo usermod -a -G frr,frrvty <your Linux Server Username>`
* Enable routing daemons (note, only select the ones you plan to use to limit resource usage. You can edit this later if need be. Here I am just booting the BGP daemon). To launch a daemon, go to `/etc/frr/daemons` file and then change the state from `no` to `yes` for the daemon you want to enable.
* Restart FRR: `$sudo systemctl restart frr.service`
* Verify daemons have started: `$ps -ef | grep frr`  (you should see three: zebra, staticd, and bgp at this point)
* Enable kernel forwarding by editing the `/etc/sysctl.conf` file and uncommenting the `net.ipv4.ip_forward=1` line (do same for ipv6 if you plan to use).
* Note: in earlier versions there was a bug where this kernel edit wouldn't hold in 18.x versions of Ubuntu Server.  I didn't have this issue in 20.x and ignored the workaround suggested.

## Initial Router Configuration
* Enter your router terminal: `$vtysh` (you should enter a command prompt similar to `Router#`)
* Create device password: 
```
Router# configure terminal
Router(config)# password <something>
Router(config)# enable password <something>
Router(config)# exit
Router# write memory
Router# exit
``` 

## Finalize setup/final tasks
* (optional) Allow telnet to router if you want to use a standard config tool like putty. Note: run this in your FRR (ubuntu server) instance, NOT ON THE EVE-NG Server:
```
sudo sed -i 's/GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty"/' /etc/default/grub
sudo update-grub
``` 
* Commit all these setup changes to finalize our QEMU image for repeat deployments:
	* shutdown the FRR (ubuntu server) node: `$sudo shutdown -h now`
	* On Eve-ng server, commit our changes to our image and modify permissions for repeat deployments:
```
$/opt/qemu/bin/qemu-img commit virtioa.qcow2
$/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
``` 
		
## Device deployments in FRR
We can now use our base FRR image we just created to launch multiple instances within eve-ng (eg. each device corresponding to a router, switch, etc.) To launch a device:
* Select `+` to add new node, select `Linux` and then select our recently created image `linux-frr-8.1`
* Select how many interface we need the device to have (we will do 3 for now). Click ok. You should now have two "Linux" nodes in your Eve-ng environment that correspond to FRR routers (you can rename them).
* Drag a connection between your two routers and select which interface your want to use for connection. 
* Add two VPCs using the same add node process. Connect these to your routers on a different interface.
* Power on all systems and let them boot (may take a bit). At this point, everything is on, but nothing should have connectivity because we have not configured anything yet. We will need to create two subnets for our local 'Lan' network, and one for our router-router connection. You can select anything, but I followed the image below:
![alt text](https://github.com/KarlOlson/Eve-NG-BGP-Lab-Setup/blob/main/Images/network%20.png "Testnet")

* Note: to enter the router configuration command prompt type `$ vtysh` from your FRR host. This will enter the base FRR router configuration software. Follow image commands to setup a BGP routable network.
* After saving your router configuration using `Router1# write memory` this will save your configuration file. However, if you shut off the lab, it does not automatically reload this file and you may notice that your config is lost on reboot. You can reload this configuration by typing `$ vtysh -b` to reload the config and then `$ vtysh` to enter the router config prompt with the pre-loaded configuration.

## Eve-NG Wireshark Configuration
You will need to cache the Eve-NG VM server credentials before being able to use Wireshark for packet capture in Eve. If you don't, you will get a "End of file on pipe magic during open" Error. To fix:
* Open a Windows CMD Prompt and move to your EVE-NG directory: `C:\Program Files\EVE-NG\` It should have a 'plink.exe' file (if you installed the integration pack). 
* run plink to generate a connection and cache the certificate: `C:\Program Files\EVE-NG\plink.exe root@<your EVE-VM IP>` Select `Yes` on if you want to cache the certificate locally. You can now try wireshark (right click a device in Eve and select `Capture`) and it should work.

## In-line Proxy Setup
The proxies are configured with a L2 Bridged interface between two Ethernet ports to pass traffic transparantly. I had looked at other solutions, but this seems to be the best workable placeholder. In bridged mode you can monitor the traffic passing through via the `br0` interface and hopefully use that as a tapped interface in the future (need to look into how linux uses bridge interfaces). 

To configure the proxy bridge interfaces, use the `/etc/netplan/00-installer-config.yaml`  configuration file to set the network interface configuration with a bridged interface between the ethernet ports:
```
network:
  ethernets:
    ens3:
      dhcp4: no
    ens4: 
      dhcp4: no
    ens5:
      addresses: [192.168.1.1/24]
      gateway4: 192.168.1.2
  bridges:
    br0:
      interfaces:
        - ens3
	- ens4
version: 2
```
* Note: the ens5 configuration is to tie into the cloud/public network per the earlier Eve-ng setup. Only required if you need this device to have internet (for updates, etc.)
* Repeat process for second proxy.

## Deploying Ethereum Chain and Nodes:
This section covers the deployment of the ethereum blockchain, initial genesis launch, and interacting with the chain in the lab via the two nodes (the previously configured proxies)

*TBD

## Launching Lab
* Highlight all devices and click start. Give it 5 min for everything to boot. The VPCs will start instantly, but servers will take a bit.
* for VPCs - I haven't figured out how to make the config load automatically, but you can just run `> load config` and the VPC configuration will load.
* for the BGP ASes - I need to make a startup script, but until I do, only thing you need to do is run `$ vtysh -b` to load the BGP config after startup. After that you can join the router command prompt by using `$ vtysh` to make any changes.
* The proxies are configured with `br0` interface and will operate without any involvement.
* You can check interface configurations with `ip a` command in the linux box or a `sh interface brief` or a shorthand `sh int b` in FRR router config mode.

## Known Bugs/Issues
* See [Eve-ng Issues](https://github.com/SmartFinn/eve-ng-integration/blob/master/README.md) for common problems and fixes.
