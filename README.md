# OpenShift Container Platform 3.5 with Username / Password authentication for OpenShift

This template deploys OpenShift Container Platform with basic username / password for authentication to OpenShift. It includes the following resources:

|Resource           	|Properties                                                                                                                          |
|-----------------------|------------------------------------------------------------------------------------------------------------------------------------|
|Virtual Network   		|**Address prefix:** 192.168.0.0/16<br />**Master subnet:** 192.168.1.0/24<br />**Node subnet:** 192.168.2.0/24                      |
|Master Load Balancer	|2 probes and 2 rules for TCP 8443 and TCP 9090 <br/> NAT rules for SSH on Ports 2200-220X                                           |
|Infra Load Balancer	|3 probes and 3 rules for TCP 80, TCP 443 and TCP 9090 									                                             |
|Public IP Addresses	|Bastion Public IP for Bastion Node<br />OpenShift Master public IP attached Master Load Balancer<br />OpenShift Router public IP attached to Infra Load Balancer            |
|Storage Accounts   	|2 Storage Accounts                                                                                                                  |
|Virtual Machines   	|1 Bastion Node - Used both to Run Ansible Playbook for OpenShift deployment and to do internal load balancing to the masters<br />1 or 3 Masters. Master 1 is used to run a NFS server to provide persistent storage.<br />1 or 3 Infra nodes<br />User-defined number of nodes<br />All VMs include a single attached data disk for Docker thin pool logical volume|
## READ the instructions in its entirety before deploying!

This template deploys multiple VMs and requires some pre-work before you can successfully deploy the OpenShift Cluster.  If you don't get the pre-work done correctly, you will most likely fail to deploy the cluster using this template.  Please read the instructions completely before you proceed. 

This template uses the On-Demand Red Hat Enterprise Linux image from the Azure Gallery.  This means there is an hourly charge for using this image.  At the same time, the instance will be registered to your Red Hat subscription so you will also be using one of your entitlements.  For this reason, this template is good for setting up temporary POCs or learning environments but not meant for production due to the "double billing".

## Prerequisites

### Generate SSH Keys

You'll need to generate an SSH key pair (Public / Private) in order to provision this template. Ensure that you do NOT include a passcode with the private key. <br/><br/>
If you are using a Windows computer, you can download puttygen.exe.  You will need to export to OpenSSH (from Conversions menu) to get a valid Private Key for use in the Template.<br/><br/>
From a Linux or Mac, you can just use the ssh-keygen command.

### Create Key Vault to store SSH Private Key

You will need to create a Key Vault to store your SSH Private Key that will then be used as part of the deployment.  I recommend creating a Resource Group specifically to store the KeyVault.  This way, you can reuse the KeyVault for other deployments and you won't have to create this every time you chose to deploy another OpenShift cluster.

1. Create KeyVault using Powershell <br/>
  a.  Create new resource group: New-AzureRMResourceGroup -Name 'ResourceGroupName' -Location 'West US'<br/>
  b.  Create key vault: New-AzureRmKeyVault -VaultName 'KeyVaultName' -ResourceGroup 'ResourceGroupName' -Location 'West US'<br/>
  c.  Create variable with sshPrivateKey: $securesecret = ConvertTo-SecureString -String '[copy ssh Private Key here - including line feeds]' -AsPlainText -Force<br/>
  d.  Create Secret: Set-AzureKeyVaultSecret -Name 'SecretName' -SecretValue $securesecret -VaultName 'KeyVaultName'<br/>
  e.  Enable for Template Deployment: Set-AzureRMKeyVaultAccessPolicy -VaultName 'KeyVaultName' -ResourceGroupName 'ResourceGroupName' -EnabledForTemplateDeployment<br/>

