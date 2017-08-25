# NETCONF/YANG on Nexus: Part 3 - Using OpenConfig YANG Models on Nexus Switches

In the first two parts of working with NETCONF/YANG on Nexus switches, we focused on using the Cisco specific NXOS YANG model. We started with learning about it in Part 1, making basic changes and then how to model BGP, adhering to the model in Part 2.

Now in Part 3, we will learn about the OpenConfig Working Group and how to use OpenConfig models on Nexus switches. We will install the OpenConfig models on the leaf switches and use it to add loopback interfaces to the leaf nodes, just like we did in part 1.


## Prerequisites

The user has an understanding and knowledge of the following:
- Complete the Introduction to YANG Data Modeling Devnet learning lab
- Complete the Introduction to the NETCONF Protocol Devnet learning lab 
- **Complete** the NETCONF/YANG on Nexus: Part 1 - Learning to use the Cisco NXOS YANG Model
- **Complete** the NETCONF/YANG on Nexus: Part 2 - Configuring BGP with the Cisco NXOS YANG Model Devnet learning lab
- Basic Python and Ansible
- Python's ncclient
- Installing and using packages on Linux
- BGP routing protocol

## Ensure that dependencies are configured

The lab exercises in part 3 have dependencies from part 1. If you are working on part 3 of the lab, directly after completing part 1 and 2, you can skip this section. If you are returning to a new lab session, the following steps will configure the dependencies from part 1, required for part 3.

Navigate to the `yang-prereqs` directory:

```shell
(python2) [root@localhost sbx_nxos]# cd /root/sbx_nxos/learning_labs/yang/yang-prereqs/
(python2) [root@localhost yang-prereqs]#
```

Execute the `devbox_setup.yml` playbook, to download the switch software, YANG models, and install the `pyang` and `ncclient` libraries, in addition to the `ntc-ansible` 3rd party Ansible module.

```shell
(python2) [root@localhost yang-prereqs]# ansible-playbook devbox_setup.yml 

PLAY [SET UP YANG DEVBOX ENVIRONMENT] ******************************************

TASK [setup] *******************************************************************
ok: [10.10.20.20]

TASK [INSTALL PYANG AND NCCLIENT] **********************************************
changed: [10.10.20.20]

TASK [CREATE THE NXOS RPMS DIRECTORY] ******************************************
ok: [10.10.20.20]

TASK [DOWNLOAD THE CISCO ARTIFACTORY RPMs] *************************************
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm)
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-infra-1.0.0-r1705191346.x86_64.rpm)
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm)
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm)
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm)
ok: [10.10.20.20] => (item=https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm)

TASK [CLONE THE YANG MODELS] ***************************************************
ok: [10.10.20.20]

TASK [ENSURE THAT THE NTC LIBRARY DIR IS ABSENT] *******************************
ok: [10.10.20.20]

TASK [CLONE THE NTC MODULE] ****************************************************
changed: [10.10.20.20]

TASK [UPDATE NTC CONFIG FILE] **************************************************
changed: [10.10.20.20]

PLAY RECAP *********************************************************************
10.10.20.20                : ok=8    changed=3    unreachable=0    failed=0   

(python2) [root@localhost yang-prereqs]# 

```

Next, run the `spine_setup.yml` playbook. This playbook will copy over the RPMS to the spine switches, install them and start the NETCONF server on the switch:

``` shell
(python2) [root@localhost yang-prereqs]# ansible-playbook spine_setup.yml 

PLAY [COPY RPMS NXOS SPINES] ***************************************************

TASK [SCP THE NATIVE YANG RPMS TO SPINE SWITCHES] ******************************
ok: [172.16.30.101] => (item=mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm)
ok: [172.16.30.102] => (item=mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm)
ok: [172.16.30.101] => (item=mtx-infra-1.0.0-r1705191346.x86_64.rpm)
ok: [172.16.30.102] => (item=mtx-infra-1.0.0-r1705191346.x86_64.rpm)
ok: [172.16.30.102] => (item=mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm)
ok: [172.16.30.101] => (item=mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm)

PLAY [INSTALL THE RPMS AND START NETCONF ON THE NXOS] **************************

TASK [INSTALL THE RPMS] ********************************************************
changed: [172.16.30.101]
changed: [172.16.30.102]

PLAY RECAP *********************************************************************
172.16.30.101              : ok=2    changed=1    unreachable=0    failed=0   
172.16.30.102              : ok=2    changed=1    unreachable=0    failed=0   

(python2) [root@localhost yang-prereqs]# 

```
Now, with the dependencies from part 1 in place, we can learn about the OpenConfig YANG model and how to work with it, to add loopback interfaces to the leaf nodes. We will begin by exploring the OpenConfig YANG models on the Nexus.

