# Automating Cisco ACI with Ansible

Ansible has quickly become a popular platform for network engineers to get started with network automation and eliminate repetitive day to day tasks.  There have been Ansible plug-ins (in the form of modules) for Cisco IOS, NX-OS, and IOS-XR for quite some time, but only recently has support been added to Ansible for Cisco Application Centric Infrastructure (ACI).

In the latest Ansible release (version 2.4), thirty-two (32) Cisco ACI modules were added to Ansible core.  This means you get this functionality as soon as you install Ansible! These modules allow you to manage ACI fabrics just like you'd manage any other device type with Ansible--through the use of resource-specific and idempotent tasks including one module that allows you to send any arbitrary JSON/XML object to the APIC.

Let's take a look and see how to get started with these modules.

## Basic Intro to Ansible

If you're new to Ansible, you only need two things to start automating with Ansible:  an inventory file (contains what devices you're automating) and a playbook (contains your automation instructions).


### ACI Inventory File

Our sample inventory file looks like this and it's just pointing to the public always-on APIC Sandbox environment:

```
[apic]
sandboxapicdc.cisco.com username=admin password=ciscopsdt
```

As you can see, it's literally just one line in a text file.  Note, in our example, we're storing credentials in the inventory file as it's the quickest way to highlight what Ansible can do for ACI.  For production, you may want to check out Ansible Vault or Tower.

### ACI Playbooks

Ansible uses playbooks written in YAML to define the tasks we want to execute against an ACI fabric (in this case).  The playbook also defines which devices those tasks will be executed against.

Here is a sample playbook that we use to retrieve tenant and EPG information from an existing ACI fabric.  Note that many modules in Ansible support the `state` parameter with the values of **present** and **absent**, the ACI modules support a 3rd value called **query** that allows us to simply _query_ and retrieve information for the given resource.  In this context, a resource is either tenant data or EPG data.

```yaml
- name: PLAY 1
  hosts: apic                        # Uses a device (or group) defined in our inventory file
  connection: local                  # Determines how to connect to the device
  gather_facts: no
  
  tasks:                                
    - name: TASK 1 – GATHER TENANTS
      aci_tenant:
        hostname: "{{ inventory_hostname }}"    
        username: "{{ username }}"      # username variable defined in our inventory file
        password: "{{ password }}"      # password variable defined in our inventory file
        validate_certs: no
        state: query                    # Task tells the module to query the APIC for Tenants

    - name: TASK 2 – GATHER EPGS
      aci_epg:
        hostname: "{{ inventory_hostname }}"
        username: "{{ username }}"
        password: "{{ password }}"
        validate_certs: no
        state: query                    # Task tells the module to query the APIC for EPGs
```

###  Executing the Playbook 

You then invoke this playbook from your Linux terminal by using the `ansible-playbook` command and passing the inventory file and playbook file:

```
ntc@ntc:~$ ansible-playbook -i inventory aci_playbook.yml -v
```

> We've add the `-v` verbose flag as this allows us to see the data being gathered from each task. You can subsequently write this data to a file using the `copy` or `template` modules to create nice automated reports.

### Using ACI Resource Specific Modules

Now that you have an understanding of how to use and create a basic playbook using ACI modules, let's take a deeper look. As stated already, the ACI modules support three `state` values:

* **present** - ensures the object exists and is configured per the task's parameters
* **absent** - ensures the object does not exist
* **query** - retrieves configurations and state for the object or object type

The **query** state considers all ACI _class_ parameters and returns the most specific results based on the parameters that are passed to the module. 

For example, this task will return just the data on the specific EPG called **web**:

```yaml
- name: RETURN DATA FOR SPECIFIC EPG
  aci_epg:
    host: "{{ inventory_hostname }}"
    username: "{{ username }}"
    password: "{{ password }}"
    validate_certs: no
    tenant: prod_tenant
    ap: intranet
    epg: web
```

This task will return all EPGs in the specified Application Profile, **intranet**, since the `epg` parameter is not provided:

