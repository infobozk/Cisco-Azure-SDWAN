# Deploy Cisco SD-WAN in Azure vWAN

## Introduction

Cisco SD-WAN, previously known as Viptela, is a software-defined networking solution in Cisco’s portfolio. Just like with many other SD-WAN solutions, the objective of this solution is to delivery routing policies, threat protection, improved application experience all in a simplified environment. 

Traditionally, setting up and maintaining a secure and reliable WAN was a complex challenge. Organizations wanting to connect (branch) offices and datacenters had to set up an MPLS connection or some other form of dedicated connection. This type of operation does not suit the modern world. With many organizations moving towards the cloud and utilizing SaaS/PaaS services a new challenge is awaiting networking teams.

Let's look at the components:

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/269a61e8cd73323f7f534d00733c6575c055e47f/images/SDWAN_Components.png)

- vManage is the management system providing a centralized environment where all management operations and monitoring can be configured and viewed. Think of this as the Azure portal but instead it is for the networking components.
- The vSmart controller is operating at the control plane, advertising routes, security, and policy information to the vEdge devices. These controllers communicate with the vEdge components.
- vBond is an orchestrator, it authenticates vSmart controllers and vEdge routers. Additionally, the vBond also informs the vEdge routers on how to connect to the vManage and vSmart controllers.
- vEdge devices, these are routers responsible for data plane operations. Can be hardware and software based. Our deployment in the vWAN Hub will be a software device.

This guide will not provide in-depth explanation on the inner workings of Cisco SD-WAN. Please refer to the Cisco documentation if you want to learn more about this topic. 

## Pre-requisites

- A vManage environment, this can be hosted in your own datacenter, or a SaaS solution provided to you. 
- vSmart and vBond controllers.
- Permissions in both the vManage and Azure environment to make changes.
- vWAN Hub deployed.
- Make sure you have your own network design, IP-addressing plan, ASN's, site ID's, Cisco licenses and device templates ready to go.

It is assumed that a functional SD-WAN is already in place. We will be focusing on extending the SD-WAN connection into Azure. This can be done in one of two ways:

- Deployment of vEdge devices into an Azure Virtual Network. 
- Deployment of vEdge devices into an Azure vWAN Hub.

This guideline will cover the vWAN deployment. 

## Design

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/269a61e8cd73323f7f534d00733c6575c055e47f/images/WAN_Architecture.png)

We will be deploying a straightforward design. In this environment we have:

- Secured vWAN Hub in the West-Europe Region.
- Virtual Network Spoke.
- On-premises datacenter already in place with vEdge devices operating.

## Device Template

During the deployment of the vEdge 8000v, we need to provide UUID's as part of the configuration. To make sure we won't be running into this issue, please make sure you follow these steps if you haven't already done so. If templates have already been configured and assigned to the vEdge devices, skip this step.

Open the Cisco Softwre Central Environemt and navigate to the Plug and Play Connect page.
Select 'Add Software Device' as illustrated.

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/123f0609f07f9bd09df067a39742f33d93eacbb4/images/SDWAN_Software%20Central.png)


Proceed to the vManage portal to Sync Smart Account.
![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/b154a88fdab19d8ddbb61171f06395c5d95f859d/images/SDWAN_SmartSync.png)

After the sync is complete, open the Templates page and create a new template:

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/f33793d764c0d1b497129d617c8cb64002e954d6/images/SDWAN_DeviceTemplate.png)

The final step is to assign our template to a device, please make sure you fill in the required information that is applicable to your environment:

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/8fb9d5f7fb0ddeea73f46e568b3b1e5bd3b1eb35/images/SDWAN_AssignDevice.png)

## Connect vManage to Azure
In the vManage portal, browse to the Cloud OnRamp for Multicloud page as shown here:

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/fec7fd2116fdef4c94e61b0d973a337406e5c895/images/SDWAN_cloudOnRamp.png)