## Exploring the use of OpenConfig YANG Models on Nexus

### Introduction to OpenConfig

The [OpenConfig](http://openconfig.net) Working Group, is a network operator driven working group focused on creating operationally relevant data models, now written in YANG. The resulting YANG models helps abstract vendor and platform dependencies. The models also allow for vendors to *augment* OpenConfig models with vendor/platform specific ones. So if operators are leveraging vendor specific features within a technology, that is still possible. 

An OpenConfig (OC) BGP model implementation on a Cisco IOS XE device would be identical to an OpenConfig BGP model on a Cisco Nexus switch, for example.  Moreover, an OC model for others would be identical as well!

A major advantage of using vendor neutral models such as those from the OpenConfig Working Group is realized in the tooling used to automate devices.  Using neutral models operators can use the same tools/programs to more easily interface with various types of devices and platforms.

### Getting Started with OpenConfig Models on Nexus Switches

In order to explore and work with the OpenConfig implementation on the Nexus platform we will need to install the necessary packages.

In Part 1, we installed three (3) `mtx` RPMs on the spine switches to deploy the Cisco specific NXOS YANG model and the NETCONF agent. 

Log into either of the spine switches to validate this:

``` shell
nx-osv9000-1# run bash sudo su
bash-4.2# yum list installed | grep mtx
mtx-device.x86_64                      1.0.0-r1705191346              @/mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64
mtx-infra.x86_64                       1.0.0-r1705191346              @/mtx-infra-1.0.0-r1705191346.x86_64
mtx-netconf-agent.x86_64               1.0.1-r1705191346              @/mtx-netconf-agent-1.0.1-r1705191346.x86_64
bash-4.2# 

```

In this part, we're going install OpenConfig models on both of the **leaf** switches. We will continue using the NETCONF agent on the switch as the interface to interact with the models.

For this exercise, we will install the following packages on the leaf devices:

``` 
mtx-infra-1.0.0-r1705191346.x86_64.rpm
mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm
mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm
mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm 
mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm 
mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm
```

> Take note that there is a different file for each and every OpenConfig model supported on a device.  This is in contrast to a more monolithic device model such as the NXOS model we've been using thus far in Part 1 and the first half of Part 2.

Just like we saw in Part 1, these packages are freely available for download  at [the Cisco Nexus Artifactory](https://devhub.cisco.com/artifactory/open-nxos-agents/7.0-3-I6-1/x86_64/).

For the lab, these RPMs are downloaded and available in the `nxos_rpms` directory. Validate this by navigating to the directory and listing it:

``` 
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

Next copy the `mtx-infra-*`, `mtx-device-*`, `mtx-netconf-agent-*`, and OpenConfig models (`mtc-openconfig*`) to each leaf switch. 

Log into the leaf devices and execute the following copy commands:

``` 
nx-osv9000-3# copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-infra-1.0.0-r1705191346.x86_64.rpm bootflash: vrf management 
The authenticity of host '10.10.20.20 (10.10.20.20)' can't be established.
ECDSA key fingerprint is SHA256:IZJVFckeMZy8BIdcqaFRl5gFxs7LTD8L5Uu7XvvJVmo.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.20.20' (ECDSA) to the list of known hosts.
root@10.10.20.20's password: 
mtx-infra-1.0.0-r1705191346.x86_64.rpm           100%   10MB   1.7MB/s   00:06

Copy complete, now saving to disk (please wait)...
```


```
nx-osv9000-3# copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm bootflash: vrf management 
root@10.10.20.20's password: 
mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm                                                                                  100%   15MB   1.8MB/s   00:08    
Copy complete, now saving to disk (please wait)...
```

```
nx-osv9000-3# copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm bootflash: vrf management 
root@10.10.20.20's password: 
mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm                                                                                      100% 2855KB   2.8MB/s   00:01    
Copy complete, now saving to disk (please wait)...
```


``` 
nx-osv9000-3# copy scp://root@10.10.20.20/root/sbx_nxos/learning_labs/yang/nxos_rpms/mtx-openconfig* bootflash: vrf management 
root@10.10.20.20's password: 
mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm                                                                          100%   20MB   5.0MB/s   00:04    
mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm                                                                        100%   10MB   5.1MB/s   00:02    
mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm                                                                   100% 5377KB   5.3MB/s   00:01    
Copy complete, now saving to disk (please wait)...
nx-osv9000-3# 

```


The above shows the copy commands only for the **nx-osv9000-3** switch. 

You should now do this for **nx-osv9000-4**.

After copying the necessary RPMs, in order to install them, enable the `bash` feature and drop into the bash shell of the Nexus device:

``` shell
nx-osv9000-3# configure terminal 
Enter configuration commands, one per line. End with CNTL/Z.   
nx-osv9000-3(config)# feature bash-shell 
nx-osv9000-3(config)# 
nx-osv9000-3(config)# run bash sudo su
bash-4.2# 

```

Navigate to the `bootflash` directory where we copied over the RPMs.


``` 
bash-4.2# cd /bootflash/
bash-4.2# ls -l
total 807252
drwxrwxrwx 3 admin network-admin      4096 Aug  6 12:04 home
-rw-rw-rw- 1 admin network-admin  15231416 Aug  6 12:08 mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm
-rw-rw-rw- 1 admin network-admin  10420941 Aug  6 11:57 mtx-infra-1.0.0-r1705191346.x86_64.rpm
-rw-rw-rw- 1 admin network-admin   2923015 Aug  6 11:58 mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm
-rw-rw-rw- 1 admin network-admin  21075904 Aug  6 11:59 mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm
-rw-rw-rw- 1 admin network-admin  10645230 Aug  6 11:59 mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm
-rw-rw-rw- 1 admin network-admin   5505579 Aug  6 12:00 mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm
-rw-rw-rw- 1 root  root          759941120 May 17 06:46 nxos.7.0.3.I6.1.bin
-rw-rw-rw- 1 admin root                  0 Aug  4 20:58 platform-sdk.cmd
drwxrwxrwx 2 root  root               4096 Aug  2 20:45 scripts
drwx------ 2 root  root               4096 Aug  2 20:45 virt_strg_pool_bf_vdc_1
drwxrwxrwx 3 root  root               4096 Aug  2 20:44 virtual-instance
-rw-rw-rw- 1 root  root                 59 Aug  2 20:44 virtual-instance.conf
bash-4.2# 

```

You should now install the RPMs using the `yum` package manager.

``` 
bash-4.2# yum install -y mtx*rpm                                                                                                                               [5/762]
Loaded plugins: downloadonly, importpubkey, localrpmDB, patchaction, patching, protect-packages
groups-repo                                                                                                                                    | 1.1 kB     00:00 ... 
localdb                                                                                                                                        |  951 B     00:00 ... 
patching                                                                                                                                       |  951 B     00:00 ... 
thirdparty                                                                                                                                     |  951 B     00:00 ... 
Setting up Install Process
Examining mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm: mtx-device-1.0.0-r1705191346.x86_64
Marking mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64.rpm to be installed
Examining mtx-infra-1.0.0-r1705191346.x86_64.rpm: mtx-infra-1.0.0-r1705191346.x86_64
Marking mtx-infra-1.0.0-r1705191346.x86_64.rpm to be installed
Examining mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm: mtx-netconf-agent-1.0.1-r1705191346.x86_64
Marking mtx-netconf-agent-1.0.1-r1705191346.x86_64.rpm to be installed
Examining mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm: mtx-openconfig-bgp-1.0.0-r1705170158.x86_64
Marking mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64.rpm to be installed
Examining mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm: mtx-openconfig-if-ip-1.0.0-r1705170202.x86_64
Marking mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64.rpm to be installed
Examining mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm: mtx-openconfig-interfaces-1.0.0-r1705190423.x86_64
Marking mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64.rpm to be installed
Resolving Dependencies
--> Running transaction check
---> Package mtx-device.x86_64 0:1.0.0-r1705191346 will be installed
---> Package mtx-infra.x86_64 0:1.0.0-r1705191346 will be installed
---> Package mtx-netconf-agent.x86_64 0:1.0.1-r1705191346 will be installed
---> Package mtx-openconfig-bgp.x86_64 0:1.0.0-r1705170158 will be installed
---> Package mtx-openconfig-if-ip.x86_64 0:1.0.0-r1705170202 will be installed
---> Package mtx-openconfig-interfaces.x86_64 0:1.0.0-r1705190423 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=====================================================================================================================================================================$
 Package                              Arch              Version                       Repository                                                                 Size
=====================================================================================================================================================================$
Installing:
 mtx-device                           x86_64            1.0.0-r1705191346             /mtx-device-7_0_3_I6_1.1.0.0-r1705191346.x86_64                            52 M
 mtx-infra                            x86_64            1.0.0-r1705191346             /mtx-infra-1.0.0-r1705191346.x86_64                                        33 M
 mtx-netconf-agent                    x86_64            1.0.1-r1705191346             /mtx-netconf-agent-1.0.1-r1705191346.x86_64                                11 M
 mtx-openconfig-bgp                   x86_64            1.0.0-r1705170158             /mtx-openconfig-bgp-7_0_3_I6_1.1.0.0-r1705170158.x86_64                    54 M
 mtx-openconfig-if-ip                 x86_64            1.0.0-r1705170202             /mtx-openconfig-if-ip-7_0_3_I6_1.1.0.0-r1705170202.x86_64                  28 M
 mtx-openconfig-interfaces            x86_64            1.0.0-r1705190423             /mtx-openconfig-interfaces-7_0_3_I6_1.1.0.0-r1705190423.x86_64             15 M

Transaction Summary
======================================================================================================================================================================
Install       6 Packages

Total size: 193 M
Installed size: 193 M
Downloading Packages:
Running Transaction Check
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : mtx-infra-1.0.0-r1705191346.x86_64                                                                                                                 1/6 
  Installing : mtx-openconfig-bgp-1.0.0-r1705170158.x86_64                                                                                                        2/6 
  Installing : mtx-openconfig-if-ip-1.0.0-r1705170202.x86_64                                                                                                      3/6 
  Installing : mtx-openconfig-interfaces-1.0.0-r1705190423.x86_64                                                                                                 4/6 
  Installing : mtx-device-1.0.0-r1705191346.x86_64                                                                                                                5/6 
  Installing : mtx-netconf-agent-1.0.1-r1705191346.x86_64                                                                                                         6/6 
Please run the following command to start the netconf service:
netconfctl start

Installed:
  mtx-device.x86_64 0:1.0.0-r1705191346               mtx-infra.x86_64 0:1.0.0-r1705191346                  mtx-netconf-agent.x86_64 0:1.0.1-r1705191346              
  mtx-openconfig-bgp.x86_64 0:1.0.0-r1705170158       mtx-openconfig-if-ip.x86_64 0:1.0.0-r1705170202       mtx-openconfig-interfaces.x86_64 0:1.0.0-r1705190423      

Complete!

```


Start the NETCONF agent on the switch:

``` shell
bash-4.2# netconfctl start
Starting Netconf Agent: [OK]
bash-4.2# 

```

Repeat the installation steps on **nx-osv9000-4** and start the NETCONF service on it too.

#### Using OpenConfig to add Loopback interfaces on the leaf switches:

In Part 1, we used the Cisco NXOS YANG model to configure loopback interfaces on **nx-osv9000-1** and **nx-osv9000-2**.  Now in Part 3, we will use the OpenConfig model to configure loopback interfaces on **nx-osv9000-3** and **nx-osv9000-4**

To construct the XML string needed, use `pyang` to visualize the `openconfig-interfaces` YANG model.

Navigate to the directory where the YANG modules have been downloaded:

``` 
(python2) [root@localhost yang]# cd /root/sbx_nxos/learning_labs/yang/yang/vendor/cisco/nx/7.0-3-I6-1/                                                                              
(python2) [root@localhost 7.0-3-I6-1]# ls -l
total 4432
-rwxr-xr-x. 1 root root    2153 Aug  3 14:08 check-models.sh
-rw-r--r--. 1 root root   51450 Aug  3 14:08 cisco-nx-openconfig-bgp-deviations.yang
-rw-r--r--. 1 root root     741 Aug  3 14:08 cisco-nx-openconfig-bgp-multiprotocol-deviations.yang
-rw-r--r--. 1 root root   60073 Aug  3 14:08 cisco-nx-openconfig-if-ip-deviations.yang
-rw-r--r--. 1 root root    2193 Aug  3 14:08 cisco-nx-openconfig-local-routing-deviations.yang
-rw-r--r--. 1 root root   15224 Aug  3 14:08 cisco-nx-openconfig-routing-policy-deviations.yang
-rw-r--r--. 1 root root 4028679 Aug  3 14:08 Cisco-NX-OS-device.yang
-rw-r--r--. 1 root root   35660 Aug  3 14:08 iana-if-type.yang
-rw-r--r--. 1 root root   16833 Aug  3 14:08 ietf-inet-types.yang
-rw-r--r--. 1 root root   26325 Aug  3 14:08 ietf-interfaces.yang
-rw-r--r--. 1 root root   17939 Aug  3 14:08 ietf-yang-types.yang
-rw-r--r--. 1 root root    2003 Aug  3 14:08 netconf-capabilities.xml
-rw-r--r--. 1 root root   19880 Aug  3 14:08 openconfig-bgp-multiprotocol.yang
-rw-r--r--. 1 root root   11560 Aug  3 14:08 openconfig-bgp-operational.yang
-rw-r--r--. 1 root root   11955 Aug  3 14:08 openconfig-bgp-types.yang
-rw-r--r--. 1 root root   31151 Aug  3 14:08 openconfig-bgp.yang
-rw-r--r--. 1 root root    2124 Aug  3 14:08 openconfig-extensions.yang
-rw-r--r--. 1 root root    4436 Aug  3 14:08 openconfig-if-aggregate.yang
-rw-r--r--. 1 root root    7902 Aug  3 14:08 openconfig-if-ethernet.yang
-rw-r--r--. 1 root root    3596 Aug  3 14:08 openconfig-if-ip-ext.yang
-rw-r--r--. 1 root root   25765 Aug  3 14:08 openconfig-if-ip.yang
-rw-r--r--. 1 root root   29064 Aug  3 14:08 openconfig-interfaces.yang
-rw-r--r--. 1 root root   10962 Aug  3 14:08 openconfig-local-routing.yang
-rw-r--r--. 1 root root    4320 Aug  3 14:08 openconfig-policy-types.yang
-rw-r--r--. 1 root root   27020 Aug  3 14:08 openconfig-routing-policy.yang
-rw-r--r--. 1 root root    2895 Aug  3 14:08 openconfig-types.yang
-rw-r--r--. 1 root root    5272 Aug  3 14:08 openconfig-vlan-types.yang
-rw-r--r--. 1 root root    9983 Aug  3 14:08 openconfig-vlan.yang
-rw-r--r--. 1 root root   13176 Aug  3 14:08 README.md
```

The models we will need, to add the loopback interface are defined within the `openconfig-interfaces.yang` and the `openconfig-if-ip.yang` files. Use `pyang` to see the tree output for these models:

``` 
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree openconfig-interfaces.yang openconfig-if-ip.yang -o /tmp/nxos_oc_interfaces.txt
(python2) [root@localhost 7.0-3-I6-1]# 

```

Open the generated file using a text editor:

The section used to create the new interface corresponds to:

``` shell
module: openconfig-interfaces
    +--rw interfaces
       +--rw interface* [name]
          +--rw name                   -> ../config/name
          +--rw config
          |  +--rw type           identityref
          |  +--rw mtu?           uint16
          |  +--rw name?          string
          |  +--rw description?   string
          |  +--rw enabled?       boolean
...
...
```

In order to narrow down the tree and nodes that are needed to assign the IP address, we can use the `tree-path` option to help generate a focused output:

``` 
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree --tree-path="/interfaces/interface/subinterfaces/subinterface/ipv4" openconfig-interfaces.yang openconfig-if-ip.y
ang -o /tmp/nxos_oc_ip_intf.txt
```

Open the new file `/tmp/nxos_oc_ip_intf.txt` using a text editor:

``` shell
module: openconfig-interfaces
    +--rw interfaces
       +--rw interface* [name]
          +--rw subinterfaces
             +--rw subinterface* [index]
                +--rw oc-ip:ipv4
                   +--rw oc-ip:addresses
                   |  +--rw oc-ip:address* [ip]
                   |     +--rw oc-ip:ip        -> ../config/ip
                   |     +--rw oc-ip:config
                   |     |  +--rw oc-ip:ip?              inet:ipv4-address-no-zone
                   |     |  +--rw oc-ip:prefix-length?   uint8
                   |     +--ro oc-ip:state
                   |     |  +--ro oc-ip:ip?              inet:ipv4-address-no-zone
                   |     |  +--ro oc-ip:prefix-length?   uint8
                   |     |  +--ro oc-ip:origin?          ip-address-origin
...
...
```

The OpenConfig model for IP addressing seems to suggest that a sub-interface is involved. The sub-interface in this context refers to an `interface index`. This index is set to `0` when referencing the main interface.

Studying the tree in the file `/tmp/nxos_oc_interfaces.txt`, the nodes needed to create the loopback interface are:

`/interfaces/interface/name`

`/interfaces/interface/config/name`

`/interfaces/interface/config/type`

`/interfaces/interface/config/description`


And from `/tmp/nxos_oc_ip_intf.txt`, to assign an IP address:

`/interfaces/interface/subinterfaces/subinterface/ipv4/addresses/address/ip`

`/interfaces/interface/subinterfaces/subinterface/ipv4/addresses/address/config/ip`


Use this information to build the XML config string:

``` xml
add_oc_interface = """<config>
<interfaces xmlns="http://openconfig.net/yang/interfaces">
    <interface>
        <name>lo103</name>
        <config>
            <description> using OpenConfig Model </description>
            <name>lo103</name>
            <type>ianaift:softwareLoopback</type>
        </config>
        <subinterfaces>
            <subinterface>
                <index>0</index>
                <ipv4>
                    <addresses>
                        <address>
                            <config>
                                <ip>10.103.1.1</ip>
                                <prefix-length>24</prefix-length>
                            </config>
                            <ip>10.103.1.1</ip>
                        </address>
                    </addresses>
                </ipv4>
            </subinterface>
        </subinterfaces>
    </interface>
</interfaces>
</config>"""

```


Now navigate to the sample code directory `03-yang` and run the `add_oc_loopback.py` script to add new loopback interfaces to the the switches **nx-osv9000-3** and **nx-osv9000-4**

``` 
(python2) [root@localhost 03-yang]# cd /root/sbx_nxos/learning_labs/yang/03-yang/
(python2) [root@localhost 03-yang]# python add_oc_loopback.py 

Now adding IP address 10.103.1.1 to interface Loopback103 on device (nx-osv9000-3) 172.16.30.103...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:63458855-3790-4307-b0f5-e494105ab1fe">
    <ok/>
</rpc-reply>


Now adding IP address 10.104.1.1 to interface Loopback104 on device (nx-osv9000-4) 172.16.30.104...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:fc9917ea-7591-4256-b7d5-bc79221a7825">
    <ok/>
</rpc-reply>

(python2) [root@localhost 03-yang]# 


```

The above output tells us that the loopback interfaces 103 and 104 were successfully added to the switches **nx-osv9000-3** and **nx-osv9000-4**, respectively. 

Log into the devices and validate the configuration. 

**nx-osv9000-3**:

```
nx-osv9000-3# sh running interface  loopback103

!Command: show running-config interface loopback103
!Time: Mon Aug  7 21:17:14 2017

version 7.0(3)I6(1)

interface loopback103
  description Configured using OpenConfig Model
  ip address 10.103.1.1/24

nx-osv9000-3# 
```

**nx-osv9000-4**

``` 
nx-osv9000-4# sh running interface loopback104

!Command: show running-config interface loopback104
!Time: Mon Aug  7 21:17:57 2017

version 7.0(3)I6(1)

interface loopback104
  description Configured using OpenConfig Model
  ip address 10.104.1.1/24

nx-osv9000-4# 
```




#### Using the BGP OpenConfig Model

Earlier in this module, we learned how to use the Cisco NXOS YANG model to collect the ASN and BGP router if the devices. In this section, we will learn how to collect that same information through the OpenConfig model. 

As before, use the `pyang` utility, to build an understanding about the OpenConfig BGP model.

Navigate to the repository containing the YANG Nexus models:

``` shell
(python2) [root@localhost 01-yang]# cd /root/sbx_nxos/learning_labs/yang/yang/vendor/cisco/nx/7.0-3-I6-1/
(python2) [root@localhost 7.0-3-I6-1]# 

```

Execute `pyang` against the OpenConfig BGP YANG model:

``` shell
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree openconfig-bgp.yang -o /tmp/nxos_oc_bgp.txt
(python2) [root@localhost 7.0-3-I6-1]# 

```

Open this file with a text editor:

``` shell
module: openconfig-bgp
    +--rw bgp!
       +--rw global
       |  +--rw config
       |  |  +--rw as           inet:as-number
       |  |  +--rw router-id?   yang:dotted-quad
       |  +--ro state
       |  |  +--ro as                inet:as-number
       |  |  +--ro router-id?        yang:dotted-quad
       |  |  +--ro total-paths?      uint32
       |  |  +--ro total-prefixes?   uint32
       |  +--rw route-selection-options
       |  |  +--rw config
       |  |  |  +--rw always-compare-med?           boolean
       |  |  |  +--rw ignore-as-path-length?        boolean
       |  |  |  +--rw external-compare-router-id?   boolean
       |  |  |  +--rw advertise-inactive-routes?    boolean

```


From this model, we can see that the ASN and the router ID are part of the `global` tree. Also we can see that there are two YANG containers that differentiate the configuration data versus the current state date. 

The `config` container nodes are `rw` whereas the `state` nodes are `ro`. 

Using this model, we can construct an XML filter to gather the data we are interested in:

``` xml
get_oc_bgp = """
<bgp xmlns="http://openconfig.net/yang/bgp">
    <global>
        <state/>
    </global>
</bgp>
"""        
```

Navigate to the sample code directory `03-yang`. This contains a Python script that uses the filter above and collects the ASN and router id information from **nx-osv9000-3** and **nx-osv9000-4**.

Navigate back to `03-yang`:

``` 
(python2) [root@localhost 7.0-3-I6-1]# cd /root/sbx_nxos/learning_labs/yang/03-yang/
(python2) [root@localhost 03-yang]# 

```

Execute the `get_oc_bgp.py` script that uses the `get_oc_bgp` XML filter:

``` 
(python2) [root@localhost 03-yang]# python get_oc_bgp.py 
ASN number:65533, Router ID: 192.168.0.3 for (nx-osv9000-3) 172.16.30.103
ASN number:65534, Router ID: 192.168.0.4 for (nx-osv9000-4) 172.16.30.104
(python2) [root@localhost 03-yang]# 

```


## Conclusion

In Part 1, we built an understanding of the Cisco specific NXOS YANG model and used the model as a reference to retrieve and configure Nexus switches.  In Part 2, we explored the NXOS model further, specifically focusing on BGP. Finally, in Part 3, we introduced OpenConfig models and how they differ (in structure) from the NXOS model.  Note that OpenConfig models are vendor-neutral and may not support all the vendor, or Nexus, specific features, but intends to satisfy the majority of operator use cases

In both cases, the YANG models (both native and OpenConfig) are evolving.  However, you should realize that you don't actually *work with* YANG models.  You simply need to ensure your XML strings or objects adhere to those given YANG models!


