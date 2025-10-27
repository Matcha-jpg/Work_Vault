This document contains a high level overview of the Lab Environment and the systems that it consists of, as well as detailed, step-by-step instruction set to replicate the COSMIAC Lab environment, granted with a few exceptions like Public IP allocations. Following the instruction set should net you a near-exact replica of the COSMIAC Lab environment.
# High-Level Overview

## Palo Alto Firewall

Since the lab environment will be utilizing commercial internet services, a firewall is necessary to maintain the security and integrity of the COSMIAC Lab Environment. The Palo Alto Firewall utilized in the Lab Environment is extremely customizable. This allows for very granular and verbose configuration, but this also leads to a more complex process for a proper, secure firewall configuration. The Graphical User Interface (GUI) of the Palo Alto Firewall is somewhat intuitive, but is more user-friendly to those who are proficient in securing network infrastructures.

As is stated above, there is a variety of customizable settings within the Palo Alto firewall. The most important customizations that need to be made for the lab environment is within the Network Address Translation (NAT) and Security Policies, and in addition, Services. Services are just the types of ports and protocols you are allowing through NAT Policies, while the Security Policies handle how traffic moves within the intra-zone and inter-zone interfaces.

There are more items that you can configure for more granularity, but those settings are not of concern in our Lab deployment. In addition, this firewall can allow customers and Subject Matter Experts (SMEs) alike the ability to interact with the environment remotely for Lab demonstrations/assistance with infrastructure setup if necessary.

## ESXi Hypervisor

ESXi is a Bare-metal Hypervisor from VMWare that overhauls a server's OS (in our case, a Dell Server), which allows for the efficient provisioning of server hardware and resources to host and deploy Virtual Machines (VMs) on. The ESXi hypervisor is what will be utilized to deploy all associated virtual devices necessary for the COSMIAC lab environment.

The GUI of ESXi is relatively intuitive and user friendly, but it having background knowledge in hosting virtual environments helps with navigation and deployment immensely. There are a lot of tools at the operator's disposal, with the ability to configure many aspects of the deployment. <div style="page-break-after: always"></div>

There are configurations for Virtual Machines, Storage, and Networking. Virtual Machines is where you will be deploying and configuring your devices. Storage is where you store any .iso, .ova files that your VMs will use, as well as hosting/representing any deployed VM and their contents. Networking is where you will configure your virtual Network infrastructure, creating vSwitches, port groups, and the such for your VMs to connect to via a Network Adapter on the VM.

The area of configuration we are concerned with is primarily the Network setup. It is of utmost importance to configure the network settings correctly in ESXi, as the end goal is to replicate a Commercial/Business deployment of an SD WAN Network infrastructure. With our Dell Server, we have a limited amount of physical Network Interface Cards (NICs) with which to facilitate internet connections, so proper allocation is necessary. Each vSwitch can be attached to multiple physical NICs, which are denoted as uplinks/vmnics in ESXi, and this allows for redundancy in normal deployments. However, the vSwitches in our setup will only have one NIC/uplink assigned to it.

The system will be utilizing 1 NIC for connection to the Palo Alto Firewall, 1 NIC for the StarLink Modem connection, and 1 NIC for the EC2 Connection, leaving 3 NICs unused. There are plans to enhance the lab environment with Satellite Communication Internet Service Providers (SATCOM ISPs) other than SpaceX StarLink, such as ViaSat and Hughes-Net, but these services will not be utilized in our primary tests. They will however serve to dissect and learn more information about the functionality and limitations of the SD WAN Network infrastructure, which will be noted at a later date in a revision/different document.

## Control Plane

The Cisco Control Plane consists of 3 devices: 
* vSmart (also known as the Controller)
* vManage (also known as the Manager)
* vBond (also known as the Validator). 
You will need to generate and install certificates for these devices, and may need specific personnel to do so. The important controller to get up and running is vBond, as it acts as the initial gateway onto your SD-WAN fabric. It validates and confirms that the devices establishing a connection to it before letting said device join into the fabric. 

In this setup, vBond requires a publicly routable ip address. This means that it can be behind a NAT (Network Address Translation) Device, but it has to be configured to have a static, 1:1 NAT with the public ip of the NAT device (in our case, the PaloAlto firewall). 

The rest of the control plane also resides behind this NAT device, and that internal communication has to be handled via NAT Hairpinning/Loopback. This allows the other control plane devices to have the correct routes mapped to the device for complete control plane connectivity. 

The **complete control plane connectivity** is important, as vSmart handles advertising the correct network communication policies between distant edges, and vManage allows for the viewing and analysis of associated systems, as well as being the central point of management within the fabric.

These three devices are all deployed within the Dell Server running as our ESXi Bare-metal Hypervisor.

## Edge Devices (Data Plane)

The Edge devices will comprise your Data Plane (Where your Data Communications happen). Your edge devices are your Edge Routers/Distant Edge Devices within your network infrastructure. The COSMIAC SD-WAN StarLink setup consists of two Edge Routers (Cisco Catalyst 8000v cEdges). Like the Control Plane devices, these Edge routers will also be deployed within our ESXi Hypervisor. 

The initial setup for both edges is quite simple, though to configure the proper interface for StarLink, there are additional configurations you must make either at the CLI or Feature Template level of configuration. The Feature Template configuration is relatively straightforward, and there is a Cisco Community Blogpost that exists to delineate the feature template creation, but the same actions can be done through the CLI.

The edges also need to be programmed with the public ip of vBond, and the associated udp port 12346 for DTLS (Datagram Transport Layer Security) protocol. This allows them to begin the system onboarding process to get the edge to join onto the SD-WAN Fabric. The first step is the initial bootstrap CLI configuration. This process involves accessing vManage's GUI through http, and generating a bootstrap configuration for a WAN edge there. This step will be outlined later in this documentation

The two edges will require at minimum 2 Network Adapters configured on the ESXi host. Network Adapter 1/GigabitEthernet1 (As denoted on a cEdge/vEdge) will be our WAN (Wide Area Network) (public facing/untrust zone) interface, and Network Adapter 2/GigabitEthernet2 will be our LAN (Local Area Network) (internal/trusted zone) interface.  The LAN interface will double as a DHCP server, allowing clients connected to the same port group to be leased a private IP address within the associated LAN subnet.<div style="page-break-after: always"></div>

After the edges are configured and have complete control plane connectivity, tunnels between edges will be created, and connectivity between sites should be established. The *Software Defined* portion of SD WAN handles the tunnel creation through the advertisement of routing and policies from the vSmart (Controller) to the Distant Edge Devices.

## Quirks With Our Lab Setup

There were multiple problems with the initial setup of this lab environment, so here are some key things to point out so that future deployments do not get stuck in the same place.

* ### Using EC2 as a NAT Gateway/Router
	This Lab environment utilizes an EC2 Instance not as a device that directs traffic through its interfaces from one place to another, but as another router (NOT EDGE ROUTER) that acts as our NAT device that our Edge router is connected to. Edge routers being behind NAT devices in commercial deployments is not uncommon, and a valid way to showcase layered Network infrastructure compatibility with SD WAN deployments, but there are important NAT caveats to pay attention to and avoid as they will inherently fail to work in an SD WAN environment. That list is delineated in this image below. ![[NAT-Compatibility.png]]

* ### Edge Router Behind NAT Device
	This is an important issue to outline as this deals with what IPs get advertised in the Edge router's TLOCs (Transport Locators). If directly connecting to the Control Plane on a LAN interface, the Edge Router will advertise the other interfaces it has on the Transport VPN using the configured IP on the device.<div style="page-break-after: always"></div>
	
	This means that, if the router is behind a NAT Device, then the Edge Router will advertise its ***PRIVATE*** IP to the SD WAN Fabric. This is an issue, as other routers will try to establish a tunnel to this ***PRIVATE*** IP through the ***PUBLIC*** internet. This is inherently impossible in normal networking. The way that this is subverted is by connecting to vBond through the public internet, as vBond has an embedded STUN (Simple Traversal Utilites for NAT) Server, and can thus help Edge routers/Control Plane devices behind a NAT Device become *NAT Aware*. This is the case for our lab setup, as our Sink Edge Router is connected behind the EC2 Instance that acts as this NAT Device.

* ### NOT An Air-Gapped Environment
	Initially, this lab was based off of an air-gapped system from a prior preliminary test on SD WAN infrastructure. In addition, there was confusion as to how the EC2 Instance fits into the picture in our infrastructure. Once again, the EC2 acts as a NAT Device, and while Network Emulation can likely be appended to the operational features of the EC2, it is not an inherent part of our environment. That is because we are NOT simulating network traffic, as we are dealing with realistic network traffic metrics as we are utilizing Commercial Internet Service Providers for our internet connectivity.
	
	The ISPs in question are SpaceX StarLink, and AWS, as the EC2 doubles as a gateway to the public internet through the AWS Fabric. The Source Edge is connected to our StarLink Modem, while the Sink Edge is connected to the EC2 Nat Device.
	
	The Non-Air-Gapped environment nuance is also important because, as stated earlier, we are utilizing Commercial Internet Services to construct our SD WAN fabric, leading us to a deployment that is much closer to a Commercial/Business deployment of Cisco SD WAN infrastructure. This means that the control plane has to be publicly routable to the internet. There are security risks associated with this, and to address those security risks, the Palo Alto Firewall is our answer, only allowing traffic from associated IPs and Ports to access the control plane.

* ### Control Plane all behind ONE NAT Device with only ONE Public IP
	This is a particular quick in our setup that caused much confusion and frustration, as since we only had a **singular public IP** to work with, that made communication to the control plane much more complex than that of a normal commercial/business deployment. Normal, Cisco suggested deployment methods mark the simplest deployment as giving each Control Plane device a public IP, but that can be expensive depending on the budget, or complications with Company-Owned IP allocations. There are deployments around this, although less suggested due to the complexity they require.<div style="page-break-after: always"></div>
	
	One such issue that arises is if they are all ALSO behind the SAME NAT Device, such as in our lab environment. This necessitates the use of what is called NAT Hairpinning/Loopback, which in simpler terms is just a way for devices/clients of the NAT Device to utilize the NAT Device's public IP to communicate with others in the same/different LAN. This is the method that we utilized in our setup. Other solutions include setting up a DMZ (De-Materialized Zone) For vSmart and vManage or just vBond, but this has yet to be explored as the previous NAT Hairpinning/Loopback solution worked to solve the problem in our case.
	
	NAT Hairpinning/Loopback is considered complex, as you need to have a good understanding of how firewalls and routing works to accomplish configuring the correct security rules and NAT policies to put in place, let alone knowing if your particular router/firewall is capable of this functionality in the first place.

* ### DTLS And Interesting Firewall Issues
	There was a problem in establishing complete control plane connectivity on the edge router side due to some granular firewall issues. In no place did we configure on the firewall to drop any specific packets from our trusted sources, yet any DTLS Packets that were attempting to connect to the vSmart or vManage devices behind the Palo Alto Firewall were dropped. 
	
	Upon granular analysis of network traffic through the usage of a router with the capability of running `tcpdump`, we were able to identify that DTLSv1.0 packets originating from the edge devices were dropped, but DTLSv1.2 packets made it through the firewall to the vManage and vSmart devices. With further inspection, we speculate that the firewall was mis-classifying our Client Hello DTLS packets (The type of packet that is utilized to establish a connection via handshake) as using a different protocol, and thus not properly identified and routed to the correct destination.
	
	The solution for this was to force vSmart and vManage to use TLS for control plane connections over DTLS, on two distinct ports. This allowed for the devices to still understand their public-private IP and port mappings behind a NAT Device, and establish connections to the distant edge devices.
	
	Again, this is purely speculation on our part, as there could be a potential bug with the version of the SD WAN Software that the lab environment uses, and thus the issue could be a SD WAN Software issue rather than a Firewall behavior issue. We are focusing efforts forward with this solution in mind, as with the current equipment, this is our path of least resistance.<div style="page-break-after: always"></div>

