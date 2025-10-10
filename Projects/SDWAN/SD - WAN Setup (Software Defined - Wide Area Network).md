This document contains a high level overview of the Lab Environment and the systems that it consists of, as well as detailed, step-by-step instruction set to replicate the COSMIAC Lab environment, granted with a few exceptions like Public IP allocations. Following the instruction set should net you a near-exact replica of the COSMIAC Lab environment.
# High-Level Overview

## Palo Alto Firewall

Since the lab environment will be utilizing commercial internet services, a firewall is necessary to maintain the security and integrity of the COSMIAC Lab Environment. The Palo Alto Firewall utilized in the Lab Environment is extremely customizable. This allows for very granular and verbose configuration, but this also leads to a more complex process for a proper, secure firewall configuration. The Graphical User Interface (GUI) of the Palo Alto Firewall is somewhat intuitive, but is more user-friendly to those who are proficient in securing network infrastructures.

As is stated above, there is a variety of customizable settings within the Palo Alto firewall. The most important customizations that need to be made for the lab environment is within the Network Address Translation (NAT) and Security Policies, and in addition, Services. Services are just the types of ports and protocols you are allowing through NAT Policies, while the Security Policies handle how traffic moves within the intra-zone and inter-zone interfaces.

There are more items that you can configure for more granularity, but those settings are not of concern in our Lab deployment. In addition, this firewall can allow customers and Subject Matter Experts (SMEs) alike the ability to interact with the environment remotely for Lab demonstrations/assistance with infrastructure setup if necessary.

## ESXi Hypervisor

ESXi is a Bare-metal Hypervisor from VMWare that overhauls a server's OS (in our case, a Dell Server), which allows for the efficient provisioning of server hardware and resources to host and deploy Virtual Machines (VMs) on. The ESXi hypervisor is what will be utilized to deploy all associated virtual devices necessary for the COSMIAC lab environment.

The GUI of ESXi is relatively intuitive and user friendly, but it having background knowledge in hosting virtual environments helps with navigation and deployment immensely. There are a lot of tools at the operator's disposal, with the ability to configure many aspects of the deployment. 

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

The two edges will require at minimum 2 Network Adapters configured on the ESXi host. Network Adapter 1/GigabitEthernet1 (As denoted on a cEdge/vEdge) will be our WAN (Wide Area Network) (public facing/untrust zone) interface, and Network Adapter 2/GigabitEthernet2 will be our LAN (Local Area Network) (internal/trusted zone) interface.  The LAN interface will double as a DHCP server, allowing clients connected to the same port group to be leased a private IP address within the associated LAN subnet.

After the edges are configured and have complete control plane connectivity, tunnels between edges will be created, and connectivity between sites should be established. The *Software Defined* portion of SD WAN handles the tunnel creation through the advertisement of routing and policies from the vSmart (Controller) to the Distant Edge Devices.

## Quirks With Our Lab Setup

There were multiple problems with the initial setup of this lab environment, so here are some key things to point out so that future deployments do not get stuck in the same place.

* ### Using EC2 as a NAT Gateway/Router
	This Lab environment utilizes an EC2 Instance not as a device that directs traffic through its interfaces from one place to another, but as another router (NOT EDGE ROUTER) that acts as our NAT device that our Edge router is connected to. Edge routers being behind NAT devices in commercial deployments is not uncommon, and a valid way to showcase layered Network infrastructure compatibility with SD WAN deployments, but there are important NAT caveats to pay attention to and avoid as they will inherently fail to work in an SD WAN environment. That list is delineated in this image below. ![[NAT-Compatibility.png]]

* ### Edge Router Behind NAT Device
	This is an important issue to outline as this deals with what IPs get advertised in the Edge router's TLOCs (Transport Locators). If directly connecting to the Control Plane on a LAN interface, the Edge Router will advertise the other interfaces it has on the Transport VPN using the configured IP on the device.
	
	This means that, if the router is behind a NAT Device, then the Edge Router will advertise its ***PRIVATE*** IP to the SD WAN Fabric. This is an issue, as other routers will try to establish a tunnel to this ***PRIVATE*** IP through the ***PUBLIC*** internet. This is inherently impossible in normal networking. The way that this is subverted is by connecting to vBond through the public internet, as vBond has an embedded STUN (Simple Traversal Utilites for NAT) Server, and can thus help Edge routers/Control Plane devices behind a NAT Device become *NAT Aware*. This is the case for our lab setup, as our Sink Edge Router is connected behind the EC2 Instance that acts as this NAT Device.

