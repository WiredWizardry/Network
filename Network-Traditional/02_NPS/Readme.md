# AAA Radius– Windows Server 2022 *Network Policy Server (NPS)- AAA RADIUS* Proof of concept Basic configuration

This lab activity delves into the implementation of AAA (Authentication, Authorization, and Accounting) in a Cisco IOS environment using a RADIUS server. AAA provides a robust framework that manages user access to network devices, enforces policies, and audits usage. This lab uses a Windows Server-based RADIUS server for AAA services, offering a common scenario in a real-world environment.

The AAA services, when configured on a Cisco device, interact with a RADIUS server that centralizes user access control and policy enforcement. Authentication verifies a user's identity, authorization determines what a user is allowed to do, and accounting tracks what the user does.

When it comes to configuring AAA, it's important to note that it's different from setting up simple password security on interfaces. AAA leverages local or external databases (like the RADIUS server in this case) for increased security and centralized control.

Typically, AAA is set up to first try to authenticate or authorize using an external server (like RADIUS), and if that server is unavailable, it falls back to local user accounts stored on the router or switch.

The main advantages of using AAA with a RADIUS server include centralized user management, scalable access control, and enhanced auditing and accountability.

The main steps in the lab are:

1.  Configure Switch and router with local user account
2.  Configure windows server network adapter
3.  Install and configure the Network Policy Server (NPS) role on the Windows Server to act as a RADIUS server
4.  Set up a local user account and group on the Windows Server
5.  Configure the Cisco device to use RADIUS for AAA, and set it up to use the local user database as a fallback authentication method.
6.  Test the setup by logging in with the user accounts created on the Windows Server and tracking the accounting logs.

## Key topics covered:

## Understanding of AAA services

## Setup of a Windows Server-based RADIUS server

## Configuration of AAA on Cisco IOS devices

## Understanding of fallback to local accounts when RADIUS server is unavailable

## Hands-on experience with user account and access management on a network device.

We will use the following topology:

![](media/609b7fa9be86aa1b8705305e1f74fd4b.png)

Here are some of the information on lab devices

| **Hostname** |                                             |
|--------------|---------------------------------------------|
| **R1**       | Cisco Router running IOSv Version 15.9(3)M2 |
| **S1**       | Cisco Switch running vios_l2 Version 15.2   |
| **DC-2**     | Microsoft Server 2022                       |

## 

Prerequires:

-   GNS3
-   Windows server 2022 -2016 (The steps are the same for any of these versions.) You can download the evaluation version from Microsoft.
-   Cisco images for GNS3

## Step 1: Configure Switch

First we have to configure the switch for connectivity

![](media/05af1e23dfe0ce6585e7f8d6f4ac2d7f.png)

| no ip domain name lookup no logging console cdp run host S1 int vlan 1 ip address 192.168.100.100 255.255.255.0 no shut username user privilege 15 secret password line con 0 login local line vty 0 4 login local |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

Let's go through each of the commands listed:

1.  **no ip domain name lookup**: This command disables DNS lookup. Without this, any mistyped command in the console is interpreted as a hostname by the router, and it will attempt to resolve it via DNS, which can cause a delay.
2.  **no logging console**: This command disables logging to the console. By default, the router sends all log messages to its console port. Therefore, disabling this can be helpful in not interrupting CLI access with log messages.
3.  **cdp run**: This command enables the Cisco Discovery Protocol (CDP). CDP is a Cisco proprietary protocol used to discover Cisco devices in your network.
4.  **host S1**: This command changes the hostname of the device to "S1".
5.  **int vlan 1**: This command enters the configuration mode for VLAN 1.
6.  **ip address 192.168.100.100 255.255.255.0**: This command assigns the IP address 192.168.100.100 with a subnet mask of 255.255.255.0 to VLAN 1.
7.  **no shut**: This command brings up the VLAN interface if it's administratively down. It's equivalent to "enable this interface".
8.  **username user privilege 15 secret password:** This command creates a user with the username "user", assigns it a privilege level of 15 (the highest level, equivalent to root or admin), and sets the password to "password". The keyword "secret" indicates that the password will be stored in a hashed format.
9.  **line con 0**: This command enters line configuration mode for the console port.
10. **login local**: This command sets the login method to use the local user database for authentication. It's used here for the console and VTY lines, meaning that the username and password set earlier will be used for console and remote logins.
11. **line vty 0 4**: This command enters line configuration mode for the first 5 VTY lines (0-4). VTY lines are used for Telnet and SSH access to the device.

