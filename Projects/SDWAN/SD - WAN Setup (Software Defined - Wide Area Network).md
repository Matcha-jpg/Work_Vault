This document contains a high level overview of the Lab Environment and the systems that it consists of, as well as detailed, step-by-step instruction set to replicate the COSMIAC Lab environment, granted with a few exceptions like Public IP allocations. Following the instruction set should net you a near-exact replica of the COSMIAC Lab environment.

# High-Level Overview

### Palo Alto Firewall

Since the lab environment will be utilizing commercial internet services, a firewall is necessary to maintain the security and integrity of the COSMIAC Lab Environment. The Palo Alto Firewall utilized in the Lab Environment is extremely customizable. This allows for very granular and verbose configuration, but this also leads to a more complex process for a proper, secure firewall configuration. The Graphical User Interface (GUI) of the Palo Alto Firewall is somewhat intuitive, but is more user-friendly to those who are proficient in securing network infrastructures.

As is stated above, there is a variety of customizable settings within the Palo Alto firewall. The most important customizations that need to be made for the lab environment is within the Network Address Translation (NAT) and Security Policies, and in addition, Services. Services are just the types of ports and protocols you are allowing through NAT Policies, while the Security Policies handle how traffic moves within the intra-zone and inter-zone interfaces.

There are more items that you can configure for more granularity, but those settings are not of concern in our Lab deployment. In addition, this firewall can allow customers and Subject Matter Experts (SMEs) alike the ability to interact with the environment remotely for Lab demonstrations/assistance with infrastructure setup if necessary.

### ESXi Hypervisor

ESXi is a Bare-metal Hypervisor from VMWare that overhauls a server's OS (in our case, a Dell Server), which allows for the efficient provisioning of server hardware and resources to host and deploy Virtual Machines (VMs) on. The ESXi hypervisor is what will be utilized to deploy all associated virtual devices necessary for the COSMIAC lab environment.

The GUI of ESXi is relatively intuitive and user friendly, but it having background knowledge in hosting virtual environments helps with navigation and deployment immensely. There are a lot of tools at the operator's disposal, with the ability to configure many aspects of the deployment. 

There are configurations for Virtual Machines, Storage, and Networking. Virtual Machines is where you will be deploying and configuring your devices. Storage is where you store any .iso, .ova files that your VMs will use, as well as hosting/representing any deployed VM and their contents. Networking is where you will configure your virtual Network infrastructure, creating vSwitches, port groups, and the such for your VMs to connect to via a Network Adapter on the VM.

The area of configuration we are concerned with is primarily the Network setup. It is of utmost importance to configure the network settings correctly in ESXi, as the end goal is to replicate a Commercial/Business deployment of an SD WAN Network infrastructure. With our Dell Server, we have a limited amount of physical Network Interface Cards (NICs) with which to facilitate internet connections, so proper allocation is necessary. Each vSwitch can be attached to multiple physical NICs, which are denoted as uplinks/vmnics in ESXi, and this allows for redundancy in normal deployments. However, the vSwitches in our setup will only have one NIC/uplink assigned to it.

The system will be utilizing 1 NIC for connection to the Palo Alto Firewall, 1 NIC for the StarLink Modem connection, and 1 NIC for the EC2 Connection, leaving 3 NICs unused. There are plans to enhance the lab environment with Satellite Communication Internet Service Providers (SATCOM ISPs) other than SpaceX StarLink, such as ViaSat and Hughes-Net, but these services will not be utilized in our primary tests. They will however serve to dissect and learn more information about the functionality and limitations of the SD WAN Network infrastructure, which will be noted at a later date in a revision/different document.

### Control Plane

The Cisco Control Plane consists of 3 devices: 
* vSmart (also known as the Controller)
* vManage (also known as the Manager)
* vBond (also known as the Validator). 
You will need to generate and install certificates for these devices, and may need specific personnel to do so. The important controller to get up and running is vBond, as it acts as the initial gateway onto your SD-WAN fabric. It validates and confirms that the devices establishing a connection to it before letting said device join into the fabric. 

In this setup, vBond requires a publicly routable ip address. This means that it can be behind a NAT (Network Address Translation) Device, but it has to be configured to have a static, 1:1 NAT with the public ip of the NAT device (in our case, the PaloAlto firewall). 