* ### NOT An Air-Gapped Environment
	Initially, this lab was based off of an air-gapped system from a prior preliminary test on SD WAN infrastructure. In addition, there was confusion as to how the EC2 Instance fits into the picture in our infrastructure. Once again, the EC2 acts as a NAT Device, and while Network Emulation can likely be appended to the operational features of the EC2, it is not an inherent part of our environment. That is because we are NOT simulating network traffic, as we are dealing with realistic network traffic metrics as we are utilizing Commercial Internet Service Providers for our internet connectivity.
	
	The ISPs in question are SpaceX StarLink, and AWS, as the EC2 doubles as a gateway to the public internet through the AWS Fabric. The Source Edge is connected to our StarLink Modem, while the Sink Edge is connected to the EC2 Nat Device.
	
	The Non-Air-Gapped environment nuance is also important because, as stated earlier, we are utilizing Commercial Internet Services to construct our SD WAN fabric, leading us to a deployment that is much closer to a Commercial/Business deployment of Cisco SD WAN infrastructure. This means that the control plane has to be publicly routable to the internet. There are security risks associated with this, and to address those security risks, the Palo Alto Firewall is our answer, only allowing traffic from associated IPs and Ports to access the control plane.

* ### Control Plane all behind ONE NAT Device with only ONE Public IP
	This is a particular quick in our setup that caused much confusion and frustration, as since we only had a **singular public IP** to work with, that made communication to the control plane much more complex than that of a normal commercial/business deployment. Normal, Cisco suggested deployment methods mark the simplest deployment as giving each Control Plane device a public IP, but that can be expensive depending on the budget, or complications with Company-Owned IP allocations. There are deployments around this, although less suggested due to the complexity they require.
	
	One such issue that arises is if they are all ALSO behind the SAME NAT Device, such as in our lab environment. This necessitates the use of what is called NAT Hairpinning/Loopback, which in simpler terms is just a way for devices/clients of the NAT Device to utilize the NAT Device's public IP to communicate with others in the same/different LAN. This is the method that we utilized in our setup. Other solutions include setting up a DMZ (De-Materialized Zone) For vSmart and vManage or just vBond, but this has yet to be explored as the previous NAT Hairpinning/Loopback solution worked to solve the problem in our case.
	
	NAT Hairpinning/Loopback is considered complex, as you need to have a good understanding of how firewalls and routing works to accomplish configuring the correct security rules and NAT policies to put in place, let alone knowing if your particular router/firewall is capable of this functionality in the first place.

* ### DTLS And Interesting Firewall Issues
	There was a problem in establishing complete control plane connectivity on the edge router side due to some granular firewall issues. In no place did we configure on the firewall to drop any specific packets from our trusted sources, yet any DTLS Packets that were attempting to connect to the vSmart or vManage devices behind the Palo Alto Firewall were dropped. 
	
	Upon granular analysis of network traffic through the usage of a router with the capability of running `tcpdump`, we were able to identify that DTLSv1.0 packets originating from the edge devices were dropped, but DTLSv1.2 packets made it through the firewall to the vManage and vSmart devices. With further inspection, we speculate that the firewall was mis-classifying our Client Hello DTLS packets (The type of packet that is utilized to establish a connection via handshake) as using a different protocol, and thus not properly identified and routed to the correct destination.
	
	The solution for this was to force vSmart and vManage to use TLS for control plane connections over DTLS, on two distinct ports. This allowed for the devices to still understand their public-private IP and port mappings behind a NAT Device, and establish connections to the distant edge devices.
	
	Again, this is purely speculation on our part, as there could be a potential bug with the version of the SD WAN Software that the lab environment uses, and thus the issue could be a SD WAN Software issue rather than a Firewall behavior issue. We are focusing efforts forward with this solution in mind, as with the current equipment, this is our path of least resistance.