So, in essence, this configuration script is setting up a basic network device with a single IP interface, a local user for administration, and with console and remote login using this user. It also disables some default settings like DNS lookup on mistyped commands and console logging.

## Step 2: Configure Router

Next, we have to configure the Router for connectivity and then validate connectivity based on the network topology.

![](media/b6fb45264428d31d18129dd155493a12.png)

| no ip domain name lookup no logging monitor no logging console cdp run host R1 int GigabitEthernet0/0 ip address 192.168.100.101 255.255.255.0 no shut username user privilege 15 secret password line con 0 login local |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

Next, we have to validate that R1 can reach S1

![](media/1ba3a962907635f699256c3b82a537d3.png)

The command **ping 192.168.100.100** is used to test the network connectivity from the router (R1) to SW1 with the IP address 192.168.100.100.

The output provides information about this process:

-   **Type escape sequence to abort**.: This line indicates that you can stop the ping process by entering the escape sequence (typically Ctrl+Shift+6 twice).
-   **Sending 5, 100-byte ICMP Echos to 192.168.100.100, timeout is 2 seconds**:: This line tells you that the ping command is sending 5 ICMP Echo Request packets, each containing 100 bytes of data, to the IP address 192.168.100.100. The router will wait for a response to each request for up to 2 seconds.
-   **!!!!!**: Each '!' represents a successful reply (ICMP Echo Reply) from the device with the IP address 192.168.100.100. In this case, the router received a reply for all 5 Echo Requests, indicating that network connectivity to this IP address is good.
-   **Success rate is 100 percent (5/5), round-trip min/avg/max = 7/9/15 ms**: This line gives a summary of the ping operation. It shows that all 5 ICMP Echo Requests were successfully replied to, for a success rate of 100 percent. It also provides the minimum, average, and maximum round-trip times for the packets, which are 7, 9, and 15 milliseconds respectively. Round-trip time is the time it takes for a packet to go from the sending endpoint to the receiving endpoint and back. The lower this value is, the better the network connection is between the two endpoints.

## Step 3: Configure windows server network adapter

1.  open the **Control Panel**, then go to **Network and Internet** \> **Network and Sharing Center** \> **Change adapter setting**
    1.  Find and right-click on the network connection you want to configure (for example, Ethernet), then select **Properties**.
    2.  In the Properties window, scroll down and select **Internet Protocol Version 4 (TCP/IPv4)** , After selecting it, click **Properties**.
    3.  In the Properties window, select Use the following IP address.
    4.  Enter the IP address, Subnet mask, and Default gateway that you want to use. Below, select Use the following DNS server addresses. Enter the Preferred DNS server.

        ![](media/09593d3d0c26d677a794b392d57a9ae5.png)

    5.  Click **OK** to save your settings, then **OK** again to close the Properties window.
    6.  Next we validate connection. Search for "cmd" or "Command Prompt" in the Start menu.
