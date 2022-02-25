---
title: eNodeB and SAS Setup
parent: Infrastructure
---

# LibreNMS Network Manager Configuration

Seattle Community Networks uses SNMP to monitor network nodes. LibreNMS is used for Network Management, Dashboard generation and Alerting.

## LibreNMS Manager Installation:
[Install LibreNMS](https://docs.librenms.org/Installation/Install-LibreNMS/)
[Install and Configure LibreNMS on Ubuntu with nginx](https://computingforgeeks.com/how-to-install-and-configure-librenms-on-ubuntu-with-nginx/)

## Network-Specific Configuration:
Change active user to librenms:
```sudo su - librenms```

Edit /opt/librenms/config.php:

```php
<?php

$config['user'] = 'librenms';
$config['base_url'] = "/";
$config['snmp']['community'] = array('<SNMP COMMUNITY STRING>');
$config['auth_mechanism'] = "mysql"; # default, other options: ldap, http-auth
$config['nets'][] = "10.0.0.0/24"; # Replace with your Management Network Subdomain
$config['rrd_purge'] = 0;
$config['enable_billing'] = 1;
$config['show_services'] = 1;
```

As user 'librenms', run /opt/librenms/snmp-scan.php, to scan the configured network for snmp hosts

## Adding Baicells OS configuration to LibreNMS

As user 'librenms' on the librenms server, create the following files and update their contents accordingly:
* For OS detection, ~librenms/includes/definitions/rts.yaml:
```
	os: rts
		text: 'Baicells RTS'
		type: network
		icon: rts
		over:
		- { graph: device_bits, text: 'Device Traffic' }
		- { graph: device_processor, text: 'CPU Usage' }
		- { graph: device_mempool, text: 'Memory Usage' }
		discovery:
		- sysDescr:
			- 'CELL'
```

* For defining custom RTS OS sensors, ~librenms/includes/definitions/discovery/rts.yaml:

```
mib: BAICELLS-MIB
modules:
	os:
    	hardware: BAICELLS-MIB::hardwareVersion.0
    	serial: BAICELLS-MIB::sn.0
    	version: BAICELLS-MIB::softwareVersion.0
	sensors:
    	count:
        	data:
            	-
                	oid: ulThroughput
                	num_oid: '.1.3.6.1.4.1.53058.190.7.{{ $index }}'
                	descr: 'Upload Throughput'
                	group: 'Throughput'
                	index: 'ulthroughput.{{ $index }}'
            	-
                	oid: dlThroughput
                	num_oid: '.1.3.6.1.4.1.53058.190.8.{{ $index }}'
                	descr: 'Download Throughput'
                	group: 'Throughput'
                	index: 'dlThroughput.{{ $index }}'
            	-
                	oid: ulPrbUtilization
                	num_oid: '.1.3.6.1.4.1.53058.190.9.{{ $index }}'
                	descr: 'Upload PRB Utilization'
                	group: 'Utilization'
                	index: 'ulPrbUtilization{{ $index }}'
            	-
                	oid: dlPrbUtilization
                	num_oid: '.1.3.6.1.4.1.53058.190.10.{{ $index }}'
                	descr: 'Download PRB Utilization'
                	group: 'Utilization'
                	index: 'dlPrbUtilization.{{ $index }}'
    	frequency:
        	data:
            	-
                	oid: carrierBwMhz
                	num_oid: '.1.3.6.1.4.1.53058.100.7.{{ $index }}'
                	divisor: 5
                	descr: 'Carrier Bandwidth'
                	index: 'carrierBwMhz.{{ $index }}'
    	percent:
        	data:
            	-
                	oid: eRABEstablishSuccessRate
                	num_oid: '.1.3.6.1.4.1.53058.190.3.{{ $index }}'
                	descr: 'ERAB Establishment Success Rate'
                	group: 'LTE'
                	index: 'eRABEstablishSuccessRate.{{ $index }}'
            	-
                	oid: hoSuccInterEnbS1Rate
                	num_oid: '.1.3.6.1.4.1.53058.190.4.{{ $index }}'
                	descr: 'Inter MME S1 Handover Success Rate'
                	group: 'LTE'
                	index: 'heSuccInterEnbS1Rate.{{ $index }}'
            	-
                	oid: hoSuccInterEnbRate
                	num_oid: '.1.3.6.1.4.1.53058.190.5.{{ $index }}'
                	descr: 'Inter MME Handover Success Rate'
                	group: 'LTE'
                	index: 'hoSuccInterEnbRate.{{ $index }}'
            	-
                	oid: rrcBuildSuccessRate
                	num_oid: '.1.3.6.1.4.1.53058.190.6.{{ $index }}'
                	descr: 'RRC Build Success Rate'
                	group: 'LTE'
                	index: 'rrcBuildSuccessRate.{{ $index }}'
```

* For defining a custom OS class to use Wireless sensors, ~librenms/LibreNMS/OS/Rts.php (note: pay attention to capitalization)

```php
<?php
namespace LibreNMS\OS;

use LibreNMS\Device\WirelessSensor;
use LibreNMS\Interfaces\Discovery\Sensors\WirelessClientsDiscovery;
use LibreNMS\Interfaces\Discovery\Sensors\WirelessUtilizationDiscovery;
use LibreNMS\OS;

class Rts extends OS implements WirelessClientsDiscovery
{
	public function discoverWirelessClients()
	{
    	$oid = '.1.3.6.1.4.1.53058.100.11.0'; //BAICELLS-MIB::ueConnections.0
    	return array(
        	new WirelessSensor('clients', $this->getDeviceId(), $oid, 'rts', 1, 'UE Connections')
    	);
	}
}
```

* A nice looking logo, ~librenms/html/images/os/rts.png
[Download an example Baicells Logo Here](https://imgur.com/9AOohPr.png)

* Download the baicells mib from [this link](https://na.baicells.com/download/RTS%203.6%20BAICELLS-MIB.mib), and save it to ~librenms/mibs/BAICELLS-MIB (note: no file extension)