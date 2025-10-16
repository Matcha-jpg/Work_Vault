This specific instruction set is assuming all necessary setup and configuration has been complete up to the point where the Virtual Machines can be provisioned in ESXi, and the networking configurations allow for full connectivity once the Edge Router has been created.

#### EC2 Sink Edge

1. Click on `Create / Register VM` in the `Virtual Machines` GUI Window.
2. Upon opening the creation window, choose the `Deploy a virtual machine from an OVF or OVA file` option, and click on the `Next` button at the bottom right of the creation window.
3. Name the Virtual Machine `EC2 Sink Edge` or whatever your naming scheme dictates. 
4. Click the `Click to Select Files or Drag/Drop` option right below the `Enter a Name` box, and select the `c8000v-universalk9.17.15.02a.ova` OVA file, if you have it available. Otherwise you may have to download this file from the ESXi host client itself, or procure it from the associated personnel that has access to this file.
	   1. To check if this file is in your ESXi host client, navigate to the `Storage` menu using the far left sidebar on your ESXi GUI.
	   2. In the `Storage` Menu, there exists a table denoting the drives that are available for your ESXi host client. Above this table is a bar that should have the `Datastore browser` button. Click this button.
	   3. This will bring up another File system-esque menu. Look through your storage options/ask associated personnel for the location where `.ova` files are stored within the ESXi host client, if any. For the COSMIAC-RAPID Lab Environment, this location was in the `ExtraStore` storage partition, in the `OVAs` directory.
	   4. Once the file has been located/you have found the associated personnel who has access to this file, download the file onto your local machine.
   **NOTE**: This is the file that you can utilize to spin up as many edges as your Cisco account is allotted.
5. Choose one of the standard storage options for your new Virtual Machine, and an associated datastore. This should be a partition that can host the virtual machine, and should be listed in a table on this window. Choose whichever datastore you like, unless otherwise specified by procedures. The datastore used by the COSMIAC-RAPID Lab Environment to host all VMs was `datastore1`. Do not worry about memory persistence, as these images have ways to enforce persistent memory despite being under Standard storage. Then, click the `Next` button.
6. Select three (3) Network Mappings. You can remove a redundant one or add another if necessary for your setup later. For the source, `GigabitEthernet1` should be `EC2Connection` (or the Port Group you configured for your EC2 Connection), and `GigabitEthernet2` should be `sink-vpn-100`. More Network mappings can be made later to accommodate for other SATCOM or Terrestrial ISPs.
7. Choose `8x Large - 16GB Disk` as the deployment type
8. Choose `Thick` as the Disk provisioning
9. It is your choice to power on automatically or not. If planning to add/remove a Network Mapping, I suggest making sure this box is ***unchecked***. After 6-9 have been completed, click the `Next` button.
10. Review the final page, and click `Finish` at the bottom VM Provisioning should be complete. 
### Edge Routers

#### Initial Bootstrap Configuration
**NOTE**: If you intended for 3 total network interfaces, ***SKIP*** steps 1 and 2.
1. In the case that you did not enable automatic startup in the previous `VM Deployment` section, power the machine off.
2. Click on the `Edit` button in the window of your source edge VM, and add/remove network adapters as needed. After this is complete, power on the VM.
3. Wait for the VM to start up. This should take roughly 5-10 minutes.
4. Once the VM looks to be up and running, click into the terminal of the VM (Unfortunately, but also fortunately, you'll know when your mouse is within the VM's console if it disappears) and attempt to enter configuration by pressing `Enter`. This should bring up a prompt. Type `yes` to this prompt.
5. Another prompt will follow the first one. Also type `yes` in response.
6. This will enter you into configuration mode. This configuration will be to set your passwords. As with general security practices, ensure these passwords are secure, note them down and store them in a secure space.
7. If the setup requests that you specify an interface to communicate with the management/control devices, enter `GigabitEthernet1`, or your desired WAN facing interface. It will then ask you to provide the desired public IP and subnet mask, which if you are DHCP'ed an IP (for example, StarLink uses DHCP to allocate your public IP if using an enterprise level or higher StarLink), you can leave the defaults by just hitting enter. Otherwise, provide a static IP and the subnet mask of the interface you are on.
   For example:
   IP : 10.10.20.2
   Subnet Mask: 255.255.255.0
   This is the IP and Subnet Mask utilized for the Data Sink Edge Router in the COSMIAC-RAPID Lab Environment.
