# NETCONF/YANG on Nexus: Part 1 - Learning to use the Cisco NXOS YANG Model

Did you know that a Nexus switch can be represented using one or more data structures?  More specifically, a Nexus switch's configuration and operational state can be represented in various data formats such as JSON and XML, but this data must adhere to the data models that the Nexus switch supports.  In this module, we'll walk through the various types of models supported and take a look at how we communicate and automate Nexus switches using data that adheres to those models!  Let's get started.

You should know by now that data modeling provides a means for defining the schema supported by devices, e.g. configuration and operational state syntax and semantics, and the most pervasive modeling language for networking devices is called YANG--we're focused on describing the different YANG models supported by Nexus in this module! 

Note that the Nexus switches have support for a native, Cisco NXOS specific YANG model and various OpenConfig YANG models. As you'll see in this module, the Cisco specific model is a single model that accounts for all features, but OpenConfig models are more concise in that they are purpose-built for specific features.

While we're only covering NETCONF and YANG models in module, you should also realize that from a programmabilty standpoint, this is independent from the NX-API CLI and NX-API Object Model APIs (although the object model API supports Cisco specific YANG model). For additional information refer to learning labs on those topics. 

From a hands-on perspective in this lab, you'll view and navigate the Cisco specific Nexus model and in Part 2, you'll continue to use this model building out a Layer 3 switch fabric, and finally in Part 3, you'll use OpenConfig YANG data models on Nexus switches.  Additionally, while utilizing XML data that adheres to those models, you'll configure Nexus switches using the NETCONF programmable interface.

## Prerequisites

The user has an understanding and knowledge of the following:
- Complete the Introduction to YANG Data Modeling Devnet learning lab
- Complete the Introduction to the NETCONF Protocol Devnet learning lab 
- Basic Python and Ansible
- Python's ncclient
- Installing and using packages on Linux
- BGP routing protocol

## Objectives

- Learn how to install and setup the proper device packages on Nexus switches to start using NETCONF/YANG.
- Learn how to use the XML representation of the YANG models to configure Nexus switches using NETCONF via the ncclient.
- Understand the difference between the Cisco NXOS YANG model and OpenConfig YANG models, both of which Nexus supports (after Part 1, Part 2, and Part 3).

**For an introduction to NETCONF, YANG, and ncclient please see the learning labs focused on these technologies**

## Preparing the Switches

**STOP:** You should ensure that the NXOS Sandbox lab is reserved before starting as it takes a few minutes to get started!

### Copying the Installation Packages to the Nexus Switches

