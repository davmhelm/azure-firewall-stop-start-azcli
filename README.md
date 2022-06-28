# Stop / Start Azure Firewall using Azure CLI

Special thanks to Adam Stuart for his script writeups here: https://github.com/adstuart/azure-firewall-deallocate

# Introduction

Azure Firewall has capabilties to deallocate (stop) and allocate (start) the resource instances in order to save costs. Azure PowerShell provides this functionality natively on instances of the PSAzureFirewall object. The operating model is nice, very PowerShell-y -- Get the firewall, store it as an object, modify the object, pass the modified object back to Azure. Very flexible.

When using Azure CLI, there is no equivalent _currently_. This could change someday. For now, I've written two simple scripts that achieve the same thing, thinking more in Bash/Az CLI models.

My scripts base their approach on source code for PSAzureFirewall object. https://github.com/Azure/azure-powershell/blob/main/src/Network/Network/Models/AzureFirewall/PSAzureFirewall.cs

Hopefully, someday the Azure CLI for firewalls will include native support for an allocate and deallocate command (or start and stop like for VMs). 

My current implementation of Allocate is not as performant as the PowerShell approach because I'm having to make multiple update calls. I could figure out how to format the JSON properties and arrays ahead of time so I'd only make one call to `az network firewall update` in any scenario. But I'd probably be better served learning how to write the commands in Azure CLI natively. :smile:

# Scripts
These scripts assume you're running in a Bash shell. They won't work on Windows CMD, and haven't been tested with any other shells (zsh, ksh, etc.)

## Azure Firewall Stop ([az-firewall-stop.azcli](az-firewall-stop.azcli))

This part is relatively easy. 

1. Given a firewall and its resource group, you check the SKU name. 
1. If the SKU indicates it's part of a VWAN hub, there'll be a Virtual Hub ID. 
    1. Clear the Virtual Hub ID on the firewall resource. Deallocated!
1. If the SKU is anything else, it's part of a VNet. 
    1. Remove the IP configurations on the firewall to deallocate. 
1. But! It might use a separate management subnet (force tunneling), meaning there's a management IP configuration you must remove at the same time.
    1. In this context, there's no harm in telling the Azure Firewall to remove something that doesn't already exist. We want it gone anyway.
    1. So in both cases, just remove all IP configurations and the management IP configuration. Deallocated!

## Azure Firewall Start ([az-firewall-start.azcli](az-firewall-start.azcli))

This requires more checks to validate.

1. Given a firewall and its resource group, you check the SKU name.
1. If the SKU indicates it's part of a VWAN hub, you need to provide a Virtual Hub name and resource group.
    1. Script will find the resource ID of the virtual hub, and
    1. Configure the firewall to use that virtual hub ID. **If the command is successul, your firewall is allocated and running in the virtual hub.**
1. If the SKU is anything else, it's part of a VNet, so you'll need to at least provide:
    1. a Virtual Network name
    > [!NOTE]
    > Currently the Azure CLI commands used do not allow you to specify a Virtual Network by name + resource group or by resource ID. This script **will not work** for you if your firewall is deployed in a resource group that is *different from* the resource group of your VNet.
    1. one or more public IP resource name(s) and their resource group(s) to use
    1. Optionally, you can include a public IP to use for the management interface on a Firewall that was originally deployed in force tunneling mode
    
    > [!TIP]
    > There is no built-in indication whether a firewall was deployed in force tunneling mode or not. You could adopt a practice of using a resource tag and value to indcate this, like "AzFwForceTunnel=true/false" and add in the logic to check that tag's value for additional validation.

1. The script will gather the resource IDs of all provided public IPs, and if a management IP is provided, gather the resource ID of that as well.
1. The script will apply the first IP configuration. For a deallocated firewall, this must include the VNet name. If a management IP has been provided, the script must apply the management IP configuration as part of the same operation as adding the first IP configuration. **If the command is successful, your firewall is allocated and running in the VNet.**
1. If there are additional public IPs to add, the script will add those one by one to the firewall. The firewall remains allocated regardless of whether these subsequent commands are successful or not, though if any commands fail you may wind up with NAT rules that don't work because the public IP wasn't assigned to the firewall properly.
