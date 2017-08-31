# Power On Auto Provisioning (POAP) on Nexus Switches Part 1 - Setting up POAP Infrastructure

POAP automates the process of device instantiation for Nexus switches. This includes system OS installation and initial configuration, upon first boot of Nexus switches when taking them out of the box and powering them up for the first time. In this lab, you will learn to set up a complete POAP infrastructure and use it to provision Nexus switches. For more details about POAP, please refer to the [POAP documentation.]( https://developer.cisco.com/site/nx-os/docs/automation/poap/index.gsp )

The lab is divided into two parts. In the first part (this lab), we will set up the server side components needed for the POAP process to work successfully on the Nexus switches. In the second part, we will initiate the POAP process on the devices and observe the automatic provisioning process.

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
> Please note that the settings used to generate the DHCP server configurations are configured within the playbook itself: `/root/sbx_nxos/learning_labs/poap/devbox_dhcp_setup.yml`. If you would like to use the playbook in your environment, be sure to update the file with details specific to you.

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

> Please note that the settings for the TFTP server configurations are made in `/root/sbx_nxos/learning_labs/poap/tftp_server/defaults/main.yml`. If you are planning to use this playbook for your environment, please update this file with configuration specific to you.

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

> Please note that the `poap_script.py` is the script that is downloaded and executed on the NXOS device. This is an open-sourced script, available for download at [Cisco's POAP lab repo.](https://github.com/<TBD>) You can use this script directly after updating the `options` dictionary to point it to your internal POAP server.

We've now setup the DHCP and TFTP services preparing to execute the POAP process.  When the switch boots and receives the IP address of the TFTP server and the name of a script to execute, it downloads this script from the TFTP server.

In the next section, we'll examine a sample POAP Python-based script that's hosted on our TFTP server.

> Note: these scripts can also be written in Tool Command Language (TCL).


### Getting the Configuration and Software Server Ready

The next server to get ready is the one that'll host configuration files and OS images that'll be copied down to each switch during the POAP process. Throughout this lab, we often refer to this as a _POAP Server_.

The POAP server we're using in this lab was written using the Python Flask web framework. It is a basic web server that handles HTTP requests from the switch and provides installation information back. It also functions as a HTTP based file server and serves the device configuration. 

> The code to setup the POAP server is open-sourced and can be downloaded from  [Cisco's POAP Lab repo](https://github.com/<TBD>)

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


> Note: If you are planning to use this POAP server for a production deployment, the poap_app.wsgi file can be used to host the app using Apache, on port 80/443


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
    eth_hdw_id: 1

  - hostname: nxos-spine2
    id: aaaa.aaaa.aaab
    mgmt0_ip: 172.16.30.102/24
    lo0_ip: 192.168.0.102/24
    system_image: nxos.7.0.3.I6.1.bin
    kickstart_image: ""
    eth_hdw_id: 2

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
  mac-address fa16.3e00.{{ eth_hdw_id }}001
  no shutdown

interface Ethernet1/2
  description Ethernet1/2
  no switchport
  mac-address fa16.3e00.{{ eth_hdw_id }}002
  no shutdown

interface Ethernet1/3
  description Ethernet1/3
  no switchport
  mac-address fa16.3e00.{{ eth_hdw_id }}003
  no shutdown

interface Ethernet1/4
  description Ethernet1/4
  no switchport
  mac-address fa16.3e00.{{ eth_hdw_id }}004
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

We're now ready to actually see POAP in action, in part 2 of this lab!