```yaml
- name: RETURN DATA FOR ALL EPGS IN SPECIFIED APP PROFILE
  aci_epg:
    host: "{{ inventory_hostname }}" 
    username: "{{ username }}"
    password: "{{ password }}"
    validate_certs: no
    tenant: prod_tenant
    ap: intranet
```

This task will return all "web" EPGs in the specified Tenant regardless of Application Profile since the `ap` parameter is not provided:

```yaml
- name: RETURN DATA FOR ALL WEB EPGS IN SPECIFIED TENANT
  aci_epg:
    host: "{{ inventory_hostname }}"
    username: "{{ username }}"
    password: "{{ password }}"
    validate_certs: no
    tenant: prod_tenant
    epg: web
```

This task will return all EPGs configured on the APIC since it does not pass any ACI _class_ parameters:

```yaml
- name: RETURN DATA FOR ALL EPGS
  aci_epg:
    host: "{{ inventory_hostname }}"
    username: "{{ username }}"
    password: "{{ password }}"
    validate_certs: no
```

Each of the possible combinations have potential use cases; we will look at gathering all App Profiles for a specified tenant. This data could be useful to gather potential impacted Applications for a maintenance window. We will use the `aci_ap` module to collect the data, the `set_fact` module to extract just the useful data, and the `debug` module to display the App Profiles.

```yaml
---

- name: FIND POTENTIAL APP PROFILES IMPACTED BY MAINTENANCE
  hosts: apic
  connection: local
  gather_facts: no

  tasks:
    - name: COLLECT LIST OF APP PROFILES FOR TENANT
      aci_ap:
        host: "{{ inventory_hostname }}"
        username: "{{ username }}"
        password: "{{ password }}"
        validate_certs: no
        tenant: prod_tenant
        state: query
      register: prod_apps

      # ASSUMES THERE IS AT LEAST 1 AP IN THE TENANT
    - name: CREATE A LIST OF JUST APP NAMES USING A QUERY FILTER
      set_fact: 
        apps: "{{ prod_apps | json_query('existing[0].fvTenant.children[*].fvAp.attributes.name') }}"
 
    - name: PRINT LIST OF APPS IN OUR TENANT
      debug: var=apps    
    

```

From the terminal, we can invoke this playbook and see the list of apps for the "prod_tenant":

```
$ ansible-playbook find_tenant_apps - i inventory

PLAY [FIND POTENTIAL APP PROFILES IMPACTED BY MAINTENANCE] *************************************************

TASK [COLLECT LIST OF APP PROFILES FOR TENANT] *************************************************************
ok: [sandboxapicdc.cisco.com]

TASK [CREATE A LIST OF JUST APP NAMES USING A QUERY FILTER] ************************************************
ok: [sandboxapicdc.cisco.com]

TASK [PRINT LIST OF APPS IN OUR TENANT] ********************************************************************
ok: [sandboxapicdc.cisco.com] => {
    "apps": [
        "benefits", 
        "intranet", 
        "storage", 
        "ticketing"
    ]
}

PLAY RECAP *************************************************************************************************
sandboxapicdc.cisco.com    : ok=3    changed=0    unreachable=0    failed=0  
```

> In this case, we did not need to use the `-v` flag to see the data collected as we _registered_ and saved the return data, massaged the data to just get the app names, and then _printed_ them using the `debug` module.


The results from this execution show 4 apps belonging to the tenant, e.g. *benefits, intranet, storage,* and *ticketing*.

So far all we did was gather data simply because it's a great way to start automating and many examples out there only focus on configuration.  However, all you would need to do is change the `state` parameter to `present` to ensure a given configuration exist making it quite easy to either configure, un-configure, or simply _query_ and fetch data from an APIC.

Finally, let's take a look at the `aci_rest` module.


### Using ACI REST Module 

The `aci_rest` module allows us to make _any_ request to the APIC.  The previous modules were _resource-specific_, .e.g they're purpose built for a given object like tenants, EPGs, contracts, filters, etc.  The `aci_rest` is a "catch all" that allows you to make ANY API call, but it does require you to fully understand the API and the ACI object model.