8. Afterwards, confirm the bootstrap configuration, and the router should prompt you with this message:
   `Press RETURN to get started!`
9. Once you see the above message, hit `Enter` on your keyboard. When following the next set of instructions, the commands will be in this structure:
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
10. This will restart your VM. Wait for it to fully launch once more, and again this should take 5-10 minutes depending on hardware. The VM will likely complain using the error message: `Smart Lisencing has encountered an internal software error`. You can safely ignore this and hit `Enter` on your keyboard, and login to the router.
11. The credentials used to login will be:
    username: admin
    password: admin
	This should prompt you to set a new password and confirm it, once again following best security practices, use a strong password and note it down in a secure location.
12. Once logged in, you are ready to follow the router configuration portion of this setup. 

#### Router Configuration Section
This section will contain the commands in a list form that you have to type in order to configure the router accordingly. When following the next set of instructions, the commands will be in this structure:
`hostname> command`
or
`hostname# command`
Depending on the step your are on. Replace `command` with the respective command as shown below.
In addition, you will see interfaces named as `GigabitEthernetx`. The x in this naming scheme corresponds to the Network Adapter configured on VM Deployment, so any connection on Network Adapter 1 will be synonymous with `GigabitEthernet1` in the respective edge router. Also, unless otherwise specified, use the last x value you were prompted to provide for the configurations.
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
**NOTE**: Repeat steps 14-20 for as many WAN facing interfaces you have. 
13. `hostname(config-system)# sdwan`
14. `hostname(config-sdwan)# interface GigabitEthernetx tunnel-interface` 
    **NOTE**: Replace `x` in `GigabitEthernetx` with a number corresponding to the Network Adapter of your WAN facing interfaces.

15. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
16. `hostname(config-tunnel-interface)# color (insert desired color here)`
     [Cisco SD WAN Color & Carrier Overview][https://www.networkacademy.io/ccie-enterprise/sdwan/tloc-color-and-carrier]

17. `hostname(config-tunnel-interface)# allow-service all`
18. `hostname(config-tunnel-interface)# exit`
19. `hostname(config-interface-GigabitEthernetx)# exit`
20. `hostname(config-sdwan)# ip route 0.0.0.0 0.0.0.0 (insert the default gateway IP used on GigabitEthernetx)`
    This should be the default gateway of the current WAN facing interface. For example, our StarLink WAN interface uses the default gateway IP of `143.105.158.1`

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
31. `hostname(config)# ip dhcp pool vrf-100-GigabitEthernetx`
    `GigabitEthernetx` is the LAN facing interface associated with the corresponding Network Adapter.

32. `hostname(dhcp-config)# vrf 100`
33. `hostname(dhcp-config)# lease 1 0 0`
34. `hostname(dhcp-config)# default-router 192.168.x.x`
    `192.168.x.x` subnet is often used in production environments as the private subnet. Other viable options would be the `10.x.x.x` and `172.16.x.x - 172.31.x.x` subnets. For example, the command for the Data Source Edge Router that the COSMIAC-RAPID lab environment utilized was:
    `hostname(dhcp-config)# default-router 192.168.52.1`
    Which notionally is the Default Gateway of the LAN Facing Interface.

35. `hostname(dhcp-config)# network 192.168.52.0 255.255.255.0`
36. `hostname(dhcp-config)# exit`
37. `hostname(config)# ip dhcp use vrf remote`
38. `hostname(config)# ip forward-protocol nd`
39. `hostname(config)# no ip ftp passive`
40. `hostname(config)# ip name-server 8.8.8.8`
41. `hostname(config)# ip name-server vrf 100 8.8.8.8`
42. `hostname(config)# ip scp server enable`
43. `hostname(config)# ip bootp server`
44. `hostname(config)# no ip source route`
45. `hostname(config)# no ip ssh bulk-mode` 
46. `hostname(config)# ip tcp RST-count 10 RST-window 5000` 
47. `hostname(config)# no ip http server` 
48. `hostname(config)# no ip http secure-server` 
49. `hostname(config)# ip http client source-interface GigabitEthernetx` 
50. `hostname(config)# ip nat pool natpool1 192.168.x.x 192.168.x.x prefix length n` 
    This is the NAT Address pool for the LAN interface. This will specify how large your private subnet is. For example, this is the command that was used to set this up on the Data Source Edge Router:
    `hostname(config)# ip nat pool natpool1 192.168.52.1 192.168.52.255 prefix length 24` 

51. `hostname(config)# ip nat inside source list global-list pool natpool1 vrf 100 match-in-vrf overload` 
52. `hostname(config)# ip nat inside source list nat-dia-vpn-hop-access-list interface GigabitEthernetx overload`
    Repeat this command for every WAN facing interface you have, with each corresponding x being associated with your WAN facing interfaces. For example:
    `hostname(config)# ip nat inside source list nat-dia-vpn-hop-access-list interface GigabitEthernet1 overload`

53. `hostname(config)# ip nat translation tcp-timeout 3600` 
54. `hostname(config)# ip nat translation udp-timeout 60`
55. `hostname(config)# ip nat settings central-policy`
56. `hostname(config)# ipv6 unicast-routing`
57. `hostname(config)# ipv6 cef load-sharing algorithm include-ports source destination`
58. `hostname(config)# ipv6 rip vrf-mode enable`
59. `hostname(config)# cdp run`
60. `hostname(config)# policy-map shape_GigabitEthernetx` - This x should correspond to your WAN facing interface.
61. `hostname(config-pmap)# class class-default`
62. `hostname(config-pmap-c)# shape average 12000000`
63. `hostname(config-pmap-c)# exit`
64. `hostname(config-pmap)# exit`
65. `hostname(config)# interface GigabitEthernetx` - This x should correspond to your LAN facing interface.
66. `hostname(config-if)# no shutdown`
67. `hostname(config-if)# arp timeout 1200`
68. `hostname(config-if)# vrf forwarding 100`
69. `hostname(config-if)# ip address 192.168.x.x 255.255.x.x`
     For Example:
     `hostname(config-if)# ip address 192.168.52.1 255.255.255.0`
70. `hostname(config-if)# no ip redirects`
71. `hostname(config-if)# ip mtu 1500`
72. `hostname(config-if)# load-interval 30`
73. `hostname(config-if)# mtu 1500`
74. `hostname(config-if)# negotiation auto`
75. `hostname(config-if)# exit`
76. `hostname(config)# commit`

###### **StarLink Interface Configuration**
***Skip this set of instructions if setting up the Data Sink Edge Router.***
This is the interface linked to the connection for StarLink. Do note that *only one router at a time* can hold the StarLink's allocated public ip. Also depending on how you configured the deployment for the edge router, this connection may not match the one that is Utilized by the COSMIAC-RAPID Lab Environment. 

This interface was used with the Data Source Edge Router in the COSMIAC-RAPID Lab Environment. In an ideal environment, there would also be a StarLink connection on the Data Sink Edge Router, but as stated previously, the COSMIAC-RAPID Lab only had access to one StarLink that allowed for a public IP Allocation.
77. `hostname(config)# interface GigabitEthernetx` 
    Replace x with the corresponding GigabitEthernet interface used for your StarLink WAN facing Network Adapter. The corresponding interface was `GigabitEthernet1` for the COSMIAC-RAPID Lab Environment on the Data Source Edge Router. 

78. `hostname(config-if)# no shutdown`
79. `hostname(config-if)# ip address dhcp`
80. `hostname(config-if)# arp timeout 1200`
81. `hostname(config-if)# ip dhcp client client-id GigabitEthernetx`
82. `hostname(config-if)# no ip redirects`
83. `hostname(config-if)# ip dhcp client default-router distance 1`
84. `hostname(config-if)# ip nat outside`
85. `hostname(config-if)# ip mtu 1500`
86. `hostname(config-if)# load-interval 30`
87. `hostname(config-if)# mtu 1500`
88. `hostname(config-if)# negotiation auto`
89. `hostname(config-if)# service-policy output shape_GigabitEthernetx`
90. `hostname(config-if)# exit`
91. `hostname(config)# interface Tunnel0`
92. `hostname(config-if)# no shutdown`
93. `hostname(config-if)# ip unnumbered GigabitEthernetx`
94. `hostname(config-if)# no ip redirects`
95. `hostname(config-if)# ipv6 unnumbered GigabitEthernetx`
96. `hostname(config-if)# no ipv6 redirects`
97. `hostname(config-if)# cts manual`
98. `hostname(config-if-cts-manual)# no propagate sgt`
99. `hostname(config-if-cts-manual)# exit`
100. `hostname(config-if)# tunnel source GigabitEthernetx`
101. `hostname(config-if)# tunnel mode sdwan`
102. `hostname(config-if)# exit`
103. `hostname(config)# sdwan`
104. `hostname(config-sdwan)# interface GigabitEthernetx tunnel-interface`
105. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
106. `hostname(config-tunnel-interface)# color blue`
107. `hostname(config-tunnel-interface)# allow-service all`
108. `hostname(config-tunnel-interface)# no border`
109. `hostname(config-tunnel-interface)# low-bandwidth-link`
110. `hostname(config-tunnel-interface)# no vBond-as-stun-server`
111. `hostname(config-tunnel-interface)# vmanage-connection-preference 5`
112. `hostname(config-tunnel-interface)# port-hop`
113. `hostname(config-tunnel-interface)# carrier default`
114. `hostname(config-tunnel-interface)# nat-refresh-interval 5`
115. `hostname(config-tunnel-interface)# hello-interval 6000`
116. `hostname(config-tunnel-interface)# hello-tolerance 600`
117. `hostname(config-tunnel-interface)# exit`
118. `hostname(config-interface-GigabitEthernetx)# bandwidth-downstream 250000`
119. `hostname(config-interface-GigabitEthernetx)# qos-adaptive`
120. `hostname(config-qos-adaptive)# downstream 200000`
121. `hostname(config-qos-adaptive)# downstream range 80000 250000`
122. `hostname(config-qos-adaptive)# upstream 12000`
123. `hostname(config-qos-adaptive)# upstream range 2000 20000`
124. `hostname(config-qos-adaptive)# exit`
125. `hostname(config-interface-GigabitEthernetx)# exit`
126. `hostname(config-sdwan)# exit`
127. `hostname(config)# commit`

###### **EC2 Interface Configuration**
***Skip this set of instructions if setting up the Data Source Edge Router.***  
This is the interface linked to the connection for the AWS EC2 Instance. Note that you can have many routers behind this EC2 instance, as it acts as a NAT Gateway/Router that the edge router would likely be behind in a commercial SD WAN Deployment. Also depending on how you configured the deployment for the edge router, this connection may not match the one that is Utilized by the COSMIAC-RAPID Lab Environment.

This interface was used with the Data Sink Edge Router in the COSMIAC-RAPID Lab Environment. 
128. `hostname(config)# interface GigabitEthernetx`
     Replace x with the corresponding GigabitEthernet interface used for the Network Adapter that connects to the EC2 Instance. The corresponding interface was `GigabitEthernet1` for the COSMIAC-RAPID Lab Environment on the Data Sink Edge Router.

129. `hostname(config-if)# no shutdown`
130. `hostname(config-if)# ip address x.x.x.x m.m.m.m`
     Here, we are setting a static IP address that is on the subnet that your EC2 is handling routing on. For the COSMIAC-RAPID Lab Environment, the command was the following:
     `hostname(config-if)# ip address 10.10.20.2 255.255.255.0`
     This command entails that on interface `GigabitEthernetx`, the router gets the IP `10.10.20.2` and is on the `10.10.20.1/24` subnet due to the `255.255.255.0` subnet mask.
     **NOTE**: DO NOT USE `10.10.20.2` IF IN THE COSMIAC-RAPID LAB ENVIRONMENT. Use anything between `10.10.20.3` - `10.10.20.254` inclusive.

131. `hostname(config-if)# arp timeout 1200`
132. `hostname(config-if)# no ip redirects`
133. `hostname(config-if)# ip nat outside`
134. `hostname(config-if)# ip mtu 1500`
135. `hostname(config-if)# load-interval 30`
136. `hostname(config-if)# mtu 1500`
137. `hostname(config-if)# negotiation auto`
138. `hostname(config-if)# service-policy output shape_GigabitEthernetx`
139. `hostname(config-if)# exit`
140. `hostname(config)# interface Tunnel0`
141. `hostname(config-if)# no shutdown`
142. `hostname(config-if)# ip unnumbered GigabitEthernetx`
143. `hostname(config-if)# no ip redirects`
144. `hostname(config-if)# ipv6 unnumbered GigabitEthernetx`
145. `hostname(config-if)# no ipv6 redirects`
146. `hostname(config-if)# tunnel source GigabitEthernetx`
147. `hostname(config-if)# tunnel mode sdwan`
148. `hostname(config-if)# exit`
149. `hostname(config)# sdwan`
150. `hostname(config-sdwan)# interface GigabitEthernetx tunnel-interface`
151. `hostname(config-tunnel-interface)# encapsulation ipsec weight 1`
152. `hostname(config-tunnel-interface)# color blue`
153. `hostname(config-tunnel-interface)# allow-service all`
154. `hostname(config-tunnel-interface)# no border`
155. `hostname(config-tunnel-interface)# low-bandwidth-link`
156. `hostname(config-tunnel-interface)# no vBond-as-stun-server`
157. `hostname(config-tunnel-interface)# vmanage-connection-preference 5`
158. `hostname(config-tunnel-interface)# port-hop`
159. `hostname(config-tunnel-interface)# carrier default`
160. `hostname(config-tunnel-interface)# nat-refresh-interval 5`
161. `hostname(config-tunnel-interface)# hello-interval 6000`
162. `hostname(config-tunnel-interface)# hello-tolerance 600`
163. `hostname(config-tunnel-interface)# exit`
164. `hostname(config-interface-GigabitEthernetx)# exit`
165. `hostname(config-sdwan)# exit
166. `hostname(config)# commit`

##### **Control Plane Onboarding**
167. `hostname# show clock` - Clock will likely be in UTC format. If time is off, adjust using next steps. Otherwise, skip step 168.
168. `hostname# clock set hh:mm:ss Day Month Year` - Where `hh` is the hour (0-23), `mm` is minutes (0-59), `ss` is seconds (0-59), `Day` is the day of the month (0-31), `Month` is the full name of the month, and `Year` is the current year in the format of `YYYY`.
169. Access the vManage GUI through a web browser on a device connected to the Control Plane network. For example, in the COSMIAC RAPID Lab Environment, a Laptop was physically connected to the network in which the control plane resided, so vManage was accessed through a Firefox browser through the IP of vManage on that network (Which was 10.10.0.42 for the COSMIAC-RAPID Lab Environment
170. Hover over `Configuration` on the left side bar, and click on `WAN Edges` under the `Devices` heading. This will take you to the list of WAN Edges allocated to your Cisco SD WAN account.
171. Generate a bootstrap config by clicking on the `...` under the `Actions` column on an available WAN Edge, and click on `Generate Bootstrap Configuration`.
172. Ensure that for the configuration, `Cloud-Init` is the selected option. Click on the `OK` button at the bottom right of this window.
173. Note the `uuid` and the `otp` that the bootstrap configuration generates. This will be ***VERY IMPORTANT***.
174. Go back to your ESXi GUI and navigate back to your Source edge router. Log back into the router if necessary. Then enter the command in the following step
175. `hostname# request platform software sdwan vedge_cloud activate chassis-number (uuid) token (otp)`
    For example:
    `hostname# request platform software sdwan vedge_cloud activate chassis-number C8K-8943C275-7B78-1E88-B761-4F8045EB38A1 token e60cbdd624774ef7bd45fab8bc40c5ac`
176. The prior command is lengthy, so **ensure** that you have the ***correct values WITHOUT any typos***. This will onboard the edge to the SD WAN fabric.
177. Verify that your edge router has complete control plane connectivity.