2.  In the Command Prompt window, type ping followed by the 192.168.100.100 or the IP address of the device you want to check the connection to

    \` ![](media/a0173ac34370ee06fed3ecc71cd415e8.png)

    1.  Validate that you can reach **SW1(192.168.100.100)** and **R1(192.168.100.101)** from your windows server 2022

        ![](media/76ffd2e88e0da5d8a8550f1679e1a035.png)

        \*you will be unable to ping the server from S1 and R1 because windows server blocks inbound ICMP by default. You can disable this in windows firewall if wanted/needed

## Step 4: Install Network Policy and Access Services Role

1.  Open **Server Manager**.

    ![](media/01886b8ef5d0f7b30aeb45b7815c5a72.png)

2.  Click on **Manage** \> **Add Roles and Features**.

    ![](media/4f44abc35d5e970e4bb4391cb840a38a.png)

3.  Follow the wizard until you get to the **Server Roles** step.
4.  Select **Network Policy and Access Services** and click **Next**.

    ![](media/05cef44c50be9f6775b48835f17e3942.png)

5.  Follow the remaining steps to complete the installation.

## Step 5: Define RADIUS Clients

1.  Open the **NPS console**. (under **tools** **Network Policy and Access Services**)

    ![](media/32db7a03094b1e53ee79b28ee53d7329.png)

2.  In the NPS console, navigate to **RADIUS Clients and Servers.**
3.  Right-click on **RADIUS Clients** and select **New**.
4.  Enter the **Friendly name, Address (IP or DNS)**, and **Shared Secret** (using **88888888** for this lab\*\*)\*\* for the Cisco device. Click OK.

| R1  | 192.168.100.101 |
|-----|-----------------|
| SW1 | 192.168.100.100 |

![](media/06b4692546665e19145116e09d02987f.png)

## Step 6: Create local Group and local user accounts on the Windows Server

1.  Open **Computer Management** right clicking the start menu then clicking on **Computer Management**.
2.  In the left pane, navigate to **System Tools** \> **Local Users and Groups** \> **Groups**.
3.  Right-click on the **Groups** folder and select **New Group**. Create two groups (Group 1 & Group 15)

![](media/484edc7ebf919bb04f0fd3e7b21636ab.png)

![](media/551100374949655f442d61f4bc0cce8f.png)

\*Repeat these steps to create additional local Group accounts as needed.

1.  In the left pane, navigate to **System Tools** \> **Local Users and Groups** \> **Users**.
2.  Right-click on the **Users** folder and select **New User**.
3.  Create Read only user account with below information. Uncheck everything and only check **User cannot change Password** & **Password never expires**

| User name: | cisco    |
|------------|----------|
| Password   | Test12!! |

![](media/cbf2b424f77879444cb66c832f47c996.png)

1.  Do the same thing with the second account (Admin account) with below information. Uncheck everything and only check **Password never expires**

| User name: | Netadmin   |
|------------|------------|
| Password   | Secret\$\$ |

![](media/6ea9eeea73f0bd9a962bfb2a19101217.png)

\*Repeat these steps to create additional local user accounts as needed.

## Step 7: Add local User to local Group on the Windows Server

1.  In the left pane, navigate to **System Tools** \> **Local Users and Groups** \> **Groups**.
2.  click on the **Groups** folder and double click on **Group1**
3.  Under **Group1** click on **Add**
4.  ![](media/c9f8b4455b7cb785126fae0f2be9ea63.png)On the Select Users **menu** type in **cisco** under **Enter the object names to select** than click on **Check Names** for windows to resolve the local user.
5.  Click OK than OK again to save the user to the local Group
6.  Add the second user ( **Netadmin**) to **Group15.** Double click on **Group15** and under **Group15** click on **Add.** On the Select Users **menu** type in **Netadmin** under **Enter the object names to select,** then click on **Check Names** for windows to resolve the local user.

    ![](media/e9c46c85cc6c22f77528ac0f4c8887ba.png)

7.  Click OK than OK again to save the user to the local Group

## Step 8. Create a Network Policy for Local Accounts

1.  Go back to **Network Policy and Access Services**. (Open the **NPS console**. (under **tools** **Network Policy and Access Services**)
2.  Navigate to **Policies** \> **Network Policies**.
3.  Remove any existing policies(Right click and select Delete)

    ![](media/f2777b6c3ab475900d5b38489c80d77b.png)

4.  Right-click on **Network Policies** and select **New**.
5.  Enter a **PRIV15_Policy** for the first policy name, then select **Type of network access server** as **Unspecified**, click **Next**.
6.  In the **Conditions** page, click **Add**, select **Windows Groups** and add the **Windows** group. Make sure to type in **group15** then click on Check Names for windows to resolve the local group.

    ![](media/6d46842e7ddbc8cd552e747aeee4ddd2.png)

7.  Click OK than OK again to save Group, click **Next**.
8.  On the **Access Granted** page, select **Access granted**, click **Next**.
9.  In the **Authentication Methods** page, uncheck everything and only check **Unencrypted authentication (PAP, SPAP)**. Click **Next**. If any popups comes up click **NO**

    ![](media/d3451882c0967d7343cdfb963f139a3e.png)

10. In the **Authentication Methods** page keep the default settings, then click **Next.**
11. In the **Configure Settings** under Navigate to **RADIUS Attributes** **\>** **Standard** and Remove **Framed-Protocol**

    ![](media/b801a28c3dd101db3cb749499bb27901.png)

12. Edit **Service-Type.** Select **Others** and change it to **Login.**

    ![](media/8db783096fe6d70a74544e9cea2a0c3a.png)

13. In the **Configure Settings** Navigate to **RADIUS Attributes** **\>** **Vendor Specific** click on **ADD**
14. Scroll to the bottom of the list and select **Vendor-Specific**

    ![](media/e230717ae9510412b1d06c916493dbc8.png)

15. Click on Add with **Vendor-Specific highlighted**
16. Click on **Add**
17. Under **Select form list** select **Cisco** than click on **Yes it conforms**

    ![](media/92608bad3043fb579ace9058851fdecf.png)

18. Select **Configure Attribute** and input the below information

| **Vendor assigned attribute number:**                             | **1**                 |
|-------------------------------------------------------------------|-----------------------|
| **Attribute format:**                                             | **String**            |
| For admin policy ( **Priv15**) under **Attribute value:**         | **shell:priv-lvl=15** |
| For Readonly policy ( **Priv1**) under **Attribute value** enter: | **shell:priv-lvl=1**  |

**![](media/7b0a75a56f7048718ee1beaacd73f420.png)**

1.  Click on **OK \> OK \> OK \> Add \> Close**

\*\*  
\*\*

\*\*  
\*\*

1.  Validate that in the **Configure Settings** under **RADIUS Attributes** **\>** **Vendor Specific the value is displayed**

    ![](media/81d2304d63e5e105936b1f99483784ad.png)

2.  Click **Next** than **Finish**
3.  Repeat Steps 4 – 21 for **PRIV1_Policy** or to create additional Policies as needed.

## Step 9. Configure AAA with RADIUS on S1 and R1

1.  Configure S1 to use RADIUS

    Use the below commands

| aaa new-model radius-server host 192.168.100.1 key 88888888 aaa authentication login default group radius local aaa authorization exec default group radius local aaa authentication enable default group radius enable aaa authorization console |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

![](media/86459b57937f43d1dbc1dd5f1b5a9a1e.png)

The commands listed are used to enable and configure AAA (Authentication, Authorization, and Accounting) using RADIUS on a Cisco IOS device:

-   **aaa new-model:** This command enables AAA, replacing the older RADIUS and TACACS+ authentication methods.
-   **radius-server host 192.168.100.1 key 88888888**: This command sets up a RADIUS server at the IP address 192.168.100.1, and the shared secret key for communication between the router and RADIUS server is "88888888".
-   **aaa authentication login default group radius local**: This command sets the default login authentication method. It first tries to authenticate using the defined RADIUS server(s) (group radius). If that fails, for example, if the RADIUS server is not available, it falls back to using the local user database on the router (local).
-   **aaa authorization exec default group radius local**: This command sets up authorization (what users are allowed to do once they're authenticated) for exec sessions. The router will first attempt to authorize users against the RADIUS server, and if that fails, it will use the local user database.
-   **aaa authentication enable default group radius enable**: This command sets the authentication method for enable mode. It specifies that the RADIUS server should be used for authentication when the enable command is entered. If the RADIUS server is unavailable, the enable password on the router will be used.
-   **aaa authorization console**: This command enables AAA authorization for all network-related service requests that are not associated with a user terminal session, including SLIP and PPP, ARAP, NASI, X.25 PAD connections, and LAT terminal server host-initiated connections.

In summary, these commands are enabling AAA on the router, setting up a RADIUS server for AAA services, and configuring the router to use the RADIUS server and then the local user database for authentication and authorization for both login and enable mode. It's also enabling AAA authorization for non-terminal service requests.

\*Note that the commands listed may not match with your device there might be some syntax changes as some of the command may have been deprecated or be different please adjust accordingly.

1.  Validate that AAA is working by logging out the device and logging in with the read-only user

    ![](media/7da9a343eb27f1c003ade88ad1e6ea31.png)

| Username: | cisco    |
|-----------|----------|
| Password: | Test12!! |

\*Read-only user is only able to be EXEC Mode and is unable to enter Privileged EXEC MODE

1.  Validate the Admin user is working

![](media/3978cf9c2034871ac8a2dbcbe538b795.png)

| Username: | Netadmin   |
|-----------|------------|
| Password: | Secret\$\$ |

\*Netadmin account is set for privilege level 15 so you are automatically put in Privileged EXEC MODE

1.  Repeat steps 1-3 for R1 and validate that **R1** is also able to reach the RADIUS Server.

![](media/523b716f429df08b2e64eea9ad2aa8ba.png)

Finished!
