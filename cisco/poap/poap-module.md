
# Understanding Power On Auto Provisioning (POAP) on Nexus Switches

POAP automates the process of device instantiation for Nexus switches. This includes system OS installation and initial configuration, upon first boot of Nexus switches when taking them out of the box and powering them up for the first time. In this lab, you will learn to set up a complete POAP infrastructure and use it to provision Nexus switches.

## Objectives

 - Understand how the POAP process works
 - Understand the requirements for POAP
 - Set up external services required for POAP:
   - DHCP Server
   - TFTP Server
   - Configuration Server
 - Understand how the POAP Python script works that executes on the switch
 - Learn how to walk-through and execute POAP on Nexus switches 

## Prerequisites

The lab makes the following assumptions:
 - The user has an understanding of DHCP, HTTP, and TFTP
 - The user has working knowledge of Python
 - The user has working knowledge of Ansible 
 - The user has knowledge of nano or other Linux based text editors
 
## Understanding the POAP Process

The Cisco Nexus Power On Auto Provisioning (POAP) feature allows the network administrator to bring a new _out-of-the-box_ Nexus switch up, with a desired operating system and configuration.  When a brand new Cisco Nexus switch boots for the first time (or after erasing its startup configuration), it starts the POAP process.

The first step on the POAP process involves the switch going through the DHCP discovery process.  Like any other device that supports DHCP, it receives an IP address from a DHCP server.  The DHCP server assigns the switch an IP address, either from a specific DHCP pool, or a pre-defined IP address (DHCP reservation) based on the MAC address.  In order for the POAP process to continue, the DHCP server must also be configured to set DHCP Option 150, which tells the switch the IP address of the TFTP Server and the name of the Python script found on the TFTP server, that will be used to provision the Nexus switch. 

*Note: The DHCP server can be configured to use either the MAC Address of the interface or a "client-identifier". This identifier could be the Serial Number of the switch. For this option the DHCP server will need to support RFC 6842* 
![POAP Flowchart](images/POAP_flow.png)

The flowchart above illustrates the following steps that are part of the POAP process:

1. The switch is powered on, for the first time (or without a startup configuration) and kicks off the POAP process.

2. If the operator chooses to configure the switch manually, the process is interrupted.

3. The DHCP Process occurs on the switch in which it receives an IP address along with the IP address of the TFTP server and Python script name of which it'll download and execute from the TFTP server.

4. The switch downloads a script (Python, TCL) and executes it (after optionally comparing the MD5 Checksum to ensure integrity) 
   
At a high level here is how our sample Python script (you can create a custom script that meets your requirements) works that gets executed on the Nexus switch:

5. The script essentially retrieves a unique ID of the switch using generic show commands (MAC address or Serial number as examples).  It uses this data in a HTTP or TFTP request it makes to the server that holds switch-specific configurations.  In the response, the server also notifies the switch of it's desired target operations system version.

6. The script then checks whether the current image on the device matches the intent it received from the server. If there was a difference, it downloads the desired image over TFTP and installs it. 

7. Then, the desired configuration (specific for that switch) is downloaded and saved to the switch, but is **not** yet applied.

8. The switch reboots.

9. When the switch powers back up, it has the desired OS, and loads the desired configuration it had previously stored in Step 7.

10. The new configuration is saved into the NVRAM.
   
At this point the POAP process is complete and the device reboots one more time with the desired image and configuration.

We've seen what the high level POAP process is like--let's now take a deeper look into the components required in order to deploy and execute POAP.


## Setting up the POAP Infrastructure 

In order to use POAP for bootstrapping Nexus switches, you'll need a few services deployed.  These include DHCP, TFTP, and a server on which to host configuration files and OS images (which is also a TFTP or HTTP server).

![POAP Architecture](images/POAP_components.png)

In the following sections we will look at each of these components, starting with the DHCP server, and you will perform the tasks required to install and setup each component in order to execute POAP on the four (4) Nexus switches in your pod.

Please note that, though the lab uses a single server to function as
the DHCP, TFTP, and configuration server, these can be different systems
and located anywhere in the network, as long as they are IP reachable
from the switch.  

### Preparing the DHCP server

The DHCP server is a critical cog in the POAP process. It assigns the IP address to the device and also instructs the switch about the TFTP server and name of the POAP script. To set up the DHCP server for this lab, we will be using an open source DHCP server called **ISC-DHCP**.  The dhcpd configuration file located at `/etc/dhcp/dhcpd.conf` is what you'll be configuring.

