+++
author = "Lucas Huang"
date = '2025-05-11T07:48:22+08:00'
title = "Azure Monitor Windows Agent Extension Operations: Handling ValidateDataStorePersistence Errors"
# description = "This article demonstrates how to deploy a Hugo web application to Azure Static Web Apps"
categories = [
    "Azure"
]
tags = [
    "Azure Monitor Windows Agent",
    "Azure Monitor",
]
image = "cover.png"
# draft = true
+++
## Introduction  
In this post, we’ll tackle a common issue affecting the Azure Monitor Agent (AMA) on Windows - **ValidateDataStorePersistence** failures. Starting with AMA version 1.30.0.0, a security check was added ([CVE-2024-38097](https://nvd.nist.gov/vuln/detail/CVE-2024-38097)) to ensure the integrity of the agent’s data store. While this improves security, it can lead to heartbeats stopping and Agent processes failing to start if the persistence key or registry entry goes missing or is out of sync. I’ll walk you through how to diagnose the problem and get your AMA back online quickly.

## Symptoms  
When the AMA extension enters a transition state or fails to launch, you may observe:  
- Heartbeats from the agent stop in Azure Monitor  
- [MonAgent* processes]({{< ref "/post/azure/ama-win-processes" >}}) do not appear in Task Manager  
- The VM extension status remains “transitioning” indefinitely  
- ExtensionHealth.*.log (on the VM or Azure Arc machine) contains one or more of these errors: 
  ``` text
  The key file exists but the registry value does not. Cannot launch AMA. 
  The registry value exists but the key file does not. Cannot launch AMA.
  The key in the file does not match the expected key. Cannot launch AMA. 
  ```
  Log locations for ExtensionHealth.*.log:  
  - Azure VM: 
    `C:\WindowsAzure\Logs\Plugins\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\{version}\ExtensionHealth.*.log`
  - Azure Arc: 
    `C:\ProgramData\GuestConfig\extension_logs\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\ExtensionHealth.*.log`  


## Root Cause  
The extension’s persistence check ensures that the agent’s local data store hasn’t been tampered with via symbolic links, junctions, deletions or other redirections. If the `.persistencekey` file and the corresponding registry entry go out of sync - or if an attacker attempts to redirect SYSTEM calls - AMA refuses to start as a security precaution.

## Step-by-Step Resolution  
1. Validate the integrity of your data store directory (no unexpected symbolic links, junctions or file deletions).  

2. Disable the Azure Monitor Agent locally  
   • Follow the [AMA Disable Guide]({{< ref "/post/azure/ama-win-restart" >}}) to diasble the AMA locally.  

3. Remove the Stale Persistence Key  
   a. Delete the registry key if exist:  
      Path: HKLM\SOFTWARE\Microsoft\AzureMonitorAgent\Secrets  
      Name: PersistenceKeyCreated  

   b. Delete the `.persistencekey` file from the data store directory if exist:  
      - Azure VM:  
        C:\WindowsAzure\Resources\{dataStoreName}\.persistencekey  
      - Azure Arc:  
        C:\Resources\Directory\{dataStoreName}\.persistencekey  

4. Re-enable the Azure Monitor Agent locally  
   - Follow the [AMA Enable Guide]({{< ref "/post/azure/ama-win-restart" >}}) to re-enable the AMA locally.
   - The extension’s health check will recreate the `.persistencekey` file and the registry entry automatically.  

5. Verify the Fix  
   - Monitor the ExtensionHealth.*.log for any new ValidateDataStorePersistence errors.  
   - Check that the [MonAgent* processes]({{< ref "/post/azure/ama-win-processes" >}}) are running and heartbeats resume in Log Analytics Workspace.  

## Conclusion
By performing a quick disable - delete - enable cycle on the Azure Monitor Windows Agent extension, you restore the data store persistence alignment and eliminate the ValidateDataStorePersistence errors. This not only gets your agent back online but also maintains the security safeguard introduced in AMA 1.30.0.0. As always, keep your agents up to date and monitor your extension logs for early detection of any anomalies. 