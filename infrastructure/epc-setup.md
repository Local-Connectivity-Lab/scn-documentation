---
title: EPC Setup
parent: Infrastructure
---

# EPC Installation and Setup

The CoLTE package runs on Ubuntu 18.04 LTS (Bionic Beaver), 64-bit version. Install is tested on a fresh Ubuntu Desktop or Ubuntu Server installation. This tutorial assumes that the install is being performed on an x86-based mini-PC.

Note: You should choose the “basic install” option that doesn’t install extra unnecessary software. In prior installs this has led to version conflicts.

## I. LTE Architecture

![Diagram of LTE architecture including 4 main sections: User equipment (UE), eNodeB base station, Evolved Packet Core (EPC), Upstream IP networks/Internet](https://i.imgur.com/dMZQVDl.png)


CoLTE simplifies implementation and configuration of the Evolved Packet Core (EPC) elements of an LTE network using the Open5GS package.

The EPC provides Control plane functions, such as subscriber and mobile management, and User plane functions, such as routing user traffic to the internet. It connects to the radio base station, known as the eNodeB, which in turn will associate with User Equipment (UE).

## II. CoLTE Installation
Ensure all Ubuntu packages are up-to-date:

```bash
sudo apt update && sudo apt -y dist-upgrade
```

### A. Option 1: Install CoLTE from package repositories

CoLTE can be installed from its debian repository, or built from source. Building from source is fairly simple, and ensures that the most up-to-date version is installed. The following repositories include packages for CoLTE dependencies open5gs and haulage. More information about CoLTE can be found at https://github.com/uw-ictd/colte:

```bash
echo "deb [signed-by=/usr/share/keyrings/colte-archive-keyring.gpg] http://colte.cs.washington.edu $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/colte.list
sudo wget -O /usr/share/keyrings/colte-archive-keyring.gpg http://colte.cs.washington.edu/colte-archive-keyring.gpg
sudo apt install software-properties-common
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
sudo apt -y install colte
```

### B. Option 2: Build CoLTE from source
Install build dependencies. Note: the version of NodeJS in the Ubuntu repositories is out of date. These commands add repositories directly provided by Node:

```bash
curl -sL https://deb.nodesource.com/setup_10.x | sudo bash -s
sudo apt install build-essential default-mysql-client default-mysql-server nodejs curl
echo "deb [signed-by=/usr/share/keyrings/colte-archive-keyring.gpg] http://colte.cs.washington.edu $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/colte.list
sudo wget -O /usr/share/keyrings/colte-archive-keyring.gpg http://colte.cs.washington.edu/colte-archive-keyring.gpg
sudo apt install software-properties-common
sudo add-apt-repository ppa:open5gs/latest
sudo apt update
```

Download CoLTE from its git repository, and build locally:

```bash
git clone https://github.com/uw-ictd/colte
cd colte
make
```

If all goes well, a package will be installed into the BUILD subdirectory. Install with apt:

```bash
sudo apt install ./BUILD/colte_<VERSION>.deb
```

## III. Network Interface Configuration

### A. Recommendation Configuration
This requires an EPC machine with 2 or more ethernet ports (here named enp1s0 and enp4s0). The upstream interface receives an IP address via DHCP as usual from the upstream router, which passes its traffic to and from the Internet. The downstream interface connecting to the eNB is assigned two IP addresses and subnets, which are configured statically.

For Ubuntu 18.04, create a file in the `/etc/netplan` directory named `99-colte-config.yaml`, and add the following lines, substituting the correct interface names and subnets for your configuration:

```yaml
# CoLTE network configuration
network:
  ethernets:
    enp1s0: # name of interface used for upstream network 
      dhcp4: yes
    enp4s0: # name of interface going to the eNB
      dhcp4: no
      addresses:
        - 192.168.150.2/24 # list all downstream networks
        - 192.168.151.2/24
  version: 2
```

Note: Netplan will apply configuration files in this directory in the numerical order of the filename prefix (ie., 00-\*, 01-\*, etc.). Any interfaces configured in an earlier file will be overwritten by higher-numbered configuration files, so we create a file with the prefix 99-\* in order to supersede all other configuration files.

To enter the appropriate values into the configuration, you will need to find out the names of your ethernet interfaces. Use the command “ip a” on the command line. A list of network interfaces will appear in the terminal. Find the ones corresponding to your ethernet ports (their names usually start with “eth,” “enp,” or “enx”). For the recommended configuration shown, **we require an EPC machine with 2 or more ethernet ports**. The first ethernet interface, enp1s0, is used as the WAN port, which accesses upstream IP networks and the Internet. The last ethernet interface, enp4s0, will connect to the eNodeB, and out to our LTE network. (Our mini-PC model has 4 ethernet ports.)

Below we also provide an alternate configuration in case you do not yet have a machine with 2 ethernet ports or a USB to ethernet adapter dongle. However, only the first configuration is recommended for deployments for security reasons. **The alternative should be used for testing only**.

### B. NOT Recommended for deployment
If you don’t yet have a machine with 2 ethernet ports or a USB to ethernet adapter dongle, you can temporarily use a machine with a single ethernet port along with a simple switch or router. If using a simple switch, you can follow the same instructions but connect all three of the EPC, eNB, and upstream Internet router to the switch. If using a router, you may instead need to configure the router to assign 2 private static IPs to each of the EPC (i.e. `192.168.150.2`, `192.168.151.2`) and eNB (i.e. `192.168.150.1`, `192.168.151.1`), such that it will correctly NAT upstream traffic and also route local traffic between the EPC and eNB.

```yaml
# Network config EPC with single ethernet card
# A switch is used to connect all devices
network:
  ethernets:
    enp1s0: # name of ethernet interface
      dhcp4: true
      addresses:
        - 192.168.150.2/24 # list all downstream networks
        - 192.168.151.2/24
  version: 2
```

Once this file (or your router configuration) has been modified, restart the network daemon to apply the configuration changes:

```bash
sudo netplan try
sudo netplan apply
```
If the eNB will be plugged into its own dedicated EPC ethernet port, as in the recommended configuration above, you may need to connect that EPC ethernet port to something (e.g. the eNB, a switch, another machine) via an ethernet cable to wake the interface up (so that it becomes active and takes on the assigned IP addresses). The open5gs MME needs to bind its S1 interface to one of those IP addresses (in this case `192.168.0.2`). Until those IPs exist on your machine, the MME will continually throw errors if you try to run it.

## IV. CoLTE Configuration

### A. Using `colteconf`
CoLTE simplifies LTE network configuration by consolidating relevant configuration files into the directory `/etc/colte`. The primary configuration file is `/etc/colte/config.yml`. Update this file as below:

```yaml
# REMEMBER TO CALL "sudo colteconf update" AFTER CHANGING THIS FILE!

# basic network settings
enb_iface_addr: 192.168.150.2 # local IP for eNB interface
wan_iface: enp1s0 # ethernet WAN (upstream) interface name
network_name: YourNetworkName
lte_subnet: 10.45.0.0/16 # End User subnet

# PLMN = first 5 digits of IMSI = MCC+MNC
mcc: 910
mnc: 54

# advanced EPC settings
dns: 8.8.8.8

# database connection settings (for Haulage + WebGui + WebAdmin)
mysql_user: haulage_db
mysql_password: haulage_db
mysql_db: haulage_db

# use these vars to turn services ON (also starts at boot) or OFF
metered: true
nat: true
epc: true
```

Once this is done, run: 

```bash
$ sudo colteconf update 
```

This will update the configuration and reload services.
**TODO: list service names**

### B. Monitoring CoLTE
Ubuntu’s built-in logging and monitoring services can be used to monitor the CoLTE services:

```bash
$ sudo journalctl -fu open5gs-mmed.service
```

OR

```bash
$ sudo systemctl status open5gs-mmed.service
```

*Tab complete may be able to fill in the service name for systemctl at least*

## V. 'Persist' CoLTE Configuration

CoLTE configures IPTables rules to make sure packets are routed correctly within the EPC. IPTables rules must be made persistent across reboots with the `iptables-persistent` package:

```bash
sudo apt install iptables-persistent
```

Installation of this package will save the current iptables rules to it’s configuration file, `/etc/iptables/rules.v4`.

Note: `iptables-persistent` reads the contents of this file at boot and applies all iptables rules it contains. If you need to update the rules, or re-apply manually, you may use the following commands. This should not be necessary under normal circumstances:

```bash
sudo iptables-save > /etc/iptables/rules.v4
sudo iptables-restore < /etc/iptables/rules.v4
```

## VI. User Administration and Management

### A. Command line using `coltedb`
CoLTE comes with the command `coltedb` which can be used to modify the user database via the command line. Run `coltedb` without any arguments to see a summary of the available commands. 

To add a new user with a given SIM card, you will need several pieces of information for each SIM card. These values should be made available to you as a spreadsheet or text file by the SIM card manufacturer when you buy them. **PLEASE KEEP THIS INFO SECRET!!!** This is essential for the privacy and security of your network.
* IMSI 
    * unique identifier for SIM card
    * manufacturer provides
* MSISDN 
    * an arbitrary number representing the user’s “phone number”
    * could be the last 5 or more digits of the IMSI- make this up if not provided to you
* IP Address
    * this value sets a private static IP for each SIM card
    * you’re also free to set this
* Key
    * user’s private key used in LTE encryption
    * manufacturer provides
* OPC 
    * “carrier” private key used in LTE encryption
    * manufacturer provides
* APN (*optional)* 
    * access point name
    * for some CBRS LTE phone models such as the LG G8 ThinQ, the APN sent by the phone is hard-coded to be the string “ims”, so the only solution we’ve found is to set the APN on the EPC to match.
    
To add a single new user in the command line, use the following command format:

```bash
sudo coltedb add imsi msisdn ip key opc [apn]
```

For example, a line with some dummy values inserted could look like this (no APN):

```bash
sudo coltedb add 460660003400030 30 192.168.151.30 0x00112233445566778899AABBCCDDEEFF 0x000102030405060708090A0B0C0D0E0F
```

### B. Bulk add using a script
The shell script “bulk_add.sh” is provided for your convenience in the [conf/](https://github.com/uw-ictd/colte/tree/main/conf) folder of the github repo. It takes a single argument, the filename (full path if not in the same directory) of a file (let’s say user_sims.txt) that contains the SIM card info of multiple users, one per line.

Here’s an example of 3 lines from such a user_sims.txt file (with dummy SIM info, and the APN set for each user):

```
460660003400032 32 192.168.151.32 0x00112233445566778899AABBCCDDEEFF 0x000102030405060708090A0B0C0D0E0F ims
460660003400033 33 192.168.151.33 0x00112233445566778899AABBCCDDEEFF 0x000102030405060708090A0B0C0D0E0F ims
460660003400034 34 192.168.151.34 0x00112233445566778899AABBCCDDEEFF 0x000102030405060708090A0B0C0D0E0F ims
```
Then, to add them all at once to the database, you would run: 

```bash
sudo bulk_add.sh user_sims.txt
```