The rest of the control plane also resides behind this NAT device, and that internal communication has to be handled via NAT Hairpinning/Loopback. This allows the other control plane devices to have the correct routes mapped to the device for complete control plane connectivity. 

The **complete control plane connectivity** is important, as vSmart handles advertising the correct network communication policies between distant edges, and vManage allows for the viewing and analysis of associated systems, as well as being the central point of management within the fabric.

These three devices are all deployed within the Dell Server running as our ESXi Bare-metal Hypervisor.

### Edge Devices (Data Plane)

The Edge devices will comprise your Data Plane (Where your Data Communications happen). Your edge devices are your Edge Routers/Distant Edge Devices within your network infrastructure. The COSMIAC SD-WAN StarLink setup consists of two Edge Routers (Cisco Catalyst 8000v cEdges). Like the Control Plane devices, these Edge routers will also be deployed within our ESXi Hypervisor. 

The initial setup for both edges is quite simple, though to configure the proper interface for StarLink, there are additional configurations you must make either at the CLI or Feature Template level of configuration. The Feature Template configuration is relatively straightforward, and there is a Cisco Community Blogpost that exists to delineate the feature template creation, but the same actions can be done through the CLI.

The edges also need to be programmed with the public ip of vBond, and the associated udp port 12346 for DTLS (Datagram Transport Layer Security) protocol. This allows them to begin the system onboarding process to get the edge to join onto the SD-WAN Fabric. The first step is the initial bootstrap CLI configuration. This process involves accessing vManage's GUI through http, and generating a bootstrap configuration for a WAN edge there. This step will be outlined later in this documentation

The two edges will require at minimum 2 Network Adapters configured on the ESXi host. Network Adapter 1/GigabitEthernet1 (As denoted on a cEdge/vEdge) will be our WAN (Wide Area Network) (public facing/untrust zone) interface, and Network Adapter 2/GigabitEthernet2 will be our LAN (Local Area Network) (internal/trusted zone) interface.  The LAN interface will double as a DHCP server, allowing clients connected to the same port group to be leased a private IP address within the associated LAN subnet.

After the edges are configured and have complete control plane connectivity, tunnels between edges will be created, and connectivity between sites should be established. The *Software Defined* portion of SD WAN handles the tunnel creation through the advertisement of routing and policies from the vSmart (Controller) to the Distant Edge Devices.

### Quirks With Our Lab Setup

There were multiple problems with the initial setup of this lab environment, so here are some key things to point out so that future deployments do not get stuck in the same place.

* #### Using EC2 as a NAT Gateway/Router
	This Lab environment utilizes an EC2 Instance not as a device that directs traffic through its interfaces from one place to another, but as another router (NOT EDGE ROUTER) that acts as our NAT device that our Edge router is connected to. Edge routers being behind NAT devices in commercial deployments is not uncommon, and a valid way to showcase layered Network infrastructure compatibility with SD WAN deployments, but there are important NAT caveats to pay attention to and avoid as they will inherently fail to work in an SD WAN environment. That list can be found here < **PLACE A LINK TO THE THING OR PROVIDE SNIPPET**

* #### Edge Router Behind NAT Device
	This is an important issue to outline as this deals with what IPs get advertised in the Edge router's TLOCs (Transport Locators). If directly connecting to the Control Plane on a LAN interface, the Edge Router will advertise the other interfaces it has on the Transport VPN using the configured IP on the device.
	
	This means that, if the router is behind a NAT Device, then the Edge Router will advertise its ***PRIVATE*** IP to the SD WAN Fabric. This is an issue, as other routers will try to establish a tunnel to this ***PRIVATE*** IP through the ***PUBLIC*** internet. This is inherently impossible in normal networking. The way that this is subverted is by connecting to vBond through the public internet, as vBond has an embedded STUN (Simple Traversal Utilites for NAT) Server, and can thus help Edge routers/Control Plane devices behind a NAT Device become *NAT Aware*. This is the case for our lab setup, as our Sink Edge Router is connected behind the EC2 Instance that acts as this NAT Device.