## Most Important Lesson Learned and Next Steps

With all this said, the most important point here is that the Control Plane **_NEEDS_** to be publicly routable for this setup to work, especially if one or more edge routers/control plane devices are behind a NAT Device.

## Moving Forward and Future Impacts

The COSMIAC team has purchased a new Palo Alto Firewall including a core security subscription bundle. The current system does not have an active license, which may or may not be attributed to our plane control issues, nor do we have a clientless VPN access available to us. This new hardware and license would provide a more robust system security like advanced thread prevention, advance URL filtering, or even external access to our environment. Our future SD-WAN deployments should be streamlined through the application of our lessons learned, technical insights gained from the current experimentation, and proper documentation of the accumulated knowledge.

## How does this tie directly to the SDN as a whole?

The current commercial SD-WAN deployment serves as a representative model for the Hybrid Space Network architecture that Joint Forces may utilize for resilient data transport and communication. The current architecture utilizes a multiple hop approach from representative Trusted Network Device (Cisco virtual router) to another. Edge users reside between a commercial SATCOM vendor, a hybrid cloud environment providing proper tunnel routing functionality for the Satellite-to-terrestrial connectivity bridge, and a fiber connection from an on-premises AWS Outpost server. This initial phase of testing provides a path towards a variety of use cases for edge users utilizing a hybrid network.

Studying the components and functionality of the commercial SD-WAN at hand provides a foundation for developing a hybrid Space Data Network by leveraging mature, field-tested technologies, that offer a significant head start to the network orchestration capabilities that are required to serve interconnected DoD agencies. The RAPID program can adapt proven SD-WAN frameworks that already incorporate essential features needed for such a network such as dynamic path selection, automated failover, a centralized management, to even cloud deployment for information or components residing a high impact level.

The current experimentation the COSMIAC team is working on aims to provide foundational knowledge necessary to accelerate a Hybrid SDN development by establishing performance baselines, operational procedures, and current methodologies for commercial SD-WAN deployment over a hybrid infrastructure. Validating the technical practicality of this technology over space and ground-based transport provides necessary information for defense agencies to confidently invest in.<div style="page-break-after: always"></div>


# Step-by-Step Instructions

These instructions will only outline the software configurations of the setup. This section of this document assumes that all hardware has been properly set up, so no steps delineating anything hardware-related will be discussed.

This instruction set will also be separated into sections relating to the specific component(s) the set of instructions will be concerned with. The three categories are as follows: ESXi, Palo Alto Firewall, Control Plane Devices, and Data Plane Devices, with the Control and Data Plane Devices under the umbrella of "VM Deployment".

Although the set of instructions are separated by component, following this instruction set top down should ensure full system functionality, as long as the differences in lab setups are handled accordingly.

Certain names within this setup documentation may differ from existing names and naming schemes in existing COSMIAC-RAPID Lab Environment infrastructure. This may be due to a premature naming scheme, a lack thereof, or missing/lacking access to related systems and/or lacking access to communication with associated personnel that have access to the system.

Things to have at the ready before you start:
1. Your StarLink Public IP
   For the COSMIAC-RAPID Lab Environment, this was `143.105.158.123`
   
2. Your EC2's Elastic IP (Which is its public IP)
   For the COSMIAC-RAPID Lab Environment, this was `13.218.134.192`
   
3. The Public IP of your Palo Alto Firewall
   For the COSMIAC-RAPID Lab Environment, this was `138.199.102.114`

***PLEASE READ THE ENTIRE STEP BEFORE ENTERING A COMMAND***<div style="page-break-after: always"></div>

## AWS Setup

This will be a brief AWS setup, which will consist of configuring a security group to allow SD-WAN related traffic, and configuring an EC2 instance with that security group to act as the public facing WAN interface for the Cisco Edge router in the COSMIAC-RAPID Lab Environment. If you are familiar with AWS and how to set up and launch an EC2 instance, skip to the `EC2 Linux Box Configuration` section, where we will discuss the `iptables` commands utilized in ensuring proper network communication happens through the EC2 instance.

This setup will utilize the AWS Web Console rather than the AWS command line interface, as the Console GUI is more user-friendly for a general audience, and thus will expedite Lab Environment setup. 

### VPC and Security Groups

Security groups are essentially your virtual firewall rules. Security Groups in AWS allow you to configure in your network the allowed ports/types of traffic, and whether that traffic is allowed in either the inbound or outbound direction. This ensures that your network is as open or as secure as you require it to be in your implementation/Lab Environment. 

To reach your security group configuration, you will have to first navigate to the Virtual Private Cloud (VPC) service in AWS. VPCs are essentially your own private, virtualized network infrastructure in the cloud, where you can apply/create multiple security groups associated with the VPC, and assign them as needed to the services you create/utilize within the AWS cloud infrastructure. This is also where you would configure the other aspects of your Cloud network, such as Network Access Control Lists (NACLs), Network Load Balancers, and a range of other essential network configuration tools that help make your cloud environment as secure as you require.

Security Groups are not to be confused with Network Firewalls among the VPC Services. Security groups allow for a per-instance firewall, meaning it can apply to singular instances, or a smaller group of instances. Network Firewalls however have a much broader scope, and encompasses the entire VPC it is assigned to, rather than individual/group instances within the VPC. Network Firewalls also allow for a much more granular inspection of Network Traffic, since it operates/encompasses the VPC level of functionality.

This section will first create a VPC where you will then set up your security group in, primarily for good networking practice and network segmentation. You *can* use the default VPC that is created within your account, but it is advised you construct a specific VPC to handle the SD WAN pieces that you have contained within the AWS Cloud space.

#### VPC Setup

1. Navigate to the VPC Console Menu by utilizing the search bar on the AWS Web Console GUI. ![[Search_VPC.png]]
   
2. Once in the VPC Console landing page, click on the `Create VPC` button in the middle. This will bring you to the VPC Creation Menu. ![[Create_VPC.png]]

3. For the `Resources to create`, choose VPC Only.
4. For the `Name tag`, this field is optional, but recommended to name, so you have an identifier for your VPC. Follow your Lab Environment's naming convention, but you may use the example below for our simplicity.
5. For the IPv4 CIDR (Classless Inter-Domain Routing) Block, choose the IP range that you require, though you may use the example below for simplicity, with the only exception being that the range is already in use by another VPC in your account. The `/(Number)` parameter portion of the CIDR Block essentially denotes the amount of IPs you can allocate/have within that CIDR block. In a `/24` configuration, we have 256 total IP addresses, and 254 total ***addressable*** IP addresses.<div style="page-break-after: always"></div>
6. Leave the rest of the settings at the default, scroll all the way down, and then click on the `Create VPC` button at the bottom right.![[Create_SDWAN_VPC.png]]![[Create_VPC2.png]]<div style="page-break-after: always"></div>

#### Subnet Setup

1. In the VPC Console Menu, open the dashboard using the Hamburger icon menu if the left sidebar has not already been opened, and click on the `Subnets` option. Afterwards, click on the `Create subnet` button at the top right of the screen to enter Subnet creation. ![[Create_Subnet.png]]
   
2. Give your subnet a name. 
3. Choosing `No preference` for the availability zone will have AWS pick arbitrarily which AZ your subnet is a part of. You may leave that as your choice, unless specifically stated elsewhere if you are only basing your setup off of the COSMIAC-RAPID Lab Environment. 
4. Your VPC CIDR Block should already be specified as the default that was created earlier. Your subnet CIDR Block can be set as the default that AWS Suggests, (which in the configuration, if using the same exact settings, is `10.0.1.0.28`) but you still have to fill in the blank yourself. ![[Subnet_Config.png]]
5. Add any additional subnets you deem necessary by using the `Add new subnet` button provided at the end of the configuration menu, but note that the COSMIAC-RAPID Lab Environment only utilized one (1) subnet.
6. Click on the `Create subnet` button to finish creating the subnet.
#### Security Group Setup

Security Groups in AWS as stated above, act as sort of instance-based virtual firewalls. They apply specific policies that can apply to one specific instance in your VPC, or multiple instances at the same time if there is more than one service that needs the same rules. These rules are **stateful**, meaning that any connection made from an internal source (within the VPC that has the associated security group attached) will implicitly allow return traffic that utilizes that same connection.

Now, while the rules are stateful, we will still explicitly allow some services to ensure their functionality through the fabric. 

1. In the VPC Console Menu, open the dashboard using the Hamburger icon menu if the left sidebar has not already been opened, scroll down to the `Security Group` option, and click on it. Afterwards, click on the `Create security group` button at the top right of the screen. ![[Click_Create_SG.png]]
   
2. Name your Security Group.
3. Add a description for your security group
4. Choose the VPC that was created earlier, unless using default VPC.![[SG_Basic_Details.png]]

5. The primary goal is to allow the necessary inbound traffic so that SD-WAN functionality is unimpeded. Click on `Add rule` in the `Inbound rules` Section. There are multiple rules that need to be added, which will be listed below. ![[SG_Add_Rule.png]]<div style="page-break-after: always"></div>
   
   The structure is as follows:
   - Type: Type of communication
   - Protocol: Internet protocol used to facilitate the type of communication
   - Port Range: Range of ports the communication encompasses.
   - Source: Origin of network traffic associated with the communication. This is in the form of an IP, which has to be denoted in CIDR Block Format (x.x.x.x/s)
   - Description: Optional field utilized for a descriptor for the rule that was made.

	1. `DTLS` 
	    - Type: `Custom UDP` 
	    - Port Range: `12346-12445` 
	    - Source: `Custom`, Palo Alto Firewall Public IP - (For COSMIAC-RAPID Lab Environment, this value was `138.199.102.114/32`) 
	      The resulting rule should look similar to the below. ![[DTLS_SG.png]]
	      
	2. TLS 
		- Type: `Custom TCP` 
	    - Port Range: `23456-24757` (This is the range used for the COSMIAC-RAPID Lab Environment, though this could very well be different in another implementation. Ranges for TLS were slightly confusing and hard to find documentation for) 
	    - Source: `Custom`, Palo Alto Firewall Public IP - (For COSMIAC-RAPID Lab Environment, this value was `138.199.102.114/32`) 
	      The resulting rule should look similar to the below. ![[TLS_SG.png]]
	      
	3. IPSec (No NAT)
		- Type: `Custom UDP` 
	    - Port Range: `500` 
	    - Source: `Custom`, StarLink Public IP - (For COSMIAC-RAPID Lab Environment, this value was `143.105.158.123/32`) 
	      The resulting rule should look similar to the below. ![[IPSec_NoNat_SG.png]]<div style="page-break-after: always"></div>
	      
	4. IPSec (NAT)
		- Type: `Custom UDP` 
	    - Port Range: `4500` 
	    - Source: `Custom`, StarLink Public IP - (For COSMIAC-RAPID Lab Environment, this value was `143.105.158.123/32`) 
	      The resulting rule should look similar to the below. ![[IPSec_NAT_SG.png]]
	      
	5. BGP
		- Type: `Custom UDP` 
	    - Port Range: `3784-3785` 
	    - Source: `Custom`, StarLink Public IP - (For COSMIAC-RAPID Lab Environment, this value was `143.105.158.123/32`) 
	      The resulting rule should look similar to the below. ![[BGP_SG.png]]
	      
	6. BGP (Multi-Hop)
		- Type: `Custom UDP` 
		- Port Range: `4784` 
	    - Source: `Custom`, StarLink Public IP - (For COSMIAC-RAPID Lab Environment, this value was `143.105.158.123/32`) 
	      The resulting rule should look similar to the below. ![[BGP_Multihop_SG.png]]
	      
	7. SSH (From Company Network (LaunchPad @ Cosmiac))
		- Type: `Custom UDP` 
		- Port Range: `4784` 
	    - Source: `Custom`, Your Company's Public IP - (For COSMIAC-RAPID Lab Environment, this value was `138.199.102.101/32`) 
	      The resulting rule should look similar to the below. ![[SSH_LaunchPad_SG.png]]
	8. SSH (From Palo Alto Firewall)
		- Type: `Custom UDP` 
		- Port Range: `4784` 
	    - Source: `Custom`, Palo Alto Firewall Public IP - (For COSMIAC-RAPID Lab Environment, this value was `143.105.158.123/32`) 
	      The resulting rule should look similar to the below. ![[SSH_PAF_SG.png]]
