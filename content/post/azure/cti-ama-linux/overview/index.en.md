+++
author = "Lucas Huang"
date = '2025-06-24T13:45:00+08:00'
title = "Deep Dig into Linux Change Tracking and Inventory with Azure Montior Agent"
# description = "This article demonstrates how to deploy a Hugo web application to Azure Static Web Apps"
categories = [
    "Azure"
]
tags = [
    "Azure Monitor Agent",
    "Change Tracking and Inventory"
]
image = "cover.png"
series = ["Linux CT&I With AMA"]
# draft = true
+++


## Architecture

### Overview
As for a big picture, Azure Monitor Agent (AMA) is responsible for uploading Change Tracking and Inventory (CT&I) data to Azure backend, while CT&I agent is just responsible for collecting data from Operation System and pass them to AMA.

![CT&I Architecture](CT&I-Architecture.png)

### Extension Portal view 

![CT&I Extension Portal View](CT&I-Extension-Portal-View.png)

## How Change Tracking & Inventory for AMA Linux Data is Collected

This section has two parts involved, namely what AMA does and what CT&I Agent does. 

### Part 1: AMA Initialization

![AMA Linux Initialization](AMA-Linux-Initialization.png)


1. Azure Monitor Agent (AMA) fetches all the DCRs, including CT&I DCR, and stores those DCRs at `/etc/opt/microsoft/azuremonitoragent/config-cache/configchunks`

2. As per the DCRs details, AMA loads the datasources and initializes the LocalSink(cache) at `/var/opt/microsoft/azuremonitoragent/events`. Specifically for CT&I DCR, below folders will be created under the above cache directory.

    - dcr-xxx:CTDataSource-Linux:CONFIG_CHANGE_BLOB

    - dcr-xxx:CTDataSource-Linux:CONFIG_CHANGE_BLOB_V2

    - dcr-xxx:CTDataSource-Linux:CONFIG_DATA_BLOB_V2

    - Similarly, Windows *3 (though won't be used)

3. AMA initializes ODSUploader Tasks for these cache. So that cache data will be uploaded to backend at an interval. 

4. AMA initializes extension sockets. (These are virtual sockets, not a socket file on OS)
    - `@CAgentStream_CloudAgentInfo_config_default_fluent.socket` - CT&I agent will connect to this socket to obtain DCR config from AMA. 

    - `@CAgentStream_CloudAgentInfo_MaExtensionDiagnostics_default_fluent.socket` - so far CT&I Agent don't use this. 

    - `@CAgentStream_CloudAgentInfo_ChangeTracking-Windows_default_fluent.socket` - isn't used by Linux  CT&I Agent.

    - `@CAgentStream_CloudAgentInfo_ChangeTracking-Linux_default_fluent.socket` - CT&I will write collected data(Software ,Services, files) to AMA via this socket.  

    ```bash
    root@LucasTestCT:/# ss -x | grep CAgentStream

    u_str              ESTAB               0                    0                    @CAgentStream_CloudAgentInfo_ChangeTracking-Linux_default_fluent.socket@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@ 34144                                    * 33175 

    ```


### Part 2: CT&I Agent Workflow

1. CT&I Agent gets DCR from AMA via  `@CAgentStream_CloudAgentInfo_config_default_fluent.socket`. This behaviour refreshes every 5 mins. 
    > **Note**: CT&I Agent does not support multiple DCRs. At a time only 1 DCR is supported. If there are multiple CT DCRs assigned to AMA, which DCR finally CT&I Agent gets is random.
2. Per the frequency defined in DCR, Files Scheduler, Services Scheduler, Software Scheduler will be initialized. These schedulers will then trigger corresponding workers to fetch the CT&I data. Please refer to detailed workflow chart as below.
3. When CT&I data are collected, CT&I Agent forwards these data to AMA via `@CAgentStream_CloudAgentInfo_ChangeTracking-Linux_default_fluent.socket`. AMA will be then responsible for data upload to Azure backend. 

#### Services
![CT&I Linux Agent Services Collection Workflow](CT&I-Linux-Agent-Services-Collection-Workflow.png)

Equivalence:
```
systemctl -a list-unit-files *.service | grep .service | grep -v "@" | awk '{print $1}' | xargs systemctl -a --no-pager --no-legend -p "Names,WantedBy,Description,SubState,FragmentPath,UnitFileState" show
```


#### Software
![CT&I Linux Agent Software Collection Workflow](CT&I-Linux-Agent-Software-Collection-Workflow.png)



- Distros with dpkg (Ubuntu, Debian, etc.): 
```
dpkg-query -W -f 'Name:{Package}\nDescription:{Description}\nMaintainer:{Maintainer}\n'Unknown';4\nSize:{Installed-Size}\nVersion:{Version}\nStatus:{Status}\n{db:Status-Status}\nArch:{Architecture}\n\n'
```


- Distros with rpm (Redhat, CentOS, SUSE, etc.): 
```
rpm -qa --queryformat "%{NAME}; %{SUMMARY}; %{PACKAGER}; %{INSTALLTIME}; %{SIZE}; %{EPOCH}:%{VERSION}-%{RELEASE}; installed; %{ARCH}\n@@" | sed "s/(none)/0/g"
```



#### Files
![CT&I Linux Agent Files Collection Workflow](CT&I-Linux-Agent-Files-Collection-Workflow.png)


### What is DB in above workflows
---
The CT&I Agent will store collected data in the DB file `/opt/microsoft/changetrackingdata/db/changetracking.db`. The DB file can be directly open by any text editor like below, or use https://github.com/ShoshinNikita/boltBrowser. 

![CT&I Linux Agent Database](CT&I-Linux-Agent-Database.png)

Though it contains all the data, it is not recommended to analyze the DB file directly as it is hard to read. We have friendly readable json files at below folder for Azure Arc and Azure VM respectively, which corresponds to the workers results, and are the same with the DB's content.

- Azure VM: `/var/log/azure/Microsoft.Azure.ChangeTrackingAndInventory.ChangeTracking-Linux`
- Azure Arc: `/var/lib/waagent/Microsoft.Azure.ChangeTrackingAndInventory.ChangeTracking-Linux-<version>`
  - File.json             
  - Packages.json         
  - Services.json         

Please note that both DB and the json files only stores the latest collected data (last worker run result). 

## Documentation 

Here is the link to CT&I with AMA documents: 

- [Overview of change tracking and inventory using Azure Monitoring Agent](https://learn.microsoft.com/azure/automation/change-tracking/overview-monitoring-agent)

- [Supported Regions](https://learn.microsoft.com/azure/automation/change-tracking/region-mappings-monitoring-agent)

- [Enable Change Tracking and Inventory using Azure Monitoring Agent](https://learn.microsoft.com/azure/automation/change-tracking/enable-vms-monitoring-agent?tabs=singlevm)

- [Configure Alerts](https://learn.microsoft.com/azure/automation/change-tracking/configure-alerts)

- [Extension Version Details and Known Issues](https://learn.microsoft.com/azure/automation/change-tracking/extension-version-details)
