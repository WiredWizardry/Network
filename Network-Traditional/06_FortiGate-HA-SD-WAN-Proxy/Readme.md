# FortiGate Basic HA W/ Basic SD-WAN Lab Guide

```Use Case

```

-   GNS3 / *EVE*-*NG environment*
-   FortiGate firewall images
-   Cisco/ Tunnel termination images

```Required Components

```

1.  **Network devices preconfigured; This lab assumes that you have a network device acting as an ISP connection, already configured and providing access to the internet. Ensure that your network is operational up to this point.**
2.  **FortiGate KVM image**: You will need a FortiGate virtual machine (KVM image) where all the configurations in this lab will be implemented. Ensure that you have this VM set up and ready for use.

```Key Steps

```

1.  **Network Topology**: We'll design the layout of our network, including the placement of FortiGate firewalls, to ensure proper traffic flow and security.
2.  **Initial Configuration**: Set up the basic configurations on the FortiGate firewall.
3.  **Configure FortiGate FW1A**: We'll focus on configuring one of the FortiGate devices for High Availability, ensuring redundancy and fault tolerance.
4.  **Configure FortiGate FW1B**: Similar to Step III, we'll configure the second FortiGate device for High Availability to establish a robust HA setup.
5.  **Configure Basic SD-WAN** –We'll delve into SD-WAN configuration to optimize traffic routing over two ISP links, enhancing network performance and reliability.
6.  **Configure Static Route -** We'll set up static routes to enable efficient routing over the ISP links, directing traffic appropriately.
7.  **Configure Firewall Policy and Test connectivity - In this final step, we'll create firewall policies to control traffic flow and test connectivity to ensure that our network operates as expected.  
    **

```Lab Topology

```

A detailed topology will be outlined in the lab guide, indicating the network layout and connections between the control node and the FortiGate firewall.

![](media/9e12c6efc75cb82d8bb282d9261a92bd.png)

\*please note that in highlighted in red we don’t spend much time on the configuration its up to you to decide to build this out the same

```Lab Devices Information

```

| Hostname      | Description                                                            | Route-table                                                                                                       |
|---------------|------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| FW1A/FW1B     | FortiGate firewalls where we will setup HA                             | 192.168.1.1/24 (WAN1) 192.168.2.1/24 (WAN2) 10.0.1.254/24 (LAN) 10.254.254.200/24(MGT) 0.0.0.0/0 (Static Default) |
|  L2-INT-SW1/2 | I86bi_linux_l2 Cisco basic L2 switch                                   | L2                                                                                                                |
|  L2-EXT-SW1/2 | I86bi_linux_l2 Cisco basic L2 switch                                   | L2                                                                                                                |
| ISP1/2        | Pfsense-2.6.0 Acting as ISP and Default GW for FortiGate ISP links 1/2 | 192.168.1.254/24 (LAN1) 192.168.1.254/24 (LAN2)                                                                   |

# Step 0: Initial configurations

We will start with the initial Firewall configuration. Configuring LAN, WAN1/2 and MGT

