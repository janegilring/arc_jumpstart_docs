---
type: docs
title: "VMware vSphere Linux VMs"
linkTitle: "VMware vSphere Linux VMs"
weight: 2
description: >
---

## Scaled onboarding of VMware vSphere Linux VMs to Azure Arc using VMware PowerCLI

The following Jumpstart scenario will guide you on how to use the provided [VMware PowerCLI](https://code.vmware.com/web/dp/tool/vmware-powercli/) script so you can perform an automated scaled deployment of the "Azure Arc Connected Machine Agent" in multiple VMware vSphere virtual machines and as a result, onboard these VMs as Azure Arc-enabled servers.

This guide assumes you already have an exiting inventory of VMware Virtual Machines and will leverage the PowerCLI PowerShell module to automate the onboarding process of the VMs to Azure Arc.

## Prerequisites

- Clone the Arc Jumpstart GitHub repository

    ```shell
    git clone https://github.com/microsoft/azure_arc.git
    ```

- [Install or update Azure CLI to version 2.65.0 and above](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest). Use the below command to check your current installed version.

  ```shell
  az --version
  ```

- Install VMware PowerCLI

  > **Note:** This guide was tested with the latest version of PowerCLI as of date (12.0.0) but earlier versions are expected to work as well.

  - Supported PowerShell Versions - VMware PowerCLI 12.0.0 is compatible with the following PowerShell versions:
    - Windows PowerShell 5.1
    - PowerShell 7
    - Detailed installation instructions can be found [here](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.esxi.install.doc/GUID-F02D0C2D-B226-4908-9E5C-2E783D41FE2D.html) but the easiest way is to use the VMware.PowerCLI module from the PowerShell Gallery using the below command.

    ```powershell
    Install-Module -Name VMware.PowerCLI
    ```

- To be able to read the VM inventory from vCenter as well as invoke a script on the VM OS-level, the following permissions are needed:

  - [VirtualMachine.GuestOperations](https://docs.vmware.com/en/VMware-vSphere/7.0/com.vmware.vsphere.security.doc/GUID-6A952214-0E5E-4CCF-9D2A-90948FF643EC.html) user account

  - VMware vCenter Server user assigned with a ["Read Only Role"](https://docs.vmware.com/en/VMware-vSphere/6.7/com.vmware.vsphere.security.doc/GUID-93B962A7-93FA-4E96-B68F-AE66D3D6C663.html)

- An operating system user account on the Linux guest VM. This user account must not prompt for password on sudo commands. To configure passwordless sudo:

  - Login to the linux VM.

  - Run the below command.
  
    ```shell
    sudo visudo
    ```

    Or you could also edit the /etc/sudoers file directly with the command:

     ```shell
    vi /etc/sudoers
    ```

  - Append the following line replacing <username> with the appropriate user name.

    ```shell
    <username> ALL=(ALL) NOPASSWD:ALL
    ```

- Create Azure service principal (SP)

    To connect the VMware vSphere virtual machine to Azure Arc, an Azure service principal assigned with the "Contributor" role is required. To create it, login to your Azure account run the below command (this can also be done in [Azure Cloud Shell](https://shell.azure.com/)).

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "<Unique SP Name>" --role "Contributor" --scopes /subscriptions/$subscriptionId
    ```

    For example:

    ```shell
    az login
    subscriptionId=$(az account show --query id --output tsv)
    az ad sp create-for-rbac -n "JumpstartArc" --role "Contributor" --scopes /subscriptions/$subscriptionId
    ```

    Output should look like this:

    ```json
    {
    "appId": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "displayName": "JumpstartArc",
    "password": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "tenant": "XXXXXXXXXXXXXXXXXXXXXXXXXXXX"
    }
    ```

    > **Note:** If you create multiple subsequent role assignments on the same service principal, your client secret (password) will be destroyed and recreated each time. Therefore, make sure you grab the correct password.

    > **Note:** The Jumpstart scenarios are designed with as much ease of use in-mind and adhering to security-related best practices whenever possible. It is optional but highly recommended to scope the service principal to a specific [Azure subscription and resource group](https://learn.microsoft.com/cli/azure/ad/sp?view=azure-cli-latest) as well considering using a [less privileged service principal account](https://learn.microsoft.com/azure/role-based-access-control/best-practices).

- Azure Arc-enabled servers depends on the following Azure resource providers in your subscription in order to use this service. Registration is an asynchronous process, and registration may take approximately 10 minutes.

  - Microsoft.HybridCompute
  - Microsoft.GuestConfiguration
  - Microsoft.HybridConnectivity

      ```shell
      az provider register --namespace 'Microsoft.HybridCompute'
      az provider register --namespace 'Microsoft.GuestConfiguration'
      az provider register --namespace 'Microsoft.HybridConnectivity'
      ```

      You can monitor the registration process with the following commands:

      ```shell
      az provider show --namespace 'Microsoft.HybridCompute'
      az provider show --namespace 'Microsoft.GuestConfiguration'
      az provider show --namespace 'Microsoft.HybridConnectivity'
      ```

## Automation Flow

Below you can find the automation flow for this scenario:

1. User edit the [*vars.ps1*](https://github.com/microsoft/azure_arc/blob/main/azure_arc_servers_jumpstart/vmware/scaled_deployment/powercli/linux/vars.ps1) PowerCLI script

2. Upon execution of the [*scale_deploy.ps1*](https://github.com/microsoft/azure_arc/blob/main/azure_arc_servers_jumpstart/vmware/scaled_deployment/powercli/linux/scale_deploy.ps1) PowerShell script:

    - The script will auto-generate a *vars.sh* shell script with the user's Azure environment variables.

    - The script execution will initiate authentication against vCenter and will scan the targeted VM folder where Azure Arc candidate VMs are located and will copy both the auto-generated *vars.sh* and the *install_arc_agent.sh* shell scripts to VM Linux OS located in */vmware/scaled_deploy/powercli/linux* to each VM in that VM folder.

3. The *install_arc_agent.sh* shell script will run on the VM guest OS and will install the "Azure Arc Connected Machine Agent" in order to onboard the VM to Azure Arc

## Pre-Deployment

To demonstrate the before & after for this scenario, the below screenshots shows a dedicated, empty Azure resource group, a vCenter VM folder with candidate VMs and the */var/opt/* directory showing no agent is installed.

![An empty Azure resource group](./01.png)

![Vanilla VMware vSphere VM with no Azure Arc agent](./02.png)

![Vanilla VMware vSphere VM with no Azure Arc agent](./03.png)

## Deployment

Before running the PowerCLI script, you must set the [environment variables](https://github.com/microsoft/azure_arc/blob/main/azure_arc_servers_jumpstart/vmware/scaled_deployment/powercli/linux/vars.ps1) which will be used by the *install_arc_agent.sh* script. These variables are based on the Azure service principal you've just created, your Azure subscription and tenant, and your VMware vSphere credentials and data.

- Retrieve your Azure subscription ID and tenant ID using the *`az account list`* command

- Use the Azure service principal ID and password created in the prerequisites section

    ![Export environment variables](./04.png)

- From the [*azure_arc_servers_jumpstart\vmware\scaled_deploy\powercli\linux*](https://github.com/microsoft/azure_arc/blob/main/azure_arc_servers_jumpstart/vmware/scaled_deployment/powercli/linux/) folder, open PowerShell session as an Administrator and run the *scale_deploy.ps1* script.

    ![scale_deploy PowerShell script](./05.png)

    ![scale_deploy PowerShell script](./06.png)

    ![scale_deploy PowerShell script](./07.png)

- Upon completion, the VM will have the "Azure Arc Connected Machine Agent" installed as well as the Azure resource group populated with the new Azure Arc-enabled servers.

    ![Azure Arc Connected Machine Agent installed](./08.png)

    ![New Azure Arc-enabled servers in an Azure resource group](./09.png)

    ![New Azure Arc-enabled servers in an Azure resource group](./10.png)