You will configure the DHCP server by executing a pre-written Ansible playbook. 

After connecting to the DevNet sandbox lab environment, connect to your Linux devbox, and navigate to the `poap` directory where the playbooks are stored.

```
(python2) [root@localhost]# cd /root/sbx_nxos/learning_labs/poap
```

Now execute the Ansible playbook as follows:

```
(python2) [root@localhost poap]# ansible-playbook   devbox_dhcp_setup.yml 
```

After the playbook executes, the file `/etc/dhcp/dhcpd.conf` will be
updated and the DHCP process will be started on the devbox.

Open the file using `nano` or a text editor of your choosing. The relevant sections of the `dhcpd.conf` configuration related to the POAP DHCP process are shown here:

``` 
option tftp-server-address code 150 = ip-address;

subnet 172.16.30.0 netmask 255.255.255.0 {
  option routers 172.16.30.254;
  option tftp-server-address 10.10.20.20;
  option bootfile-name "poap_script.py";
}
host nx-osv9000-1 {
  hardware ethernet aa:aa:aa:aa:aa:aa;
  fixed-address 172.16.30.101;
}
host nx-osv9000-2 {
  hardware ethernet aa:aa:aa:aa:aa:ab;
  fixed-address 172.16.30.102;
}
host nx-osv9000-3 {
  hardware ethernet aa:aa:aa:aa:aa:ac;
  fixed-address 172.16.30.103;
}
host nx-osv9000-4 {
  hardware ethernet aa:aa:aa:aa:aa:ad;
  fixed-address 172.16.30.104;
}

```

You can see in the first line we are setting DHCP option 150, which provides IP addresses of TFTP servers, that we'll define in the actual DHCP scope in the scope definition.

> NOTE: Depending on the DHCP server software these configuration settings could vary.

In the lab DHCP server configuration above, we define `172.16.30.0/24` as the subnet from which IP addresses will be assigned to the Nexus devices. The configuration defines the IP address of the TFTP server, from where to download the POAP script and the name of script, to be downloaded. It also sets the default gateway to `172.16.30.254`.

The `host` section, for each device, defines the reservation binding an IP address to a given device MAC address.

The DHCP server service is now setup and we can move onto setting up the TFTP server process (also on the same server for us).

### Preparing the TFTP server

As we stated previously, once the DHCP process is complete, the switch will have an IP address assigned, and it'll be given the IP address of a TFTP server to download the target Python script.  In this section, we will set up the TFTP server for the lab.

In the same directory you executed the playbook to install and setup DHCP, now execute the following Ansible command to install and setup TFTP:

```
(python2) [root@localhost poap]# ansible-playbook   devbox_tftp_setup.yml
```


This playbook handles setting up the TFTP server configuration, the
necessary Security-Enhanced Linux (SELinux) settings and starting the service. After the playbook executes, use nano to examine at the TFTP configuration file found at `/etc/xinetd.d/tftp`.

``` shell
service tftp
{
  socket_type  = dgram
  protocol     = udp
  wait         = yes
  user         = root
  server       = /usr/sbin/in.tftpd
  server_args  = --secure /root/sbx_nxos/learning_labs/poap/poap_app/templates
  disable      = no
  per_source   = 11
  cps          = 100 2
  flags        = IPv4
}

```

In the example TFTP server configuration above, we are configuring the TFTP server, to serve files from the following directory:

```
/root/sbx_nxos/learning_labs/poap/poap_app/templates
```

This directory contains the POAP script (`poap_script.py`), all device
configurations, and potentially image files (for a real production deployment ) that we want to install on the target system.  Note we aren't dealing with OS images as we're using the virtual Nexus 9000 series switch!

We've now setup the DHCP and TFTP services preparing to execute the POAP process.  When the switch boots and receives the IP address of the TFTP server and the name of a script to execute, it downloads this script from the TFTP server.

In the next section, we'll examine a sample POAP Python-based script that's hosted on our TFTP server.

> Note: these scripts can also be written in Tool Command Language (TCL).


### Getting the Configuration and Software Server Ready

The next server to get ready is the one that'll host configuration files and OS images that'll be copied down to each switch during the POAP process. Throughout this lab, we often refer to this as a _POAP Server_.