6. (Optional) Make a tag to associate your security group with, for housekeeping and auditing purposes.
7. Finish creating your security group by scrolling down and clicking `Create security group`. ![[Finish_Create_SG.png]]

### EC2 Instance Launch Configuration

AWS EC2, otherwise known as Amazon Web Services Elastic Compute Cloud, is a service provided by AWS that allows you to host an instance of a machine on the cloud. It is essentially a virtual machine provisioning service that allows you to deploy an instance, or multiple instances, of a particular machine image and run your services (web or otherwise) on it, while AWS Hosts the physical hardware.

1. Navigate to the `EC2` console by using the search bar at the top of the Management Console webpage. ![[Search_EC2.png]]<div style="page-break-after: always"></div>
   
2. Open the EC2 Console Dashboard using the hamburger icon at the top left of the webpage if not already open, and then click on the `Instances` option in the sidebar. Then, click on the `Launch instances` button at the top right of the webpage. ![[Launch_Instance.png]]
   
3. Name the EC2 instance according to your naming scheme/standards, or use the example below.
4. Click on the `Quick Start` tab if not already on it, and then choose the `Ubuntu` AMI. It should default to an Ubuntu Server 24.04 image, but if not, choose an Ubuntu 24.04 image. Leave the rest of the settings as defaults, unless you see discrepancies with the defaults in your setup vs the example below. ![[EC2_Config_1.png]]<div style="page-break-after: always"></div>
   
5. Choose an instance type as depicted by your setup instructions from associated personnel. If wishing to mirror the COSMIAC-RAPID Lab Environment, choose `c6id.large`. ![[Select_Instance_Type.png]]
   
6. If you do not already have a designated test/lab key pair from associated personnel, follow the below instructions to make a key pair specific for this SD WAN Configuration. Otherwise, select your preferred key pair. This is the way you will gain SSH access to the instance.
	1. Click on the `Create new key pair` button. This will bring up the key pair creation window. ![[Create_Key_Pair.png]]
	2. Name your new Key Pair according to your naming scheme, or use the example below.
	3. Choose `RSA` as the `Key pair type`.
	4. Choose `.pem` as the Private key file format. This is important as this is the private key format that SSH Utilizes.
	5. Create the key pair using the `Create key pair` button ![[Confirm_KP.png]]

7.  Reload available key pairs by clicking on the clockwise circle arrow, and select your preferred key pair. ![[Select_KP.png]]<div style="page-break-after: always"></div>
   
8. In the `Network settings` section, first click on the `Edit` Button at the top right. Afterwards, a more granular set up section should replace the `Network settings` section. ![[Edit_Network_Settings.png]]
   
9. Choose the VPC created earlier. This will give you access to the subnets and security groups associated with the VPC.![[Select_VPC.png]]
10. If your VPC was preconfigured for you, or you are selecting an existing/in the default VPC, you may have access to more subnets than what you created. Choose the subnet that you created earlier in this documentation if one was not provided for you by associated personnel. ![[Select_Subnet.png]]

11. Enable the `Auto-assign public IP` setting. ![[Enable_AutoAssign_IP.png]]<div style="page-break-after: always"></div>

12. Click on `Select existing security group`, and then choose the Security Group(s) that you created earlier/were instructed to use. ![[Select_SG.png]]

13. Leave the rest as default. review the summary at the far right of the webpage, and confirm that the settings are correct, then click `Launch instance`. ![[Launch_EC2.png]]<div style="page-break-after: always"></div>

### EC2 Network Interface Configuration

The AWS setup portion of this instruction set was done within AWS Commercial **Cloud**, not on AWS Commercial **Outpost**. This section will be updated in the future, as the network configuration is different on the outpost than in cloud, as all the cloud interfaces are virtualized and provisioned by AWS themselves, while an AWS Outpost is a physical server hosted on-prem (In this case, in the COSMIAC-RAPID Lab Environment at COSMIAC LaunchPad), and the on-prem outpost has more Network Interface configurability as we have access to the physical hardware provided by AWS.
 
### EC2 Instance Linux Box Configuration

1. SSH into the EC2 instance with a machine/device on one of the allowed networks from the Security group using the command:
   `ssh -i <your_key_pair_file.pem> ubuntu@<youe_elastic_ip_here>`
   For example, the command to SSH into the COSMIAC-RAPID EC2 Instance was:
   `ssh -i cosmiac-rapid-ec2.pem ubuntu@13.218.134.192`
   **NOTE**: That is not the actual name of the file. It is obfuscated for security reasons.
2. When in the EC2 instance, run the following commands:
	1. `sudo iptables -A FORWARD -i ens6 -o ens5 -j ACCEPT`
	2. `sudo iptables -A FORWARD -i ens5 -o ens6 -m state --state RELATED,ESTABLISHED -j ACCEPT`
	3. `sudo iptables -t nat -L POSTROUTING -n -v`
	4. `sudo iptables -t nat -L POSTROUTING -o ens5 -j MASQUERADE`
	**NOTE**: `ens5` and `ens6` are likely to be different in your configuration. For reference, `ens5` is the WAN facing/Public/Untrust Zone (in other words, the interface used to access public internet), and `ens6` is the LAN facing/private/Trusted Zone (in other words, the interface for the LAN interface, where your client devices connect to. In this case, our only 'client' is the virtual edge router.)<div style="page-break-after: always"></div>

## ESXi Hypervisor

Connect to the host machine via its DHCP Ip given to it by the Palo Alto Firewall. In our setup, this IP was `10.10.0.2`.

This is what the host landing page looked like for the COSMIAC-RAPID Lab Environment
![[Default_Host_Window.png]]

### Network Configuration

This configuration is going to be done under the `Networking` tab of the ESXi host machine. Click on the `Networking` Tab on the far left Navigation panel on the ESXi host GUI to access the Networking GUI and the necessary Network Configuration tools for the ESXi Host Machine.

This is what the `Networking` Window looks like.
![[Networking_Landing_Page.png]]<div style="page-break-after: always"></div>
#### Physical NICs

In the Networking GUI Window, click on the `Physical NICs` Tab. This will display the available physical NICs on the server. The Dell Server utilized in the COSMIAC/RAPID Lab Environment has 6 physical NICs immediately available for use. The list of physical NICs are labeled `vmnic0-vmnic5` respectively.

These physical NICs will be your interfaces for internet communication, and the associated hardware connections to your commercial internet services will be accessed utilizing these vmnics. These vmnics will be attached to your switches in the following sections as your uplinks.
![[Physical_NICs.png]]
#### vSwitches

In the Networking GUI Window, click on the `Virtual Switches` Tab. This will bring up the configuration menu of your vSwitches on the ESXi Host Machine. Most of the configuration settings of the vSwitches will be left at a default. 

The COSMIAC/RAPID Lab environment utilizes three (3) total vSwitches. Follow an intuitive or set naming scheme for these switches, so that it is understood what each switch is being utilized for/what uplinks are connected to it. For simplicity, this instruction set will refer to the three (3) vSwitches as `vSwitch0`, `vSwitch1`, and `vSwitch2`.

Since the settings for each vSwitch will be kept at default, follow the steps below to set up the vSwitches. 

1. Click on `Add standard virtual switch`![[Add_vSwitch.png]]
2. Give your vSwitch a name. These are the names we alocated to the vSwitches in the COSMIAC/RAPID Lab Environment: PA-Switch, EC2-Switch, SatCom-WAN
3. Click on the dropdown box for `Uplink 1` and choose the vmnic that is most appropriate for the switch. (e.g. If `vmnic1` is your EC2 Fiber connection to the AWS Outpost, then you would associate this vmnic with the EC2-Switch labeled above) 
   **NOTE**: You can have multiple uplinks, and this is normally advised/encouraged in the case of internet outages. In the case of the SatCom-WAN Switch labeled above, you may have more than one uplink configured, but these will have to be VLAN'ed carefully to ensure the correct devices get the correct SatCom interface(s).
4. Click on the `Add` button at the bottom of the `Add standard virtual switch` Window.
#### Port Groups

In the Networking GUI Window, click on the `Port Groups` tab. This will bring up the configuration menu of the port groups that will be attached to your vSwitches. These port groups are essentially providing your Network infrastructure with segmentation, dictating what policies are served to each group of devices connected to the switch in that particular port group. Port Groups can also be used for VLAN Tagging, in order to properly VLAN Devices/interfaces correctly with the VLAN settings of the switch or other networking device that the ESXi is/may be attached to.

The COSMIAC/RAPID Lab Environment utilizes seven (7) port groups, which will be allocated to one of the above vSwitches from the earlier section. This port group section will delineate all the configurations for the seven (7) port groups utilized by the Lab Environment.

In each of the Port Groups below, you will click on the `Add port group` button as shown below.
![[Add_PG.png]]

##### VM Network

This port group will be utilized to give the Control Plane devices the ability to be publicly routable, which from the high level overview, is crucial for the COSMIAC/RAPID Lab Environment. The reason for this is so that the WAN Interfaces of the Edge Devices are able to reach and establish complete connectivity with the Control Plane, so that the tunnels between edges can be facilitated.

This port group should only have connections from the Control Plane devices during a test run, but can be used by Client devices to gain internet access for a time to install necessary packages from `apt` for further network analysis. Otherwise, connection to this port group should be disabled or removed from devices other than the Control Plane devices.

1. Click on the `Add port group` button.
2. Name the new port group `VM Network`, or whatever the naming scheme dictates.
3. Do not assign a VLAN ID unless your network infrastructure requires you to do so. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group did not require a VLAN ID.
4. Choose the corresponding vSwitch that is connected to the Palo Alto firewall. In the COSMIAC/RAPID Deployment, this was PA-Switch.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.

##### EC2 Connection

This port group will be utilized to serve as the internet connection for the Sink Edge Device in the Lab Environment. This connection to an EC2 Instance will be facilitated through a Local Link cable, connecting the ESXi Host directly to an AWS Outpost server that will be hosting this EC2 Instance. 

This EC2 instance is the NAT Device mentioned in the High Level Overview that the Sink Edge will be utilizing as a stand in in place of a secondary StarLink unit, as of the time of this deployment, only one Enterprise level StarLink terminal was procured. 