With all this said, the most important point here is that the Control Plane ***NEEDS*** to be publicly routable for this setup to work, especially if one or more edge routers/control plane devices are behind a NAT Device.


# Step-by-Step Instructions

These instructions will only outline the software configurations of the setup. This section of this document assumes that all hardware has been properly set up, so no steps delineating anything hardware-related will be discussed.

This instruction set will also be separated into sections relating to the specific component(s) the set of instructions will be concerned with. The three categories are as follows: ESXi, Palo Alto Firewall, Control Plane Devices, and Data Plane Devices, with the Control and Data Plane Devices under the umbrella of "VM Deployment".

Although the set of instructions are separated by component, following this instruction set top down should ensure full system functionality, as long as the differences in lab setups are handled accordingly.

## ESXi Hypervisor

Connect to the host machine via its DHCP Ip given to it by the Palo Alto Firewall. In our setup, this IP was `10.10.0.2`.
### Network Configuration

This configuration is going to be done under the `Networking` tab of the ESXi host machine. Click on the `Networking` Tab on the far left Navigation panel on the ESXi host GUI to access the Networking GUI and the necessary Network Configuration tools for the ESXi Host Machine.
#### Physical NICs

In the Networking GUI Window, click on the `Physical NICs` Tab. This will display the available physical NICs on the server. The Dell Server utilized in the COSMIAC/RAPID Lab Environment has 6 physical NICs immediately available for use. The list of physical NICs are labeled `vmnic0-vmnic5` respectively.

These physical NICs will be your interfaces for internet communication, and the associated hardware connections to your commercial internet services will be accessed utilizing these vmnics. These vmnics will be attached to your switches in the following sections as your uplinks.
#### vSwitches

In the Networking GUI Window, click on the `Virtual Switches` Tab. This will bring up the configuration menu of your vSwitches on the ESXi Host Machine. Most of the configuration settings of the vSwitches will be left at a default. 

The COSMIAC/RAPID Lab environment utilizes three (3) total vSwitches. Follow an intuitive or set naming scheme for these switches, so that it is understood what each switch is being utilized for/what uplinks are connected to it. For simplicity, this instruction set will refer to the three (3) vSwitches as `vSwitch0`, `vSwitch1`, and `vSwitch2`.

Since the settings for each vSwitch will be kept at default, follow the steps below to set up the vSwitches. 

1. Click on `Add standard virtual switch`
2. Give your vSwitch a name. These are the names we alocated to the vSwitches in the COSMIAC/RAPID Lab Environment: PA-Switch, EC2-Switch, SatCom-WAN
3. Click on the dropdown box for `Uplink 1` and choose the vmnic that is most appropriate for the switch. (e.g. If `vmnic1` is your EC2 Fiber connection to the AWS Outpost, then you would associate this vmnic with the EC2-Switch labeled above) 
   **NOTE**: You can have multiple uplinks, and this is normally advised/encouraged in the case of internet outages. In the case of the SatCom-WAN Switch labeled above, you may have more than one uplink configured, but these will have to be VLAN'ed carefully to ensure the correct devices get the correct SatCom interface(s).
4. Click on the `Add` button at the bottom of the `Add standard virtual switch` Window.
#### Port Groups

In the Networking GUI Window, click on the `Port Groups` tab. This will bring up the configuration menu of the port groups that will be attached to your vSwitches. These port groups are essentially providing your Network infrastructure with segmentation, dictating what policies are served to each group of devices connected to the switch in that particular port group. Port Groups can also be used for VLAN Tagging, in order to properly VLAN Devices/interfaces correctly with the VLAN settings of the switch or other networking device that the ESXi is/may be attached to.

The COSMIAC/RAPID Lab Environment utilizes seven (7) port groups, which will be allocated to one of the above vSwitches from the earlier section. This port group section will delineate all the configurations for the seven (7) port groups utilized by the Lab Environment.

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

##### StarLink-PG