The POAP server we're using in this lab was written using the Python Flask web framework. It is a basic web server that handles HTTP requests from the switch and provides installation information back. It also functions as a HTTP based file server and serves the device configuration. 

Let's setup the POAP server. In order to understand the different communication flows, the POAP server will be run in debug mode within the lab. 

The first step is to install the dependencies required for the POAP server.  This includes things like the Flask web framework. This is done by executing the following pre-written Ansible playbook:


```
(python2) [root@localhost poap]# ansible-playbook   devbox_poap_reqs.yml
```


> IMPORTANT: Our POAP server relies on the `/root/sbx_nxos/learning_labs/poap/podvars.yml` file to generate the device specific configurations.  You'll take a look at this in the next section.


You can start the web server by invoking the Python script as follows:

``` shell
(python2) [root@localhost poap]#cd poap_app
(python2) [root@localhost poap_app]#
(python2) [root@localhost poap_app]# python poap_app.py 
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 117-915-351

```

This starts the central web service on the devbox. It listens on port 5000 (by default). Leave this terminal session open for now.  We will come back to this terminal session to observe the debug logs.


> Note: For a production deployment, the poap_app.wsgi file can be used to host the app using Apache, on port 80/443


#### Setting the Configuration Parameters for Your Switches

When a switch starts executing the script, it makes HTTP calls to the POAP server. As stated previously, the POAP server used for this lab uses the `podvars.yml` file to render the specific configuration per device. 

Now, let us look at how we, as the operator, define switch specific settings that we want to apply to each switch.