We will focus on making GET requests, but if you start to use this for POST requests, the body can use YAML syntax if that is preferred over JSON or XML.

One of the extra abilities we get with the REST module is the ability to query for health, faults, and event-logs using query filters:

```yaml
- name: VIEW EPG HEALTH
  hosts: apic
  connection: local
  gather_facts: False
  
  tasks:
    - name: COLLECT EPG HEALTH
      aci_rest:
        hostname: "{{ inventory_hostname }}"
        username: "{{ username }}"
        password: "{{ password }}"
        validate_certs: false
        method: get
        path: "api/mo/uni/tn-{{ tenant }}/ap-{{ ap }}/epg-{{ epg }}.json?rsp-subtree-include=health,faults,event-logs"
      register: epg_health

    - name: PRINT LIST OF HEALTH CHILDREN
      debug:
        var: epg_health.imdata.0.fvAEPg.children

```

This playbook uses variables for the path as indicated by the double curly braces (`{{ variable }}`). We will pass these variables in as we call the playbook from the terminal as Ansible _extra vars_ using the `--extra-vars command line flag.

Let's take a look:

```
```ntc@ntc:~$ ansible-playbook -i inventory aci-pb.yml --extra-vars "tenant=prod_tenant ap=intranet epg=web"

PLAY [VIEW EPG HEALTH] *************************

TASK [COLLECT EPG HEALTH] **********************
ok: [sandboxapicdc.cisco.com]

TASK [PRINT LIST OF HEALTH CHILDREN] ***********
ok: [sandboxapicdc.cisco.com] => {
    "epg_health.imdata.0.fvAEPg.children": [
        {
            "healthNodeInst": {
                "attributes": {
                    "childAction": "", 
                    "chng": "-1", 
                    "cur": "95", 
                    "maxSev": "cleared", 
                    "nodeId": "102", 
                    "prev": "96", 
                    "rn": "nodehealth-102", 
                    "status": "", 
                    "twScore": "95", 
                    "updTs": "2017-10-18T18:20:21.941+00:00", 
                    "weight": "3"
                }
            }
        }, 
        {
            "healthNodeInst": {
                "attributes": {
                    "childAction": "", 
                    "chng": "-1", 
                    "cur": "95", 
                    "maxSev": "cleared", 
                    "nodeId": "101", 
                    "prev": "96", 
                    "rn": "nodehealth-101", 
                    "status": "", 
                    "twScore": "95", 
                    "updTs": "2017-10-18T18:15:21.054+00:00", 
                    "weight": "3"
                }
            }
        }, 
        {
            "healthInst": {
                "attributes": {
                    "childAction": "", 
                    "chng": "0", 
                    "cur": "95", 
                    "maxSev": "cleared", 
                    "prev": "95", 
                    "rn": "health", 
                    "status": "", 
                    "twScore": "95", 
                    "updTs": "2017-10-18T18:20:28.522+00:00"
                }
            }
        }
    ]
}
```

This example only had children for the "health" filter, so that is all that is returned. The EPG has a health score of "95" as do 2 of the nodes that have End Points connected to them; likely their is a minor issue with those two leaf nodes.

As you can see, the `aci_rest` module provides full access to the APIC REST API, but the downside is that it requires the user to know how to build the URL, including any query strings that help narrow the returned data.


This was meant purely to whet your appetite with getting started with Ansible to automate your ACI infrastructure.  There is no need to be in the APIC GUI clicking around when you can leverage a platform like Ansible.

Interested to learn more about using Ansible to automate ACI?  Check out these DevNet Learning Labs:
  * List labs


### About the Author

Jacob McGill is a network automation engineer at [Network to Code](http://networktocode.com), a network automation solutions provider that provides network automation training and professional services.  He spends his time helping customers automate their networks using tools including like Python and Ansible. Jacob is also an Ansible developer who's written and contributed to many of the ACI modules that were released in 2.4. Feel free to reach out to Jacob at jacob@networktocode.com 