1. Click on the `Add port group` button.
2. Name the new port group `EC2Connection`, or whatever the naming scheme dictates.
3. Do not assign a VLAN ID unless your network infrastructure requires you to do so. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group did not require a VLAN ID.
4. Choose the corresponding vSwitch that is connected to the EC2 Instance. In the COSMIAC/RAPID Deployment, this was EC2-Switch.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.

The EC2 instance itself does not inherently service ips through DHCP, so if a device is connected to this port group, or any other port group associated with the EC2-Switch vSwitch, they need to be manually assigned an IP within the range of the subnet defined in the EC2 instance.

##### StarLink-Source-PG

This port group will be used to service the WAN facing interface on the Source Edge in the Lab Environment. Ideally, the lab environment would utilize StarLink on both ends, but as of the time of development, the COSMIAC-RAPID Lab Environment only had access to a single StarLink terminal. 

This will be the primary source of data-flow, as the overarching goal of the system is to run Network performance analyses on commercial SATCOM internet service providers. The StarLink service is the first of three to be integrated with our Environment, and will be the main point of focus for this series of Network performance tests.

1. Click on the `Add port group` button.
2. Name the new port group `StarLink-Source-PG`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 200. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID due to the hardware configuration of the Lab environment.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.<div style="page-break-after: always"></div>
   
##### ViaSat-PG

1. Click on the `Add port group` button.
2. Name the new port group `ViaSat-PG`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 201. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID due to the hardware configuration of the Lab environment.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
##### HughesNet-PG

1. Click on the `Add port group` button.
2. Name the new port group `HughesNet-PG`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 202. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID due to the hardware configuration of the Lab environment.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
##### vpn-100-source

This port group will be used to service the LAN facing interface of the StarLink Source Edge. This port group will also be utilized to service IPs through DHCP from the Source Edge. The Data Source Client will have a connection to this port group in order to utilize the SD WAN Fabric to, as the name implies, be the TX source of data that will be used to analyze the Network Performance with. 

1. Click on the `Add port group` button.
2. Name the new port group `vpn-100-source`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 200. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment,  this port group required a VLAN ID so that it would still be segmented away from the main network that allowed for internet access, as the goal was to ensure that the DHCP server running on the Edge Routers could use this port group to service private IPs to the corresponding systems.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
##### vpn-100-sink

This port group will be used to service the LAN facing interface of the EC2 Sink Edge. This port group will also be utilized to service IPs through DHCP from the Sink Edge. The Data Sink Client will have a connection to this port group in order to utilize the SD WAN Fabric to, as the name implies, be the RX point of reception of the data that will be used to analyze the Network Performance.

1. Click on the `Add port group` button.
2. Name the new port group `vpn-100-source`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID that does not have a proper VLAN mapping, so that this port group cannot service an internet connection. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID so that it would still be segmented away from the main network that allowed for internet access, as the goal was to ensure that the DHCP server running on the Edge Routers could use this port group to service private IPs to the corresponding systems.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
## Palo Alto Firewall

The COSMIAC-RAPID Lab Environment is utilizing an already-existing Palo Alto firewall that had basic firewall configurations already made, and thus this instruction set will go over only the portion that was necessary for Cisco SD-WAN Functionality/Connectivity. What this means is this instruction set will assume that basic firewall configuration needs have been met prior to configuring the firewall for SD-WAN Functionality. 

### Services Configuration
You need to specify the service ports that the Cisco SD-WAN control plane requires to communicate within the control plane and by your distant edge routers. We've found that a range of ports is required for use in both DTLS and TLS. DTLS and TLS protocols are used to facilitate secure communication.
We've also utlizied port offset from the default port ranges for vManage and vSmart since we are using only one public IP address for all 3 controllers. Below is a table of required ports. 

***Note: The range of ports needed may vary depending on which version of Cisco SD-WAN is being implimented.***

| Controllers | DTLS Ports (UDP)                                                                                 | TLS Ports (TCP)                                                                                  |
|-------------|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|
| vBond       | 12346, 12366, 12386, 12406, 12426, 12446, 12466, 12486, 12506, 12526, 12546, 12566, 12586, 12606 |                                                                                                  |
| vSmart      | 12347, 12367, 12387, 12407, 12427, 12447, 12467, 12487, 12507, 12527, 12547, 12567, 12587, 12607 | 23456, 23556, 23656, 23756, 23856, 23956, 24056, 24156, 24256, 24356, 24456, 24556, 24656, 24756 |
| vManage     | 12348, 12368, 12388, 12408, 12428, 12448, 12468, 12488, 12508, 12528, 12548, 12568, 12588, 12608 | 23457, 23557, 23657, 23757, 23857, 23957, 24057, 24157, 24257, 24357, 24457, 24557, 24657, 24757 |

Here are the steps for setting up to services:
1. Sign into the Firewall using an internet browser and the firewall's IP address (for our setup we are connecting using the internal connection for which the firewall is our default gateway so we use default gateway address 10.10.0.1).
2. Once in the web interface click the ribbon for 'Objects'. ![img.png](img.png)
3. In objects click 'Services', then click 'Add'. ![img_1.png](img_1.png)<div style="page-break-after: always"></div>
4. This should open a 'Service' pop-up. In the pop-up define the name (for example for vManage dtls we named it "CiscoSdWAN_vmanage_dtls"). Define the protocol (TCP for TLS and UDP for DTLS). Then for the specified ports use the previously defined table and the ports should be placed in the 'Desination Port' section. Once those 3 are filled out click ok. Create the services for vBond, vSmart, and vManage. Below is an example of vManages services. ![img_2.png](img_2.png) ![img_3.png](img_3.png)
5. Once all the services are define in 'Objects' click 'Service Groups', and then click 'Add'. ![img_4.png](img_4.png)
6. You want to create service groups to combine the TLS and DTLS services into one service group for each controller and one service group for all the SD-WAN services. It should look something like the image below.![img_5.png](img_5.png)
7. Once the services and service groups are created click 'Commit'. ![img_7.png](img_7.png)
8. This should open a pop-up. If you're not the only one making changes make sure the radio button is on "Commit Changes Made by: [YOUR_LOGGED_IN_ACCOUNT]". Then click commit. ![img_22.png](img_22.png)
9. Once the commit is finished you can then move to making the security policies.
### Security Policies

For security policies we are going to verbosely set allowances for the ports the control plane requires. We'll make two security policies, one for the nat hairpinning and another for port-forwarding.
Here are the security policy steps:
1. Click the Policies Ribbon, Security, then click add. ![img_24.png](img_24.png)<div style="page-break-after: always"></div>
2. A 'Security Policy Rule' pop-up should show up, and in 'General' tab create a name. For first Security policy lets configure security policy for nat hairpinning. Name the policy something like "nat_hairpin_to_vbond". ![img_25.png](img_25.png)
3. In the 'Source' tab under 'Source Zone' click add then choose LAN as the source zone (other names could also be Inside or Trust), 'Source Address' should be any. ![img_33.png](img_33.png)<div style="page-break-after: always"></div>
4. Skip the 'User' tab.
5. In the 'Destination' tab under 'Destination Zone' click add then choose WAN as destination zone (other names could also be Outside or Untrust), in 'Destination Address' click add and then add the firewall's public address. ![img_26.png](img_26.png)
6. Skip the 'Application' tab.
7. In the 'Service/URL Category' in the dropdown switch to select and then add the sd-wan service group that has all the services for sd-wan. ![img_27.png](img_27.png)
8. You should be able to leave the actions group as default (assuming that allow is the default action). ![img_28.png](img_28.png)
9. Then click 'Ok'.
10. This should've created the security policy that look something like the following. ![img_30.png](img_30.png)
11. Next lets create the security rule for port-forwarding. Under the Policies Ribbon, Security, click add.
12. A 'Security Policy Rule' pop-up should show up, and in 'General' tab create a name. This security policy is for nat port-forwarding. Name the policy something like "cisco_sd_wan_port-forwarding". ![img_31.png](img_31.png)
13. In the 'Source' tab under 'Source Zone' click add then choose WAN as source zone (other names could also be Outside or Untrust), 'Source Address' should be any. ![img_32.png](img_32.png)
14. Skip the 'User' tab.
15. In the 'Destination' tab under 'Destination Zone' click add then choose LAN as destination zone (other names could also be Inside or Trust), in 'Destination Address' click add and then add the firewall's public address. ![img_34.png](img_34.png)<div style="page-break-after: always"></div>
16. Skip the 'Application' tab.
17. In the 'Service/URL Category' in the dropdown switch to select and then add the sd-wan service group that has all the services for sd-wan created previously. ![img_27.png](img_27.png)
18. You should be able to leave the actions group as default (assuming that allow is the default action). ![img_28.png](img_28.png)
19. Then click 'Ok'.
20. This should've created the security policy that look something like the following. ![img_35.png](img_35.png)

Once all the security policies are created you will need to order them as the firewall follows policy rules from highest to lowest priority.
Priority order should be the new policies you created above the default policies that were there previously.
To change the order try the following:
1. In Policies Ribbon, Security, there should be a drop down by the name of the policy. Click the move option. ![img_36.png](img_36.png)
2. A pop-up should show up for moving your policy, set a rule and then click either move before or after (which button to choose will depend on the current order of your rules).![img_37.png](img_37.png)
3. Repeat moving rules until all the security policies you created are near the top. It should something like this when you've reprioritized your security policies. ![img_38.png](img_38.png)

Next commit the security policies you created.
1. Once all the security policies are created click 'Commit'. ![img_39.png](img_39.png)<div style="page-break-after: always"></div>
2. This should open a pop-up. If you are not the only one making changes make sure the radio button is on "Commit Changes Made by: [YOUR_LOGGED_IN_ACCOUNT]". Then click commit. ![img_23.png](img_23.png)
3. Once the commit is finished you can then move to making the NAT policies.

### NAT Policies

Since we are using only one public IP address for 3 controllers (vbond, vmanage, vsmart), we will need to setup port-fowarding and nat hair-pinning.

We use port-forwarding to forward traffic to the correct controller from packets coming in from outside the network.
With port-forwarding when a packet with a specified port comes to our public address it will forward the packet to our internal network to the correct private address.

We use nat hair-pinning polices to have vSmart and vManage traffic look like its being routed from our public address. This is required as vBond advertises the IP addresses that vManage and vSmart use to communicate with vBond to distant edge routers.

Here are the steps for setting up port-forwarding:
1. Click the Policies Ribbon, NAT, then click add. ![img_8.png](img_8.png)
2. A 'NAT Policy Rule' pop-up should show up, and in 'General' tab create a name. For the first NAT policy lets configure port-forwarding for vBond. Name the policy something like "cisco_sd_wan_vbond". 'NAT Type' should be ipv4. ![img_9.png](img_9.png)<div style="page-break-after: always"></div>
3. Next click 'Original Packet' tab. Add WAN as source zone (other names could also be Outside or Untrust). For 'Destination Zone' also choose WAN. 'Destination Interface' and 'Source Address' should both be any. Service should be the service group created previously for vBond for both TLS and DTLS traffic. For 'Destination Address' click add, and add the firewalls public address. See below image as an example. ![img_10.png](img_10.png)
4. Next click 'Translated Packet' tab. 'Source Address Translation' should be none. Under 'Destination Address Translation' 'Translation Type' should be 'Static IP', 'Transalted Address' should be vBonds private address, 'Translated Port' should be blank and 'Enable DNS Rewrite' should be unchecked. See below image for example.![img_11.png](img_11.png)  
5. Next click 'OK' and the policy created should show up in NAT. It should look like something similar to the following. ![img_12.png](img_12.png)
6. Next create port-forwarding rules for vSmart and vManage. It should follow the same pattern just make sure you choose the appropriate group service and private IP addresses for vSmart and vManage.