Associate your Azure account, make sure the managed identity has the required permissions.
![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/1e68aac42e4eecd864c1b2384d4e57e4b1220b23/images/SDWAN_AssociateAccount.png)

Configure the Cloud Global settings:

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/6921b3e079fc1809f5c2e3ef2c38636bafbfad0c/images/SDWAN_GlobalSettings.png)

Please be aware:

```
In the IP Subnet Pool field, specify the IP subnet pool to be used for the Azure virtual WAN hub. 
A subnet pool needs prefixes between /16 and /24. 
A single /24 subnet pool can support one cloud gateway only. 
You cannot modify the pool if other cloud gateways are already using the pool. Overlapping subnets are not allowed.
The IP subnet pool is meant for all vWAN Hubs inside an Azure Virtual WAN, one /24 prefix per Virtual WAN Hub. 
Ensure that you allocate enough /24 subnets for all the Virtual WAN Hubs you plan to create within the Virtual WAN. 
If a Virtual WAN Hub is already created in Azure, discover it through vManage.
Use the existing subnet pool for the discovered hub.
```


## Deploy Cloud Gateway

Before deploying the NVA into our vWAN Hub, let's check the routing table on our on-premises device.
![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/bb6569b154db5bdfa3133a930f8c5477970e61f7/images/SDWAN_InitialRT.png)

We see no entries to our Virtual Network in Azure: 10.6.1.0/24, confirming that we do not have any connectivity between our Azure environment and on-premises.

Start the wizard to deploy the vEdge into our Azure vWAN Hub.

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/5db7168655e173f4faa1e5129dad0ecdc225e17d/images/SDWAN_NVADeployment.png)

Once the deployment is complete, you should be able to login to the devices through the vManage SSH terminal. This deployment will associate a public IP to your vEdge devices. Make sure to modify your security settings to protect management connections.

At this point there still is no connectivity between our on-premises environment and the spoke virtual network. SD-WAN operates with intent based routing. Through the vManage portal, we can discover virtual networks inside of our Azure environment and using tags group them together.

First, we assign a tag to our virtual network through vManage. Now we can tell our SD-WAN that we want tagged virtual networks to communicate with our on-premises environment. In this demo, this is VPN111 which is the one running on the on-premises devices. SD-WAN connects all locations together with VPN-tunnels. This is once again outside of the scope of this guide, but important to know that this is how we connect our virtual networks to our other locations using SD-WAN.

Enabling this intent will do two things: 

- the vEdge in Azure will start advertizing routes for VPN111
- the vEdge in Azure it will start learning and advertizing routes to VPN111 learned from the vWAN Hub Router

![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/a9c25f4c45525a95cae22ad43534baaaa5667315/images/SDWAN_vNetConnectivity.png)

Check the on-premises routing table entries and we now see a route to our virtual network in Azure.
![](https://github.com/infobozk/Cisco-Azure-SDWAN/blob/0fd58f00481047f652ca006f9368bb8869df6b2d/images/SDWAN_RouteTableOnPrem.png)

## Notes

- We now have connectivity between our Azure Virtual network and on-premises datacenter.
- Looking at the routing table in the 'Effective VM Routes' on the VM NIC, we see that the next-hop is our vEdge in the vWAN Hub.
- It might be preferred to have traffic flowing between Azure and on-premises to pass through the Azure Firewall. 

We can achieve this by setting an UDR on a Routing Table in the vWAN Hub and associating this Routing Table to the virtual network. Alternatively, the same thing can also be achieved by using the routing intent inside the Azure Firewall Policies. Both methods perform the same: overriding the default BGP-injected route. 

Please note that the 'Default Routing Table' should not have any type of custom route. When a route gets introduced into this routing table, the vWAN Hub Router will advertise it to the vEdge appliances. As a result, the vEdge appliance will advertise these routes to the other vEdge devices. Resulting in unwanted routing entries, such as default routes all pointing towards the Azure Firewall. Use custom routing tables instead.
