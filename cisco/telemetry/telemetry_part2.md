# Get Started with Streaming Telemetry on Open NX-OS - Part 2

In Part 1 to the introduction to streaming Telemetry on Open NX-OS, we saw how to configure telemetry and collect data from a device using _show commands_. In this section we will look at a more production-like scenario and stream BGP related telemetry from 4 NX-OS devices to a central collector, leveraging the Data Modeling Engine.


# Ensure that Dependencies are Configured

If you are working on Part 2 of the lab, directly after completing Part 1, you may skip this section. If not, please install the Python `flask` library on the devbox as follows:

Log into the devbox and perform the following:

```shell
(python2) [root@localhost sbx_nxos]# pip install flask
Collecting flask
  Downloading Flask-0.12.2-py2.py3-none-any.whl (83kB)
    100% |████████████████████████████████| 92kB 827kB/s 
Collecting itsdangerous>=0.21 (from flask)
  Downloading itsdangerous-0.24.tar.gz (46kB)
    100% |████████████████████████████████| 51kB 2.2MB/s 
Collecting click>=2.0 (from flask)
  Downloading click-6.7-py2.py3-none-any.whl (71kB)
    100% |████████████████████████████████| 71kB 1.9MB/s 
Collecting Werkzeug>=0.7 (from flask)
  Downloading Werkzeug-0.12.2-py2.py3-none-any.whl (312kB)
    100% |████████████████████████████████| 317kB 1.5MB/s 
Requirement already satisfied: Jinja2>=2.4 in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from flask)
Requirement already satisfied: MarkupSafe in /root/sbx_nxos/venv/python2/lib/python2.7/site-packages (from Jinja2>=2.4->flask)
Building wheels for collected packages: itsdangerous
  Running setup.py bdist_wheel for itsdangerous ... done
  Stored in directory: /root/.cache/pip/wheels/fc/a8/66/24d655233c757e178d45dea2de22a04c6d92766abfb741129a
Successfully built itsdangerous
Installing collected packages: itsdangerous, click, Werkzeug, flask
Successfully installed Werkzeug-0.12.2 click-6.7 flask-0.12.2 itsdangerous-0.24
(python2) [root@localhost sbx_nxos]#
```

## Collecting the data using DME

As mentioned in part 1, the NX-OS devices support Model Based Telemetry or MBT using a Data Modeling Engine (DME). Using the DME to stream telemetry data is the preferred way as there are resource utilization implications to using NX-API based show commands to collect analytics data from switches. In this section we will explore a more real world scenario and monitor BGP on multiple NX-OS devices using the DME.

### Configure BGP

We will use an Ansible playbook to "wire up" the initial network configuration. This playbook will configure IP addresses across the different interfaces of the spine and leaf switches. It will also configure the BGP routing protocol on all four devices and advertise a set of networks between them.  

Once this network is up and running, you'll use the DME modeling of BGP,  to collect the BGP telemetry data.

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

This is done by sourcing the environment variable file `.ansible-env` within that directory:

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

### Verifying the BGP Configuration

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

Now that a base BGP configuration is deployed, we're going to configure telemetry, using gRPC to collect BGP analytics.


## Configuring telemetry that uses the DME 

Log into each of the NX-OS switches and configure the sensor, collector and subscription. The configuration is identical on all the devices:

``` shell
nx-osv9000-2# conf t
Enter configuration commands, one per line. End with CNTL/Z.
nx-osv9000-2(config)# feature telemetry
nx-osv9000-2(config)# telemetry 
nx-osv9000-2(config-telemetry)# sensor-group 101
nx-osv9000-2(conf-tm-sensor)# path sys/bgp unbounded
nx-osv9000-2(conf-tm-sensor)# destination-group 101
nx-osv9000-2(conf-tm-dest)# ip address 10.10.20.20 port 8888 protocol HTTP encoding JSON
nx-osv9000-2(conf-tm-dest)# 
nx-osv9000-2(conf-tm-dest)# subscription 101
nx-osv9000-2(conf-tm-sub)# snsr-grp 101 sample-interval 10000
nx-osv9000-2(conf-tm-sub)# dst-grp 101
nx-osv9000-2(conf-tm-sub)# 

```

When we configure this new sensor-group, we are not specifying the data-type like before. This is because, the default data-type used is the DME -  we are leveraging a sensor based on the device's Data Modeling Engine (DME), by specifying the path as `sys/bgp` compared to a show command on the switch. Finally, for the destination group, we continue to use the simple collector application from the previous example.

On any one of the devices, execute the following show commands:

Verify the transport:

