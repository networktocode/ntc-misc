# NETCONF/YANG on Nexus: Part 2 - Configuring BGP with the Cisco NXOS YANG Model

In Part 2 of using NETCONF and YANG on Nexus switches, we will further our understanding of the Cisco specific NXOS model by using it to advertise a network over BGP. 

## Prerequisites

The user has an understanding and knowledge of the following:
- Complete the Introduction to YANG Data Modeling Devnet learning lab
- Complete the Introduction to the NETCONF Protocol Devnet learning lab 
- **Complete** the NETCONF/YANG on Nexus: Part 1 - Using the Cisco NXOS YANG Model Devnet learning lab
- Basic Python and Ansible
- Python's ncclient
- Installing and using packages on Linux
- BGP routing protocol


## Ensure that dependencies are configured

The lab exercises in part 2 have dependencies from part 1. If you are working on part 2 of the lab, directly after completing part 1, you can skip this section. If you are returning to a new lab session, the following steps will configure the dependencies from part 1, required for part 2.

Navigate to the `yang-prereqs` directory:

```shell
(python2) [root@localhost sbx_nxos]# cd /root/sbx_nxos/yang/yang-prereqs/
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
Finally, navigate to the `01-yang` directory and execute the loopback creation script. This will add interfaces `loopback101` and `loopback102` on `nx-osv9000-1` and `nx-osv9000-2` respectively.

``` shell
(python2) [root@localhost yang-prereqs]# cd /root/sbx_nxos/yang/01-yang

(python2) [root@localhost 01-yang]# python add_loopback_full.py 

Now adding IP address 10.101.1.1/24 to intf Loopback101 on device (nx-osv9000-1) 172.16.30.101...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:d1a93f52-4d02-4f27-8a39-afa6cd64c3b6">
    <ok/>
</rpc-reply>


Now adding IP address 10.102.1.2/24 to intf Loopback102 on device (nx-osv9000-2) 172.16.30.102...

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:7e25c6c4-33cc-434f-bc5b-8e244f899787">
    <ok/>
</rpc-reply>

(python2) [root@localhost 01-yang]# 


```

Now, with the configurations from part 1 in place, we can learn about the BGP YANG model and how to work with it to advertise prefixes, in this part. We will begin by configuring a baseline BGP configuration across our network topology.



## Using Ansible to Setup a Baseline BGP Configuration

We will use an Ansible playbook to "wire up" the initial network configuration. This playbook will configure IP addresses across the different interfaces of the spine and leaf switches. It will also configure the BGP routing protocol on all four devices and advertise a set of networks between them.  

Once this network is up and running, you'll use NETCONF with YANG models to update the BGP configuration the fabric.

Topology reference drawing:

![BGP drawing](images/basic-l3-fabric-topology.png)


Navigate to the directory that contains the playbook. 

``` shell
(python2) [root@localhost sbx_nxos]# cd /root/sbx_nxos/ansible-playbooks/
(python2) [root@localhost ansible-playbooks]# ls -l
total 28
-rw-r--r--. 1 root root  144 Jun  9 14:23 ansible.cfg
drwxr-xr-x. 2 root root 4096 Jun  9 14:23 basic_l3_network
drwxr-xr-x. 2 root root   20 Jun  9 14:23 group_vars
-rw-r--r--. 1 root root  420 Jun  9 14:23 hosts
drwxr-xr-x. 2 root root  102 Aug  1 22:21 host_vars
drwxr-xr-x. 2 root root   74 Jun  9 14:23 readme_images
-rw-r--r--. 1 root root 9395 Jun  9 14:23 README.md
-rw-r--r--. 1 root root   35 Jun  9 14:23 requirements.txt
drwxr-xr-x. 2 root root   31 Jun  9 14:23 switch_admin
(python2) [root@localhost ansible-playbooks]# 

```


To run the playbook, we first need to define the switch credentials as environment variables so that Ansible can properly login to each device.

This is done by sourcing the environment variable file `.ansible-env` within that directory
``` shell
(python2) [root@localhost ansible-playbooks]# source .ansible_env
```

The playbook consists of four plays to deploy L3 and BGP configurations on the switches.  They are as follows:

1. Configure IP addressing on the "physical" interfaces
2. Create and assign IP addresses on loopback interfaces
3. Enable BGP
4. Configure BGP

You can also see the **name** of each Ansible Play and Ansible Task using the `--list-tasks` option of the `ansible-playbook` command.


``` 