Next are the steps to create nat-hairpinning. We need to hair-pin vSmart and vManage communication to vBond.
1. Ensure you are still in Policies ribbon, NAT, then click add (see step 1 of port-forwarding if you need to confirm your in the correct page).
2. A 'NAT Policy Rule' pop-up should show up, and in 'General' tab create a name. For first NAT policy lets configure the nat hairpin for vSmart. Name the policy something like "nat_hairpin_vsmart_to_vbond". 'NAT Type' should be ipv4. ![img_13.png](img_13.png) 
3. Next click 'Original Packet' tab. Add LAN as source zone (other names could also be Inside or Trust). For 'Destination Zone' choose WAN (other names could also be Outside or Untrust). 'Destination Interface' should be any. 'Source Address' click add, and you should add vSmart's IP address. Service can be the service group for all control group services. For 'Destination Address' click add, and add the firewalls public address. See below image as an example. ![img_14.png](img_14.png)
4. Next click 'Translated Packet' tab. 'Source Address Translation' the 'Translation Type' should be 'Static IP' and for the 'Translation Address' it should be the firewalls public IP. Under 'Destination Address Translation' 'Translation Type' should be 'Static IP', 'Translation Address' should be vBonds private address, 'Translated Port' should be vBond port (here we are using 12346, refer to the starting range for dtls for vBond) and 'Enable DNS Rewrite' should be unchecked. See below image for example. ![img_15.png](img_15.png)
5. Next click 'OK' and the policy created should show up in NAT. It should look like something similar to the following. ![img_16.png](img_16.png)
6. Next create nat hairpin rules for vManage. It should follow the same pattern you should use the same group service as the previous nat hairpinning, for source address use vManage's IP and Translated Packet tab should be the same as vSmart's.<div style="page-break-after: always"></div>

Once all the port-forwarding and hairpinning policies are created you will need to order them as the firewall follows policy rules from highest to lowest priority.
Priority order should be the new policies you created above the default policies that were there previously.
To change the order try the following:
1. In Policies Ribbon, NAT, there should be a drop down by the name of the policy. Click the move option. ![img_17.png](img_17.png)
2. A pop-up should show up for moving your policy, set a rule and then click either move before or after (which button will depend on the current order of your rules).![img_18.png](img_18.png)<div style="page-break-after: always"></div>
3. Repeat moving rules until all the NAT policies you created are near the top. It should something like this when you've reprioritized your NAT policies. ![img_19.png](img_19.png)

Next commit the NAT policies you created.
1. Once all the NAT policies are created click 'Commit'. ![img_20.png](img_20.png)<div style="page-break-after: always"></div>
2. This should open a pop-up. If you are not the only one making changes make sure the radio button is on "Commit Changes Made by: [YOUR_LOGGED_IN_ACCOUNT]". Then click commit. ![img_23.png](img_23.png)
3. Once the commit is finished you should be done with all the required firewall configurations.<div style="page-break-after: always"></div>

## VM Deployment

This configuration will be done under the `Virtual Machines` tab. This will bring up the configuration menu for all of the Virtual Machines that you will be utilizing. Many, if not all of the devices will end up being a Virtual Machine, unless you can configure connectivity of an external device, such as a Laptop, with connectivity to the edge routers.

Though the COSMIAC/RAPID deployment is entirely virtualized, the hardware components utilized in the setup enabled the virtual components to represent a full physical/commercial deployment of a small SD WAN Network Infrastructure. 

This section will be split into two instruction sets, one for the Control Plane, and the other for the Data Plane. The button mentioned in the first step of each VM's creation is shown here.
![[Create_A_VM.png]]
### Control Plane Devices

The Control Plane devices are the central brains of the system, and are all required for Cisco SD WAN Functionality. There are three (3) control plane devices, vBond (also known as the Validator), vSmart (also known as the Controller), and vManage (also known as the Manager), all of which will be detailed in the following sections. Note that depending on the images you receive, extra VM options may need to be configured, which will be stated at the start of each section. Unfortunately, there will be no screenshots for these extra configurations, as we at the COSMIAC-RAPID Lab Environment had access to an image that had these settings already instantiated on the machine.<div style="page-break-after: always"></div>
#### vBond

The vBond (Validator) control plane device handles the validation of all devices, control and data plane alike. All Cisco SD WAN Devices must make first contact with vBond before any of the other SD WAN components. vBond is the window/gateway into the SD WAN Fabric, and is what helps facilitate initial route establishments between edge devices and control plane devices. 

What makes vBond so important in the entire scheme as the first point of contact is its role as the Validator, as it ensures that the devices connecting to it/attempting to join into the fabric are legitimate non-malicious devices to the best of its ability. Essentially, it acts as the initial "security guard" of the system.

**Extra Configurations**
 - Memory: 4 GB
 - Processors: 2

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `vBond` or whatever your naming scheme dictates. 

4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `viptela-bond-20.15.20genericx86-64.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.<div style="page-break-after: always"></div>
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine.
   **NOTE**: This is the file that you can utilize to spin up as many vBond control plane devices as your Cisco account is allotted.![[Open_Datastore-browser.png]]![[OVA_Location.png]]
   
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. ![[Select_Storage.png]]<div style="page-break-after: always"></div>

 6. Select two (2) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the vBond in the COSMIAC-RAPID Lab environment, `VM Network` was `VM Network`, and `VM Network1` was removed before the launch of vBond. More Network mappings can be made later as necessary.
    **NOTE**: If you do not know how many Network Adapters/interfaces you need refer to related diagrams/documentation for how many Network Adapters should exist in the respective control plane device. For the COSMIAC-RAPID Lab Environment, there were **two** (2) total network adapters for each edge device.

7. Choose `Thick` as the Disk provisioning
8. Uncheck `Power on automatically`. After 6-9 have been completed, click the `Next` button.![[Choose_Controller_Adapters.png]]<div style="page-break-after: always"></div>
9. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. 
10. As mentioned in step 7, we will now remove the `VM Network1` interface. Navigate to your newly created vBond, and click on `Edit`. ![[To_vBond.png]]![[Edit_Bond.png]]<div style="page-break-after: always"></div>

11.  In this edit menu, first choose the necessary adapter for `Network Adapter 1`, and then remove `Network Adapter 2`. Once finished, click `Save` at the bottom right. ![[Edit_Control_Plane_Network_Interfaces.png]]
12. The setup for vBond has completed upon this step. DO NOT START THE MACHINE until later.<div style="page-break-after: always"></div>
#### vManage

The vManage (Manager) Control Plane Device handles the management and administration orchestration of the Cisco SD WAN Fabric. It is meant to be the central hub/authority on all management of the SD WAN Fabric, with the ability to make and replicate configurations for edge devices and control plane devices outside of vBond to make larger deployments easier and faster.

The advantage of this is that if you have a default router configuration that your company routers should follow, all default acting routers just need to have been onboarded to the SD WAN Fabric with default factory settings on their router, and afterwards, vManage handles pushing templates and associated certificates onto the device.

vManage also allows you to monitor the health and logs of your SD WAN Fabric, allowing for streamlined standard operating procedures regarding the management and auditing of your network infrastructure.

**Extra Configurations**
- Set Disk Size = 30 GB
- Create new disk size = 100 GB

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `vManage` or whatever your naming scheme dictates. 

4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `viptela-vmanage-20.15.20genericx86-64.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.![[Open_Datastore-browser.png]]![[OVA_Location.png]]
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine.
   **NOTE**: This is the file that you can utilize to spin up as many vManage control plane devices as your Cisco account is allotted.
   
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. ![[Select_Storage.png]]

 6. Select two (2) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the vBond in the COSMIAC-RAPID Lab environment, `VM Network` was `VM Network`, and `VM Network1` was removed before the launch of vBond. More Network mappings can be made later as necessary.
    **NOTE**: If you do not know how many Network Adapters/interfaces you need refer to related diagrams/documentation for how many Network Adapters should exist in the respective control plane device. For the COSMIAC-RAPID Lab Environment, there were **two** (2) total network adapters for each edge device.

7. Choose `Thick` as the Disk provisioning
8. Uncheck `Power on automatically`. After 6-9 have been completed, click the `Next` button. ![[Choose_Controller_Adapters.png]]
9. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. <div style="page-break-after: always"></div>
#### vSmart

The vSmart (Controller) Control Plane Device handles the advertisement of the OMP TLOCs (Transport Locators) and route policies received from each edge device. Edge devices will communicate with vSmart, which will then advertise how the initial edge should establish, or not establish, connections within the SD WAN Fabric.

**Extra Configurations**
- Increase CPUs from 2 -> 4

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `vSmart` or whatever your naming scheme dictates. 

4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `viptela-smart-20.15.20genericx86-64.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.
	    ![[Open_Datastore-browser.png]]![[OVA_Location.png]]
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine.
   **NOTE**: This is the file that you can utilize to spin up as many vSmart control plane devices as your Cisco account is allotted.
   
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. ![[Select_Storage.png]]

 6. Select two (2) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the vBond in the COSMIAC-RAPID Lab environment, `VM Network` was `VM Network`, and `VM Network1` was removed before the launch of vBond. More Network mappings can be made later as necessary.
    **NOTE**: If you do not know how many Network Adapters/interfaces you need refer to related diagrams/documentation for how many Network Adapters should exist in the respective control plane device. For the COSMIAC-RAPID Lab Environment, there were **two** (2) total network adapters for each edge device.

7. Choose `Thick` as the Disk provisioning
8. Uncheck `Power on automatically`. After 6-9 have been completed, click the `Next` button.![[Choose_Controller_Adapters.png]]
9. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. <div style="page-break-after: always"></div>

### Edge Devices
#### StarLink Source Edge

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `StarLink Source Edge` or whatever your naming scheme dictates. 

4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `c8000v-universalk9.17.15.02a.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.<div style="page-break-after: always"></div>
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.![[Open_Datastore-browser.png]]![[OVA_Location.png]] 
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine.
   **NOTE**: This is the file that you can utilize to spin up as many edges as your Cisco account is allotted. ![[Select_OVA.png]]<div style="page-break-after: always"></div>
   
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. ![[Choose_storage_option.png]]

 6. Select three (3) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the Data Source Edge in the COSMIAC-RAPID Lab environment, `GigabitEthernet1` was `StarLink-source-PG`, and `GigabitEthernet2` was `source-vpn-100`. More Network mappings can be made later to accommodate for other SATCOM or Terrestrial ISPs. 
    **NOTE**: If you do not know how many Network Adapters/interfaces you need refer to related diagrams/documentation for how many Network Adapters should exist in the respective edge device. For the COSMIAC-RAPID Lab Environment, there were **two** (2) total network adapters for each edge device.

7. Choose `8x Large - 16GB Disk` as the deployment type
8. Choose `Thick` as the Disk provisioning<div style="page-break-after: always"></div>
9. It is your choice to power on automatically or not. If planning to add/remove a Network Mapping, I suggest making sure this box is ***unchecked***. After 6-9 have been completed, click the `Next` button. ![[Configure_Router_Example.png]]
10. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. ![[Review_Router.png]]