```shell

nx-osv9000-1# show telemetry transport 

Session Id      IP Address      Port       Encoding   Transport  Status    
-----------------------------------------------------------------------------------
0               10.10.20.20     8888       JSON       HTTP       Connected 
nx-osv9000-1(conf-tm-sub)# 


```

Verify the sensor:

``` shell
nx-osv9000-1# show telemetry data collector details 

--------------------------------------------------------------------------------
Successful     Failed         Skipped        Sensor Path
--------------------------------------------------------------------------------
 10             0              0             sys/bgp
nx-osv9000-1(conf-tm-sub)# 

```

Verify the data collector type:

``` shell
nx-osv9000-1# show telemetry control database subscriptions 
Subscription Database size = 1
--------------------------------------------------------------------------------
Subscription ID      Data Collector Type 
--------------------------------------------------------------------------------
101                  DME 
nx-osv9000-1(conf-tm-sub)# 
```


### Validating the Data Collected

Our collector application, continues to write the collected data into the `/tmp/nxos.log` file as before. `tail` this file (or open it in a text editor) to look at the BGP data being collected from each of the NX-OS devices.

A sample data snippet is shown below:

``` shell
{ "version_str": "1.0.0", "node_id_str": "nx-osv9000-3", "encoding_path": "sys/bgp", "collection_id": 555, "collection_start_time": 0, "collection_end_time": 0, "msg_timestamp": 0, "subscription_id": [ ], "sensor_group_id": [ ], "data_source": "DME", "data": { "bgpEntity": { "attributes": { "adminSt": "enabled", "childAction": "", "dn": "sys/bgp", "modTs": "2017-08-29T13:33:32.625+00:00", "monPolDn": "uni/fabric/monfab-default", "name": "bgp", "operErr": "", "operSt": "enabled", "persistentOnReload": "true", "rn": "bgp", "uid": "0" }, "children": [ { "bgpInst": { "attributes": { "activateTs": "2017-08-29T13:42:15.880+00:00", "adminSt": "enabled", "affGrpActv": "0", "asPathDbSz": "172", "asn": "65533", "attribDbSz": "784", "childAction": "", "createTs": "2017-08-29T13:42:15.168+00:00", "ctrl": "fastExtFallover", "disPolBatch": "disabled", "disPolBatchv4PfxLst": "", "disPolBatchv6PfxLst": "", "dn": "sys/bgp/inst", "epeActivePeers": "0", "epeConfiguredPeers": "0", "fabricSoo": "unknown:unknown:0:0", "flushRoutes": "disabled", "isolate": "disabled", "lnkStClnt": "inactive", "lnkStSrvr": "inactive", "medDampIntvl": "0", "memAlert": "normal", "modTs": "2017-08-29T13:42:13.853+00:00", "monPolDn": "uni/fabric/monfab-default", "name": "bgp-Inst", "numAsPath": "6", "numRtAttrib": "7", "operErr": "", "persistentOnReload": "true", "rn": "inst", "snmpTrapSt": "disable", "srgbMaxLbl": "none", "srgbMinLbl": "none", "syslogLvl": "err", "uid": "0", "ver": "v4", "waitDoneTs": "2017-08-29T13:42:15.880+00:00" }, "children": [ { "bgpEvtHist": { "attributes": { "childAction": "", "dn": "sys/bgp/inst/evthist-cli", "modTs": "2017-08-29T13:42:13.853+00:00", "persistentOnReload": "true", "rn": "evthist-cli", "size": "default", "type": "cli", "uid": "0" } } }, { "bgpEvtHist": { "attributes": { "childAction": "", "dn": "sys/bgp/inst/evthist-events", "modTs": "2017-08-29T13:42:13.853+00:00", "persistentOnReload": "true", "rn": "evthist-events", "size": "default", "type": "events", "uid": "0" } } }, { "bgpEvtHist": { "attributes": { "childAction": "", "dn": "sys/bgp/inst/evthist-errors", "modTs": "2017-08-29T13:42:13.853+00:00", "persistentOnReload": "true", "rn": "evthist-errors", "size": "default", "type": "errors", "uid": "0" } } }, { "bgpDom": { "attributes": { "always": "disabled", "bestPathIntvl": "300", "bgpCfgFailedBmp": "", "bgpCfgFailedTs": "00:00:00:00.000", "bgpCfgState": "0", "cfgSrcCtrlr": "disabled", "childAction": "", "clusterId": "", "dn": "sys/bgp/inst/dom-default", "firstPeerUpTs": "2017-08-29T13:42:38.791+00:00", "holdIntvl": "180", "id": "1", "kaIntvl": "60", "localAsn": "", "maxAsLimit": "0", "modTs": "2017-08-29T13:42:15.912+00:00", "mode": "fabric", "monPolDn": "uni/fabric/monfab-default", "name": "default", "numEstPeers": "2", "numPeers": "2", "numPeersPending": "0", "operRtrId": "192.168.0.3", "operSt": "up", "persistentOnReload": "true", "pfxPeerTimeout": "30", "pfxPeerWaitTime": "90", "rd": "unknown:unknown:0:0", "reConnIntvl": "60", "rn": "dom-default", "routerMac": "00:00:00:00:00:00", "rtrId": "192.168.0.3", "uid": "0", "vnid": "0", "vtepIp": "0.0.0.0", "vtepVirtIp": "0.0.0.0" }, "children": [ { "bgpPeer": { "attributes": { "activePfxPeers": "0", "addr": "172.20.0.1", "adminSt": "enabled", "affGrp": "0", "asn": "65531", "bgpCfgFailedBmp": "", "bgpCfgFailedTs": "00:00:00:00.000", "bgpCfgState": "0", "capSuppr4ByteAsn": "disabled", "childAction": "", "connMode": "", "ctrl": "", "curPfxPeers": "0", "dn": "sys/bgp/inst/dom-default/peer-[172.20.0.1]", "dynRtMap": "", "epe": "disabled", "epePeerSet": "", "holdIntvl": "180", "inheritContPeerCtrl": "", "kaIntvl": "60", "logNbrChgs": "none", "lowMemExempt": "disabled", "maxCurPeers": "0", "maxPeerCnt": "0", "maxPfxPeers": "0", "modTs": "2017-08-29T13:42:18.317+00:00", "monPolDn": "uni/fabric/monfab-default", "name": "nx-osv9000-1", "password": "", "peerImp": "", "peerType": "fabric-internal", "persistentOnReload": "true", "privateASctrl": "none", "rn": "peer-[172.20.0.1]", "sessionContImp": "", "srcIf": "eth1/1", "totalPfxPeers": "0", "ttl": "0", "ttlScrtyHops": "0", "uid": "0" }, "children": [ { "bgpPeerEntry": { "attributes": { "addr": "172.20.0.1", "advCap": "as4,dynamic,dynamic-gr,dynamic-mp,dynamic-old,dynamic-refresh,gr,ipv4-ucast,refresh,refresh-old", "childAction": "", "connAttempts": "1", "connDrop": "0", "connEst": "1", "connIf": "eth1/1", "dn": "sys/bgp/inst/dom-default/peer-[172.20.0.1]/ent-[172.20.0.1]", "fd": "79", "flags": "cap-neg,direct-connect,gr-enabled", "holdIntvl": "180", "kaIntvl": "60", "lastFlapTs": "2017-08-29T13:42:38.791+00:00", "localIp": "172.20.0.2", "localPort": "43494", "maxConnRetryIntvl": "60", "modTs": "never", "monPolDn": "", "name": "", "operSt": "established", "passwdSet": "disabled", "peerIdx": "3", "persistentOnReload": "false", 
```

