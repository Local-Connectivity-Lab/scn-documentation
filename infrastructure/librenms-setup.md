---
title: LibreNMS Agent Configuration
parent: Infrastructure
---

# Librenms Agent Configuration

## Adding a New Node to LibreNMS

Both the eNodeB and the ePC must be configured individually in order for them to report statistics to the SNMP Manager. Since the eNodeB is not directly accessible from the management VPN, we configure an SNMP proxy on the ePC to pass SNMP statistics to the Management host.

## ePC SNMP Configuration

* Install snmpd to the ePC node:
``` $ sudo apt install snmpd ```

* Modify /etc/snmp/snmpd.conf:

```
sysLocation 	<SITE NAME STRING>
sysContact  	lcl@seattlecommunitynetwork.org
sysServices 	72
master      	agentx
agentAddress	udp:161
com2sec readonly <SNMP Manager IP Address> <SNMP COMMUNITY STRING>
com2sec -Cn ctx_baicells readonly <SNMP Manager IP Address> enodeb
group readonlygroup v2c readonly
view all included .1
access readonlygroup "" v2c noauth exact all none none
access readonlygroup ctx_baicells v2c noauth prefix all none none
proxy -Cn ctx_baicells -v 2c -c private 192.168.151.1 .1.3
```

This configuration allows us to access SNMP data on the EPC with the standard community string (refer to internal standards documentation). but will proxy the Baicells SNMP data when we send the community string ‘enodeb’

* Update the snmpd service file to automatically restart snmpd on crash:
   * Edit /lib/systemd/system/snmpd.service, modify the 'ExecStart' line, and add the 'ExecReload', 'Restart', and 'RestartSec' lines:

```
[Unit]
Description=Simple Network Management Protocol (SNMP) Daemon.
After=network.target
ConditionPathExists=/etc/snmp/snmpd.conf

[Service]
Type=simple
ExecStartPre=/bin/mkdir -p /var/run/agentx
ExecStart=/usr/sbin/snmpd -LO2w -u Debian-snmp -g Debian-snmp -I -smux,mteTrigger,mteTriggerConf -f -p /run/snmpd.pid
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

* Enable and restart snmpd:
```
Sudo systemctl daemon-reload
sudo systemctl enable snmpd
sudo systemctl restart snmpd
```

## Baicells SNMP configuration
* Log into the Baicells configuration console:
```https://<Baicells IP Address>```

* From the left menu, select System

* Select SNMP
![Example Screenshot: enabling SNMP in the Baicells Console](https://i.imgur.com/YanPtMs.png)
   * Under ‘SNMP Switch,’ select ‘Enable’
   * Configure the following options:
      * Community String: private
      * Contact: lcl@seattlecommunitynetwork.org
      * Location: \<SITE NAME STRING\> (String should not have any spaces)
      * Source: Any

## Adding the Node to LibreNMS
* If the ePC is running, librenms should be able to auto-discover it. Run this command from a shell on the management host:
```sudo -u librenms lnms scan```

* LibreNMS should print a status message that it was able to add a new device.

* When first discovered, the ePC will show up generically as it’s ip address. Edit the hostname, but clicking ‘Edit Device’ (gear icon):
   * Click the red pencil icon, and change the ip address to the hostname
   * Fill ‘Overwrite IP’ with the ePC IP address
      * *Note: If the IP is not changed to the hostname, you will not be able to add the eNodeB by it’s IP address*
![Example Screenshot: Updating ePC Hostname in LibreNMS](https://i.imgur.com/LHeL3Zq.png)

* The Baicells eNB needs to be added manually: From LibreNMS, select Devices and click “Add Device”
![Example Screenshot: Manually adding eNodeB to LibreNMS](https://i.imgur.com/Tlqpbh3.png)

* Add a new device, with the following configurations:
   * Hostname: <IP Address of the site’s EPC>
   * Community: ‘enodeb’
   * Force Add: On

* *Note: If you receive an error message stating that a device with the specified IP already exists, make sure that you have* successfully changed the eNodeB’s hostname per the previous step.

* Once the device is added, click the ‘Edit Device’ icon (gear icon) and update the following values:
   * Display name: \<eNB Cell Name\>
   * Overwrite device contact: lcl@seattlecommunitynetwork.org

## Other helpful notes:

* [Baicells eNB config guide](https://img.baicells.com//Upload/20210810/FILE/195c7e84-47d9-4acb-aa00-cba0e080d885.pdf)

* How to SSH into Baicells eNB:
   * SSH using port 27149 (username same as normal web-based login)
   * Convert the MAC address of this eNB to link local address: http://www.sput.nl/internet/ipv6/ll-mac.html

