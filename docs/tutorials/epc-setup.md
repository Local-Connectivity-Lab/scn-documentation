---
title: Step 1. LTE Core Network Setup
---
# Introduction and Overview

## LTE Architecture

![Diagram of LTE architecture including 4 main sections: User equipment (UE), eNodeB base station, Evolved Packet Core (EPC), Upstream IP networks/Internet](https://i.imgur.com/dMZQVDl.png)

The LTE Evolved Packet Core (EPC) provides core software functions such as subscriber management and routing user traffic to the Internet. It connects to the radio "base station", called the eNodeB (eNB), which then talks to the User Equipment (UE)- i.e., your cell phone or access device.

The most important component to know about in the LTE core is the "MME," which manages the process of the eNB and any end-user devices attaching themselves to the network (you can think of this as "signing on") so they can start sending data. In the case of users, the MME has to ask the HSS software component (essentially a user database) for credentials (shared secret keys unique to each user) to verify that a given SIM card is allowed to join the network. The MME is the software component whose output logs you should check on first for error messages if something is going wrong with the network.

SCN currently runs the LTE-specific components of the [Open5GS](https://open5gs.org/) 4G/5G Non-Standalone (NSA) core. 

You can find more detailed documentation and diagrams of the Open5GS software architecture at the Open5GS [Quickstart](https://open5gs.org/open5gs/docs/guide/01-quickstart/) page. Their software supports both 4G and 5G, and you only need to run a subset of the software components for 4G.

## Operating System Support

In SCN we will typically perform these installation steps using a fresh install of Ubuntu 22.04 on an x86-64-based computer; however, any operating system that `open5gs` supports should work.

Note: When you're installing Ubuntu, we suggest choosing the "minimal install" option that doesn’t install extra unnecessary software. In prior installs this has led to version conflicts.

## Software Components

As of November 2024, in the [Open5GS software package](https://github.com/open5gs/open5gs), the LTE-specific components (which run on Ubuntu as [systemd](http://manpages.ubuntu.com/manpages/bionic/man1/systemd.1.html) services) are as follows:

* MME - Mobility Management Entity: `open5gs-mmed.service`
* HSS - Home Subscriber Server: `open5gs-hssd.service`
* PCRF - Policy and Charging Rules Function: `open5gs-pcrfd.service`
* SGWC - Serving Gateway Control Plane: `open5gs-sgwcd.service`
* SGWU - Serving Gateway User Plane: `open5gs-sgwud.service`
* PGWC/SMF - Packet Gateway Control Plane / (component contained in Open5GS SMF): `open5gs-smfd.service`
* PGWU/UPF - Packet Gateway User Plane / (component contained in Open5GS UPF): `open5gs-upfd.service`

We would also recommend running the optional WebUI (Web User Interface) service: `open5gs-webui.service`.

The following steps will walk you through this installation process.

# Step 1: Install Open5GS (Notes and Pointers)

Install Open5GS following the [Open5GS Quickstart documentation](https://open5gs.org/open5gs/docs/guide/01-quickstart/) based on your operating system and desired implementation (e.g. "bare metal" directly on the operating system vs. [Docker](https://github.com/wildeyedskies/docker-open5gs-basic-config)).
There are even [VoLTE](https://open5gs.org/open5gs/docs/tutorial/02-VoLTE-setup/) and [Dockerized VoLTE](https://open5gs.org/open5gs/docs/tutorial/03-VoLTE-dockerized/) implementations of Open5GS.
A similar step-by-step tutorial to this one can be found [here](https://medium.com/networkers-fiit-stu/setting-up-open5gs-a-step-by-step-guide-or-how-we-set-up-our-lab-environment-5da1c8db0439).

In SCN we have run Open5GS successfully using Ubuntu 20.04 and 22.04, on bare metal or in Virtual Machines, installed via the `apt` package manager (see Step "2. Install Open5GS with a Package Manager" of the [Quickstart](https://open5gs.org/open5gs/docs/guide/01-quickstart/)).
First install MongoDB as described in the Quickstart. Then follow instructions under the "Ubuntu" section to install Open5GS via apt.

Note: If installing over a `ssh` connection, we recommend using `tmux` or another program in case you get disconnected from the session in the process. 

## Configure MME and SGWU

Note that for our LTE setup, the MME and SGWU are the only components whose config files you will really need to change from the defaults.

### MME
Edit the `/etc/open5gs/mme.yaml` file (as root or using `sudo`) as follows:
- Under `mme:` -> `s1ap:` -> `server:` -> `address:`, set the IP address you will assign to the network interface (likely an ethernet port) on your EPC computer which will be connecting to the eNB. In this tutorial (to match with the Network Configuration section that follows), we will use `192.168.150.1`.
- Under both `mme:` -> `gummei:` and `mme:` -> `tai:`, you will need to change the `plmn_id` (`mcc` and `mnc` values) to match the PLMN you are using for your network.
- Note that for the purposes of eNB config later, the Tracking Area Code (or TAC) listed under `tai:` -> `tac:` will need to match the TAC number configured on the eNB (using the default of 1 is fine). 

**Quick explanation:** "PLMN" refers to the [Public Land Mobile Network](https://en.wikipedia.org/wiki/Public_land_mobile_network), in which every network has to have a unique carrier ID defined by the 3-digit "mobile country code (MCC)" and a 2 or 3-digit "mobile network code (MNC)". Alternately, for iPhone compatibility in the US, SCN uses the CBRS "private LTE" PLMN assigned by Apple as described in [this doc](https://support.apple.com/guide/deployment/support-for-private-5g-and-lte-networks-depac6747317/web).

- Optional: Edit `network_name:` (full and short) and `mme_name:` as desired. One of these names will show up on smartphones' lock screens as the "carrier" when the phone is attached to the network.

### SGWU
Edit the `/etc/open5gs/sgwu.yaml` file (as root or using `sudo`) as follows:
- Under `sgwu:` -> `gtpu:` -> `server:` -> `address:`, set the IP address you will assign to the network interface on your EPC computer which will be connecting to the eNB (this should be the same as the IP address of the MME set above, if the MME and SGWU are running on the same machine). In this tutorial we will use `192.168.150.1`.

As mentioned in the Quickstart, after changing the config files, you will need to restart the corresponding Open5GS daemons:
```bash
sudo systemctl restart open5gs-mmed
sudo systemctl restart open5gs-sgwud
```
However, the MME will likely not start correctly until networking is configured, as described below.

# Step 2: Configure Networking

Remember to follow all the network configuration steps in the [Open5GS Quickstart documentation](https://open5gs.org/open5gs/docs/guide/01-quickstart/). For SCN's Ubuntu machines, this means:

- Allowing IP forwarding on your machine, e.g. via the following command:
```bash
sudo sysctl -w net.ipv4.ip_forward=1
```
- Using Netplan to configure network interfaces with IP addresses in the desired way.
- Setting up NAT rules using `iptables` so that traffic from the eNB can reach the Internet and vice versa

The latter two steps are explained in detail below. 

## Netplan Configuration
### A. Recommended

For this recommended configuration, **we require an EPC
machine with 2 or more ethernet ports** (_in our case_, the ethernet interfaces corresponding to these ports are named enp1s0
and enp4s0). The ethernet port named "enp1s0" is used as the [WAN](https://en.wikipedia.org/wiki/Wide_area_network) port, which accesses upstream networks and eventually the Internet. It is physically connected via an ethernet cable to a router that can give it Internet access (e.g. our ISP's router). The one named "enp4s0" will connect to our private LTE network, and is physically connected via an ethernet cable to the eNB radio. (Our mini-PC model has 4 ethernet ports.)

To enter the appropriate values _in your case_, you will need to figure
out the names of your computer's ethernet interfaces. Use the command `ip a` on the command
line. A list of network interfaces will appear in the terminal. Find the ones
corresponding to your ethernet ports (their names usually start with “eth,”
“enp,” or “enx”).

For Ubuntu 22.04, we're currently using the Netplan program to manage our network configuration.
Create a file in the `/etc/netplan` directory (i.e. a folder) named
`99-open5gs-config.yaml`, and add the following lines, substituting the correct
interface names and subnets for your configuration:

```yaml
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

Note: Netplan will apply configuration files in this directory in the numerical
order of the filename prefix (ie., 00-\*, 01-\*, etc.). Any interfaces
configured in an earlier file will be overwritten by higher-numbered
configuration files, so we create a file with the prefix 99-\* in order to
supersede all other configuration files.

**Quick explanation:**
In order to get Internet connectivity to the EPC, we configure the "upstream" or "WAN" ethernet interface (enp1s0) to request an IP address via [DHCP](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)
from an upstream router it's connected to (as your computer usually does when you plug it into a typical home router), which passes its traffic to and from the global Internet. That's why we have the line `dhcp4: yes` under our interface name `enp1s0`. We don't need this interface to have any other IP addresses.

The "downstream" ethernet interface (enp4s0) connected to the eNB is assigned two IP addresses and
subnets, which are configured statically (_not_ by DHCP, hence the `dhcp4: no`).
In our case, we need this interface to talk to the Baicells Nova 233 eNB we use.
Our eNB has the default local (LAN) IP address of `192.168.150.1`. We also need to set its WAN address (for whatever reason this is required to be different) to `192.168.151.1`, as in [this eNB setup tutorial](https://docs.seattlecommunitynetwork.org/infrastructure/sas-setup). That's why we have the `addresses:` section that sets the static IP addresses of the EPC to `192.168.150.2/24` and `192.168.151.2/24`. Since these IP addresses are in the same subnet as the eNB IP addresses, they will be able to talk to each other automatically _without a router in between_ helping to route communications packets between the two addresses.

Below we also provide an alternate configuration in case you do not yet have a
machine with 2 ethernet ports or a USB to ethernet adapter dongle. However, only
the first configuration is recommended for deployments for security reasons.
**The alternative should be used for testing only**.

### B. NOT Recommended for deployment

If you don’t yet have a machine with 2 ethernet ports or a USB to ethernet
adapter dongle, you can temporarily use a machine with a single ethernet port
along with a simple switch or router. If using a simple switch, you can follow
the same instructions but connect all three of the EPC, eNB, and upstream
Internet router to the switch. If using a router, you may instead need to
configure the router to assign 2 private static IPs to each of the EPC (i.e.
`192.168.150.2`, `192.168.151.2`) and eNB (i.e. `192.168.150.1`,
`192.168.151.1`), such that it will correctly NAT upstream traffic and also
route local traffic between the EPC and eNB.

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

Once this file (or your router configuration) has been modified, restart the
network daemon to apply the configuration changes:

```bash
sudo netplan try
```
and if the Netplan syntax check succeeds, hit the `Enter` or `Return` key to accept the configuration change.

If the eNB will be plugged into its own dedicated EPC ethernet port, as in the
recommended configuration above, you may need to connect that EPC ethernet port
to something (e.g. the eNB, a switch, another machine) via an ethernet cable to
wake the interface up (so that it becomes active and takes on the assigned IP
addresses). This is because the open5gs MME needs to "bind" (or associate) its S1 interface to one of those IP
addresses (in this case `192.168.0.2`). Until those IP addresses exist on your machine,
the MME will continually throw errors if you try to run it.

## Setting `iptables` NAT rules to connect the eNB to the Internet

As explained above, the eNB currently has the IP addresses `192.168.150.1` and `192.168.151.1`-- _[private IP addresses](https://en.wikipedia.org/wiki/Private_network)_ that cannot be used on the public Internet. Therefore, to successfully route the eNB's network traffic to the Internet, we have to add a routing rule in the EPC computer that performs NAT, allowing packets from the eNB's subnet to exit the WAN port of the EPC _masquerading as_ coming from the EPC's IP address to the upstream network.

There might be an easier way to do this, but we've found the cleanest and most reliable way so far to be using the `iptables` command line tool. In the Terminal on the EPC, run the following command to add a NAT rule for the eNB's subnet:

```bash
sudo iptables -t nat -A POSTROUTING -s 192.168.151.0/24 -j MASQUERADE
```

**Quick explanation:** The `-t nat` option tells IPTables to install the rule in the correct "table" containing all the NAT rules, and the `-A` option means we're **A**dding the rule as opposed to **D**eleting it (`-D`). `POSTROUTING` is the "chain," or particular list of rules, that this type of NAT rule should go in (more on that [here](https://rlworkman.net/howtos/iptables/chunkyhtml/c962.html) and in this [diagram](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg) if you're interested). `-s 192.168.151.0/24` means that we're applying this rule to packets from the **S**ource IP addresses described by the subnet `192.168.151.0/24`. `-j MASQUERADE` means the action we'll be **J**umping to as a result of this rule is "masquerading" the source IP address as my EPC's WAN IP address.

### 'Persist' IPTables Configuration

We use IPTables rules to make sure packets are routed correctly within the EPC. IPTables rules must be made persistent across reboots with the `iptables-persistent` package:

```bash
sudo apt install iptables-persistent
```

Installation of this package will save the current iptables rules to its
configuration file, `/etc/iptables/rules.v4`.

Note: `iptables-persistent` reads the contents of this file at boot and applies
all iptables rules it contains. If you need to update the rules, or re-apply
manually, you may use the following commands. This should not be necessary under
normal circumstances:

```bash
sudo iptables-save > /etc/iptables/rules.v4
sudo iptables-restore < /etc/iptables/rules.v4
```

# Step 3: Start and monitor Open5GS software services

Ubuntu’s built-in logging and monitoring services can be used to monitor the core network services. For example, for seeing the output logs of the MME software component we described in the first section, run the following command in the Terminal:

```bash
sudo journalctl -f -u open5gs-mmed.service
```

OR

```bash
sudo systemctl status open5gs-mmed.service
```

_Tab complete may be able to fill in the service name for systemctl at least._

Learning to read output logs is really important for managing software infrastructure! Simply Googling output messages that seem important but that you don't understand can be a good first step to figuring out how a system is working. Another interesting tool to investigate is [Wireshark](https://www.wireshark.org/), which is essentially a graphical user interface (GUI) version of the [tcpdump](https://www.tcpdump.org/) command line tool that can show you the communications [packets](https://en.wikipedia.org/wiki/Network_packet) flowing through the various network cards on your computer.

Here are some more useful commands for managing systemd services, which can be used to start, stop, and reload the software components after you've changed their configuration or they've run into errors and need to be restarted:

```bash
sudo systemctl start open5gs-mmed.service
sudo systemctl stop open5gs-mmed.service
sudo systemctl restart open5gs-mmed.service
sudo systemctl status open5gs-*
```

The following command will start only the systemd services required for LTE. However, you do not need to stop or disable the other components of the 5G core for it to run 4G LTE network hardware correctly- the full Open5GS 5G core is backwards compatible with LTE hardware if you configure the LTE components correctly.

```bash
sudo systemctl start open5gs-hssd.service open5gs-mmed.service open5gs-sgwud.service open5gs-sgwcd.service open5gs-pcrfd.service open5gs-upfd.service open5gs-smfd.service
```

### Install and Start the WebUI

The WebUI is another systemd service and runs by default on your local computer at port 9999. 
It requires some more dependencies to install, such as `nodejs` (see Step "3. Install the WebUI of Open5GS" in the [Quickstart](https://open5gs.org/open5gs/docs/guide/01-quickstart/)). You can reach it by navigating to `http://localhost:9999` in your web browser.

If not already started, start it with the following command:

```bash
sudo systemctl start open5gs-webui.service
```

The default WebUI login credentials are as follows:
- Username : admin
- Password : 1423

# Step 4: Add Users to Open5GS database

(Note that an important pre-condition to adding users is to have SIM cards or eSIMs to give to the users for authentication, along with their respective IMSIs and secret keys to register them onto the EPC. These must be procured separately. 
WIP- We will endeavor to make guides for these processes available soon.)

You can manage users using the Open5GS WebUI, or using a script provided in the [Open5GS GitHub repository](https://github.com/open5gs/open5gs).
Our preferred strategy is to use the script, which supports automation better and does not require the WebUI to be running. 
Clone the repository into the EPC machine:

```bash
git clone https://github.com/open5gs/open5gs.git
```

The script can be found in `misc/db/open5gs-dbctl` from the top level of the repository (`open5gs` folder). For example, you could run a command to add a user like this from within the `open5gs/misc/db` folder:

```bash
sudo ./open5gs-dbctl add 460660003400030 192.168.20.30 0x00112233445566778899AABBCCDDEEFF 0x000102030405060708090A0B0C0D0E0F
```

Running the `./open5gs-dbctl` command on its own will output a list of allowed command syntax, of which the following can be particularly handy:
- `add {imsi key opc}: adds a user to the database with default values`
- `add {imsi ip key opc}: adds a user to the database with default values and a IPv4 address for the UE`
- `remove {imsi}: removes a user from the database`
- `static_ip {imsi ip4}: adds a static IP assignment to an already-existing user`
- `add_ue_with_apn {imsi key opc apn}: adds a user to the database with a specific apn`

The help text also tells you that "default values are as follows: APN "internet", dl_bw/ul_bw 1 Gbps, PGW address is 127.0.0.3, IPv4 only".

# Step 5: Maintenance and Management

## Updating Open5GS
WIP: We are working on an Ansible-based management script for updates and will post updates as they occur.

## Backup and Restore
WIP: We are working on our backup and restore strategies and will update this with a repo soon.

# Deprecated: CoLTE/EPC (LTE Core Network) Setup

Our core networks formerly used the [CoLTE project](https://github.com/uw-ictd/colte) maintained by the [UW ICTD Lab](https://ictd.cs.washington.edu/).

For information on how to install and configure CoLTE, visit the [tutorial](https://docs.colte.network/tutorials/epc-setup.html) we wrote with them, on which this document is based.

# Comments and Feedback
Please get in touch with us at [support@seattlecommunitynetwork.org](mailto:support@seattlecommunitynetwork.org) if you have questions or feedback about this tutorial! We want your feedback so we can make this better.