This port group will be used to service the WAN facing interface on the Source Edge in the Lab Environment. This will be the primary source of data-flow, as the overarching goal of the system is to run Network performance analyses on commercial SATCOM internet service providers. The StarLink service is the first of three to be integrated with our Environment, and will be the main point of focus for this series of Network performance tests.

1. Click on the `Add port group` button.
2. Name the new port group `StarLink-PG`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 200. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID due to the hardware configuration of the Lab environment.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
##### ViaSat-PG
##### HughesNet-PG
##### vpn-100-source

This port group will be used to service the LAN facing interface of the StarLink Source Edge. This port group will also be utilized to service ips through DHCP from the Source Edge. The Data Source Client will have a connection to this port group in order to utilize the SD WAN Fabric to, as the name implies, be the TX source of data that will be used to analyze the Network Performance with. 

1. Click on the `Add port group` button.
2. Name the new port group `vpn-100-source`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID of 200. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment,  this port group required a VLAN ID so that it would still be segmented away from the main network that allowed for internet access, as the goal was to ensure that the DHCP server running on the Edge Routers could use this port group to service private IPs to the corresponding systems.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
##### vpn-100-sink

This port group will be used to service the LAN facing interface of the EC2 Sink Edge. This port group will also be utilized to service ips through DHCP from the Sink Edge. The Data Sink Client will have a connection to this port group in order to utilize the SD WAN Fabric to, as the name implies, be the RX point of reception of the data that will be used to analyze the Network Performance.

1. Click on the `Add port group` button.
2. Name the new port group `vpn-100-source`, or whatever the naming scheme dictates.
3. Assign the port group a VLAN ID that does not have a proper VLAN mapping, so that this port group cannot service an internet connection. 
   **NOTE**: In the COSMIAC/RAPID Lab Environment, this port group required a VLAN ID so that it would still be segmented away from the main network that allowed for internet access, as the goal was to ensure that the DHCP server running on the Edge Routers could use this port group to service private IPs to the corresponding systems.
4. Choose the corresponding vSwitch that is connected to the SatCom uplinks. In the COSMIAC/RAPID Deployment, this was SatCom-WAN.
5. Leave the Security settings as the defaults.
6. Click on `Add` at the bottom right of the window.
   
## Palo Alto Firewall

### Security Policies

### Services Configuration

### NAT Policies

## VM Deployment

This configuration will be done under the `Virtual Machines` tab. This will bring up the configuration menu for all of the Virtual Machines that you will be utilizing. Many, if not all of the devices will end up being a Virtual Machine, unless you can configure connectivity of an external device, such as a Laptop, with connectivity to the edge routers.

Though the COSMIAC/RAPID deployment is entirely virtualized, the hardware components utilized in the setup enabled the virtual components to represent a full physical/commercial deployment of a small SD WAN Network Infrastructure. 

This section will be split into two instruction sets, one for the Control Plane, and the other for the Data Plane.
### Control Plane Devices

Talk to Luis about some of this, convert vBond pdf to this. 
#### vBond

#### vManage

#### vSmart

### Edge Devices
#### StarLink Source Edge


1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `StarLink Source Edge` or whatever your naming scheme dictates. 
4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `c8000v-universalk9.17.15.02a.ova` OVA file. This is the file that you can utilize to spin up as many edges as your Cisco account is allotted.
5. Choose one of the standard storage options for your new Virtual Machine. This should be a partition that can host the virtual machine to begin with. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage.
6. Select three (3) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the source, `GigabitEthernet1` should be `StarLink-PG`, and `GigabitEthernet2` should be `vpn-100-source`. More Network mappings can be made later to accommodate for other SATCOM or Terrestrial ISPs.
7. Choose `8x Large - 16GB Disk` as the deployment type
8. Choose `Thick` as the Disk provisioning
9. It is your choice to power on automatically or not. If planning to remove a Network Mapping, I suggest making sure this box is ***unchecked***.

#### EC2 Sink Edge

### Data Plane Devices

#### Client Devices

##### Source Client

##### Sink Client

## Control Plane Setup
## Data Plane Setup