#### EC2 Sink Edge

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `EC2 Sink Edge` or whatever your naming scheme dictates. 
4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `c8000v-universalk9.17.15.02a.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.
	    ![[Open_Datastore-browser.png]]![[OVA_Location.png]]<div style="page-break-after: always"></div>
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine. ![[Select_OVA.png]]
   **NOTE**: This is the file that you can utilize to spin up as many edges as your Cisco account is allotted.<div style="page-break-after: always"></div>
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. ![[Choose_storage_option.png]]
 6. Select three (3) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the Data Sink Edge Router in the COSMIAC-RAPID Lab Environment, `GigabitEthernet1` was `EC2Connection`, and `GigabitEthernet2` was `sink-vpn-100`. More Network mappings can be made later to accommodate for other SATCOM or Terrestrial ISPs.
    **NOTE**: If you do not know how many Network Adapters/interfaces you need refer to related diagrams/documentation for how many Network Adapters should exist in the respective edge device. For the COSMIAC-RAPID Lab Environment, there were **two** (2) total network adapters for each edge device.

7. Choose `8x Large - 16GB Disk` as the deployment type
8. Choose `Thick` as the Disk provisioning<div style="page-break-after: always"></div>
9. It is your choice to power on automatically or not. If planning to add/remove a Network Mapping, I suggest making sure this box is ***unchecked***. After 6-9 have been completed, click the `Next` button. ![[Configure_Router_Example.png]]

10. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. ![[Review_Router.png]]

### Client Devices

These Client devices will be integral to our test, as these will be where the network traffic **originates from** and gets *delivered to*, being the **source** and *sink* of data respectively. On the clients, we use an Ubuntu x64 Linux distribution. Installed on the virtual machines are the `iperf3` and `traceroute` packages from `apt`, as well as `mgen` from the Naval Research Laboratory. These tools are essential for our network performance analysis, as they all provide ways to inspect network performance metrics (`iperf3`, `mgen`) and traffic flow (`traceroute`).

#### Source Client

Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
1. Upon opening the creation window, choose the `Create a new virtual machine` option, and click on the `Next` button at the bottom right of the creation window.
2. Name the Virtual Machine `Data Source Client` or whatever your naming scheme dictates.
3. Leave the `Compatibility` selection as the default selection unless otherwise specified.
4. In the `Guest OS family` selection, choose `Linux` from the dropdown.
5. In the `Guest OS version` selection, scroll down in the dropdown until you see `Ubuntu x64` as an option.
6. After steps 3-6 are complete, click the `Next` button.
7. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. 
8. In the `Customize settings` portion of the creation, you will see a menu denoting all virtual hardware and VM Options. Adjust these to your requirements. If setting up a client in the COSMIAC-RAPID Lab Environment, these are the suggested settings<div style="page-break-after: always"></div>
	1. For `CPU`, this denotes the number of virtual CPUs present on the virtual machine. Choose `4` from the dropdown box. ![[Choose_CPUs.png]]
	2. For `Memory`, this denotes how much memory is allocated to the virtual machine from the host ESXi client. In the left input box, enter `8`, and then on the right dropdown box, switch from `MB`  to `GB`. ![[Choose_Memory.png]]
	3. Leave the `SCSI Controller 0`, `SATA Controller 0`, and `USB controller 1` as their defaults unless otherwise specified in your setup.<div style="page-break-after: always"></div>
	4. For your `Network Adapter 1`, choose `vpn-100-source`, or equivalent port group from the dropdown.![[Choose_Network_Adapter.png]]
	5. For `CD/DVD Drive 1`, select `Datastore ISO file`, and in the window that pops up, navigate to where your Ubuntu ISO is stored. This should have already been uploaded into your ESXi Host Client, but if it is not present, contact associated personnel or documentation for assistance. The COSMIAC_RAPID Lab Environment utilized the iso `ubuntu-24.04.1-RAPID-desktop-amd64.iso`, which is a custom ISO created by Zack Daniels of the COSMIAC-RAPID Lab Group.
	   This iso is based off of the `ubuntu-24.04.1-desktop-amd64.iso`.![[Choose_ISO_Source.png]] ![[Select_ISO_Source.png]]
	6. Leave the rest of the settings as default and click on `Next` at the bottom right of the window.
9. Review the configuration and click on `Finish` at the bottom right to complete your creation of the Virtual Machine.<div style="page-break-after: always"></div>

#### Sink Client

Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
1. Upon opening the creation window, choose the `Create a new virtual machine` option, and click on the `Next` button at the bottom right of the creation window.
2. Name the Virtual Machine `Data Sink Client` or whatever your naming scheme dictates.
3. Leave the `Compatibility` selection as the default selection unless otherwise specified.
4. In the `Guest OS family` selection, choose `Linux` from the dropdown.
5. In the `Guest OS version` selection, scroll down in the dropdown until you see `Ubuntu x64` as an option.
6. After steps 3-6 are complete, click the `Next` button.
7. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button. 
8. In the `Customize settings` portion of the creation, you will see a menu denoting all virtual hardware and VM Options. Adjust these to your requirements. If setting up a client in the COSMIAC-RAPID Lab Environment, these are the suggested settings
	1. For `CPU`, this denotes the number of virtual CPUs present on the virtual machine. Choose `4` from the dropdown box. ![[Choose_CPUs_Sink.png]]
	2. For `Memory`, this denotes how much memory is allocated to the virtual machine from the host ESXi client. In the left input box, enter `8`, and then on the right dropdown box, switch from `MB`  to `GB`. ![[Choose_Memory_Sink.png]]
	3. Leave the `SCSI Controller 0`, `SATA Controller 0`, and `USB controller 1` as their defaults unless otherwise specified in your setup.
	4. For your `Network Adapter 1`, choose `vpn-100-source`, or equivalent port group from the dropdown.![[Choose_Network_Adapter_Sink.png]]
	5. For `CD/DVD Drive 1`, select `Datastore ISO file`, and in the window that pops up, navigate to where your Ubuntu ISO is stored. This should have already been uploaded into your ESXi Host Client, but if it is not present, contact associated personnel or documentation for assistance. The COSMIAC_RAPID Lab Environment utilized the iso `ubuntu-24.04.1-RAPID-desktop-amd64.iso`, which is a custom ISO created by Zack Daniels of the COSMIAC-RAPID Lab Group.
	   This iso is based off of the `ubuntu-24.04.1-desktop-amd64.iso`.![[Choose_ISO_Sink.png]] ![[Select_ISO_Source.png]]
	6. Leave the rest of the settings as default and click on `Next` at the bottom right of the window.
9. Review the configuration and click on `Finish` at the bottom right to complete your creation of the Virtual Machine.<div style="page-break-after: always"></div>

## Control Plane CLI Setup
### vBond

1. Log into the CLI
   - `vbond login: admin`
   - `password: admin`
2. Set new password. (Recommend using a secure password, but for an already secure lab environment, a default `admin1` password should work). 
3. `vbond# config`
4. `vbond(config)# system`
5. `vbond(config-system)# hostname vBond`
6. `vbond(config-system)# system-ip 1.1.1.13`
7. `vbond(config-system)# site-id 100`
8. `vbond(config-system)# organization-name "RAPID-Laboratory`
9. `vbond(config-system)# vbond 138.199.102.114 local`
10. `vbond(config-system)# exit`
11. `vbond(config)# commit`
12. `vBond(config)# vpn 0`
13. `vBond(config-vpn-0)# interface eth0`
14. `vBond(config-interface-eth0)# no ipv6 dhcp-client`
15. `vBond(config-interface-eth0)# ip address 10.10.0.40`
16. `vBond(config-interface-eth0)# no shutdown`
17. `vBond(config-interface-eth0)# tunnel-interface`
18. `vBond(config-tunnel-interface)# allow-service ssh`
19. `vBond(config-tunnel-interface)# allow-service netconf`
20. `vBond(config-tunnel-interface)# no allow-service dhcp`
21. `vBond(config-tunnel-interface)# commit`
22. `vBond(config-tunnel-interface)# exit`
23. `vBond(config-interface-eth0)# exit`
24. `vBond(config-vpn-0)# ip route 0.0.0.0/0 10.10.0.1`
25. `vBond(config-vpn-0)# commit`
26. `vBond(config-vpn-0)# exit`
27. `vBond(config)# exit`
28. `vBond# show clock`
29. `vBond# clock set HH:MM:SS Day(1-31) Month(String) Year(YYYY)` - If not already in correct UTC Time)
30. Install associated Cisco Certificate with assistance of associated personnel.
   

### vSmart

1. Log into the CLI
   - `vsmart login: admin`
   - `password: admin`
2. Set new password. (Recommend using a secure password, but for an already secure lab environment, a default `admin1` password should work). 
3. `vsmart# config`
4. `vsmart(config)# system`
5. `vsmart(config-system)# hostname vSmart`
6. `vsmart(config-system)# system-ip 1.1.1.12`
7. `vsmart(config-system)# site-id 100`
8. `vsmart(config-system)# organization-name "RAPID-Laboratory`
9. `vsmart(config-system)# vbond 138.199.102.114`
10. `vsmart(config-system)# exit`
11. `vsmart(config)# security`
12. `vsmart(config-security)# control`
13. `vsmart(config-control)# protocol TLS`
14. `vsmart(config-control)# exit`
15. `vsmart(config-security)# exit`
16. `vsmart(config)# commit`
17. `vSmart(config)# vpn 0`
18. `vSmart(config-vpn-0)# interface eth0`
19. `vSmart(config-interface-eth0)# ip address 10.10.0.41`
20. `vSmart(config-interface-eth0)# no shutdown`
21. `vSmart(config-interface-eth0)# tunnel-interface`
22. `vSmart(config-tunnel-interface)# allow-service ssh`
23. `vSmart(config-tunnel-interface)# allow-service netconf`
24. `vSmart(config-tunnel-interface)# commit`
25. `vSmart(config-tunnel-interface)# exit`
26. `vSmart(config-interface-eth0)# exit`
27. `vSmart(config-vpn-0)# ip route 0.0.0.0/0 10.10.0.1`
28. `vSmart(config-vpn-0)# commit`
29. `vSmart(config-vpn-0)# exit`
30. `vSmart(config)# exit`
31. `vSmart# show clock`
32. `vSmart# clock set HH:MM:SS Day(1-31) Month(String) Year(YYYY)` - If not already in correct UTC Time)
33. Install associated Cisco Certificate with assistance of associated personnel.

### vManage

1. Log into the CLI
   - `vmanage login: admin`
   - `password: admin`
2. Set new password. (Recommend using a secure password, but for an already secure lab environment, a default `admin1` password should work). 
3. Select disk created for storage
4. 
	1. `COMPUTER_AND_DATA`
	2. `DATA`
	3. `COMPUTE`
Select persona for vManage \[1, 2, or 3]: 1
5. Are you sure? \[y/n]: y
6. Available storage devices:
	1) sdb
7. Select storage device to use: 1
8. would you like to format sdb? \[y/n]: y
9. Wait 5~ minutes
10. Log back into CLI:
   - `vmanage login: admin`
   - `password: admin1`