* #### NOT An Air-Gapped Environment
	Initially, this lab was based off of an air-gapped system from a prior preliminary test on SD WAN infrastructure. In addition, there was confusion as to how the EC2 Instance fits into the picture in our infrastructure. Once again, the EC2 acts as a NAT Device, and while Network Emulation can likely be appended to the operational features of the EC2, it is not an inherent part of our environment. That is because we are NOT simulating network traffic, as we are dealing with realistic network traffic metrics as we are utilizing Commercial Internet Service Providers for our internet connectivity.
	
	The ISPs in question are SpaceX StarLink, and AWS, as the EC2 doubles as a gateway to the public internet through the AWS Fabric. The Source Edge is connected to our StarLink Modem, while the Sink Edge is connected to the EC2 Nat Device.
	
	The Non-Air-Gapped environment nuance is also important because, as stated earlier, we are utilizing Commercial Internet Services to construct our SD WAN fabric, leading us to a deployment that is much closer to a Commercial/Business deployment of Cisco SD WAN infrastructure. This means that the control plane has to be publicly routable to the internet. There are security risks associated with this, and to address those security risks, the Palo Alto Firewall is our answer, only allowing traffic from associated IPs and Ports to access the control plane.

* #### Control Plane all behind ONE NAT Device with only ONE Public IP
	This is a particular quick in our setup that caused much confusion and frustration, as since we only had a **singular public IP** to work with, that made communication to the control plane much more complex than that of a normal commercial/business deployment. Normal, Cisco suggested deployment methods mark the simplest deployment as giving each Control Plane device a public IP, but that can be expensive depending on the budget, or complications with Company-Owned IP allocations. There are deployments around this, although less suggested due to the complexity they require.
	
	One such issue that arises is if they are all ALSO behind the SAME NAT Device, such as in our lab environment. This necessitates the use of what is called NAT Hairpinning/Loopback, which in simpler terms is just a way for devices/clients of the NAT Device to utilize the NAT Device's public IP to communicate with others in the same/different LAN. This is the method that we utilized in our setup. Other solutions include setting up a DMZ (De-Materialized Zone) For vSmart and vManage or just vBond, but this has yet to be explored as the previous NAT Hairpinning/Loopback solution worked to solve the problem in our case.
	
	NAT Hairpinning/Loopback is considered complex, as you need to have a good understanding of how firewalls and routing works to accomplish configuring the correct security rules and NAT policies to put in place, let alone knowing if your particular router/firewall is capable of this functionality in the first place.

* #### DTLS And Interesting Firewall Issues
	There was a problem in establishing complete control plane connectivity on the edge router side due to some granular firewall issues. In no place did we configure on the firewall to drop any specific packets from our trusted sources, yet any DTLS Packets that were attempting to connect to the vSmart or vManage devices behind the Palo Alto Firewall were dropped. 
	
	Upon granular analysis of network traffic through the usage of a router with the capability of running `tcpdump`, we were able to identify that DTLSv1.0 packets originating from the edge devices were dropped, but DTLSv1.2 packets made it through the firewall to the vManage and vSmart devices. With further inspection, we speculate that the firewall was mis-classifying our Client Hello DTLS packets (The type of packet that is utilized to establish a connection via handshake) as using a different protocol, and thus not properly identified and routed to the correct destination.
	
	The solution for this was to force vSmart and vManage to use TLS for control plane connections over DTLS, on two distinct ports. This allowed for the devices to still understand their public-private IP and port mappings behind a NAT Device, and establish connections to the distant edge devices.
	
	Again, this is purely speculation on our part, as there could be a potential bug with the version of the SD WAN Software that the lab environment uses, and thus the issue could be a SD WAN Software issue rather than a Firewall behavior issue. We are focusing efforts forward with this solution in mind, as with the current equipment, this is our path of least resistance.

With all this said, the most important point here is that the Control Plane ***NEEDS*** to be publicly routable for this setup to work, especially if one or more edge routers/control plane devices are behind a NAT Device.


# Step-by-Step Instructions

### Palo Alto Firewall


### ESXi Hypervisor

#### Network Configuration

##### Physical NICs
##### vSwitches

##### Port Groups

#### VM Deployment

##### Control Plane Devices

##### Edge Devices

##### Client Devices

### Control Plane

#### vBond

#### vManage

#### vSmart

### Edge Devices

#### StarLink Source Edge

#### EC2 Sink Edge

### Clients
