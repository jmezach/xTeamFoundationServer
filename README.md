# xTeamFoundationServer
PowerShell DSC resources for configuring TFS agents

This module provides two PowerShell DSC resources that can be used to configure either a TFS Build Controller (for TFS 2013) or a TFS Build Agent (for TFS 2015).

##xTFSBuildController##
This resource provides the ability to configure a TFS 2013 Build Controller on a machine. Note that it requires TFS 2013 to be already installed on the target machine (perhaps by using the built-in Package resource). It requires the following parameters:

* ControllerName - Name of the controller that is to be configured.
* CollectionUrl - URL of the TFS Team Project collection to which the build controller should be associated.
* Ensure - Either "Present" or "Absent", indicating whether the build controller should be configured or unconfigured.
* ServiceAccount - Credential of the service account under which the build controller should run.
* NumberOfAgents - Number of agents to configure with the controller.

##xTFSBuildAgent##
This resource provides the ability to configure a TFS 2015 Build Agent on a machine. No prior installation of TFS is required, since the agent itself is downloaded from the associated TFS server automatically. You'll probably want to install Visual Studio on the target machine as well though in order to make it a usuable build server. This resource requires the following parameters:

* AgentName - Name of the agent that is to be configured.
* PoolName - Name of the agent pool to associate the agent with.
* ServerUrl - URL to the TFS Server (note: this is not a collection URL, but the URL of the server itself) to associate the agent with.
* AgentFolder - Path to a location on disk where the agent should be installed on the target machine.
* RunAsWindowsService - Indicates whether the agent should be run as a Windows Service
* WindowsServiceCredential - Credential of the service account under which the agent should run.

##Authentication##
Both these resources require access to the network during installation. This means that in order to be able to complete the installation these resources run commands under the credentials of the service account under which the agent or controller are installed. To do so, CredSSP is used, so before these resources can be used on a machine, CredSSP must be configured. This can be easily accomplished by using the xCredSSP PowerShell DSC module. A typical configuration might look like this (showing the xTFSBuildAgent resource in this case, but the same applies to the xTFSBuildController resource):

```powershell
Configuration NewBuildServer
{
    param([string[]]$ComputerName, [PSCredential]$ServiceAccount)

    Import-DscResource -ModuleName xCredSSP
    Import-DscResource -ModuleName xTeamFoundationServer

    Node $ComputerName
    {
        Group Administrators
        {
            GroupName = "Administrators"
            MembersToInclude = $ServiceAccount.UserName
            Credential = $ServiceAccount
        }

        xCredSSP CredSSPServer
        {
            Ensure = "Present"
            Role = "Server"
        }

        xCredSSP CredSSPClient
        {
            Ensure = "Present"
            Role = "Client"
            DelegateComputers = $ComputerName
        }

        xTFSBuildAgent Agent
        {
            AgentName = "$ComputerName-Agent"
            Ensure = "Present"
            PoolName = "default"
            ServerUrl = "http://<server>:8080/tfs"
            AgentFolder = "D:\Agent1"
            RunAsWindowsService = $True
            WindowsServiceCredential = $ServiceAccount
            DependsOn = @("[xCredSSP]CredSSPClient", "[xCredSSP]CredSSPServer")
        }
    }
}
```