11. `vmanage# config`
12. `vmanage(config)# system`
13. `vmanage(config-system)# hostname vManage`
14. `vmanage(config-system)# system-ip 1.1.1.11`
15. `vmanage(config-system)# site-id 100`
16. `vmanage(config-system)# organization-name "RAPID-Laboratory`
17. `vmanage(config-system)# vbond 138.199.102.114`
18. `vmanage(config-system)# exit`
19. `vmanage(config)# security`
20. `vmanage(config-security)# control`
21. `vmanage(config-control)# protocol TLS`
22. `vmanage(config-control)# tls-port 23557`
23. `vmanage(config-control)# exit`
24. `vmanage(config-security)# exit`
25. `vmanage(config)# commit`
26. `vManage(config)# vpn 0`
27. `vManage(config-vpn-0)# interface eth0`
28. `vManage(config-interface-eth0)# ip address 10.10.0.42`
29. `vManage(config-interface-eth0)# no shutdown`
30. `vManage(config-interface-eth0)# tunnel-interface`
31. `vManage(config-tunnel-interface)# allow-service ssh`
32. `vManage(config-tunnel-interface)# allow-service netconf`
33. `vManage(config-tunnel-interface)# commit`
34. `vManage(config-tunnel-interface)# exit`
35. `vManage(config-interface-eth0)# exit`
36. `vManage(config-vpn-0)# ip route 0.0.0.0/0 10.10.0.1`
37. `vManage(config-vpn-0)# commit`
38. `vManage(config-vpn-0)# exit`
39. `vManage(config)# exit`
40. `vManage# show clock`
41. `vManage# clock set HH:MM:SS Day(1-31) Month(String) Year(YYYY)` - If not already in correct UTC Time)
42. Install associated Cisco Certificate with assistance of associated personnel.<div style="page-break-after: always"></div>

## Data Plane CLI Setup

### Edge Routers

#### Initial Bootstrap Configuration
**NOTE**: If you intended for 3 total network interfaces, ***SKIP*** steps 1 and 2.
1. In the case that you did not enable automatic startup in the previous `VM Deployment` section, power the virtual machine off.
2. Click on the `Edit` button in the window of your source edge VM, and add/remove network adapters as needed. After this is complete, power on the VM.
   ![[Add_Remove_Network_Adapters.png]]
   `Add network adapter` is shown at the top circled in green.
   To `Remove a network adapter`, click on the `x` button associated with the Network adapter to be deleted as denoted in a red circle above. 
   
4. Wait for the VM to start up. This should take roughly 5-10 minutes.
5. Once the VM looks to be up and running, click into the terminal of the VM (Unfortunately, but also fortunately, you'll know when your mouse is within the VM's console if it disappears) and attempt to enter configuration by pressing `Enter`. This should bring up a prompt. Type `yes` to this prompt.
6. Another prompt will follow the first one. Also type `yes` in response.
7. This will enter you into configuration mode. This configuration will be to set your passwords. As with general security practices, ensure these passwords are secure, note them down and store them in a secure space.
8. If the setup requests that you specify an interface to communicate with the management/control devices, enter `GigabitEthernetW`, with `W` replaced with your desired WAN facing interface. It may then ask you to provide the desired public IP and subnet mask, which if you are DHCP'ed an IP (for example, StarLink uses DHCP to allocate your public IP if using an enterprise level or higher StarLink), you can leave the defaults by just hitting enter. Otherwise, you may have to provide a static IP and the subnet mask of the interface you are on.
   For example:
   IP : 10.10.20.2
   Subnet Mask: 255.255.255.0
   This is the IP and Subnet Mask utilized for the Data Sink Edge Router in the COSMIAC-RAPID Lab Environment.
9. Afterwards, confirm the bootstrap configuration, and the router should prompt you with this message:
   `Press RETURN to get started!`
10. Once you see the above message, hit `Enter` on your keyboard. When following the next set of instructions, the commands will be in this structure:
   `hostname> command`
   or
   `hostname# command`
   Depending on the step your are on. Replace `command` with the respective command as shown below.
	1. `hostname> enable`
	2. (type in the ***FIRST*** of your three passwords here)
	3. `hostname# control enable`
	4. Hit enter on your keyboard to confirm.
	5. Type `no` to ensure you do not abort this process
	The above sequence is required to enable your router to be in `Controller Mode`, which is synonymous to `sdwan mode`
11. This will restart your VM. Wait for it to fully launch once more, and again this should take 5-10 minutes depending on hardware. The VM will likely complain using the error message: `Smart Lisencing has encountered an internal software error`. You can safely ignore this and hit `Enter` on your keyboard, and login to the router.
12. The credentials used to login will be:
    username: admin
    password: admin
	This should prompt you to set a new password and confirm it, once again following best security practices, use a strong password and note it down in a secure location.
13. Once logged in, you are ready to follow the router configuration portion of this setup. 

#### Router Configuration Section
This section will contain the commands in a list form that you have to type in order to configure the router accordingly. When following the next set of instructions, the commands will be in this structure:
`hostname> command`
or
`hostname# command`
Depending on the step your are on. Replace `command` with the respective command as shown below.

In addition, you will see interfaces named as `GigabitEthernetW` or `GigabitEthernetL` . The `W` and `L` in this naming scheme corresponds to the Network Adapter configured on VM Deployment, where `W` denotes a WAN Interface, and `L` denotes a LAN interface. In other words, Any connection on Network Adapter 1 will be synonymous with `GigabitEthernet1` in the respective edge router, but the nuance of the connection being WAN or LAN relies on your understanding of your network interfaces. 

A good rule of thumb to remember is that if you can access google from the interface, it is the WAN interface. If you use the interface to contact other devices within the current branch of your company, say a local printer in the building, it is likely a LAN interface.

Also, unless otherwise specified, use the last `L` or `W` value you were prompted to provide for the configurations, if you have multiple WAN or LAN interfaces.

Before engaging with these steps, on the edge router, run the following command:
`hostname# show ip int br`
This command will show the current interfaces that your edge router has. Note down all that start with `GigabitEthernet`, as this will come in handy later.
##### **Disable Console Logging**
1. `hostname# config-transaction`
2. `hostname(config)# no logging con`
3. `hostname(config)# commit` 
   
##### **Change Hostname**
4. `hostname(config)# hostname (insert desired hostname here)`
5. `hostname(config)# commit`
   
##### **System Configuration**
6. `hostname(config)# system`
7. `hostname(config-system)# system-ip 1.1.1.x`
   Replace x with a number in the inclusive range of 0-255. Refer to internal documentation for system IPs already in use. This system ip can realistically be anything within the x.x.x.x range, but this is too large of a subnet for our lab environment. 
   
8. `hostname(config-system)# site-id (insert id here)`
9. `hostname(config-system)# sp-organization-name "YOUR_ORG_NAME"`
	 **NOTE**: This has to be the same org name denoted in the control plane setup, as this will cause certificate mismatches if the org name is different. The `sp-organization-name` for the COSMIAC-RAPID Lab Environment was "RAPID-Laboratory".
   
10. `hostname(config-system)# organization-name "YOUR_ORG_NAME"` 
    **NOTE**: This has to be the same org name denoted in the control plane setup, as this will cause certificate mismatches if the org name is different. The `organization-name` for the COSMIAC-RAPID Lab Environment was "RAPID-Laboratory".

11. `hostname(config-system)# vbond (insert vBond public ip here) port 12346`
12. `hostname(config-system)# commit`

##### **SD WAN Router Configurations**
**NOTE**: Repeat steps 14-20 for as many WAN facing interfaces you have. This instruction set will also repeat some of this configuration later on, for specific SD WAN configurations for the interface, as certain technologies require more granular configurations (i.e. StarLink). This is a general, barebones default template you are able to follow if nothing was specified otherwise.
13. `hostname(config-system)# sdwan`
14. `hostname(config-sdwan)# interface GigabitEthernetW tunnel-interface` 
    **NOTE**: Replace `W` in `GigabitEthernetW` with a number corresponding to the Network Adapter of the current WAN facing interface you are configuring.

15. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
16. `hostname(config-tunnel-interface)# color (insert desired color here)`
    [Cisco SD WAN Color & Carrier Overview](https://www.networkacademy.io/ccie-enterprise/sdwan/tloc-color-and-carrier)

17. `hostname(config-tunnel-interface)# allow-service all`
18. `hostname(config-tunnel-interface)# exit`
19. `hostname(config-interface-GigabitEthernetW)# exit`
20. `hostname(config-sdwan)# exit`

##### **Interface Configurations**

###### **LAN Interface Configuration**
21. `hostname(config)# vrf definition 100`
22. `hostname(config-vrf)# rd 1:100`
23. `hostname(config-vrf)# address-family ipv4`
24. `hostname(config-ipv4)# route-target export 1:100`
25. `hostname(config-ipv4)# route-target import 1:100`
26. `hostname(config-ipv4)# exit`
27. `hostname(config-vrf)# address-family ipv6`
28. `hostname(config-ipv6)# exit`
29. `hostname(config-vrf)# exit`
30. `hostname(config)# ip arp proxy disable`
31. `hostname(config)# ip dhcp pool vrf-100-GigabitEthernetL`
    `GigabitEthernetx` is the LAN facing interface associated with the corresponding Network Adapter. In the COSMIAC-RAPID Lab Environment, `GigabitEthernet2` was used.

32. `hostname(dhcp-config)# vrf 100`
33. `hostname(dhcp-config)# lease 1 0 0`
34. `hostname(dhcp-config)# default-router 192.168.x.x`
    `192.168.x.x` subnet is often used in production environments as the private subnet. Other viable options would be the `10.x.x.x` and `172.16.x.x - 172.31.x.x` subnets. For example, the command for the Data Source Edge Router that the COSMIAC-RAPID lab environment utilized was:
    `hostname(dhcp-config)# default-router 192.168.52.1`
    Which notionally is the Default Gateway of the LAN Facing Interface.

35. `hostname(dhcp-config)# network 192.168.x.x 255.255.x.x`
    This is denoting the network that this DHCP Configuration is concerned with. Again, like stated above, this is a private network, so as long as the network follows the above in the RFC 1918 private space, the configuration should work. Note that this should be the network that the above default router is in. To continue the example above:
    `hostname(dhcp-config)# network 192.168.52.0 255.255.255.0`
    would be the associated command to instantiate the network.

36. `hostname(dhcp-config)# exit`
37. `hostname(config)# ip dhcp use vrf remote`
38. `hostname(config)# no ip ftp passive`
39. `hostname(config)# ip name-server 8.8.8.8`
40. `hostname(config)# ip name-server vrf 100 8.8.8.8`
41. `hostname(config)# ip scp server enable`
42. `hostname(config)# ip bootp server`
43. `hostname(config)# no ip source route`
44. `hostname(config)# no ip ssh bulk-mode` 
45. `hostname(config)# ip tcp RST-count 10 RST-window 5000` 
46. `hostname(config)# no ip http server` 
47. `hostname(config)# no ip http secure-server` 
48. `hostname(config)# ip http client source-interface GigabitEthernetL` 
49. `hostname(config)# ip nat pool natpool1 192.168.x.x 192.168.x.x prefix-length n` 
    This is the NAT Address pool for the LAN interface. This will specify how large your private subnet is. For example, this is the command that was used to set this up on the Data Source Edge Router:
    `hostname(config)# ip nat pool natpool1 192.168.52.1 192.168.52.255 prefix length 24` 

50. `hostname(config)# ip nat inside source list global-list pool natpool1 vrf 100 match-in-vrf overload` 
51. `hostname(config)# ip nat inside source list nat-dia-vpn-hop-access-list interface GigabitEthernetW overload`
    Repeat this command for every WAN facing interface you have, with each corresponding `W` being associated with your WAN facing interfaces. For example:
    `hostname(config)# ip nat inside source list nat-dia-vpn-hop-access-list interface GigabitEthernet1 overload`

52. `hostname(config)# ip nat translation tcp-timeout 3600` 
53. `hostname(config)# ip nat translation udp-timeout 60`
54. `hostname(config)# ip nat settings central-policy`
55. `hostname(config)# ipv6 unicast-routing`
56. `hostname(config)# ipv6 cef load-sharing algorithm include-ports source destination`
57. `hostname(config)# ipv6 rip vrf-mode enable`
58. `hostname(config)# cdp run`
59. `hostname(config)# policy-map shape_GigabitEthernetW` - This `W` should correspond to your WAN facing interface.
60. `hostname(config-pmap)# class class-default`
61. `hostname(config-pmap-c)# shape average 12000000`
62. `hostname(config-pmap-c)# exit`
63. `hostname(config-pmap)# exit`
64. `hostname(config)# interface GigabitEthernetL` - This `L` should correspond to your LAN facing interface.
65. `hostname(config-if)# no shutdown`
66. `hostname(config-if)# arp timeout 1200`
67. `hostname(config-if)# vrf forwarding 100`
68. `hostname(config-if)# ip address 192.168.x.x 255.255.x.x`
     For Example:
     `hostname(config-if)# ip address 192.168.52.1 255.255.255.0`
69. `hostname(config-if)# no ip redirects`
70. `hostname(config-if)# ip mtu 1500`
71. `hostname(config-if)# load-interval 30`
72. `hostname(config-if)# mtu 1500`
73. `hostname(config-if)# negotiation auto`
74. `hostname(config-if)# exit`

###### **StarLink Interface Configuration**
***Skip this set of instructions if setting up the Data Sink Edge Router.***
This is the interface linked to the connection for StarLink. Do note that *only one router at a time* can hold the StarLink's allocated public ip. Also depending on how you configured the deployment for the edge router, this connection may not match the one that is Utilized by the COSMIAC-RAPID Lab Environment. 

This interface was used with the Data Source Edge Router in the COSMIAC-RAPID Lab Environment. In an ideal environment, there would also be a StarLink connection on the Data Sink Edge Router, but as stated previously, the COSMIAC-RAPID Lab only had access to one StarLink that allowed for a public IP Allocation.
75. `hostname(config)# interface GigabitEthernetW` 
    Replace `W` with the corresponding GigabitEthernet interface used for your StarLink WAN facing Network Adapter. 
    The corresponding interface was `GigabitEthernet1` for the COSMIAC-RAPID Lab Environment on the Data Source Edge Router. 

76. `hostname(config-if)# no shutdown`
77. `hostname(config-if)# ip address dhcp`
78. `hostname(config-if)# arp timeout 1200`
79. `hostname(config-if)# ip dhcp client client-id GigabitEthernetW`
80. `hostname(config-if)# no ip redirects`
81. `hostname(config-if)# ip dhcp client default-router distance 1`
82. `hostname(config-if)# ip nat outside`
83. `hostname(config-if)# ip mtu 1500`
84. `hostname(config-if)# load-interval 30`
85. `hostname(config-if)# mtu 1500`
86. `hostname(config-if)# negotiation auto`
87. `hostname(config-if)# service-policy output shape_GigabitEthernetW`
88. `hostname(config-if)# exit`
89. `hostname(config)# interface Tunnel0`
90. `hostname(config-if)# no shutdown`
91. `hostname(config-if)# ip unnumbered GigabitEthernetW`
92. `hostname(config-if)# no ip redirects`
93. `hostname(config-if)# ipv6 unnumbered GigabitEthernetW`
94. `hostname(config-if)# no ipv6 redirects`
95. `hostname(config-if)# cts manual`
96. `hostname(config-if-cts-manual)# no propagate sgt`
97. `hostname(config-if-cts-manual)# exit`
98. `hostname(config-if)# tunnel source GigabitEthernetW`
99. `hostname(config-if)# tunnel mode sdwan`
100. `hostname(config-if)# exit`
101. `hostname(config)# sdwan`
102. `hostname(config-sdwan)# interface GigabitEthernetW tunnel-interface`
103. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
104. `hostname(config-tunnel-interface)# color blue`
105. `hostname(config-tunnel-interface)# allow-service all`
106. `hostname(config-tunnel-interface)# no border`
107. `hostname(config-tunnel-interface)# low-bandwidth-link`
108. `hostname(config-tunnel-interface)# no vBond-as-stun-server`
109. `hostname(config-tunnel-interface)# vmanage-connection-preference 5`
110. `hostname(config-tunnel-interface)# port-hop`
111. `hostname(config-tunnel-interface)# carrier default`
112. `hostname(config-tunnel-interface)# nat-refresh-interval 5`
113. `hostname(config-tunnel-interface)# hello-interval 6000`
114. `hostname(config-tunnel-interface)# hello-tolerance 600`
115. `hostname(config-tunnel-interface)# exit`
116. `hostname(config-interface-GigabitEthernetW)# bandwidth-downstream 250000`
117. `hostname(config-interface-GigabitEthernetW)# qos-adaptive`
118. `hostname(config-qos-adaptive)# downstream 200000`
119. `hostname(config-qos-adaptive)# downstream range 80000 250000`
120. `hostname(config-qos-adaptive)# upstream 12000`
121. `hostname(config-qos-adaptive)# upstream range 2000 20000`
122. `hostname(config-qos-adaptive)# exit`
123. `hostname(config-interface-GigabitEthernetW)# exit`
124. `hostname(config-sdwan)# exit`
125. `hostname(config)# commit`

###### **EC2 Interface Configuration**
***Skip this set of instructions if setting up the Data Source Edge Router.***  
This is the interface linked to the connection for the AWS EC2 Instance. Note that you can have many routers behind this EC2 instance, as it acts as a NAT Gateway/Router that the edge router would likely be behind in a commercial SD WAN Deployment. Also depending on how you configured the deployment for the edge router, this connection may not match the one that is Utilized by the COSMIAC-RAPID Lab Environment.

This interface was used with the Data Sink Edge Router in the COSMIAC-RAPID Lab Environment. 
126. `hostname(config)# interface GigabitEthernetW`
     Replace `W` with the corresponding GigabitEthernet interface used for the Network Adapter that connects to the EC2 Instance. 
     The corresponding interface was `GigabitEthernet1` for the COSMIAC-RAPID Lab Environment on the Data Sink Edge Router.

127. `hostname(config-if)# no shutdown`
128. `hostname(config-if)# ip address x.x.x.x m.m.m.m`
     Here, we are setting a static IP address that is on the subnet that your EC2 is handling routing on. For the COSMIAC-RAPID Lab Environment, the command was the following:
     `hostname(config-if)# ip address 10.10.20.2 255.255.255.0`
     This command entails that on interface `GigabitEthernetW`, the router gets the IP `10.10.20.2` and is on the `10.10.20.1/24` subnet due to the `255.255.255.0` subnet mask.
     **NOTE**: DO NOT USE `10.10.20.2` IF IN THE COSMIAC-RAPID LAB ENVIRONMENT. Use anything between `10.10.20.3` - `10.10.20.254` inclusive.

129. `hostname(config-if)# arp timeout 1200`
130. `hostname(config-if)# no ip redirects`
131. `hostname(config-if)# ip nat outside`
132. `hostname(config-if)# ip mtu 1500`
133. `hostname(config-if)# load-interval 30`
134. `hostname(config-if)# mtu 1500`
135. `hostname(config-if)# negotiation auto`
136. `hostname(config-if)# service-policy output shape_GigabitEthernetW`
137. `hostname(config-if)# exit`
138. `hostname(config)# interface Tunnel0`
139. `hostname(config-if)# no shutdown`
140. `hostname(config-if)# ip unnumbered GigabitEthernetW`
141. `hostname(config-if)# no ip redirects`
142. `hostname(config-if)# ipv6 unnumbered GigabitEthernetW`
143. `hostname(config-if)# no ipv6 redirects`
144. `hostname(config-if)# tunnel source GigabitEthernetW`
145. `hostname(config-if)# tunnel mode sdwan`
146. `hostname(config-if)# exit`
147. `hostname(config)# sdwan`
148. `hostname(config-sdwan)# interface GigabitEthernetW tunnel-interface`
149. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
150. `hostname(config-tunnel-interface)# color blue`
151. `hostname(config-tunnel-interface)# allow-service all`
152. `hostname(config-tunnel-interface)# no border`
153. `hostname(config-tunnel-interface)# no vbond-as-stun-server`
154. `hostname(config-tunnel-interface)# vmanage-connection-preference 5`
155. `hostname(config-tunnel-interface)# port-hop`
156. `hostname(config-tunnel-interface)# carrier default`
157. `hostname(config-tunnel-interface)# nat-refresh-interval 5`
158. `hostname(config-tunnel-interface)# exit`
159. `hostname(config-interface-GigabitEthernetW)# exit`
160. `hostname(config-sdwan)# exit
161. `hostname(config)# commit`

##### **Routing**

162. `hostname(config)# ip route 0.0.0.0 0.0.0.0 (insert the default gateway IP used on GigabitEthernetW)`
    This should be the default gateway of the current WAN facing interface. For example, our StarLink WAN interface uses the default gateway IP of `143.105.158.1`

##### **Control Plane Onboarding**
163. `hostname# show clock` - Clock will likely be in UTC format. If time is off, adjust using next steps. Otherwise, skip step 168.
164. `hostname# clock set hh:mm:ss Day Month Year` - Where `hh` is the hour (0-23), `mm` is minutes (0-59), `ss` is seconds (0-59), `Day` is the day of the month (0-31), `Month` is the full name of the month, and `Year` is the current year in the format of `YYYY`.
165. Access the vManage GUI through a web browser on a device connected to the Control Plane network. For example, in the COSMIAC RAPID Lab Environment, a Laptop was physically connected to the network in which the control plane resided, so vManage was accessed through a Firefox browser through the IP of vManage on that network (Which was 10.10.0.42 for the COSMIAC-RAPID Lab Environment
166. Hover over `Configuration` on the left side bar, and click on `WAN Edges` under the `Devices` heading. This will take you to the list of WAN Edges allocated to your Cisco SD WAN account.
167. Generate a bootstrap config by clicking on the `...` under the `Actions` column on an available WAN Edge, and click on `Generate Bootstrap Configuration`.
168. Ensure that for the configuration, `Cloud-Init` is the selected option. Click on the `OK` button at the bottom right of this window.
169. Note the `uuid` and the `otp` that the bootstrap configuration generates. This will be ***VERY IMPORTANT***.
170. Go back to your ESXi GUI and navigate back to your current edge router. Log back into the router if necessary. Then enter the command in the following step
171. `hostname# request platform software sdwan vedge_cloud activate chassis-number (uuid) token (otp)`
    For example:
    `hostname# request platform software sdwan vedge_cloud activate chassis-number C8K-8943C275-7B78-1E88-B761-4F8045EB38A1 token e60cbdd624774ef7bd45fab8bc40c5ac`
172. The prior command is lengthy, so **ensure** that you have the ***correct values WITHOUT any typos***. This will onboard the edge to the SD WAN fabric.
173. Verify that your edge router has complete control plane connectivity using the following command:
     `hostname# show sdwan control connections`

#### Client Configuration Section

The Clients will be tailored to the needs of your testing/Lab Environment. You will need to structure your scripts for `mgen`, `iperf3`, and `traceroute` as your network analysis dictates. For the test that this Lab Environment was initially structured for, we conduct a TCP and UDP `mgen` test, sending data from Source to Sink, 1 packet per second, with a size of 1024 bytes. `iperf3` allows for bandwidth and throughput analysis, and `traceroute` allows us to see the route we take, ensuring that the tunnel is used. The route analyzed by traceroute is obfuscated, so it is a matter of ensuring that the correct gateways are traversed at the start and end points of the tunnel.