In order to work with the data model, we will first need to install the necessary software on the devices. The software installation RPMs are available for download, for free, at
[the Cisco Artifactory](https://devhub.cisco.com/artifactory/open-nxos-agents/)
*Note: No CCO login required*

There are three types of RPM packages to be aware of:

- NX-OS Programmable Interface Infrastructure Components
- Common Model Components
- Agent Components

Let's take a look at each so you can better understand them.

The NX-OS Programmable Interface Infrastructure comprises of two Linux RPM packages:

- **mtx-infra** — This RPM is platform-independent.
- **mtx-device-model** — This RPM is platform-dependent and must be selected to match the installed NX-OS image at the time of Cisco Artifactory download.


Common Model component RPMs provides support for Openconfig and IETF defined models. In order to enable support for one or more desired Common Models, the associated Common Model component RPMs must be downloaded and installed. 

As with the Cisco specific model (**mtx-device-model**), Common Model components are also platform-dependent and must be selected to match the installed NX-OS image at the time of Cisco Artifactory download.  You'll install a given common model such as a OpenConfig (OC) BGP model should you want to use the OC model instead of the Cisco Nexus model.

Finally, three agent packages are available: NETCONF, RESTConf and gRPC. At least one agent must be installed in order to have access to the modeled NX-OS interface. For our lab, we will be working with the NETCONF agent exclusively.


Login to the devbox to download the necessary packages. Run the following ansible playbook from the `yang-prereqs` directory:

``` shell
(python2) [root@localhost sbx_nxos]# cd /root/sbx_nxos/learning_labs/yang/yang-prereqs
(python2) [root@localhost yang-prereqs]# ansible-playbook get_rpms.yml
PLAY [ENSURE THAT THE NXOS RPMS ARE AVAILABLE] *********************************

TASK [setup] *******************************************************************
ok: [10.10.20.20]

TASK [CREATE THE NXOS RPMS DIRECTORY] ******************************************
changed: [10.10.20.20]

TASK [DOWNLOAD THE CISCO ARTIFACTORY RPMs] *************************************
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm)
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-infra-1.0.0-r1705191346.x86_64.rpm)
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm)
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm)
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm)
changed: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm)

PLAY RECAP *********************************************************************
10.10.20.20                : ok=3    changed=2    unreachable=0    failed=0   



```

This playbook creates a directory `/root/sbx_nxos/learning_labs/yang/nxos_rpms/` and downloads the required software.

Take a look by navigating within the devbox.

``` shell
(python2) [root@localhost ~]# cd /root/sbx_nxos/learning_labs/yang/nxos_rpms/
(python2) [root@localhost nxos_rpms]# 
(python2) [root@localhost nxos_rpms]# ls -l
total 64272
-rw-r--r--. 1 root root 15231416 May 19 23:21 mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm
-rw-r--r--. 1 root root 10420941 May 19 23:22 mtx-infra-1.0.0-r1705191346.x86_64.rpm
-rw-r--r--. 1 root root  2923015 May 19 23:23 mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm
-rw-r--r--. 1 root root 21075904 May 24 19:17 mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm
-rw-r--r--. 1 root root 10645230 May 24 19:17 mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm
-rw-r--r--. 1 root root  5505579 May 24 19:17 mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm
(python2) [root@localhost nxos_rpms]# 

```

Now from the devbox, log into **nx-osv9000-1** and **nx-osv9000-2** and copy the NETCONF agent and the programmable interface infrastructure packages.  

You'll be using SCP to copy the files from the devbox directly to each switch.

``` shell

### Copy the programmable interface infrastructure packages:

nx-osv9000-1#copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm bootflash: vrf management

nx-osv9000-1#copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-infra-1.0.0-r1705191346.x86_64.rpm bootflash: vrf management

# Copy the NETCONF agent package

nx-osv9000-1#copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm bootflash: vrf management

```

You now need to copy the same files over to **nx-osv9000-2**.

Now that the RPMs are copied over, let's install them.

### Installing the Packages

In order to install the packages that were just copied over, we need to use the **bash shell** Linux environment that exists on each switch. 

Now log into the **both** switches using separate terminal sessions. Then execute the following commands to enable and switch to the bash shell on each of the switches:

``` shell
nx-osv9000-1(config)#feature bash-shell
nx-osv9000-1(config)#
nx-osv9000-1(config)#run bash sudo su 
bash-4.2#

```

On **each** device, navigate to the `bootflash` directory, where we copied the RPMs over and install them using the `yum` package manager.

Install the infrastructure components (remember, this one is platform independent):

> Don't forget, this needs to be done for **nx-osv9000-1** and **nx-osv9000-2**.

``` shell
bash-4.2# cd /bootflash/
bash-4.2# 
bash-4.2# yum install mtx-infra-1.0.0-r1705191346.x86_64.rpm
Loaded plugins: downloadonly, importpubkey, localrpmDB, patchaction, patching, protect-packages
groups-repo                                                                                                                                    | 1.1 kB     00:00 ... 
localdb                                                                                                                                        |  951 B     00:00 ... 
patching                                                                                                                                       |  951 B     00:00 ... 
thirdparty                                                                                                                                     |  951 B     00:00 ... 
Setting up Install Process
Examining mtx-infra-1.0.0-r1705191346.x86_64.rpm: mtx-infra-1.0.0-r1705191346.x86_64
Marking mtx-infra-1.0.0-r1705191346.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mtx-infra.x86_64 0:1.0.0-r1705191346 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================================================================
 Package                         Arch                         Version                                 Repository                                                 Size
======================================================================================================================================================================
Installing:
 mtx-infra                       x86_64                       1.0.0-r1705191346                       /mtx-infra-1.0.0-r1705191346.x86_64                        33 M
 
Transaction Summary
======================================================================================================================================================================
Install       1 Package

Total size: 33 M
Installed size: 33 M
Is this ok [y/N]: y
Downloading Packages:
Running Transaction Check
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : mtx-infra-1.0.0-r1705191346.x86_64                                                                                                                 1/1 

Installed:
  mtx-infra.x86_64 0:1.0.0-r1705191346                                                                                                                                

Complete!

bash-4.2# 
```


Next we'll install the Cisco NXOS device YANG model (that is platform dependent) on both **nx-osv9000-1** and **nx-osv9000-2**:

``` shell
bash-4.2# yum install mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm 
Loaded plugins: downloadonly, importpubkey, localrpmDB, patchaction, patching, protect-packages
groups-repo                                                                                                                                    | 1.1 kB     00:00 ... 
localdb                                                                                                                                        |  951 B     00:00 ... 
patching                                                                                                                                       |  951 B     00:00 ... 
thirdparty                                                                                                                                     |  951 B     00:00 ... 
thirdparty/primary                                                                                                                             | 2.0 kB     00:00 ... 
thirdparty                                                                                                                                                        8/8
Setting up Install Process
Examining mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm: mtx-device-1.0.0-r1705191346.x86_64
Marking mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mtx-device.x86_64 0:1.0.0-r1705191346 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================================================================
 Package                      Arch                     Version                                Repository                                                         Size
======================================================================================================================================================================
Installing:
 mtx-device                   x86_64                   1.0.0-r1705191346                      /mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64                    52 M

Transaction Summary
==========================
============================================================================================================================================
Install       1 Package

Total size: 52 M
Installed size: 52 M
Is this ok [y/N]: Y
Downloading Packages:
Running Transaction Check
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : mtx-device-1.0.0-r1705191346.x86_64                                                                                                                1/1 

Installed:
  mtx-device.x86_64 0:1.0.0-r1705191346                                                                                                                               

Complete!


bash-4.2# 

```

> For any given technology, more than one model can be installed on the device (as we will see, when we learn about OpenConfig YANG models in Part 3). Unique namespaces are used to distinguish the different installed models.  For example, you can install the OpenConfig BGP model and have the NXOS model that supports BGP installed at the same time.

Next install the NETCONF agent on both **nx-osv9000-1** and **nx-osv9000-2**:

``` shell
bash-4.2# yum install mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm 
Loaded plugins: downloadonly, importpubkey, localrpmDB, patchaction, patching, protect-packages
groups-repo                                                                                                                                    | 1.1 kB     00:00 ... 
localdb                                                                                                                                        |  951 B     00:00 ... 
patching                                                                                                                                       |  951 B     00:00 ... 
thirdparty                                                                                                                                     |  951 B     00:00 ... 
thirdparty/primary                                                                                                                             | 2.1 kB     00:00 ... 
thirdparty                                                                                                                                                        9/9
Setting up Install Process
Examining mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm: mtx-netconf-agent-1.0.1-r1705191346.x86_64
Marking mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mtx-netconf-agent.x86_64 0:1.0.1-r1705191346 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

======================================================================================================================================================================
 Package                             Arch                     Version                             Repository                                                     Size
======================================================================================================================================================================
Installing:
 mtx-netconf-agent                   x86_64                   1.0.1-r1705191346                   /mtx-netconf-agent-1.0.1-r1705191346.x86_64                    11 M

Transaction Summary
======================================================================================================================================================================
Install       1 Package

Total size: 11 M
Installed size: 11 M
Is this ok [y/N]: y
Downloading Packages:
Running Transaction Check
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : mtx-netconf-agent-1.0.1-r1705191346.x86_64                                                                                                         1/1 
Please run the following command to start the netconf service:
netconfctl start

Installed:
  mtx-netconf-agent.x86_64 0:1.0.1-r1705191346                                                                                                                        

Complete!
bash-4.2# 


```

This completes the installation of the programmable infrastructure interface and NETCONF agent packages.  Next, we'll enable NETCONF on the switch so we can start building data structures that use one of the models we installed on the switch.

### Starting the NETCONF agent on the Nexus Switches

NETCONF is one of the 3 supported programmatic model-driven APIs available on the Nexus devices.  The agent installed in the last step will now act as a SSH sub-system that has the ability to leverage the models you also previously installed ![PII Architecture](images/PII_infra.jpg). 

On **nx-osv9000-1** and **nx-osv9000-2**, while still in the `bash` prompt execute the following command to start the NETCONF server (`netconfctl start`):

``` shell
bash-4.2# netconfctl start
Starting Netconf Agent: [OK]
bash-4.2# 
```

Now the NETCONF agent has been started, it will listen on TCP/830 on the switches even though it's still using SSH as its transport protocol.

> You need to start the netconf server only once. The server is automatically started for subsequent device reboots.

## Preparing the Control Machine (devbox)

The control machine (devbox) is where we will use a NETCONF client, allowing us to interact with the YANG models through the NETCONF agent interface. 

### The ncclient Python Library

[`ncclient`](https://ncclient.readthedocs.io/en/latest/) is an opensource Python-based NETCONF client, that maps the XML-encoded nature of NETCONF to Python constructs and idioms, and vice versa.  It's the most popular and common way in Python to write scripts communicating with NETCONF-enabled devices.


### The pyang Python library

The [`pyang`](http://www.yang-central.org/twiki/pub/Main/YangTools/pyang.1.html) Python library is an opensource YANG validator and general tool to view and also convert models to different representations. 

We will use `pyang` as a learning tool to help visualize the YANG models for this lab.


### Installing the library files

We will now install `ncclient` and `pyang`. Navigate to the `yang-prereqs` directory

``` shell
(python2) [root@localhost nxos_rpms]# cd /root/sbx_nxos/learning_labs/yang/yang-prereqs
(python2) [root@localhost yang-prereqs]#
```
Execute pip to install the requirements

``` shell
(python2) [root@localhost yang-prereqs]# pip install -r yang-requirements.txt 
Collecting ncclient (from -r yang-requirements.txt (line 1))
  Downloading ncclient-0.5.3.tar.gz (63kB)
    100% |████████████████████████████████| 71kB 771kB/s 
Collecting pyang (from -r yang-requirements.txt (line 2))
  Downloading pyang-1.7.3-py2.py3-none-any.whl (326kB)
    100% |████████████████████████████████| 327kB 2.0MB/s 
Requirement already satisfied: setuptools>0.6 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: paramiko>=1.15.0 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from ncclient->-r yang-requirements.txt (line 1))
Collecting lxml>=3.3.0 (from ncclient->-r yang-requirements.txt (line 1))
  Downloading lxml-3.8.0-cp27-cp27m-manylinux1_x86_64.whl (6.8MB)
    100% |████████████████████████████████| 6.8MB 139kB/s 
Requirement already satisfied: six in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: pyasn1>=0.1.7 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: cryptography>=1.1 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: idna>=2.1 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: asn1crypto>=0.21.0 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: enum34 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: ipaddress in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: cffi>=1.7 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Requirement already satisfied: pycparser in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from cffi>=1.7->cryptography>=1.1->paramiko>=1.15.0->ncclient->-r yang-requirements.txt (line 1))
Building wheels for collected packages: ncclient
  Running setup.py bdist_wheel for ncclient ... done
  Stored in directory: /root/.cache/pip/wheels/86/30/68/153d65b60834981c1960737f3f2de488574ba5355fe1329558
Successfully built ncclient
Installing collected packages: lxml, ncclient, pyang
Successfully installed lxml-3.8.0 ncclient-0.5.3 pyang-1.7.3
(python2) [root@localhost yang-prereqs]# 

```


## Initial Interactions with the Cisco NXOS YANG Model

When we installed the **mtx-device** package on the switches, it installed the Cisco NXOS YANG model. We then installed the NETCONF agent and started the agent on the switch. 

In this section we will use the `ncclient` library and Python scripts to interact with the NETCONF agent, running on the switches, using XML data that adheres to the NXOS YANG model. 

Open a new terminal session to the devbox and navigate to the to the `/root/sbx_nxos/learning_labs/yang/01-yang` directory:


``` shell
(python2) [root@localhost yang]# cd /root/sbx_nxos/learning_labs/yang/01-yang/
(python2) [root@localhost 01-yang]# ls -l
total 28
-rw-r--r--. 1 root root 2207 Aug  4 18:49 add_loopback_full.py
-rw-r--r--. 1 root root 1942 Aug  4 18:32 add_loopback_ip.py
-rw-r--r--. 1 root root 1309 Aug  4 18:07 add_loopback.py
-rw-r--r--. 1 root root 1639 Aug  3 20:45 add_vlans.py
-rw-r--r--. 1 root root  901 Aug  3 11:16 get_capabilities.py
-rw-r--r--. 1 root root 1320 Aug  3 16:01 get_serial.py
-rw-r--r--. 1 root root  990 Aug  3 16:59 update_hostname.py

```

Included in the sample code for this lab is a Python script called `get_capabilities.py`. 

Execute this script with the command `python get_capabilities.py`.


```
(python2) [root@localhost 01-yang]# python get_capabilities.py                                                                                                        

***Remote Devices Capabilities for device (nx-osv9000-1) 172.16.30.101***

urn:ietf:params:netconf:capability:writable-running:1.0
urn:ietf:params:netconf:capability:rollback-on-error:1.0
urn:ietf:params:netconf:capability:confirmed-commit:1.1
urn:ietf:params:netconf:capability:validate:1.1
urn:ietf:params:netconf:base:1.0
urn:ietf:params:netconf:base:1.1
urn:ietf:params:netconf:capability:candidate:1.0
http://cisco.com/ns/yang/cisco-nx-os-device

***Remote Devices Capabilities for device (nx-osv9000-2) 172.16.30.102***

urn:ietf:params:netconf:capability:writable-running:1.0
urn:ietf:params:netconf:capability:rollback-on-error:1.0
urn:ietf:params:netconf:capability:confirmed-commit:1.1
urn:ietf:params:netconf:capability:validate:1.1
urn:ietf:params:netconf:base:1.0
urn:ietf:params:netconf:base:1.1
urn:ietf:params:netconf:capability:candidate:1.0
http://cisco.com/ns/yang/cisco-nx-os-device
(python2) [root@localhost 01-yang]# 

```

> *We learned from the devnet learning lab on NETCONF that a "capability" is simply a reference to a Data Model that is supported and that withing XML they are referred to as "namespaces" (in addition to the core NETCONF capabilities).*

Pay attention in the `http://cisco.com/ns/yang/cisco-nx-os-device` namespace. This references the Cisco NXOS YANG model on the Nexus. The `get_capabilities.py` is shown below. This is similar to the code we have already seen in the introduction to the NETCONF protocol devenet learning lab.


```python
#!/usr/bin/env python

from ncclient import manager
import sys

# Set the device variables

DEVICES = ['172.16.30.101', '172.16.30.102']
DEVICE_NAMES = {'172.16.30.101': '(nx-osv9000-1)',
                '172.16.30.102': '(nx-osv9000-2)' }
USER = 'admin'
PASS = 'admin'
PORT = 830

# create a main() method
def main():
    """
    Main method that prints netconf capabilities of remote device.
    """
    for device in DEVICES:
        with manager.connect(host=device, port=PORT, username=USER,
                             password=PASS, hostkey_verify=False,
                             device_params={'name': 'nexus'},
                             look_for_keys=False, allow_agent=False) as m:

            # print all NETCONF capabilities
            print('\n***Remote Devices Capabilities for device {}  {}***\n'.format(DEVICE_NAMES[device], device))
            for capability in m.server_capabilities:
                print(capability.split('?')[0])
                

if __name__ == '__main__':
    sys.exit(main())

```

## Exploring the Cisco NXOS YANG Model

From the execution of the script in the previous step, we identified the NXOS model namespace as `http://cisco.com/ns/yang/cisco-nx-os-device`. 

Let's use `pyang`to visualize this model.

First, it's worth noting that all supported models for Nexus are posted to GitHub at https://github.com/YangModels/yang[https://github.com/YangModels/yang] and specifically within the `cisco/nx` sub-directory, which we'll see below.

On the devbox, navigate to the `yang` directory and clone this repository:

``` shell
(python2) [root@localhost yang]# cd /root/sbx_nxos/learning_labs/yang
(python2) [root@localhost yang]# git clone https://github.com/YangModels/yang
Cloning into 'yang'...
remote: Counting objects: 8057, done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 8057 (delta 6), reused 11 (delta 2), pack-reused 8027
Receiving objects: 100% (8057/8057), 15.89 MiB | 8.20 MiB/s, done.
Resolving deltas: 100% (4851/4851), done.
(python2) [root@localhost yang]#
```

This git repository contains YANG models of different vendors and open/IETF models as well. 

Navigate to the NXOS version that is running on the devices in our lab:

```
(python2) [root@localhost yang]# cd yang/vendor/cisco/nx/7.0-3-I6-1/
(python2) [root@localhost 7.0-3-I6-1]# 
(python2) [root@localhost 7.0-3-I6-1]# ls -l
total 4432
-rwxr-xr-x. 1 root root    2153 Jul 31 20:48 check-models.sh
-rw-r--r--. 1 root root   51450 Jul 31 20:48 cisco-nx-openconfig-bgp-deviations.yang
-rw-r--r--. 1 root root     741 Jul 31 20:48 cisco-nx-openconfig-bgp-multiprotocol-deviations.yang
-rw-r--r--. 1 root root   60073 Jul 31 20:48 cisco-nx-openconfig-if-ip-deviations.yang
-rw-r--r--. 1 root root    2193 Jul 31 20:48 cisco-nx-openconfig-local-routing-deviations.yang
-rw-r--r--. 1 root root   15224 Jul 31 20:48 cisco-nx-openconfig-routing-policy-deviations.yang
-rw-r--r--. 1 root root 4028679 Jul 31 20:48 Cisco-NX-OS-device.yang
-rw-r--r--. 1 root root   35660 Jul 31 20:48 iana-if-type.yang
-rw-r--r--. 1 root root   16833 Jul 31 20:48 ietf-inet-types.yang
-rw-r--r--. 1 root root   26325 Jul 31 20:48 ietf-interfaces.yang
-rw-r--r--. 1 root root   17939 Jul 31 20:48 ietf-yang-types.yang
-rw-r--r--. 1 root root    2003 Jul 31 20:48 netconf-capabilities.xml
-rw-r--r--. 1 root root   19880 Jul 31 20:48 openconfig-bgp-multiprotocol.yang
-rw-r--r--. 1 root root   11560 Jul 31 20:48 openconfig-bgp-operational.yang
-rw-r--r--. 1 root root   11955 Jul 31 20:48 openconfig-bgp-types.yang
-rw-r--r--. 1 root root   31151 Jul 31 20:48 openconfig-bgp.yang
-rw-r--r--. 1 root root    2124 Jul 31 20:48 openconfig-extensions.yang
-rw-r--r--. 1 root root    4436 Jul 31 20:48 openconfig-if-aggregate.yang
-rw-r--r--. 1 root root    7902 Jul 31 20:48 openconfig-if-ethernet.yang
-rw-r--r--. 1 root root    3596 Jul 31 20:48 openconfig-if-ip-ext.yang
-rw-r--r--. 1 root root   25765 Jul 31 20:48 openconfig-if-ip.yang
-rw-r--r--. 1 root root   29064 Jul 31 20:48 openconfig-interfaces.yang
-rw-r--r--. 1 root root   10962 Jul 31 20:48 openconfig-local-routing.yang
-rw-r--r--. 1 root root    4320 Jul 31 20:48 openconfig-policy-types.yang
-rw-r--r--. 1 root root   27020 Jul 31 20:48 openconfig-routing-policy.yang
-rw-r--r--. 1 root root    2895 Jul 31 20:48 openconfig-types.yang
-rw-r--r--. 1 root root    5272 Jul 31 20:48 openconfig-vlan-types.yang
-rw-r--r--. 1 root root    9983 Jul 31 20:48 openconfig-vlan.yang
-rw-r--r--. 1 root root   13176 Jul 31 20:48 README.md
(python2) [root@localhost 7.0-3-I6-1]# 

```

As you can see from the directory listing, there are multiple YANG models defined for the NXOS--each with it's own namespace.  Since we are working with the Cisco NXOS model for now, we will focus our attention for the time being on `Cisco-NX-OS-device.yang`

Within the `7.0-3-I6-1` directory, issue the following command: 

``` shell
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree Cisco-NX-OS-device.yang -o /tmp/nxos_native.txt
# output omitted and shown below
```

This is going to print the "tree" output of the NXOS device model to a file stored in `/tmp/nxos_native.txt`.

Now open the file `/tmp/nxos_native.txt` using vi or nano. This output is a visualization of the NXOS YANG model for the NXOS device. 

You'll see based on the output that YANG models are hierarchical, and this particular model is rooted in a container called `System`:

``` shell
module: Cisco-NX-OS-device
    +--rw System
       +--rw id?                         uint32
       +--ro mode?                       top_Mode
       +--rw address?                    address_Ipv4
       +--ro oobMgmtAddr?                address_Ipv4
       +--ro inbMgmtAddr?                address_Ipv4
       +--rw role?                       top_NodeRole
       +--ro currentTime?                uint64
       +--ro systemUpTime?               uint64
       +--ro serial?                     eqpt_Serial
       +--ro podId?                      top_PodId
       +--ro fabricId?                   top_FabricId
       +--ro fabricMAC?                  top_fabricMacAddr
       +--ro state?                      top_SystemSt
       +--ro configIssues?               top_ConfigIssues
       +--rw name?                       naming_Name
       +--rw bgp-items
       |  +--rw name?         naming_Name
       |  +--rw adminSt?      nw_AdminSt
# output omitted for brevity
...
```

Let's now "use" this model and try and collect the serial number of the devices.  By using this model, it simply means we are going to pass an XML object to the device that adheres to this YANG model.

As we saw in the devnet learning lab for YANG, we can "interact" with this YANG model by constructing an equivalent `XML` filter and using that with the `ncclient` Python library over the NETCONF protocol.

Based on the tree output above, our `xml` filter would look like this

``` python
serial_number = """
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
      <serial/>
    </System>
    """
```

You could also have done the following:

``` python
serial_number = """
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
      <serial>
      </serial>
    </System>
    """
```


This follows the YANG tree, in that we are filtering for a YANG "leaf" node called `serial` that is rooted under the "container" `System`. 

*Note:The concepts of leafs and containers are explored in the YANG learning devnet lab.*

> YANG models can effectively be thought of as a documentation/definition, or _schema_, of a particular property of the device we are interested in interacting with.

Let's continue and actually make a request to the device.


## Gathering Serial Numbers with a NETCONF GET Operation

We're now going to collect the serial number of the devices using the filter we created above using a pre-built script called `get_serial.py` from the sample code directory `01-yang`.  

You'll have to navigate back to the `01-yang` directory:

``` shell
(python2) [root@localhost 01-yang]# ls -l
total 28
-rw-r--r--. 1 root root 2207 Aug  4 18:49 add_loopback_full.py
-rw-r--r--. 1 root root 1942 Aug  4 18:32 add_loopback_ip.py
-rw-r--r--. 1 root root 1309 Aug  4 18:07 add_loopback.py
-rw-r--r--. 1 root root 1639 Aug  3 20:45 add_vlans.py
-rw-r--r--. 1 root root  901 Aug  3 11:16 get_capabilities.py
-rw-r--r--. 1 root root 1320 Aug  3 16:01 get_serial.py
-rw-r--r--. 1 root root  990 Aug  3 16:59 update_hostname.py
(python2) [root@localhost 01-yang]# 

```

Now execute the `get_serial.py` script.  You'll see the following (with changes in serial number):

```
(python2) [root@localhost 01-yang]# python get_serial.py 
The serial number for (nx-osv9000-1) 172.16.30.101 is 9PRREICVAR0
The serial number for (nx-osv9000-2) 172.16.30.102 is 936GDP2U9D2
(python2) [root@localhost 01-yang]# 

```

Using nano or any other text editor open the `get_serial.py` file to understand the code:


```python
#!/usr/bin/env python

import sys
from ncclient import manager
from lxml import etree

# Set the device variables

DEVICES = ['172.16.30.101', '172.16.30.102']
DEVICE_NAMES = {'172.16.30.101': '(nx-osv9000-1)',
                '172.16.30.102': '(nx-osv9000-2)' }
USER = 'admin'
PASS = 'admin'
PORT = 830


# create a main() method
def main():
    """
    Main method that prints NETCONF capabilities of remote device.
    """

    serial_number = """
        <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
          <serial/>
        </System>
    """


    for device in DEVICES:
        with manager.connect(host=device, port=PORT, username=USER,
                             password=PASS, hostkey_verify=False,
                             device_params={'name': 'nexus'},
                             look_for_keys=False, allow_agent=False) as m:

            # Collect the NETCONF response
            netconf_response = m.get(('subtree', serial_number))
            # Parse the XML and print the data
            xml_data = etree.fromstring(netconf_response.data_xml)
            xml_tree = etree.ElementTree(xml_data)
            # Create Namespace Map
            ns = {"ns": "http://cisco.com/ns/yang/cisco-nx-os-device"}
            serial = xml_tree.find("//ns:serial", ns).text

            print("The serial number for {} {} is {}".format(DEVICE_NAMES[device], device, serial))
            

if __name__ == '__main__':
    sys.exit(main())

```


**The key takeaway here is that, we are able to execute a simple get request (NETCONF GET operation) and collect the devices' serial numbers, by using a XML representation of the Cisco NXOS YANG model.**


## Changing the Hostname with a NETCONF EDIT Operation

Device configurations are made by performing NETCONF edit operations.  The only difference to the GET operations is that instead of passing a _filter_ to query data, you use an XML string (or object), to configure the device.  In both cases, the XML data must adhere to the YANG model. Using the tree visualization produced by `pyang`, we will now write a a script to modify the hostname of both switches. 


Once again, looking at the saved file `/tmp/nxos_native.txt`, we find:

``` shell
module: Cisco-NX-OS-device
    +--rw System
       +--rw id?                         uint32
       +--ro mode?                       top_Mode
       +--rw address?                    address_Ipv4
       +--ro oobMgmtAddr?                address_Ipv4
       +--ro inbMgmtAddr?                address_Ipv4
       +--rw role?                       top_NodeRole
       +--ro currentTime?                uint64
       +--ro systemUpTime?               uint64
       +--ro serial?                     eqpt_Serial
       +--ro podId?                      top_PodId
       +--ro fabricId?                   top_FabricId
       +--ro fabricMAC?                  top_fabricMacAddr
       +--ro state?                      top_SystemSt
       +--ro configIssues?               top_ConfigIssues
       +--rw name?                       naming_Name

```

The "read-write" leaf called `name` directly under the `System` root represents the hostname of the device. Now we can model the filter to update the hostname as:


``` python
new_name = """
    <config>
      <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
        <name>nxos-spine1</name>
      </System>
    </config>
    """
```

> Note: all configuration objects are going to be encompassed within the `<config>` and `</config>` tags.

In the sample code directory `01-yang`, execute the `update_hostname.py` script. 

> This script uses the XML string, to update the hostname of the `nx-osv9000-1` device to `nxos-spine1`:


``` 
(python2) [root@localhost 01-yang]# cd /root/sbx_root/yang/01-yang
(python2) [root@localhost 01-yang]# python update_hostname.py 
<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:2637d9ec-b8c6-46f8-9ebc-c96134ba5258">
    <ok/>
</rpc-reply>
(python2) [root@localhost 01-yang]#
```

You should receive the "OK" response back from the device. 

Validate the changes by logging into the devices and verifying:

```

(python2) [root@localhost 01-yang]# ssh admin@172.16.30.101
User Access Verification
Password: 

Cisco NX-OS Software
Copyright (c) 2002-2017, Cisco Systems, Inc. All rights reserved.
NX-OSv9K software ("NX-OSv9K Software") and related documentation,
files or other reference materials ("Documentation") are
the proprietary property and confidential information of Cisco
Systems, Inc. ("Cisco") and are protected, without limitation,
pursuant to United States and International copyright and trademark
laws in the applicable jurisdiction which provide civil and criminal
penalties for copying or distribution without Cisco's authorization.

Any use or disclosure, in whole or in part, of the NX-OSv9K Software
or Documentation to any third party for any purposes is expressly
prohibited except as otherwise authorized by Cisco in writing.
The copyrights to certain works contained herein are owned by other
third parties and are used and distributed under license. Some parts
of this software may be covered under the GNU Public License or the
GNU Lesser General Public License. A copy of each such license is
available at
http://www.gnu.org/licenses/gpl.html and
http://www.gnu.org/licenses/lgpl.html
***************************************************************************
*  NX-OSv9K is strictly limited to use for evaluation, demonstration      *
*  and NX-OS education. Any use or disclosure, in whole or in part of     *
*  the NX-OSv9K Software or Documentation to any third party for any      *
*  purposes is expressly prohibited except as otherwise authorized by     *
*  Cisco in writing.                                                      *
***************************************************************************
nxos-spine1# show hostname 
nxos-spine1 

```

Go ahead and manually reset it back to the original name:

``` shell
nxos-spine1# conf t
Enter configuration commands, one per line. End with CNTL/Z.
nxos-spine1(config)# hostname nx-osv9000-1
nx-osv9000-1(config)# exit
nx-osv9000-1# show hostname 
nx-osv9000-1 

```

## Adding a Loopback Interface with an EDIT Operation

In the previous sections we performed NETCONF `get` and `edit` operations on YANG leaf nodes that were relatively straightforward to visualize from the data model. In this section, we will add a loopback interface on each switch, once again, leveraging the Cisco NXOS YANG model.

To help focus on the YANG model definitions for the interfaces and loopback interfaces in particular, it will be helpful to narrow down the visualization of the data model to just the relevant elements in the model.

> For our example, we will be adding a new loopback address `loopback 99` and assigning an IP address to the interface. 

The YANG model for the Nexus models the interface, distinctly from the IP address assignment to that interface, as you'll soon see.

Navigate to the directory where we downloaded the YANG models from GitHub and execute the `pyang` command with a specific `tree-path` as follows:

First, collect the YANG definition for loopback interface and write to a file:

``` 

(python2) [root@localhost 7.0-3-I6-1]# cd /root/sbx_nxos/learning_labs/yang/yang/vendor/cisco/nx/7.0-3-I6-1
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree --tree-path="/System/intf-items/lb-items" Cisco-NX-OS-device.yang" Cisco-NX-OS-device.yang  -o /tmp/nxos_lbintf.txt
# output omitted

```

Next, collect the YANG definition for IP addressing an interface from the NXOS YANG model.

``` 
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree --tree-path="/System/ipv4-items/inst-items/dom-items/Dom-list/if-items/If-list" Cisco-NX-OS-device.yang" Cisco-NX-OS-device.yang  -o /tmp/nxos_ipv4.txt

```


Let's look at the model definition for the interfaces. 

Open up the `/tmp/nxos_lbintf.txt` file using vi or nano:

``` shell
module: Cisco-NX-OS-device
    +--rw System
       +--rw intf-items
          +--rw lb-items
             +--rw LbRtdIf-list* [id]
                +--rw linkLog?                       l1_LinkLog
                +--rw name?                          naming_Name
                +--rw id                             nw_IfId
                +--rw descr?                         naming_Descr
                +--rw adminSt?                       l1_AdminSt
                +--rw vrf-items
                |  +--ro name?   l3_VrfName
                +--rw lbrtdif-items
                |  +--ro ifIndex?        uint32
                |  +--ro iod?            uint64
...
```

> Above output is shortened to focus on the lab specifics.


Studying this model, we can identify the nodes that are needed (to construct the XML string) to add a new loopback interface. Walking the path in this tree we can get to the `loopback id`, `admin state`  and `description` as follows:

``` shell
/System/intf-items/lb-items/LbRtdIf-list/id
/System/intf-items/lb-items/LbRtdIf-list/adminSt
/System/intf-items/lb-items/LbRtdIf-list/descr

```

Use this information to build out our configuration XML string:

```python
add_lbintf = """
<config>
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
        <intf-items>
            <lb-items>
                <LbRtdIf-list>
                    <id>lo99</id>
                    <adminSt>up</up>
                    <descr>Interface added via NETCONF</descr>
                </LbRtdIf-list>
            </lb-items>
        </intf-items>
    </System>
</config>
"""

```

*Note: Here only lo99 is being created on each device.*


Navigate back to the `01-yang` directory:

```
(python2) [root@localhost 01-yang]# cd /root/sbx_nxos/learning_labs/yang/01-yang

```

Now execute the `add_loopback.py` script.


```
(python2) [root@localhost 01-yang]# python add_loopback.py 

Now adding Looback99 device (nx-osv9000-1) 172.16.30.101...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:002babe4-0e9e-4864-96c6-5adb09601e1d">
    <ok/>
</rpc-reply>


Now adding Looback99 device (nx-osv9000-2) 172.16.30.102...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:00edebd5-874c-41bf-b52d-f915b2bdab6c">
    <ok/>
</rpc-reply>

```


Now verify that the interfaces are available on the switches by SSH'ing to each device:

``` shell
nx-osv9000-1# show interface description | include Lo
Lo0                      Loopback
Lo99                     Interface added via NETCONF

```


As you can see from the running configuration of the interface, there is no IP address assigned.

``` shell
nx-osv9000-1# show running interface lo99

!Command: show running-config interface loopback99
!Time: Fri Aug  4 17:10:56 2017

version 7.0(3)I6(1)

interface loopback99
  description Interface added via NETCONF

```


The IP address definition is modeled in the NXOS YANG module under the `/System/ipv4-items` branch. In the previous step, we had created the visualization of the branch in the `nxos_ipv4.txt` file. 

Let's go ahead and look at it using a text editor.

```
module: Cisco-NX-OS-device
    +--rw System
       +--rw ipv4-items
          +--rw inst-items
             +--rw dom-items
                +--rw Dom-list* [name]
                   +--rw if-items
                      +--rw If-list* [id]
                         +--rw directedBroadcast?    enumeration
                         +--rw acl?                  string
                         +--rw forward?              nw_AdminSt
                         +--rw unnumbered?           nw_IfId
                         +--rw name?                 naming_Name
                         +--rw id                    nw_IfId
                         +--rw descr?                naming_Descr
                         +--rw adminSt?              nw_IfAdminSt
                         +--rw ctrl?                 ip_IfControl
                         +--rw mode?                 ip_IfMode
                         +--rw donorIf?              nw_IfId
                         +--rw addr-items
                         |  +--rw Addr-list* [addr]
                         |     +--rw seckey?       string
                         |     +--rw addr          address_Ip
                         |     +--rw type?         ip_AddrT
                         |     +--rw ctrl?         ip_AddrControl
                         |     +--rw vpcPeer?      address_Ip

```

> Note: An abbreviated version is presented above, to highlight the relevant configuration information for this lab

Analyzing this model, we can identify the nodes that need to be included within our XML string, to add a new IP address. 

This model uses:

`/System/ipv4-items/inst-items/dom-items/Dom-list/if-items/If-list/id`

to identify the loopback interface, whose IP configuration needs to be changed. The node used to make the change to the IP address itself is given by the path:

`/System/ipv4-items/inst-items/dom-items/Dom-list/if-items/If-list/addr-items/Addr-list/addr`

We can codify this as an XML string as follows:

```
add_loopback_ip = """<config>
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
    <ipv4-items>
        <inst-items>
            <dom-items>
                <Dom-list>
                    <name>default</name>
                    <if-items>
                        <If-list>
                            <id>lo99</id>
                            <addr-items>
                                <Addr-list>
                                    <addr>10.99.99.1/24</addr>
                                </Addr-list>
                            </addr-items>
                        </If-list>
                    </if-items>
                </Dom-list>
            </dom-items>
        </inst-items>
    </ipv4-items>
</System>
</config>"""


```

Navigate to the sample code directory `01-yang` and execute the python script `add_loopback_ip.py`.

``` 
(python2) [root@localhost 01-yang]# python add_loopback_ip.py 

Now adding IP address 10.99.99.1/24 to device (nx-osv9000-1) 172.16.30.101...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:25a88ed0-b8b8-49b0-8b77-a5524feb76dd">
    <ok/>
</rpc-reply>


Now adding IP address 10.99.99.2/24 to device (nx-osv9000-2) 172.16.30.102...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:ff8f7ec0-668c-4820-acc4-106bf493ea12">
    <ok/>
</rpc-reply>

(python2) [root@localhost 01-yang]# 

```


The script execution output indicates that `10.99.99.1/24` was added to switch `172.16.30.101 (nx-osv9000-1)` and `10.99.99.2/24` was added to switch `172.16.30.102 (nx-osv9000-2)`. We can verify this by logging into the devices:

**nx-osv9000-1**:

``` shell
nx-osv9000-1# show running interface lo99

!Command: show running-config interface loopback99
!Time: Fri Aug  4 17:36:20 2017

version 7.0(3)I6(1)

interface loopback99
  description Interface added via NETCONF
  ip address 10.99.99.1/24


```

**nx-osv9000-2**:

``` shell
nx-osv9000-2# show running interface lo99

!Command: show running-config interface loopback99
!Time: Fri Aug  4 17:36:54 2017

version 7.0(3)I6(1)

interface loopback99
  description Interface added via NETCONF
  ip address 10.99.99.2/24

nx-osv9000-2# 

```

## Adding the Interface and IP address in one go!

In the previous example, the interface was added using the `add_loopback.py` script, whereas the IP address assignment was done via the `add_loopback_ip.py` script. This was done to help understand the different pieces of the model that are being used separately in order to effect the final configuration of the loopback interface. 

We will now see, how to use a single XML string (and consequently a single script) to effect the same change, by adding a new loopback interfaces `Loopback101` and `Loopback102` on devices `nx-osv9000-1` and `nx-osv9000-2` respectively. Combining the two XML strings we derived from the NXOS YANG model earlier, we have:


``` 
add_ip_interface = """<config>
    <System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
    <intf-items>
        <lb-items>
            <LbRtdIf-list>
                <id>lo101</id>
                <adminSt>up</up>
                <descr>Full intf config via NETCONF</descr>
            </LbRtdIf-list>
        </lb-items>
    </intf-items>
    <ipv4-items>
        <inst-items>
            <dom-items>
                <Dom-list>
                    <name>default</name>
                    <if-items>
                        <If-list>
                            <id>lo101</id>
                            <addr-items>
                                <Addr-list>
                                    <addr>10.101.1.1/24</addr>
                                </Addr-list>
                            </addr-items>
                        </If-list>
                    </if-items>
                </Dom-list>
            </dom-items>
        </inst-items>
    </ipv4-items>
</System>
</config>"""

```


You can see how the `intf-items` branch and the `ipv4-items` branch have been used together as a single XML string, to effect the change on the devices.

From the sample code directory `01-yang`, we can now execute the Python script `add_loopback_full.py` and observe the changes.

``` 
(python2) [root@localhost 01-yang]# python add_loopback_full.py 


Now adding IP address 10.101.1.1/24 to intf lo101 on device (nx-osv9000-1) 172.16.30.101...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:984d0b53-1f21-4f09-833c-e161094cc4ec">
    <ok/>
</rpc-reply>


Now adding IP address 10.102.1.2/24 to intf lo102 on device (nx-osv9000-2) 172.16.30.102...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:59bfc1ed-fa75-44d6-a7c4-a2cea3112b96">
    <ok/>
</rpc-reply>

(python2) [root@localhost 01-yang]# 

(python2) [root@localhost 01-yang]# 

```

We can verify this change took effect on the switch by logging into the devices and displaying the configuration:


**nx-osv9000-1**:

```
nx-osv9000-1# show running-config interface loopback 101

!Command: show running-config interface loopback101
!Time: Tue Aug  8 12:30:18 2017

version 7.0(3)I6(1)

interface loopback101
  description Full intf config via NETCONF
  ip address 10.101.1.1/24

nx-osv9000-1# 

```


**nx-osv9000-2**:

```
nx-osv9000-2# show running-config interface Loopback 102

!Command: show running-config interface loopback102
!Time: Tue Aug  8 12:27:40 2017

version 7.0(3)I6(1)

interface loopback102
  description Full intf config via NETCONF
  ip address 10.102.1.2/24

nx-osv9000-2# 

```
