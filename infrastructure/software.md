---
title: Our Software
parent: Infrastructure
nav_order: 2
---

# Our Software

Here is a list of the software that we use to deploy, maintain, and plan our network sites.

## Networking

### Local Services
We use the [CoLTE project](https://github.com/uw-ictd/colte) maintained by the University of Washington [ICTD Lab](https://ictd.cs.washington.edu/)
to provide services such as network monitoring, web-based administration, and local web and DNS serving/caching.

### Evolved Packet Core (EPC)
Our EPC is powered by [Open5GS](https://github.com/open5gs/open5gs), an open-source project for 4G and 5G core networks. Currently all of our networks are 4G networks.

### Spectrum Access System (SAS)
We have a partnership with [Google SAS](https://www.google.com/get/spectrumdatabase/sas/) to gain access to CBRS spectrum.

Learn more about our SAS setup [here]({{ "infrastructure/sas-setup.html" | absolute_url }}).

### Network Monitoring and Alerting
We use [LibreNMS](https://www.librenms.org) and SNMPd to monitor our nodes and provide alerting. Our Baicells-specific Network Manager setup is documented [here]({{ "infrastructure/librenms-manager-setup.html" | absolute_url }}), and our instructions for configuring a new node can be found [here]({{"infrastructure/librenms-setup.html" | absolute_url }}).

## Field Measurement

### Network Performance Measurement Tool
The LCL Network Performance Measurement Tool is an Android App in development that will measure a variety of network metrics, including but not limited to ping, upload/download speed, signal strength. We will use this tool to easily capture and upload network metrics
in the field so that we can provide better estimates of what kind of Internet access that our users can expect to receive.

### Network Cell Info Lite
[Network Cell Info Lite](https://play.google.com/store/apps/details?id=com.wilysis.cellinfolite) is an Android App on the Google Playstore that is free to use (with advertisements)
and is capable of taking network metric measurements and recording them to upload. This is an option that we
use but are not satisfied with for many reasons, which is why we are developing our own app.

## Site Planning

### Google Earth
We primarily use the Google Earth Pro [desktop application](https://www.google.com/earth/versions/#earth-pro) to do a rough line-of-sight evaluation. We perform what is called a "viewshed analysis" that allows us to determine what is visible from a specific point on Earth (e.g. a rooftop).

### Ubiquiti Line of Sight
A [web-based line of sight tool](https://link.ui.com/) provided by Ubiquiti that contains helpful altitude data and diagrams.
A drawback is that it is specialized to provide data for Ubiquiti devices only.


## Other resources
### Facebook ISP Toolbox Line of Sight
A [web-based line of sight tool](https://www.facebook.com/isptoolbox/line-of-sight-check/) provided by Facebook Connectivity that utilizes public LiDAR data. Unfortunately LiDAR data for the Seattle area is not present yet, although there is data for some areas in Tacoma.

### Facebook ISP Toolbox Market Evaluator
A [web-based market evaluator](https://www.facebook.com/isptoolbox/market-evaluator/) provided by Facebook Connectivity that can be used
to provide more context about the areas around potential network sites. It offers information about other service providers in the area, average household income, median speeds, and current lowest broadband price available.