Open another terminal session on devbox (do not close the one that's running the web server).

In your new terminal window, open up the file `/root/sbx_nxos/learning_labs/poap/poap_app/podvars.yml` using nano or a text editor of your choice.

You'll see this file:

``` yaml
---

pod:
  tftp_server: 10.10.20.20
  http_server: 10.10.20.20:5000
  protocol: http

switches:
  - hostname: nxos-spine1
    id: aaaa.aaaa.aaaa
    mgmt0_ip: 172.16.30.101/24
    lo0_ip: 192.168.0.101/24
    system_image: nxos.7.0.3.I6.1.bin
    kickstart_image: ""

  - hostname: nxos-spine2
    id: aaaa.aaaa.aaab
    mgmt0_ip: 172.16.30.102/24
    lo0_ip: 192.168.0.102/24
    system_image: nxos.7.0.3.I6.1.bin
    kickstart_image: ""

```

The snippet above shows you an example for two devices in the pod. This "settings" file is where you define the parameters that are used by the switches to locate it's image and configuration files (for our configuration/POAP server).

For instance, the IP address of the TFTP and HTTP server and the protocol used to collect the configuration (TFTP/HTTP) are defined under the `pod` _key_.

For the switch specific settings, we  need to know one of two things:

1. The switch's identifying MAC address (MAC on mgmt0) OR
2. The switch's Serial Number

In a real world scenario, it's likely that you might know the serial number from the invoice or shipment manifest. For the lab, since we are working with virtual devices, we are going to use the MAC Address `mac` is used as the _device identifier_.

In the configuration being applied for this lab, we are only setting a few basic parameters for the device:

  * Device hostname
  * IP addresses for the management interface
  * IP address for primary Loopback interface

Since these are virtual devices, the system image is set to match the device image and should not be changed!

These settings in the YAML file are rendered with a Jinja2 template on the POAP server creating complete configuration files as switches are booting and making HTTP requests to the POAP server.

> Jinja2 is a templating engine used commonly in Python applications.

Here is a sample Jinaj2 template:

``` bash
power redundancy-mode combined force

hostname {{ hostname }}
vdc {{ hostname }} id 1
  limit-resource vlan minimum 16 maximum 4094
  limit-resource vrf minimum 2 maximum 4096
  limit-resource port-channel minimum 0 maximum 511
  limit-resource u4route-mem minimum 96 maximum 96



limit-resource u6route-mem minimum 24 maximum 24
  limit-resource m4route-mem minimum 58 maximum 58
  limit-resource m6route-mem minimum 8 maximum 8

feature telnet
feature nxapi
feature bash-shell
feature scp-server

no password strength-check
username admin password 5 $1$KuOSBsvW$Cy0TSD..gEBGBPjzpDgf51  role network-admin
username adminbackup password 5 !  role network-operator
username cisco password 5 $1$Nk7ZkwH0$fyiRmMMfIheqE3BqvcL0C1  role network-operator
username cisco role network-admin
username lab password 5 $1$buoy/oqy$.EXQz8rCn72ii8qtdldj00  role network-admin
ip domain-lookup
snmp-server user lab auth md5 0x5ceb414591539ee35159fca86fdfa101 priv 0x5ceb414591539ee35159fca86fdfa101 localizedkey engineID 128:0:0:9:3:170:170:170:170:170:170
snmp-server user admin auth md5 0x328945d53e05e8e7207f8c20b142f0b7 priv 0x328945d53e05e8e7207f8c20b142f0b7 localizedkey engineID 128:0:0:9:3:170:170:170:170:170:170
snmp-server user cisco auth md5 0x55b3c64a53fb95518e75358ee75e82e9 priv 0x55b3c64a53fb95518e75358ee75e82e9 localizedkey engineID 128:0:0:9:3:170:170:170:170:170:170
rmon event 1 log trap public description FATAL(1) owner PMON@FATAL
rmon event 2 log trap public description CRITICAL(2) owner PMON@CRITICAL
rmon event 3 log trap public description ERROR(3) owner PMON@ERROR
rmon event 4 log trap public description WARNING(4) owner PMON@WARNING
rmon event 5 log trap public description INFORMATION(5) owner PMON@INFO

vlan 1

vrf context management
  ip route 0.0.0.0/0 172.16.30.254
hardware forwarding unicast trace


interface Ethernet1/1
  description Ethernet1/1
  no switchport
  no shutdown

interface Ethernet1/2
  description Ethernet1/2
  no switchport
  no shutdown

interface Ethernet1/3
  description Ethernet1/3
  no switchport
  no shutdown

interface Ethernet1/4
  description Ethernet1/4
  no switchport
  no shutdown

interface mgmt0
  description OOB Management
  vrf member management
  ip address {{ mgmt0_ip }}

interface loopback0
  description Loopback
  ip address {{ lo0_ip }}

line console
line vty
boot nxos bootflash:/{{ system_image }}

```

Take note of all lines that have double-curly brackets.  These are all variables that are replaced by the values we define per switch from the `podvars.yml` file. As you can see, we can influence all aspects of the configuration that will be initially deployed, though in this example we are only updating a few switch configuration CLI commands.

To recap, we have now set up the DHCP, TFTP, and the POAP server (the server that holds the switch configurations) on the devbox. Additionally, we have  configured the `podvars.yml` file and established the desired configurations we would like applied to each switch. 

We're now ready to actually see POAP in action!

## Initiating the POAP Process

With all these pieces in place, we can now fire up our switches and observe POAP in action. 

### Booting up Brand New Switches

We will use a Python script to fire up a pod of brand new Nexus switches. To do this, issue the command `python virl_poap_simulation_setup.py`.

Ensure you're in the following directory:

```
/root/sbx_nxos/learning_labs/poap
```


```shell
(python2) [root@localhost poap]# python virl_poap_simulation_setup.py
Current VIRL Simulation Name: API-Test

Killing VIRL Simulation
200 SUCCESS
Waiting 20 seconds to clear simulation

Launching New Simulation
200 NXOS_POAP
  Nodes not started yet
  Nodes not started yet
  Nodes not started yet
  Nodes not started yet
  Nodes not started yet

Retrieving Console Connection Details: 
    Console to csr1000v-1 -> `telnet 10.10.20.160 17002`
    Console to nx-osv9000-4 -> `telnet 10.10.20.160 17008`
    Console to nx-osv9000-1 -> `telnet 10.10.20.160 17000`
    Console to nx-osv9000-3 -> `telnet 10.10.20.160 17006`
    Console to nx-osv9000-2 -> `telnet 10.10.20.160 17004`
(python2) [root@localhost poap]# telnet 10.10.20.160 17008

```

This will also give you the console information of the devices to connect to. 


In separate terminal sessions,open telnet sessions to each of the switches.  

You can do this using the Linux command for each device:

```
telnet 10.10.20.160 17008
```


Navigate back to the "Flask Console (HTTP Server)" which is the terminal in which you executed `python poap_app.py` earlier.  You should observe the debug logs in this window and correlate them to what you see on the console of each switch.

Here is an example message from the HTTP Server Console:

```
172.16.30.102 - - [27/Jul/2017 14:53:44] "PUT /9US2008H3JF HTTP/1.1" 200 - 
```

Observe the requests coming in from the switches that resemble the one above. Since this is a learning lab, we strategically placed print statements in the `poap_app.py` script such that the output will be displayed in the log, helping us further understand the different pieces of information being processed by the POAP server.

Here are a few of those print statements you will see:

```
{'config_protocol': 'http', 'config_file': u'9US2008H3JF.cfg',
'kickstart_image': '', 'hostname': 'nxos-spine2', 'http_server':
'10.10.20.20:5000', 'tftp_server': '10.10.20.20', 'system_image':
'nxos.7.0.3.I6.1.bin'}
```

In addition, on the console of each switch, you will see log information emitted by the `poap_script.py` script, as it goes through the different checks:

```
2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Using DHCP, information received
   over mgmt0 from 10.10.20.20
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Assigned IP address: 172.16.30.101
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Netmask: 255.255.255.0
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - DNS Server: 208.67.222.222
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Default Gateway: 172.16.30.254
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script Server: 10.10.20.20
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script Name: poap_script.py
   2017 Jul 27 14:04:42 switch %$ VDC-1 %$ %ASCII-CFG-2-CONF_CONTROL: System ready
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] -
   poap_dhcp_intf_ac_action_configuration_success: the script download
   string is [copy tftp://10.10.20.20/poap_script.py bootflash:
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - The POAP Script download has started
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - The POAP Script is being
   downloaded from [copy tftp://10.10.20.20/poap_script.py
   bootflash:scripts/script.sh vrf management ]
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_SCRIPT_DOWNLOADED: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Successfully downloaded POAP script file
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script file size 11998, MD5
   checksum 3de633668b950f555cf3b3b63a992bd9
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - MD5 checksum received from the
   script file is 3de633668b950f555cf3b3b63a992bd9
   2017 Jul 27 14:04:47 switch %$ VDC-1 %$
   %POAP-2-POAP_SCRIPT_STARTED_MD5_VALIDATED:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - POAP script execution started(MD5
   validated)
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Logfile name: /bootflash/20170727140448_poap.log - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Found 1 POAP script logs - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Collecting system S.No and MAC... - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System MAC address is: aaaa.aaaa.aaaa - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System Serial NO is: 9Y782OPGG5Q - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Sending API request to the POAP server 10.10.20.20:5000 - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Requesting http://10.10.20.20:5000/9Y782OPGG5Q... - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Install
   info collected successfully... - script.sh

```
   
As you can see, the switch initiates a HTTP request to the central POAP server. Also please note that, during this process, you have the opportunity to stop POAP at any point allowing you to manually configure the device.


#### A Look into the POAP Script 

You've now seen POAP in action in and should understand that a Python script (ours is called `poap_script.py`) is pulled down from the TFTP server to each Nexus switch during the POAP process--this was Step 4 in the flowchart we saw previously.

> Remember, when we setup the TFTP server, we configured the TFTP server to
serve files from the `/root/sbx_nxos/learning_labs/poap/poap_app/templates/` directory. And when we setup the DHCP server, we defined the script (boot file) for the switch to look for as being `poap_script.py`.

After the script is pulled down by the switch via TFTP, it is executed. The first thing the script does, is reach back out to the central POAP server with information about it's serial number and MAC address. 

So, how does the script know about the IP and port of the POAP server? This is an operator task and a common option is to define this in the Python script. 

Let's take a look at this in the actual Python script.

Open the Python script using a text editor.  It is stored here: 

```
/root/sbx_nxos/learning_labs/poap/poap_app/templates/poap_script.py`
```

The section in the script that would normally need to be updated is the following:

```python
##
# USER INPUT SECTION - DEFINE POAP SERVER IP
#

options = {
    "poap_server": "10.10.20.20",
    "port": "5000"
}

```

For the lab, this has already been set.  For assurance, you can ensure the `poap_server` is set to "10.10.20.20" and the `port` is set to `5000`. 

> Note: If any modification is made to the script, you'll need to execute the `/root/sbx_nxos/learning_labs/poap/poap_app/templates/md5sum_ztp.sh` script. This script updates the md5sum signature within the script. As noted earlier, the switch will compute the md5sum of the downloaded script locally and check if it matches the signature in the script. POAP will fail if there is a mismatch.

You've seen POAP at its finest allowing four switches to boot up and receive base configurations from our POAP server, i.e. the HTTP server that is pushing down configurations based on MAC Address of the switch itself.

This functionality only occurs due to this Python script (`poap_script.py`) that is pulled down from the TFTP server and executed on each switch.  

Let's take a look at a few main components (Python functions) inside this script:


**`poap_collect()`**

This function sends a HTTP PUT request to the central server, using the settings we defined in the `options` dictionary. The PUT request carries the serial number and MAC information back to the POAP server. 

```
2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Collecting system S.No and MAC... - script.sh
2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System MAC address is: aaaa.aaaa.aaaa - script.sh
2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System Serial NO is: 9Y782OPGG5Q - script.sh
2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Sending API request to the POAP server 10.10.20.20:5000 - script.sh
2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Requesting http://10.10.20.20:5000/9Y782OPGG5Q... - script.sh
```

The POAP server uses this info, to collect device specific settings from the `podvars.yml` file and renders the configuration into the `poap_app/templates directory` from where the script was stored. 

You should be able to do a directory listing of this directory and will observe filenames that correspond to the serial number of the devices after this function has executed. 

Here is a listing of the `templates` directory before the `poap_collect()` method was called:

``` shell
(python2) [root@localhost templates]#ls -ltr
total 20
-rwxr-xr-x. 1 root root   183 Jul 26 14:55 md5sum_ztp.sh
-rwxrwxr-x. 1 root root 11812 Jul 28 14:01 poap_script.py
-rw-r--r--. 1 root root  2372 Jul 28 14:03 conf_nxv.j2

```

After the switch initiates the request that contains its MAC and serial number, the server will render the configurations locally in the `template` directory. A subsequent listing will show:

``` shell
(python2) [root@localhost templates]#ls -ltr
total 20
-rwxr-xr-x. 1 root root   183 Jul 26 14:55 md5sum_ztp.sh
-rwxrwxr-x. 1 root root 11812 Jul 28 14:01 poap_script.py
-rw-r--r--. 1 root root  2372 Jul 28 14:03 conf_nxv.j2
-rw-r--r--. 1 root root  2372 Jul 28 15:22 9GA7T4WBRXZ.cfg
-rw-r--r--. 1 root root  2405 Jul 28 15:22 9RT3N4WBTLN.cfg
-rw-r--r--. 1 root root  2405 Jul 28 15:22 9KY5Y4WBGTL.cfg
-rw-r--r--. 1 root root  2405 Jul 28 15:22 9YH7T4WBTIP.cfg
```

The POAP server then responds back to the switch with the following information:
   
```
{'config_protocol': 'http', 'config_file': u'9C3YE9ZC2QS.cfg',
'kickstart_image': '',  'http_server':'10.10.20.20:5000',
'tftp_server': '10.10.20.20', 'system_image':
'nxos.7.0.3.I6.1.bin'}
   
```

   

**`image_install()`**

This function, collects the name of the kickstart and system image provided from the previous function to validate whether the current image on the device matches the desired image. If not, it uses the `tftp_server` information from the previous function, to download the correct image.
   
```
Checking the current image...
Target already matches current image. Skipping....
```
   
**`copy_config()`**

The `copy_config` function will use the protocol defined by the user (HTTP/TFTP) to download the device specific configuration that was generated in step 1. It uses the `config_protocol` to make a determination on the desired protocol to use. Depending on whether that was set to TFTP or HTTP, it uses the information provided by either the `tftp_server` or `http_server` to download the configuration file defined in `config_file`
   
```
Transfering config file over HTTP...
Data collected from HTTP request
Config data copied successfully to bootflash
INFO: Ready to execute terminal dont-ask ;copy bootflash:/9YARXTR3TWL.cfg scheduled-config
```
   

> Note: The copy `scheduled-config` command is used exclusively with POAP. It schedules the configuration that the switch will use on next reload and continue with the POAP process.

```
Waiting for box online to replay poap config
2017 Jul 28 15:14:48 switch %$ VDC-1 %$ %ASCII-CFG-2-CONFIG_REPLAY_STATUS: Bootstrap Replay Done.
```

At this point, the POAP process is complete and the switch will reboot into the desired configuration state.

While we only covered three of the main Python functions in `poap_script.py`, there are many other helper functions within the script that perform logging, cleanup, signal/interrupt handling, and a number of other functions.


## Applying a Base Configuration to an Existing Switch

So far we observed how the POAP process works while bringing up new, out of the box Nexus devices. In this section we will look at the use case of using POAP to re-establish an already existing device, using a baseline configuration. 

For this exercise, we will update the hostname and Loopback interface IP address for the spine1 device.

Open the file `/root/sbx_nxos/learning_labs/poap/poap_app/podvars.yml`.

``` yaml

switches:
  - hostname: nx-osv9000-1  # update this line
    id: aaaa.aaaa.aaaa
    mgmt0_ip: 172.16.30.101/24
    lo0_ip: 192.168.200.201/24  # update this line
    system_image: nxos.7.0.3.I6.1.bin
    kickstart_image: ""


```

The above snippet shows only the modifications made to the switch that was earlier `nxos-spine1`. We have updated the hostname and the loopback0 interface from `nxos-spine1` to `nx-osv9000-1` and the IP of the loopback0 interface from `192.168.0.101/24` to `192.168.200.201/24`

Now log into the switch using the correct console port:

``` shell
telnet 10.10.20.160 17008

```

Now perform a write erase and reload the device. 

```
nxos-spine1# 
nxos-spine1# conf t
Enter configuration commands, one per line. End with CNTL/Z.
nxos-spine1(config)# write erase
Warning: This command will erase the startup-configuration.
Do you wish to proceed anyway? (y/n)  [n] y
nxos-spine1(config)# reload

```


This step will erase the startup configuration on the NVRAM of the device and force it to enter the POAP process as indicated in step 1, on the POAP flowchart.

On the console, you will see the switch reboot and entering the POAP process as before:

```
2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Using DHCP, information received
   over mgmt0 from 10.10.20.20
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Assigned IP address: 172.16.30.101
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Netmask: 255.255.255.0
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - DNS Server: 208.67.222.222
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Default Gateway: 172.16.30.254
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script Server: 10.10.20.20
   2017 Jul 27 14:04:30 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script Name: poap_script.py
   2017 Jul 27 14:04:42 switch %$ VDC-1 %$ %ASCII-CFG-2-CONF_CONTROL: System ready
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] -
   poap_dhcp_intf_ac_action_configuration_success: the script download
   string is [copy tftp://10.10.20.20/poap_script.py bootflash:
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - The POAP Script download has started
   2017 Jul 27 14:04:44 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - The POAP Script is being
   downloaded from [copy tftp://10.10.20.20/poap_script.py
   bootflash:scripts/script.sh vrf management ]
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_SCRIPT_DOWNLOADED: [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Successfully downloaded POAP script file
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - Script file size 11998, MD5
   checksum 3de633668b950f555cf3b3b63a992bd9
   2017 Jul 27 14:04:46 switch %$ VDC-1 %$ %POAP-2-POAP_INFO:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - MD5 checksum received from the
   script file is 3de633668b950f555cf3b3b63a992bd9
   2017 Jul 27 14:04:47 switch %$ VDC-1 %$
   %POAP-2-POAP_SCRIPT_STARTED_MD5_VALIDATED:
   [9Y782OPGG5Q-AA:AA:AA:AA:AA:B1] - POAP script execution started(MD5
   validated)
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Logfile name: /bootflash/20170727140448_poap.log - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Found 1 POAP script logs - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Collecting system S.No and MAC... - script.sh
   2017 Jul 27 14:04:48 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System MAC address is: aaaa.aaaa.aaaa - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: System Serial NO is: 9Y782OPGG5Q - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Sending API request to the POAP server 10.10.20.20:5000 - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Requesting http://10.10.20.20:5000/9Y782OPGG5Q... - script.sh
   2017 Jul 27 14:04:49 switch %$ VDC-1 %$ %USER-1-SYSTEM_MSG: Install
   info collected successfully... - script.sh

```

The POAP server will render the updated baseline configuration for the device, using the information we updated in the `podvars.yml` file. The file will then be downloaded to the switch and copied to the scheduled configuration as before. 

The switch finally boots one last time and will come back online with the updated hostname and Loopback interface IP address.

``` shell
nx-osv9000-1# show running interface loopback0
!Command: show running-config interface loopback0
!Time: Sun Jul 23 12:48:47 2017

version 7.0(3)I6(1)

interface loopback0
  description Loopback
  ip address 192.168.200.201/24

```

Thus we have established how to use the POAP feature of the Cisco Nexus switch, to provision a brand new switch when it boots for the very first time and also how to reestablish a baseline configuration using POAP on an existing device.

You should never have to console into a new switch again!!