2. Create Key Vault using Azure CLI<br/>
  a.  Create new Resource Group: azure group create \<name\> \<location\> <br/>
         Ex: [azure group create ResourceGroupName 'East US'] <br/>
  b.  Create Key Vault: azure keyvault create -u \<vault-name\> -g \<resource-group\> -l \<location\><br/>
         Ex: [azure keyvault create -u KeyVaultName -g ResourceGroupName -l 'East US'] <br/>
  c.  Create Secret: azure keyvault secret set -u \<vault-name\> -s \<secret-name\> --file \<private-key-file-name\><br/>
         Ex: [azure keyvault secret set -u KeyVaultName -s SecretName --file ~/.ssh/id_rsa <br/>
  d.  Enable the Keyvvault for Template Deployment: azure keyvault set-policy -u \<vault-name\> --enabled-for-template-deployment true <br/>
         Ex: [azure keyvault set-policy -u KeyVaultName --enabled-for-template-deployment true] <br/>

### Red Hat Subscription Access

If you don't already have a user account to access your company's Red Hat user portal, please contact your administrator.  You will need to ensure your Red Hat subscription credentials are in working order by logging into https://access.redhat.com.<br/>

You will also need to get the Pool ID that contains your entitlements for OpenShift.  You can retrieve this from the Red Hat portal by examining the details of the subscription that has the OpenShift entitlements.  Or you can contact your Red Hat administrator to help you.

### azuredeploy.Parameters.json File Explained

1.  _artifactsLocation: URL for artifacts (json, scripts, etc.)
2.  masterVmSize: Select from one of the allowed VM sizes listed in the azuredeploy.json file
3.  nodeVmSize: Select from one of the allowed VM sizes listed in the azuredeploy.json file
4.  openshiftClusterPrefix: Cluster Prefix used to configure hostnames for all nodes - bastion, master, infra and nodes (between 1 and 5 characters)
5.  openshiftMasterPublicIpDnsLabel: A unique Public DNS host name (not FQDN) to reference the Master Node by
6.  infraLbPublicIpDnsLabel: A unique Public DNS host name (not FQDN) to reference the Node Load Balancer by.  Used to access deployed applications
7.  masterInstanceCount: Number of Masters and Infra nodes to deploy
8.  nodeInstanceCount: Number of Nodes to deploy
9.  dataDiskSize: Size of data disk to attach to nodes for Docker volume - valid sizes are 128 GB, 512 GB and 1023 GB
10. adminUsername: Admin username for both OS (VM) login and initial OpenShift user
11. openshiftPassword: Password for OpenShift user
12. cloudAccessUsername: Your Red Hat Cloud Access subscription user name
13. cloudAccessPassword: The password for your Red Hat Cloud Access subscription
14. cloudAccessPoolId: The Pool ID that contains your OpenShift entitlements
15. sshPublicKey: Copy your SSH Public Key here
16. keyVaultResourceGroup: The name of the Resource Group that contains the Key Vault
17. keyVaultName: The name of the Key Vault you created
18. keyVaultSecret: The Secret Name you used when creating the Secret (that contains the Private Key)
19. defaultSubDomainType: This will either be xipio (if you don't have your own domain) or custom if you have your own domain that you would like to use for routing
20. defaultSubDomain: The wildcard DNS name you would like to use for routing if you selected custom above.  If you selected xipio above, you must still enter something here but it will not be used

## Deploy Template

Deploy to Azure using Azure Portal: 
<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fmglantz%2Focp-azure-simple%2Fmaster%2Fazuredeploy.json" target="_blank"><img src="http://azuredeploy.net/deploybutton.png"/></a>
<a href="http://armviz.io/#/?load=https%3A%2F%2Fraw.githubusercontent.com%2Fharoldwongms%2Focp-azure-simple%2Fmaster%2Fazuredeploy.json" target="_blank">
    <img src="http://armviz.io/visualizebutton.png"/>
</a><br/>

Once you have collected all of the prerequisites for the template, you can deploy the template by clicking Deploy to Azure or populating the *azuredeploy.parameters.json* file and executing Resource Manager deployment commands with PowerShell or the Azure CLI.

### NOTE

The OpenShift Ansible playbook does take a while to run when using VMs backed by Standard Storage. VMs backed by Premium Storage are faster. If you want Premium Storage, select a DS or GS series VM.
<hr />
Be sure to follow the OpenShift instructions to create the necessary DNS entry for the OpenShift Router for access to applications.

### TROUBLESHOOTING

If you encounter an error during deployment of the cluster, please view the deployment status.  The following Error Codes will help to narrow things down.

1. Exit Code 3: Your Red Hat Subscription User Name and / or Password is incorrect
2. Exit Code 4: Your Red Hat Pool ID is incorrect or there are no entitlements available
3. Exit Code 5: Unable to provision Docker Thin Pool Volume

For further troubleshooting, please SSH into your Bastion node on port 22.  You will need to be root (sudo su -) and then navigate to the following directory: /var/lib/waagent/custom-script/download<br/><br/>
You should see a folder named '0' and '1'.  In each of these folders, you will see two files, stderr and stdout.  You can look through these files to determine where the failure occurred.

## Post-Deployment Operations

### Metrics and logging

To display metrics and logs, you need to logon to OpenShift ( https://publicDNSname:8443 ) go into the logging project, click on the Kubana route and accept the SSL exception in your brower, then do the same with the Hawkster metrics route in the openshift-infra project.

### Creation of additional users

To create additional (non-admin) users in your environment, login to your master server(s) via SSH and run:
<br><i>htpasswd /etc/origin/master/htpasswd mynewuser</i>

### Access to Cockpit

Use user 'root' and the same password as you assigned to your OpenShift admin to login to Cockpit ( https://publicDNSname:9090 ).
   
### Additional OpenShift Configuration Options
 
You can configure additional settings per the official (<a href="https://docs.openshift.com/container-platform/3.5/welcome/index.html" target="_blank">OpenShift Enterprise Documentation</a>).