(python2) [root@localhost ansible-playbooks]# ansible-playbook basic_l3_network/full_bgp.yml  --list-tasks

playbook: basic_l3_network/full_bgp.yml

  play #1 (switches): Configure IP Connectivity for L3 Fabric   TAGS: []
    tasks:
      Set Interfaces for L3 Mode and Description        TAGS: []
      Configure IPv4 Address on Interface       TAGS: []

  play #2 (switches): Configure Loopback Networks on Each Switch        TAGS: []
    tasks:
      Create Loopback Interface TAGS: []
      Configure IPv4 Address on Interface       TAGS: []

  play #3 (switches): Enable BGP Feature        TAGS: []
    tasks:
      Enable BGP Feature        TAGS: []

  play #4 (switches): Setup BGP on Switches     TAGS: []
    tasks:
      Configure BGP     TAGS: []
      Configure Neighbors       TAGS: []
      Configure Neighbors ipv4 Address Family   TAGS: []
      Configure Default ipv4 Address Family     TAGS: []

```


Next, go ahead and execute the playbook as follows:    
    

``` 

(python2) [root@localhost ansible-playbooks]# ansible-playbook basic_l3_network/full_bgp.yml                                                                      

PLAY [Configure IP Connectivity for L3 Fabric] *********************************

TASK [setup] *******************************************************************
ok: [172.16.30.104]
ok: [172.16.30.101]
ok: [172.16.30.103]
ok: [172.16.30.102]

