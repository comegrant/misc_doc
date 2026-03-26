# Databricks Network Security

We currently have network protection on Databricks’ storage account and compute layer. However, the workspace itself is accessible on the open internet. This means that if the URL of one of our workspaces and a personal access token (PAT) were to leak, anyone could connect to our Databricks with the permissions of the leaked PAT.

The goal is to add a layer of network security to Databricks so the Workspace Web UI & API can not be accessed from the open internet even if credentials were to leak. This mean that people will only be able to connect to Databricks on the office wifis or using the company VPN.

I think we have 2 main options to achieve this. The first one is to keep Databricks accessible through public networks, but have an IP whitelist to reject requests coming from unknown & untrusted IPs. The second one is to forbid access through public networks and only connect to databricks using private endpoints. At the moment, I'm leaning towards the first option, which requires less technical setup and still should offer good protection.


## What to whitelist

We need to preserve access for

* Office wifi
* Company VPN
* Container apps deployed by the ML team (+ Dash)
* PowerBI
* Github runners

No action points required for:

* Slack integration: The Slack integration for notifications goes in the direction of Databricks -> Slack. Slack does not need access to Databricks. Dash is deployed in an Azure container. We need to allow the Dash -> Databricks connection, but no action point is required for Slack -> Dash connection.
* Sana: I have not looked into how the Sana integration, but I suspect that it would be a lot of work to keep it running. From team conversation, we agree that it's ok to deprecate the Sana integration, possibly replicating some of those features in Dash.


## Option 1: IP Whitelisting

In this option, we do not forbid request to Databricks from public network. But we add a whitelist of trusted IPs that are allowed to connect to Databricks. Allowing the office wifis and the company VPN is easy since they both have a static IP (AFAIK.) Container apps used by the ML team have dynamic IP ranges. So we will need an extra layer (most likely a NAT Gateway) to give them a static egress IP that we can whitelist.

PowerBI is trickier. Whitelisting PowerBI as a whole is too broad, since it would let anyone access our Databricks through PowerBI. If we want to allow access to only our PowerBI spaces. We likely need an [On-Premise Data Gateway](https://learn.microsoft.com/en-us/power-bi/connect-data/service-gateway-onprem) deployed in a virtual machine in azure. We should then be able to connect PowerBI to Databricks through the on-prem gateway and whitelist the IP of the VM which runs the gateway.

For Github runners, we would need self-hosted runner(s) that we can give a static IP to. It's extra infrastructure to manage, but on the plus side, we get to reduce our Github runner cost (or rather move it to Azure) so Martin will be happy. I've looked into whitelisting the runner's IP at runtime (like we do when we fetch secrets from Azure Keyvault.) Unfortunately, we can't use the Azure CLI there. The IP whitelist is configured in the databricks workspace, which is the very thing we're trying to put a whitelist on. So we've got a chicken-and-egg problem, the runner isn't whitelisted on Databricks so it can't connect to Databricks to whitelist itself.


## Option 2: Private Endpoints

This option involves forbidding request to Databricks from public network and only accessing it through private endpoints. This is the more complicated option since it involves adding private endpoints to the VPN & container virtual networks or use VNet Peering. We would also need to create DNS Zones so the URLs of our Databricks workspaces lead to the private endpoints. Power BI has a [Virtual Network Data Gateway](https://learn.microsoft.com/en-us/data-integration/vnet/overview) which would let us connect Power BI to Databricks over a private network. Unfortunately, this is not currently available in our Power BI Premium-Per-User (PPU) plan and would require purchasing Fabrics capacity or something (idk, PBI's pricing model is very confusing.) But using the on-prem data gateway should also be an option here.

For Github actions, we could [run Github runners in our VNet](https://docs.github.com/en/enterprise-cloud@latest/admin/configuring-settings/configuring-private-networking-for-hosted-compute-products/about-azure-private-networking-for-github-hosted-runners-in-your-enterprise) but Martin says that's expensive so self-hosted runners are likely a better option.


## Concrete Plan

I think we should go for Option 1. Using private endpoints seems to require more infrastructure for a marginal security improvement. Turning on the actual IP whitelist can be done as the last step to minimize disruption.


1. Deploy the on-prem data gateway for Power BI on an Azure VM. (Tackling this one first because I think it's where we're the most likely to run into issues)
2. Connect Power BI to Databricks through the Gateway. Make sure everything works.
3. Deploy an on-prem github runner.
4. Change Github workflows to use the self-hosted runner. Change progressively to catch issues early and monitor load to right-size the runner.
5. Deploy a NAT gateway to give a static IP to the ML container apps & Dash. (Might need Mats' help regarding the ML deployment setup.)
6. Add an IP whitelist to our Databricks workspaces, whitelist:

   
   a. The office wifis
   b. The company VPN
   c. The machine running the PowerBI gateway
   d. The self-hosted Github runner
   e. The NAT Gateway in front of container apps.


## Cost consideration

Turning on IP whitelisting for Databricks require deploying multiple pieces of infrastructure (VMs for self-hosted runner and PBI gateway, NAT Gateway) which will increase our overall cost. I can't provide an estimate at the moment. If cost is an important consideration, we should spend some time making some estimations before moving forward with this. 