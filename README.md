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

## Creating an Admin Account in Active Directory

Restarting the domain controller kicked us out of our remote desktop session, so we'll need to log in again. In order to log in this time, though, we'll need to specify the domain we're logging into along with the user. Use `domain\user` in the username portion of the login along with the same password and you should be able to connect.

![16  logging in with domain](https://github.com/user-attachments/assets/4883188a-8065-454f-96ad-2eaaa17cfef1)

Once inside, open the Start menu and click "Windows Administrative Tools", then select "Active Directory Users and Computers" from the dropdown menu.

![17  aduc not a duck](https://github.com/user-attachments/assets/2822f50a-f109-4488-90c3-799bb738846c)

Right click your domain name, and create two new organizational units, one called `_EMPLOYEES` and one called `_ADMINS` EXACTLY. We will be using a script later to create random employees and PowerShell will look for those exact names to place the employees in.

💡 Active Directory Domain Services has a bunch of built-in user groups that you can see by clicking the "Users" folder under your domain name. They have all sorts of preconfigured permissions and overlapped levels of access, so we'll be using our own simplified groups, but feel free to explore and read about the default groups to gain insight about Active Directory permissions management.

![18  two new ous](https://github.com/user-attachments/assets/417ad9b4-845a-4c8a-b4ed-919b4d0f6f0f)

Now, we will create an administrator we can roleplay as later. Let's hire Jane Doe again. Create a new user in our admin folder by right clicking "_ADMINS" and selecting **New -> User**. Fill out her information, and set a new password for her. Untick the "User must change password at next logon" box unless you'd like to see what that UI looks like from the user's point of view. 💡 Notice the similarities in the settings between Active Directory and osTicket!

![19  jane doe admin extraordinare](https://github.com/user-attachments/assets/95ba70bb-06a2-4b0f-b92c-db022f7da409)

In order to give Jane a set of permissions that will allow her to perform admin activities on our domain, let's add her to the preconfigured "Domain Admins" user group by doing the following:

1. Right click Jane's user and select "Properties".
2. Go to "Member of", and click "Add".
3. Type "domain admins" into the text box and then press Enter. (You can also click "check names" to be sure that what you typed is a valid group. If it is, it will underline itself and correctly capitalize itself.)

![20  3 step jane admin process](https://github.com/user-attachments/assets/97e80409-6fa1-4ae7-9199-39993af31ab1)

Once you're finished, log out of your current session and try logging in as Jane! Remember to use `domain\user` in the username portion of your RDC client!

![21  jane can log in](https://github.com/user-attachments/assets/a05756f1-859d-4d80-a50d-b55e15588bec)

## Joining the Client to the Domain

Our domain controller has active directory installed and has been promoted, and our client is using it as its DNS Server, but our client isn't part of the domain yet. Remote desktop into your client computer with your original user and password you set on it (your domain name won't be recognized if you try to use it, and Jane won't be able to log in).

Once inside, do the following to add the client to the domain:

1. Right click the Windows Icon in the bottom right corner and click on "System".
2. Click on "Rename this PC (advanced)".
3. Click "Change" to change the domain the computer belongs to.
4. Type your domain name into the text box and click "OK".

![22  4 step process to join to domain](https://github.com/user-attachments/assets/18069faf-f738-4496-aa51-f02a571352fb)

The client will then ask for an account's credentials that has permission to join the computer to the domain. Input Jane's credentials, remembering to specify the domain.

![23  jane doe does sysadmin work](https://github.com/user-attachments/assets/4a851eed-c704-405f-a78d-337a898687a5)

A popup will appear behind your other menus welcoming you to the domain!

![24  welcome](https://github.com/user-attachments/assets/5812b356-456d-4c88-8a7d-3d61d7d184ff)

After this, the comptuer will restart automatically to apply the changes. Congratulations on adding a client to your Active Directory Domain! To check on client computers from your domain controller, open "Active Directory Users and Computers" and look in the "Computers" folder under your domain name. You should see your other virtual machine and be able to view properties about it!

![25  client registration](https://github.com/user-attachments/assets/b764ef28-208d-44b3-a1c1-1800a531dee4)

## Allowing "Regular" Employees to Log On to the Client

Remote desktop into your client virtual machine as Jane Doe (we can do that since the client is part of the domain now!). Remember that the username will be `domain\user`!

Once you're logged in, do the following to allow anyone on the domain to access this client remotely:

1. Right click the Windows icon in the bottom left of your screen and click "System".
2. Select "Remote Desktop" from the sidebar and click "Select users that can remotely access this PC".
3. Click "Add" in the pop-up menu.
4. Add "Domain Users" as an object.

![26  domain user remote desktop](https://github.com/user-attachments/assets/ab0ccbef-1a11-4377-9a29-194a0b9d7c9f)

We can now log onto our client as a normal, non-admin user. 💡 Take a look at the "Remote Desktop Users" dialog box (step 3). Notice how it says "...any members of the Administrators group can connect even if they are not listed." That's how Jane got to log in.

## Creating Normal Employees

Now it's time to hire a bunch of new people to have as employees so we can manage their permissions! Log onto your domain controller as Jane Doe and open PowerShell ISE as an administrator (search for it in the Start menu and right click it for the option). Click the arrow next to the word "Script" in the top right corner to open the script pane we'll be using to make our employees. Paste the contents of [this script](https://github.com/joshmadakor1/AD_PS/blob/master/Generate-Names-Create-Users.ps1) into the Script pane, adjust the `$NUMBER_OF_ACCOUNTS_TO_CREATE` variable at the top to something more easily manageable, and then press F5 to run it (The green play button icon at the top middle also runs the script)!

![27  many new employees](https://github.com/user-attachments/assets/4bb095b6-3bc3-425d-917e-0fde3a6ad52f)

Notice how in our Active Directory Users and Computers tool, we can see that we now have ten fresh hires in our system! Everyone's password is `Password1`, so our final step will be to see if we can log into the client computer as a normal employee. Let's get Ken's account, `ken.xibu` (if you followed along, your names will be different!). Your username will be `domain\user`, just like always!

![28  ken's account](https://github.com/user-attachments/assets/a8eb014d-8211-4ffa-a6b4-ace344d3ac03)

Ken shouldn't have any trouble logging in for his first day of work!

This concludes installing and configuring on-premise active directory domain services. I will be showing some examples of business scenarios that might happen and explore different aspects of Active Directory's functionality [here](https://github.com/grrob015/active-directory-examples). Thank you for reading! 🙂






