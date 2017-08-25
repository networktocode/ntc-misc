# Power On Auto Provisioning (POAP) on Nexus Switches Part 2 - Boostrapping Nexus Switches with POAP

Part 1 of this lab was focused on setting up the centralized DHCP, TFTP and Software server. In this part, we will initiate the POAP process by booting up 4 brand new switches and observing the auto-provisioning process unfold. In this part, we will also learn how to leverage the POAP process to reestablish baseline configuration on existing devices.

## Centralized server setup

If the TFTP, DHCP and POAP Server components are up and running from part 1, you can skip this step. If not, navigate to the `poap` directory and execute the following playbook to bring all the centralized server components up.

``` shell
(python2) [root@localhost poap]# cd /root/sbx_nxos/learning_labs/poap
(python2) [root@localhost poap]# ansible-playbook central_server.yml


PLAY [devbox] ******************************************************************

TASK [setup] *******************************************************************
ok: [10.10.20.20]

TASK [dhcp_server : include_vars] **********************************************
ok: [10.10.20.20] => (item=/root/sbx_nxos/learning_labs/poap/dhcp_server/vars/RedHat.yml)

TASK [dhcp_server : Install packages] ******************************************
changed: [10.10.20.20] => (item=[u'dhcp'])

TASK [dhcp_server : AppArmor fix | Check if policy file exists] ****************
skipping: [10.10.20.20]

TASK [dhcp_server : AppArmor fix | Ensure dhcpd can acces temp config file for validation (1/2)] ***
skipping: [10.10.20.20]

TASK [dhcp_server : AppArmor fix | Ensure dhcpd can acces temp config file for validation (2/2)] ***
skipping: [10.10.20.20]

TASK [dhcp_server : AppArmor fix | Restart AppArmor] ***************************
skipping: [10.10.20.20]

TASK [dhcp_server : Install includes] ******************************************

TASK [dhcp_server : Set config directory perms] ********************************
changed: [10.10.20.20]

TASK [dhcp_server : Install config file] ***************************************
changed: [10.10.20.20]

TASK [dhcp_server : Ensure service is started] *********************************
changed: [10.10.20.20]

RUNNING HANDLER [dhcp_server : restart dhcp] ***********************************
changed: [10.10.20.20]

PLAY [INSTALL THE TFTP SERVER] *************************************************

TASK [setup] *******************************************************************
ok: [10.10.20.20]

TASK [tftp_server : include_vars] **********************************************
ok: [10.10.20.20] => (item=/root/sbx_nxos/learning_labs/poap/tftp_server/vars/CentOS.yml)

TASK [tftp_server : Install packages] ******************************************
changed: [10.10.20.20] => (item=[u'libsemanage-python', u'tftp-server', u'xinetd'])

TASK [tftp_server : Ensure tftp root directory exists] *************************
changed: [10.10.20.20]

TASK [tftp_server : Ensure SELinux boolean ‘tftp_anon_write’ has the desired value] ***
changed: [10.10.20.20]

TASK [tftp_server : Ensure SELinux boolean ‘tftp_home_dir’ has the desired value] ***
changed: [10.10.20.20]

TASK [tftp_server : Ensure service is started] *********************************
changed: [10.10.20.20]

TASK [tftp_server : Install configuration file] ********************************
changed: [10.10.20.20]

TASK [tftp_server : Install foreman map file] **********************************
skipping: [10.10.20.20]

RUNNING HANDLER [tftp_server : restart tftp] ***********************************
changed: [10.10.20.20]

PLAY [INSTALL THE REQUIREMENTS FOR FLASK] **************************************

TASK [setup] *******************************************************************
ok: [10.10.20.20]

TASK [INSTALL POAP SERVER REQUIREMENTS] ****************************************
changed: [10.10.20.20]

PLAY RECAP *********************************************************************
10.10.20.20                : ok=18   changed=13   unreachable=0    failed=0   

(python2) [root@localhost poap]# 


```

> This playbook executes all the playbooks used in part 1 of this lab, setting up the DHCP, TFTP and POAP servers, so you can more easily perform just the tasks in Part 2.

At this point the DHCP and TFTP server should be up and running. The next step is to bring up the POAP server in debug mode.

``` shell
(python2) [root@localhost poap]#cd poap_app
(python2) [root@localhost poap_app]#
(python2) [root@localhost poap_app]# python poap_app.py 
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 117-915-351

```

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


    > **IMPORTANT:** It has been observed that, in the devnet sandbox environment, occasionally, a switch might not pull down the `poap_script.py` file. If this happens, rebooting that switch seems to fix the problem. You can reboot a single switch by following the steps below:


In a different terminal, navigate to the `poap` directory and execute the `restart-sbx.py` script.

``` bash
(python2) [root@localhost poap]# cd /root/sbx_nxos/learning_labs/poap
(python2) [root@localhost poap]# python restart-sbx.py
VIRL Simulation Name: API-POAP

Which node would you like to restart?
  0 - csr1000v-1: Status ACTIVE
  1 - nx-osv9000-1: Status ACTIVE
  2 - nx-osv9000-2: Status ACTIVE
  3 - nx-osv9000-3: Status ACTIVE
  4 - nx-osv9000-4: Status ACTIVE
  5 - ~mgmt-lxc: Status ACTIVE
  a - Restart All Nodes
Enter 0 - 5 to choose a node, or a for all
4
Stopping Nodes
nx-osv9000-4

Nodes not stopped yet
Nodes not stopped yet
Nodes not stopped yet
Starting Nodes
nx-osv9000-4

Nodes not started yet
Nodes not started yet
Nodes have been restarted, however it can take up to 15 minutes for all switches to fully boot and be ready.
You can monitor the startup activity at:
    Console to nx-osv9000-4 -> `telnet 10.10.20.160 17002`

```



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

    