This data contains a wealth of information regarding the BGP process on each switch. This data is as near real-time as we can possibly get and can be forwarded to data-analytics and visualization tools such as the ELK stack or influxDB, Prometheus etc. 

Thus using the Telemetry feature on the NX-OS devices, network administrators are now empowered with analytics data, that can be subscribed to, using applications. The reliance on SNMP and CLI scraping based monitoring are no longer impediments in relevant, real-time data.


### Interested in more detail on Streaming Telemetry on NX-OS?

You can learn more about telemetry on the NX-OS in the [NX-OS Programmability Guide](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/programmability/guide/b_Cisco_Nexus_9000_Series_NX-OS_Programmability_Guide_7x/b_Cisco_Nexus_9000_Series_NX-OS_Programmability_Guide_7x_chapter_011000.html)

Telemetry on the NXOS is still a very new feature and it is important to be cognizant of some of it's limitations. Telemetry is supported in Cisco NX-OS releases starting from 7.0(3)I5(1) for releases that support the [Data Management Engine](https://developer.cisco.com/site/nx-api/documents/n3k-n9k-api-ref/?shell#) (DME) Native Model.

A comprehensive list of restrictions and guidelines are [published](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/programmability/guide/b_Cisco_Nexus_9000_Series_NX-OS_Programmability_Guide_7x/b_Cisco_Nexus_9000_Series_NX-OS_Programmability_Guide_7x_chapter_011000.html#id_40825) on Cisco's website.