TASK [Set Interfaces for L3 Mode and Description] ******************************
ok: [172.16.30.103] => (item={u'prefix': 30, u'ip_address': u'172.20.0.2', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-1'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.9', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-3'})
ok: [172.16.30.104] => (item={u'prefix': 30, u'ip_address': u'172.20.0.6', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-1'})
ok: [172.16.30.101] => (item={u'prefix': 30, u'ip_address': u'172.20.0.1', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-3'})
ok: [172.16.30.103] => (item={u'prefix': 30, u'ip_address': u'172.20.0.10', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-2'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.13', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-4'})
ok: [172.16.30.101] => (item={u'prefix': 30, u'ip_address': u'172.20.0.5', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-4'})
ok: [172.16.30.104] => (item={u'prefix': 30, u'ip_address': u'172.20.0.14', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-2'})
ok: [172.16.30.101] => (item={u'prefix': 30, u'ip_address': u'172.20.0.17', u'name': u'Ethernet1/3', u'desc': u'L3 Link to nx-osv9000-2'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.18', u'name': u'Ethernet1/3', u'desc': u'L3 Link to nx-osv9000-1'})

TASK [Configure IPv4 Address on Interface] *************************************
ok: [172.16.30.104] => (item={u'prefix': 30, u'ip_address': u'172.20.0.6', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-1'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.9', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-3'})
ok: [172.16.30.101] => (item={u'prefix': 30, u'ip_address': u'172.20.0.1', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-3'})
ok: [172.16.30.103] => (item={u'prefix': 30, u'ip_address': u'172.20.0.2', u'name': u'Ethernet1/1', u'desc': u'L3 Link to nx-osv9000-1'})
ok: [172.16.30.104] => (item={u'prefix': 30, u'ip_address': u'172.20.0.14', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-2'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.13', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-4'})
ok: [172.16.30.101] => (item={u'prefix': 30, u'ip_address': u'172.20.0.5', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-4'})
ok: [172.16.30.103] => (item={u'prefix': 30, u'ip_address': u'172.20.0.10', u'name': u'Ethernet1/2', u'desc': u'L3 Link to nx-osv9000-2'})
ok: [172.16.30.102] => (item={u'prefix': 30, u'ip_address': u'172.20.0.18', u'name': u'Ethernet1/3', u'desc': u'L3 Link to nx-osv9000-1'})
...
...
...

PLAY RECAP *********************************************************************
172.16.30.101              : ok=13   changed=0    unreachable=0    failed=0   
172.16.30.102              : ok=13    changed=0    unreachable=0    failed=0   
172.16.30.103              : ok=13    changed=0    unreachable=0    failed=0   
172.16.30.104              : ok=13    changed=0    unreachable=0    failed=0   

(python2) [root@localhost ansible-playbooks]# 

```
> Note: The output is truncated for readability.


**IMPORTANT**: Sometimes the playbooks time out in the sandbox environment. If you run into this, please rerun the playbook.

#### Verifying the BGP Configuration

Once the playbook execution is complete, log into each of the switches:

```
(python2) [root@localhost 7.0-3-I6-1]# ssh admin@172.16.30.101
User Access Verification
Password: 

# outout omitted
nx-osv9000-1# 
```

Observe and verify the BGP configuration by issuing the command:

`show running | section bgp`

The output from two of the switches are shown below:

**nx-osv9000-1 (spine)**

``` shell
nx-osv9000-1# show running | section bgp
feature bgp
router bgp 65531
  router-id 192.168.0.1
  address-family ipv4 unicast
    network 172.21.1.0/24
    network 172.21.2.0/24
    network 172.21.3.0/24
    network 172.21.4.0/24
  neighbor 172.20.0.2
    remote-as 65533
    description nx-osv9000-3
    update-source Ethernet1/1
    address-family ipv4 unicast
  neighbor 172.20.0.6
    remote-as 65534
    description nx-osv9000-4
    update-source Ethernet1/2
    address-family ipv4 unicast
  neighbor 172.20.0.18
    remote-as 65532
    description nx-osv9000-2
    update-source Ethernet1/3
    address-family ipv4 unicast
    
    

```


**nx-osv9000v-3(leaf)**

``` shell

nx-osv9000-3# show running-config | section bgp
feature bgp
router bgp 65533
  router-id 192.168.0.3
  address-family ipv4 unicast
    network 172.23.1.0/24
    network 172.23.2.0/24
    network 172.23.3.0/24
    network 172.23.4.0/24
  neighbor 172.20.0.1
    remote-as 65531
    description nx-osv9000-1
    update-source Ethernet1/1
    address-family ipv4 unicast
  neighbor 172.20.0.9
    remote-as 65532
    description nx-osv9000-2
    update-source Ethernet1/2
    address-family ipv4 unicast
nx-osv9000-3# 


```

> Note: The leaf switches have BGP neighbor relationships with the spine switches. Each spine is connected to both leaves and each other.


You can validate this by issuing the `show ip bgp summary` command on **all** the devices.

On a spine device:

``` 
nx-osv9000-1# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 192.168.0.1, local AS number 65531
BGP table version is 37, IPv4 Unicast config peers 3, capable peers 3
16 network entries and 32 paths using 5504 bytes of memory
BGP attribute entries [8/1248], BGP AS path entries [7/58]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.20.0.2      4 65533      14      23       37    0    0 00:06:35 8         
172.20.0.6      4 65534      13      14       37    0    0 00:05:07 8         
172.20.0.18     4 65532      15      23       37    0    0 00:06:33 12        

```

On a leaf device:

``` 
nx-osv9000-3# show ip bgp summary 
BGP summary information for VRF default, address family IPv4 Unicast
BGP router identifier 192.168.0.3, local AS number 65533
BGP table version is 36, IPv4 Unicast config peers 2, capable peers 2
16 network entries and 28 paths using 4992 bytes of memory
BGP attribute entries [7/1092], BGP AS path entries [6/52]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
172.20.0.1      4 65531      17      16       36    0    0 00:08:31 12        
172.20.0.9      4 65532      20      17       36    0    0 00:08:29 12        
nx-osv9000-3# 

```

Now that a base BGP configuration is deployed, we're going to use NETCONF to make configuration changes to the devices using the Cisco NXOS YANG model.

## Mapping BGP configuration to the Cisco NXOS YANG model

As in Part 1, you will use `pyang` to visualize the BGP elements within the NXOS YANG model.  

Navigate to the directory containing the YANG models (this was a Git clone in Part 1):

``` shell
(python2) [root@localhost 7.0-3-I6-1]# cd /root/sbx_nxos/yang/yang/vendor/cisco/nx/7.0-3-I6-1

```

Execute the command running `pyang` as shown below.

``` shell
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree Cisco-NX-OS-device.yang  -o /tmp/nxos_bgp.txt

```

Open the file in the text editor and look for `bgp-items`.

You'll notice this isn't very efficient when you just want a specific node or element within a given model. 

To narrow down the tree output, so you can see it specifically for BGP, execute `pyang` filtering it with the `tree-path` option.  

Use the following command:

``` shell
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree --tree-path="/System/bgp-items" Cisco-NX-OS-device.yang  -o /tmp/nxos_bgp.txt

```

Open the resulting file in a text editor.

``` shell
module: Cisco-NX-OS-device
    +--rw System
       +--rw bgp-items
          +--rw name?         naming_Name
          +--rw adminSt?      nw_AdminSt
          +--ro operSt?       nw_EntOperSt
          +--ro operErr?      nw_OperErrQual
          +--rw inst-items
             +--rw asn?                   bgp_AsnNum
             +--ro ver?                   bgp_Ver
             +--ro syslogLvl?             syslog_Level
             +--ro snmpTrapSt?            snmp_SnmpTrapSt
             +--rw disPolBatch?           bgp_AdminSt
             +--rw disPolBatchv4PfxLst?   string
             +--rw disPolBatchv6PfxLst?   string
             +--ro createTs?              uint64
             +--ro activateTs?            uint64
             +--ro waitDoneTs?            uint64
             +--ro memAlert?              nw_MemAlertLevel
             +--ro numRtAttrib?           cap_Quant
             +--ro attribDbSz?            bgp_AttribDbSz
             +--ro numAsPath?             cap_Quant
             +--ro asPathDbSz?            bgp_AsPathDbSz
             +--rw isolate?               bgp_AdminSt
             +--rw medDampIntvl?          bgp_MedIntvl
             +--rw fabricSoo?             mtx_array_community
             +--rw flushRoutes?           bgp_AdminSt
             +--rw affGrpActv?            bgp_AffGrpActv
             +--ro srgbMinLbl?            bgp_SRGBRange
             +--ro srgbMaxLbl?            bgp_SRGBRange
             +--ro epeConfiguredPeers?    bgp_NumPeers
             +--ro epeActivePeers?        bgp_NumPeers
             +--ro lnkStSrvr?             bgp_LsAdminSt
             +--ro lnkStClnt?             bgp_LsAdminSt
             +--rw name?                  naming_Name
             +--rw adminSt?               nw_AdminSt
             +--rw ctrl?                  nw_InstCtrl_Inst_ctrl
             +--ro operErr?               nw_OperErrQual
             +--rw dom-items
             |  +--rw Dom-list* [name]
             |     +--ro mode?                bgp_Mode
             |     +--rw rtrId?               ip_RtrId
...
```

## Collecting the ASN and BGP Router ID

By reviewing the YANG model tree output above, the ASN can be obtained via the following path:

`/System/bgp-items/inst-items/asn`

Thus, the XML string that'll act as a NETCONF filter to query the device for the ASN, can be represented as the following:

```
asn_filter = """
<System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
    <bgp-items>
        <inst-items>
            <asn/>
        </inst-items>
    </bgp-items>
</System>

```

Let's execute a script that'll query the device for two of the Nexus switches.

First, navigate to the sample code directory for `02-yang` using the command `cd /root/sbx_nxos/yang/02-yang`.  You'll see a script called `get_bgp_asn.py` in this directory.

Execute the `get_bgp_asn.py` script.

``` 
(python2) [root@localhost 02-yang]# cd /root/sbx_nxos/yang/02-yang
(python2) [root@localhost 02-yang]# python get_bgp_asn.py 
The ASN number for (nx-osv9000-1) 172.16.30.101 is 65531
The ASN number for (nx-osv9000-2) 172.16.30.102 is 65532
(python2) [root@localhost 02-yang]# 

```


If you take another look at the tree output found in `/tmp/nxos_bgp.txt`, you'll see BGP router ID can be obtained via the following path:

`/System/bgp-items/inst-items/dom-items/Dom-list/rtrId`

Thus, the XML string that'll act as a NETCONF filter to query the device for the BGP router ID, can be represented as the following:

``` 
bgp_rtrid_filter = """
<System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
    <bgp-items>
        <inst-items>
            <dom-items>
                <Dom-list>
                    <rtrId/>
                </Dom-list>
            </dom-items>
        </inst-items>
    </bgp-items>
</System>
"""
```

To execute the script utilizing this filter, execute the `get_bgp_rtrid.py` script.

``` 
(python2) [root@localhost 02-yang]# python get_bgp_rtrid.py 
The BGP router-id for (nx-osv9000-1) 172.16.30.101 is 192.168.0.1
The BGP router-id for (nx-osv-9000-2) 172.16.30.102 is 192.168.0.2
(python2) [root@localhost 02-yang]# 

```



## Advertise Subnets over BGP Still Using the Cisco NXOS YANG Model

In Part 1, you added interfaces `loopback 101` and  `loopback 102` to switches **nx-osv9000-1** and **nx-osv9000-2** respectively. You will now advertise those subnets over BGP.

> Please see the section about configuring part 1 dependencies at the beginning of this lab.

Use the `pyang` `tree-path` again to visualize the model for BGP prefixes.

Ensure you're in the `/root/sbx_nxos/yang/yang/vendor/cisco/nx/7.0-3-I6-1/`  directory:

``` shell
(python2) [root@localhost 01-yang]# cd /root/sbx_nxos/yang/yang/vendor/cisco/nx/7.0-3-I6-1/
(python2) [root@localhost 7.0-3-I6-1]# 

```

Execute `pyang` against the NXOS YANG model, specifying the tree required for prefixes:

``` shell
(python2) [root@localhost 7.0-3-I6-1]# pyang -f tree --tree-path="/System/bgp-items/inst-items/dom-items/Dom-list/af-items/DomAf-list/prefix-items" Cisco-NX-OS-device.yang -o /tmp/nxos_bgp_prefix.txt
(python2) [root@localhost 7.0-3-I6-1]# 

```

Open the resulting file using a text editor:

``` shell
module: Cisco-NX-OS-device
    +--rw System
       +--rw bgp-items
          +--rw inst-items
             +--rw dom-items
                +--rw Dom-list* [name]
                   +--rw af-items
                      +--rw DomAf-list* [type]
                         +--rw prefix-items
                            +--rw AdvPrefix-list* [addr]
                               +--rw addr     address_Ip
                               +--rw rtMap?   string
...
...
```

From this, we infer the path needed to create the NETCONF specific XML filter for adding addresses to be advertised: 

`/System/bgp-items/inst-items/dom-items/Dom-list/af-items/DomAf-list/prefix-items/AdvPrefix-list/addr`

Translating this into XML, and populating it with the subnet we're looking to advertise, we end up with the following XML string:


``` xml
add_prefix = """
<config>
<System xmlns="http://cisco.com/ns/yang/cisco-nx-os-device">
    <bgp-items>
        <inst-items>
            <dom-items>
                <Dom-list>
                    <af-items>
                        <DomAf-list>
                            <type>ipv4-ucast</type>
                            <prefix-items>
                                <AdvPrefix-list>
                                    <addr>10.101.1.0/24</addr>
                                </AdvPrefix-list>
                            <prefix-items>
                        </DomAf-list>
                    </af-items>
                </Dom-list>
            </dom-items>
        </inst-items>
    </bgp-items>
</System>
</config>"""

```

Navigate back to the `02-yang` directory. 

Execute the script called `add_nxos_bgp_prefixes.py`.

This will advertise each device's Loopback address via BGP.


``` 
(python2) [root@localhost 7.0-3-I6-1]# cd /root/sbx_nxos/yang/02-yang
(python2) [root@localhost 02-yang]# python add_nxos_bgp_prefixes.py 

Now adding prefix 10.101.1.0/24 to device (nx-osv9000-1) 172.16.30.101..

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:88d03c59-663d-4759-93f3-b9db5369ac7d">
    <ok/>
</rpc-reply>


Now adding prefix 10.102.1.0/24 to device (nx-osv9000-2) 172.16.30.102..

<?xml version="1.0" encoding="UTF-8"?>
<rpc-reply xmlns:if="http://www.cisco.com/nxos:1.0:if_manager" xmlns:nfcli="http://www.cisco.com/nxos:1.0:nfcli" xmlns:nxos="http://www.cisco.com/nxos:1.0" xmlns:vlan_mgr_cli="http://www.cisco.com/nxos:1.0:vlan_mgr_cli" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0" message-id="urn:uuid:6c8f8db0-dc60-4062-ad6c-de682c5a2dbc">
    <ok/>
</rpc-reply>

(python2) [root@localhost 02-yang]# 

```

You should expect to see the same output above with an `<ok/>` response for each device.

Let's verify that the new prefixes have been added to the BGP configuration, by logging into the devices. Log in to nx-osv9000-1 and nx-osv9000-2 using ssh:


**nx-osv9000-1:**

``` shell
nx-osv9000-1# show running-config section bgp
show running-config | section bgp
feature bgp
router bgp 65531
  router-id 192.168.0.1
  address-family ipv4 unicast
    network 10.101.1.0/24
    network 172.21.1.0/24
    network 172.21.2.0/24
    network 172.21.3.0/24
    network 172.21.4.0/24
  neighbor 172.20.0.2
    remote-as 65533
    description nx-osv9000-3
    update-source Ethernet1/1
    address-family ipv4 unicast
  neighbor 172.20.0.6
    remote-as 65534
...
```

**nx-osv9000-1:**

``` shell
nx-osv9000-2# show running-config section bgp
show running-config | section bgp
feature bgp
router bgp 65532
  router-id 192.168.0.2
  address-family ipv4 unicast
    network 10.102.1.0/24
    network 172.22.1.0/24
    network 172.22.2.0/24
    network 172.22.3.0/24
    network 172.22.4.0/24
  neighbor 172.20.0.10
    remote-as 65533
    description nx-osv9000-3
...
```


To see the advertisements and verify the routes are being exchanged, log onto **nx-osv9000-4** and verify you see the newly advertised prefixes:

Verify you see **10.101.1.0/24** in the BGP table and routing table:

``` 
nx-osv9000-4# show ip bgp 10.101.1.0/24
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 10.101.1.0/24, version 38
Paths: (2 available, best #1)
Flags: (0x00001a) on xmit-list, is in urib, is best urib route, is in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65531 , path sourced external to AS
    172.20.0.5 (metric 0) from 172.20.0.5 (192.168.0.1)
      Origin IGP, MED not set, localpref 100, weight 0

  Path type: external, path is valid, not best reason: AS Path, no labeled nexthop
  AS-Path: 65532 65531 , path sourced external to AS
    172.20.0.13 (metric 0) from 172.20.0.13 (192.168.0.2)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.20.0.13    
```

```
nx-osv9000-4# show ip route 10.101.1.0/24
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.101.1.0/24, ubest/mbest: 1/0
    *via 172.20.0.5, [20/0], 00:11:33, bgp-65534, external, tag 65531
nx-osv9000-4# 

```

Verify you see **10.102.1.0/24** in the BGP table and routing table:

``` 
nx-osv9000-4# show ip bgp 10.102.1.0/24
BGP routing table information for VRF default, address family IPv4 Unicast
BGP routing table entry for 10.102.1.0/24, version 40
Paths: (2 available, best #1)
Flags: (0x00001a) on xmit-list, is in urib, is best urib route, is in HW

  Advertised path-id 1
  Path type: external, path is valid, is best path, no labeled nexthop, in rib
  AS-Path: 65532 , path sourced external to AS
    172.20.0.13 (metric 0) from 172.20.0.13 (192.168.0.2)
      Origin IGP, MED not set, localpref 100, weight 0

  Path type: external, path is valid, not best reason: AS Path, no labeled nexthop
  AS-Path: 65531 65532 , path sourced external to AS
    172.20.0.5 (metric 0) from 172.20.0.5 (192.168.0.1)
      Origin IGP, MED not set, localpref 100, weight 0

  Path-id 1 advertised to peers:
    172.20.0.5     
nx-osv9000-4# 

```

```
nx-osv9000-4# show ip route 10.102.1.0/24
IP Route Table for VRF "default"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.102.1.0/24, ubest/mbest: 1/0
    *via 172.20.0.13, [20/0], 00:11:38, bgp-65534, external, tag 65532
nx-osv9000-4# 

```

You can also confirm that the newly advertised prefixes are also visible from the **nx-osv9000-3**.

This brings us to the end of part 2. In the next part, we will learn about the OpenConfig YANG model implementation on the Nexus.
