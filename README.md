# active-directory-setup
Setting up on-premise active directory with Azure virtual machines.

<h2>Environments and Technologies Used</h2>

- Microsoft Azure (Virtual Machines/Compute)
- Remote Desktop
- Active Directory Domain Services
- PowerShell

<h2>Operating Systems Used </h2>

- Windows Server 2022
- Windows 10 (21H2)

## Creating our virtual machines

In this demosntration, I will be setting up and configuring Active Directory on Azure Virtual machines to simulate a business or school with many users that need different levels of permissions and access to do each of their jobs or tasks. A detailed overview can be found [here](https://www.quest.com/solutions/active-directory/what-is-active-directory.aspx) (the official Microsoft documentation is *extremely* long and complex), but Active Directory can be thought of as a system administration tool that  manages user permissions on very large scales, inside of a configured network where all computer traffic goes through the **domain controller**, which stores all the rules for each user or group that a user belongs to.

Creating Azure virtual machines in detail was covered in [Azure Basics 1](https://github.com/grrob015/azure-basics?tab=readme-ov-file#virtual-machines). For our Active Directory, create two virtual machines **on the same virtual network**. 

- The **domain controller** will use image "Windows Server 2022 Datacenter: Azure Edition x64 Gen2".
- The **client** computer will use Windows 10 Pro.

2 vcpus and 8 GiB of memory (Standard_D2s_v3) will be plenty for both virtual machines. Be sure to have notepad open to keep track of usernames and passwords across VMs.

## Domain Controller Network Settings

The first step is to make our domain controller's private IP address static. Later on, we will be assigning the domain controller as the DNS server for the client, so we need a permanent IP address on the network for our domain controller. This prevents mishaps if the domain controller is turned off and turned on again, as the default Azure settings are for the private IP address to be **dynamically** allocated. Dynamic IP addresses can change, and then our client would no longer be using our domain controller as its DNS server, causing problems to organization workflow.

In order to make our domain controller's IP address static, go to the Azure resource for your domain controller VM and go to network settings. You should see a green network interface card icon with ipconfig settings referenced, which is what we will need to modify. Click the blue text to configure your virtual network interface card.

![1  click the nic](https://github.com/user-attachments/assets/ed8f85dc-fd38-4193-911b-b17783148413)

Once inside your NIC (network interface card), click "ipconfig1", then change the "Private IP address settings" to "Static", so our client DNS settings never need to be updated.

![2  static ip address steps](https://github.com/user-attachments/assets/752e4cd7-f88f-4f85-8682-c18279046589)

Next, we'll need to log into our domain controller and disable windows firewall to make connecting to it from other machines easier. (⚠️ **This is not a cybersecurity course!** Don't do this if you're setting up active directory for a real business, _this is just to remove later steps that would be outside the scope of this demonstration!_) Get your domain controller's public IP address and log into it with remote desktop. A server manager interface should pop up, minimize that for now.

![3  windows server logon](https://github.com/user-attachments/assets/61ada8a7-d627-4639-b5df-7006b26b6e51)

Open Run either by pressing Windows + R or by searching for "run" in the start menu. Once you've opened it up, type `wf.msc` and press enter to bring up the Advanced Windows Defender settings.

![4  the transformation](https://github.com/user-attachments/assets/142846ce-e061-4ee4-92a6-a9b3e309b134)

Click "Windows Defender Firewall Properties" in the "Overview" section and disable the "Firewall State" option for all three sections - "Domain Profile", "Private Profile", and "Public Profile".

![5  disabling the firewall](https://github.com/user-attachments/assets/a5a9381b-913e-4a7e-9644-07442548f627)

Save the changes, and then log out of your domain controller's remote desktop.

## Client Network Settings

By default, the client computer will be "pointing" to Azure's DNS server. We need to change it to "point" towards our domain controller, so that our client resolves domain names with our active directory rules that we'll be setting up later. Go to your client VM's NIC settings (the green network card icon in Azure's "Network Settings", just like before) but this time, click on "DNS Servers" on the sidebar.

![6  client DNS settings](https://github.com/user-attachments/assets/ccf79eb5-88b8-4835-a359-bd0009c95a6c)

Switch the DNS Server to "Custom" and then put in your domain controller's private IP address.

## Fixing the "Invalid IP Address" Error

When changing the network settings on your client, this error might pop up:

![7  error](https://github.com/user-attachments/assets/e2814fc4-1cd9-4c98-a75d-14296e86762a)

This means that our network settings have not been properly configured. In Azure, search for Network Security Groups and click on the standard version (⚠️ **NOT** classic!). 

![8  newgen network security groups](https://github.com/user-attachments/assets/027d9fcc-0c63-44d6-bb29-5317123d257f)

Click into your client's network security group and select "Inbound security rules" from the "Settings" dropdown menu.

![9  client inbound nsg rules](https://github.com/user-attachments/assets/d99226bd-6b5b-4401-8038-1d0556128439)

Port 53 is where DNS communication between computers happens, so we need to allow any traffic coming in from port 53 and make it the top priority in our network security group. Put `53` into "Destination port ranges" and `290` into "Priority" and then create the rule. 

![10  da rules](https://github.com/user-attachments/assets/f2b4efcd-b557-4b3e-b324-f6282a05b41b)

⚠️ You will need to repeat this process for the outbound rules and your domain controller's network security group rules! Use the same destination port range and priority for each one, but create a total of **FOUR** new rules.

1. Client inbound rule
2. Client outbound rule
3. Domain Controller inbound rule
4. Domain Controller outbound rule

Once you have created all of the new rules, go to your Virtual Machines menu and restart both your client and your domain controller.

![11  restarting](https://github.com/user-attachments/assets/d8e929da-98f0-46f3-8ed5-c995ee71aaf1)

After your virtual machines restart, update the DNS settings again and it will work!

![12  hooray it work](https://github.com/user-attachments/assets/8e007684-4d08-4fdb-8fba-eeef6405ba9f)

Restart your client computer to apply its new network settings and you're finished with the client side!

💡 An extra thing you can do is to remote desktop into your client and run `ipconfig /all` into your powershell. Near the very bottom, you should see that your DNS server is the private IP address of your domain controller!

![13  cool thing](https://github.com/user-attachments/assets/8d53d0d0-aa94-419b-844b-09e51395096a)

## Installing Active Directory onto the Domain Controller

Log into your domain controller with Remote Desktop. An application called "Server Manager" should pop up, but if it doesn't you can always search for/select it in the Start Menu. Within the server manager, click "Add roles and features" to bring up an installation wizard.

![14  summon the wizard](https://github.com/user-attachments/assets/a06c3100-aa38-4382-9ab6-659d480a4625)

Inside the wizard:

1. On the "**Before You Begin**" page, click "Next".
2. On the "**Installation Type**" page, choose "Role-based or feature-based installation".
3. On the "**Server Selection**" page, select "Select a server from the server pool" and make sure your domain controller is highlighted.
4. On the "**Server Roles**" page, click "Active Directory Domain Services" ONLY (and add the features, of course).
5. On the "**Features**" page, click "Next".
6. On the "**AD DS**" page, click "Next".
7. On the "**Confirmation**" page, tick the "Restart the destination server automatically if required" box and then click "Install".

Once Active Directory is finished installing, click the notification flag in the top right corner and select "Promote this server to a domain controller".

![15  promotion](https://github.com/user-attachments/assets/8e83accb-030d-4448-9472-eb5984274c79)

Another configuration wizard will appear, this time for Active Directory Domain Services specifically. Inside the wizard:

1. On the "**Deployment Configuration**" page, select "Add a new forest" and specify a domain name. I will be using `bigplasticco.com`.
2. On the "**Domain Controller Options**" page, set a Directory Services Restore Mode password and leave the other settings default.
3. On the "**DNS Options**" page, uncheck the "Create DNS delegation" box.
4. On the "**Additional Options**" page, click "Next".
5. On the "**Paths**" page, click "Next".
6. On the "**Review Options**" page, click "Next".
7. On the "**Prerequisite Check**" page, click "Install".

Once the wizard is finished installing, your domain controller will automatically restart. Congratulations, you now have a domain controller with active directory domain services installed!

## Creating Accounts in Active Directory

Restarting the domain controller kicked us out of our remote desktop session, so we'll need to log in again. In order to log in this time, though, we'll need to specify the domain we're logging into along with the user. Use `domain\user` in the username portion of the login along with the same password and you should be able to connect.

![16  logging in with domain](https://github.com/user-attachments/assets/4883188a-8065-454f-96ad-2eaaa17cfef1)



