# Forced Tunneling of Internet traffic through Active-Active Fortinet Firewalls using Azure Route Server
## Use Case
Recently, many customers have the requirement to “force tunnel” Internet bound traffic from on-premises through Azure, for centralized management and control. This configuration is the opposite of the model where customers force tunnel all Azure-initiated traffic destined to Internet through an on-prem firewall. This is also different from the split tunnel model.  
This article describes how this force tunneling is configured in an Azure Hub-Spoke with a pair of Active-Active Fortinet Firewall Network Virtual Appliances (NVAs), with a Public Load Balancer (PLB) directing North-South traffic, and an Internal Load Balancer (ILB) directing East-West traffic, as in [FortiGate template](https://github.com/fortinetsolutions/Azure-Templates/tree/master/FortiGate/Azure%20Active-Active%20LoadBalancer%20HA-Ports). On-premises is connected to Azure by [ExpressRoute](https://docs.microsoft.com/en-us/azure/expressroute/expressroute-introduction). By configuring the Fortigates to originate default route (0/0), and by introducing [Azure Route Server](https://docs.microsoft.com/en-us/azure/route-server/overview) to reflect this default route, customers will be able to force on-prem Internet bound traffic through the Fortinet firewalls.
 
## Concepts
1.	A default route (0/0) MUST be propagated via BGP to on-prem across ExpressRoute, in order to attract Internet traffic from on-prem through Azure. Customers have unsuccessfully tried to define a static route at on-prem border router pointing to ExpressRoute circuit, but this fails because although the traffic may enter the customer edge ExpressRoute circuit interface, it will be dropped upstream at the MSEE (Microsoft Edge Routers) which has no awareness of 0/0.
2.	Both Fortigates (Active-Active) will originate the default route by redistributing the static route to 0/0 (part of standard PLB Fortigate configuration) into BGP.
3.	Azure Route Server will learn this default route sourced by Fortigate through eBGP.
4.	ExpressRoute Gateway will learn this default route through iBGP peering with the Route Server. ExpressRoute Gateway will see Next Hop as the Fortigates’ peer IP, not Route Server
5.	ExpressRoute Gateway will propagate the default route to on-prem across ExpressRoute Private Peering, via MSEE.
6.	On-prem thus learns of default route from Azure, and will route Internet traffic to Azure and the Fortigate NVAs.
7.	As in the standard Active-Active Fortigate design, Protected VNETs and Spoke subnets will have UDR pointing to ILB, for either East-West or North-South traffic. The use of the Load Balancer does NOT change for this Active-Active Fortigate configuration. Furthermore, User-Defined Routes are still required at GatewaySubnet pointing to ILB to ensure sticky, symmetrical flow path for East-West traffic.

## Configuration
This tested configuration looks as follows:
 
•	Resource Group: “RG-Fortigate”
•	VNET: “FW-FG1-VNET”, address space 172.16.136.0/22
•	Route Server: “RS-FW-FG1-VNET”
•	RouteServerSubnet BGP speakers: 172.16.139.4 and 172.16.139.5 
•	Internal Load Balancer: 172.16.136.68
•	External Load Balancer IP: 40.63.93.48
•	Firewalls Internal IPs: FortigateA 172.16.136.69, FortigateB IP 172.16.136.70
•	MSEE VXLAN IPs:  172.16.138.4 and 172.16.138.5
•	ERGW IPs: 172.16.138.12 and 172.16.138.13

### Preparation of environment:
1.	Deployment of a Fortigate firewall sandwich, from Fortinet Github solutions page:  fortinet-azure-solutions/FortiGate/Active-Active-ELB-ILB at main · 40net-cloud/fortinet-azure-solutions (github.com). (Other options including building off Azure Marketplace deployment template works fine too)  This step is unnecessary for brownfield deployments.
2.	Deployment of GatewaySubnet and ExpressRoute Gateway with Connection to provisioned ExpressRoute Circuit and Private Peering to OnPrem.  This step is unnecessary for brownfield deployments.
3.	Create a RouteServerSubnet and deploy a Route Sever in the subnet [Quickstart: Create and configure Route Server using Azure CLI | Microsoft Docs], in this example:
•	az network vnet subnet create -g "RG-Fortigate" --vnet-name "FW-FG1-VNET" --name "RouteServerSubnet" --address-prefix "172.16.139.0/27"
•	$subnet_id = $(az network vnet subnet show -n "RouteServerSubnet" --vnet-name "FW-FG1-VNET" -g "RG-Fortigate" --query id -o tsv)
•	az network routeserver create -n "RS-FW-FG1-VNET" -g "RG-Fortigate" --hosted-subnet $subnet_id

### Detailed configuration on Fortigate and on RouteServer
#### Fortigates
1.	Enable BGP and establish Peering to Route Server, from each Fortigates. 
a.	Route Server has 2 BGP speaker addresses, so each Fortigate will be configured for 2 BGP peers
b.	EBGP Multhop will be required as this is not a point-to-point BGP peer  
c.	Fortigate can use any ASN that is not reserved by Azure (65515 – 65520).  Azure Route Server will always use ASN 65515. 
This example shows onfiguration on FortigateA. FortigateA’s local ASN is assigned as 65008, and it peers to the two BGP speaker addresses of Route Server. FortigateB configuration looks identical
 
2.	Propagate default 0/0 Route in BGP on each Fortigate
a.	Redistribute the static route to default 0/0, which is already defined as a Static Route on the Fortigates (standard template configuration), into BGP. 
b.	Associate a route-map to limit the static redistribution to only 0/0
 

#### Route Server
1.	Configure Peering to both Fortigates. 
a.	Add peer to each Fortigate as a Peer, identifying each by a Name.  In this example, the Names are “FortigateA” and “FortigateB”, and the remote ASN number is 65008.
b.	The Peer address is the Internal IP (Port 2) of the Fortigates, which will force Internet bound traffic to hit the Internal interface, be processed by firewall rules, before exiting to the External interface.
 
2.	Enable “Branch to Branch” flag, which underneath the covers establishes iBGP peering between Route Server and ExpressRoute Gateway

 
### User Defined Routes 
Although BGP is introduced in the picture, UDRs are still important consideration:  
1.	Fortigate’s External Subnet: The firewall’s external subnet needs to have direct route to Internet. Any default 0/0 route learned via Route Server will need to be overridden by UDR, otherwise there will be a loop. Below example of Effective Routes on Fortigate’s External NIC (Port 1) shows how has learned 0/0 from Route Server. (This is overridden with a UDR to 0/0 with Next Hop of Internet.  This concept should become clear later in the article with additional screen captures.)
 
2.	GatewaySubnet:  To ensure flow symmetry of East-West traffic, UDRs to Protected Subnets and Protected Peered VNETs will need UDRs with Next hop IP address of the ILB
 
## Verification
Verification steps include validating Peering, Routing and Next-Hop, and finally Forwarding at each component. 
### Validate BGP Peering  
1.	At both Fortigates, validate BGP peering is established to the two Route Server instances.
 
 
2.	At RouteServer, validate BGP peering with the two Fortigates has “Succeeded”
 
3.	At ExpressRouteGateway, validate iBGP peering has formed with the two instances of RouteServer.
 
### Validate Routing Tables  
Once BGP peering among ExpressRoute Gateway, RouteServer, and Fortigates have been validated as above, routes should be properly exchanged among on-prem and Azure.
1.	Validating at the Fortigates: A healthy configuration shows Fortigate originates the default 0/0 in the BGP table, highlighted in red below. It also has learned the on-prem prefixes 2.2.2.2/32 and 192.168.1.0/24, highlighted in blue.
 
2.	Validating at the RouteServer: Shown below, both instances of RouteServer (IN_0) and (IN_1) are learning default 0/0 from Fortigate A, 172.16.136.69. They are also learning the default from FortigateB, 172.16.136.70. 
 


 
3.	Validating at the ExpressRoute Gateway: ExpressRoute Gateway is learning many routes from on-prem (highlighted in blue) and the Fortigate (highlighted in red). Notice the default 0/0 learned from peer Route Server (172.16.139.4 and 172.16.139.5) are set with nextHop of the Fortigate internal IP of 172.16.136.69, and not the RouteServer. Azure Route Server is NEVER in the data path but works at the BGP control plane level.
 
 
ExpressRoute advertises the 0/0 route it has learned downstream to the MSEE peers of 172.16.138.4 and 172.16.138.5.
  
   
4.	Validating at the MSEE: MSEE learns the default 0/0 route from the ExpressRoute Gateway (red), and learns the on-prem prefixes from ExpressRoute Private Peering (blue)
 
5.	Validate routing at on-prem. Default route 0/0 is learned from ExpressRoute Private Peering 
	
### Validate Forwarding Path	
The routes have been validated in previous steps, and now the final validation is with for data path between source and destination. This is to ensure traffic is flowing symmetrically through the firewalls, and properly out to Internet.
1.	Validate E-W traffic: 
Traceroute of Protected Hub VM to on-prem shows traffic goes through the Firewall (via ILB). 
 
	Traceroute of spoke VM to on-prem shows traffic goes through the Firewall (via ILB).
 
2.	Validate North-South traffic: 
Curl ifconfig.io shows Protected Hub VM has reachability to outside world/Internet. It uses the Fortigate’s Public Load Balancer Public IP, 40.64.93.48
 
Curl ifconfig.io shows Spoke VM has reachability to outside world/Internet. It uses the Fortigate’s Public Load Balancer Public IP, 40.64.93.48
 
3.	FINALLY! Validation on-prem goes out to Internet via Azure, via Fortigate firewalls.
The on-prem CPE router is able to SSH to a TestVM (52.183.63.77) as shown below. The on-prem host is using Fortigate’s Public Load Balancer’s PIP, 40.64.93.48 (red below). This proves the 
 



 
Traceroute shows Internet traffic sourced from on-prem follows the path to Fortigate firewalls (172.16.136.69 and 172.16.136.70), though the traceroute process is blocked and does not complete to the Internet destination.
 

## Summary
This article shows how customers can extend the Fortinet “firewall sandwich” configuration to force on-prem sourced Internet bound traffic through the Azure. This is accomplished by adding Azure Route Server to the Hub VNET and configuring the Fortigates to originate a default route 0/0 which is ultimately learned by on-prem. This configuration applies to not only on-prem, but also to ExpressRoute connected AVS [] environments. This configuration is also extensible to Site-to-Site VPNs.