| FW1A | !{ FW1A Configuration: ====================  config system global  set hostname FW1A set admintimeout 480 end  config system interface  edit port1 set alias MGT set mode static set ip 10.254.254.200 255.255.255.0 set allowaccess ping https ssh http end  config system interface  edit port3 set alias WAN-1 set mode static set ip 192.168.1.1 255.255.255.0 set allowaccess ping set role wan end   config system interface  edit port4 set alias WAN-2 set mode static set ip 192.168.2.1 255.255.255.0 set allowaccess ping set role wan end  config system interface  edit port2 set alias LAN set mode static set ip 10.0.1.254 255.255.255.0 set allowaccess ping set role lan end  config system dns set primary 8.8.8.8 set secondary 1.1.1.1 end |
|------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| FW1B | config system global  set hostname FW1A set admintimeout 480 end  config system interface  edit port1 set alias MGMT set mode static set ip 10.254.254.201 255.255.255.0 set allowaccess ping https ssh http end                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |

Validate that routing table matches expected configs using expected command.

| get router info routing-table all |
|-----------------------------------|

**FW1A**

![](media/197b3887957fbd5b3533e73569fd69b9.png)

**FW1B**

![](media/8c52a0b9d2c79a55e30a5a1589ab4c34.png)

# Step 1: Configure FortiGate FW1A

First, we login into the FortiGate;

Visit your FortiGate through the GUI using http (use the ip you just configured)

https://10.254.254.201

![](media/2bd84aa8410903277de00bac444bb0f8.png)

Configure HA

Navigate to **System -\> HA**

![](media/2dbc62aeb90524fa45871b76584475e3.png)

Select -\> **Active-Passive**

Configure the below

| Name                 | Value                | Info                                                                                         |
|----------------------|----------------------|----------------------------------------------------------------------------------------------|
| Device priority      | 150                  | (We want FW1A to be Primary)                                                                 |
| Group name           | HA-Pair              |                                                                                              |
| Password             | 123456               |                                                                                              |
| Session pickup       | Enable               | This is enabled so that session is not dropped                                               |
| Monitor Interfaces   | MGMT LAN WAN-1 WAN-2 | Interfaces to be monitored so that FortiGate fails over to passive if one of the links fails |
| Heartbeat interfaces | Port6                | Interface to use for HA                                                                      |

![](media/5934989222ddd661e7356044ac3d5fd2.png)

**Click [Ok] to commit changes**

**Go back to your lab and make sure port6 from FW1A to port6 on FW1B is in place**

**![](media/640192acf728f2165b037c60be88a763.png)**

# Step 2: Configure FortiGate FW1B

First, we login into the FortiGate;

Visit your FortiGate through the GUI using http (use the ip you just configured)

<https://10.254.254.200>

![](media/a019517209afac4a166aaf7a0c99b9a3.png)

**Configure HA**

Navigate to **System -\> HA**

![](media/2dbc62aeb90524fa45871b76584475e3.png)

Select -\> **Active-Passive**

Configure the below

| Name                 | Value                  | Info                                                                                         |
|----------------------|------------------------|----------------------------------------------------------------------------------------------|
| Device priority      | 128                    | (We want FW1A to be Primary)                                                                 |
| Group name           | HA-Pair                |                                                                                              |
| Password             | 123456                 |                                                                                              |
| Session pickup       | Enable                 | This is enabled so that session is not dropped                                               |
| Monitor Interfaces   | MGMT Port2 Port3 Port4 | Interfaces to be monitored so that FortiGate fails over to passive if one of the links fails |
| Heartbeat interfaces | Port6                  | Interface to use for HA                                                                      |

![](media/81f90203b74946fc8d813809b633885d.png)

**Click [Ok] to commit changes**

Run below command on both Fw to force synchronize if need

| execute ha synchronize start |
|------------------------------|

As FW1A is synching status will show Not synchronized

![](media/c74cb6feee1c1a3851993bf5c62a900d.png)

Cli command to see HA status:

| get system ha status |
|----------------------|

Once FW1A is finished synching status will show as synchronized

![](media/f74a0d140de6721e424af6934aafa6a2.png)

Once FW1B has been synchronized it is in a passive state and has the exact same configuration as FW1B. (You will also not be able to login into the device vail http, ssh anymore since the device now has the same mgmt device as FW1A(10.254.254.201) and is just in a passive mode.

![](media/b2339479e72f5755bfc22b2297f65af8.png)On the console you can see that FW1B has the configs of FW1A now

# Step 3: CONFIGURE BASIC SD-WAN

**First, we login into the FortiGate;**

Visit your FortiGate through the GUI using http (use the ip you just configured)

<https://10.254.254.201>

First step is to create a SD-WAN Zone:

Go to **Network \> SD-WAN Zones.** Click **Create New \> SD-WAN Zone.**

![](media/28639dd8c61b0064ed6f3a43a9eb3a6a.png)

Name the New SD-WAN Zone in this case **SDWAN** and click OK to create.

![](media/35f3641aa9594f35722a38212859c1ea.png)

The New SD-WAN Zone is ready however, it is shown as down b/c there is no interface associated with the zone.

![](media/641016a7fedcab7de43815d4ac08ac74.png)

To add interfaces to the SD-WAN Zone.

Go to **Network \> SD-WAN Zones**. Click **Create New \> SD-WAN Member.**

![](media/2089b9450e3c6fea6a0942dda8ae978e.png)

Set the Interface to **WAN-1.** Change the **SD-WAN Zone** to **SDWAN** created earlier.

Set Gateway set to **192.168.1.254**. Leave the Cost as 0. Leave the Priority to 1. Set Status to Enable, and click OK.

![](media/38c715c2e743794a714a6b31a5a4ea15.png)

Repeat the above steps for WAN-2, setting Gateway to the ISP's gateway: **192.168.2.254**

![](media/979f3a8eb648d04d86026b5e5aaec15c.png)

Once both the ISPs interfaces are added to SDWAN the SD-WAN Zone should now show up as green.

![](media/2f56d83dfd7a182d2f5a0e10ddf67a5c.png)

Configure SD-WAN Health-check:

Go to **Network \> Performance SLS**

**![](media/456e6f15379abff00449fc09a90e0048.png)**

Click **Create New and Fill out with below**

![](media/4bc67f9a2b2f76113e344cf8c2c5135b.png)

You should Now see the new SLA in the list

![](media/4cdbf8c0290ec9c3eefaa9ca8c86c52e.png)

This step configuration can also be done vail CLI as Such

| config system sdwan config zone edit "SDWAN" end  config system sdwan set status enable config members edit 1 set interface port1 set zone "SDWAN" set gateway 192.168.1.254 next edit 2 set interface port2 set zone "SDWAN" set gateway 192.168.2.254 next end config system sdwan set load-balance-mode source-dest-ip-based end  config system sdwan config health-check edit dnssrv set server 8.8.8.8 set update-static-route enable set members 1 2 next end end |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

# Step 4: Configure Static Route

The next step is to configure static routes so that your ISP links are used as the default gateway.

Go to **Network \> Static Routes**. Click **Create New.** The New Static Route page opens. Set Destination to **Subnet**, and leave the IP address and subnet mask as **0.0.0.0/0.0.0.0**.

From the Interface drop-down list, select SDWAN. Ensure that Status is Enabled. Click OK

![](media/fdeb349c2a85c4ceb8c7de6c64583c44.png)

![](media/8e169ee63c316f72a4766c82b8ce15db.png)

![](media/c677c212ee1e87276bf0b248f48b3464.png)

Configuration above can be done vail CLI as well

| config router static edit 1 set sdwan-zone SDWAN next end |
|-----------------------------------------------------------|

# Step 5: Configure Firewall Policy and Test connectivity

To allow data to move between your LAN and ISP links a firewall policy must be set up. This policy serves as a rule that tells the FortiGate device to let traffic from the LAN reach the ISP Links and vice versa.

Go to **Policy & Objects \> Firewall Policy**. Click Create New.

![](media/0abb0a54075d3cd01f3e56d94ebda876.png)

The **New Policy** page opens. Set the name, incoming interface, outgoing interface, source, destination, schedule, service, action details as below and **Enable** the policy, then click OK

![](media/70c61e8f4a6c106094d38d171ce4de02.png)

Firewall policy can be created vail CLI as well

| config firewall policy edit 1 set name LAN-to-ISP set srcintf port2 set dstintf SDWAN set srcaddr all set dstaddr all set action accept set schedule always set service ALL set logtraffic all set nat enable set status enable next end |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

**  
**

**Testing and Verification:**

Let’s ping/browse from internal LAN computer Desktop1 to the internet using two different IPs or domain in this case Youtube.com and Google.com

First, we have to assign our desktop a IP with the LAN range before we can do any testing

We can use **10.0.1.10** for Desktop1 and **10.0.1.20** for Desktop2

![](media/8529c45403b5ee22db9afa72d0ce45e7.png)

![](media/02825393c7b7d6b477a3ab8266b0cbe0.png)

![](media/9e06785689d69031be0af0cad4304843.png)

Validate that you can reach your default Gateway (10.0.1.254 in our case)

![](media/e83930df332ac39df8eb6531e35d2831.png)

![](media/d12e091dfb84e02dd4f69f34b1edc740.png)

Validate that you can ping/browse to google.com/YouTube on both Desktop thus validating that Firewall Rules are in place and routing is working to your ISP links.

![](media/e76328614bddce189f0b248bd5153212.png)

![](media/eac7cf049586ed2a861f096048df03c8.png)

![](media/3172071444adc4afaba28ed84904b4d9.png)

![](media/f5d37885e1b19b3ac433f825c62486df.png)

IF you want to test HA on the firewalls you can shut down the primary firewall to simulate a Power failure on the Active Firewall or bring down one of the monitored links which would also cause the firewall to failover. You can run a continues ping from your Desktop and notice how the session is kept alive even in the case of the simulated link/power failure.

Simulate Power failure

Ping test

![](media/72e687a2854d97730019b830fe9262f8.png)

Pings continue even after FW1A power failure

![](media/b6529858cb10e0bad976baa4b8577b01.png)

**Simulate Link Failure**

Currently FW1B is Primary after simulated Hardware failure

![](media/0e2f8d599ab318d665798e84667ba4c4.png)

Suspend Port2 on FW1B while running a ping test for a link failure simulation. Since Port2(LAN) is a monitored link, FW1B should failover to FW1A as Primary and no pings should fail.

![](media/65596c4199b68b7b1a84c054e5d9599c.png).

No ping loss and FW1A is back to being Primary

![](media/4e97879322125c8ab675f0d42420b809.png)

FW1A is HA Primary once more and you can see that on FW1B port2 is currently Down

![](media/85db9a840f53d1c3f569c33ee8a61d0c.png)
