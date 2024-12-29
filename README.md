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

By default, the client computer will be "pointing" to Azure's DNS server. We need to change it to "point" towards our domain controller, so that our client resolves domain names with our active directory rules that we'll be setting up later.

