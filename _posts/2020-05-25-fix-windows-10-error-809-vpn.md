---
layout: post
title: Fix Windows 10 Error 809, L2TP/IPSec VPN
# subtitle: Excerpt from Soulshaping by Jeff Brown
cover-img: https://miro.medium.com/max/700/0*iuVrmoYNMxFRdNl6
tags: [books, test]
---
With the work from home rising in the last period, a lot of workers are distributed to their home offices, and connect to the companies servers via VPN. Setting a VPN sometimes is not ideal like in the documentations and problems arise. This is one I had, and here is how I solved it.

## Some technical stuff

By default, Windows client and the Windows Server operating system do not support Internet Protocol security (IPsec) network address translation (NAT) Traversal (NAT-T) security associations to servers that are located behind a NAT device. Therefore, if the virtual private network (VPN) server is behind a NAT device, a Windows based VPN client computer or a Windows Server based VPN client computer cannot make a Layer Two Tunneling Protocol (L2TP)/IPsec connection to the VPN server. This scenario includes VPN servers that are running Windows Server 2008/R2, Windows Server 2012/16 and Microsoft Windows Server 2003.

Because of the way in which NAT devices translate network traffic, you may experience unexpected results when you put a server behind a NAT device and then use an IPsec NAT-T environment. Therefore, if you must have IPsec for communication, it is recommended that you use public IP addresses for all servers that you can connect to from the Internet. However, if you have to put a server behind a NAT device and then use an IPsec NAT-T environment, you can enable communication by changing a registry value on the VPN client computer and the VPN server.

## The Solution

You will need to to create and configure the `AssumeUDPEncapsulationContextOnSendRule` registry value using the windows registry.

### Disclaimer: Editing the Windows Registry file is a serious undertaking. A corrupted Windows Registry file could render your computer inoperable, requiring a reinstallation of the Windows 10 operating system and potential loss of data. Back up the Windows 10 Registry file and create a valid restore point before you proceed.

To apply the fix follow these steps:

1. Log on to the Windows client computer as a user who is a member of the Administrators group.
2. Click Start, point to All Programs, click Accessories, click Run, type `regedit`, and then click OK. If the User Account Control dialog box is displayed on the screen and prompts you to elevate your administrator token, click Continue.
3. Locate and then click the following registry subkey: `HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\PolicyAgent`
4. On the Edit menu, point to New, and then click DWORD (32-bit) Value.
5. Type `AssumeUDPEncapsulationContextOnSendRule`, and then press ENTER.
6. Right-click `AssumeUDPEncapsulationContextOnSendRule`, and then click Modify.
7. In the Value Data box, type 2. *A value of 2 configures Windows so that it can establish security associations when both the server and the Windows Vista-based or Windows Server based VPN client computer are behind NAT devices.*
8. Click OK, and then exit Registry Editor.
9. Restart the computer.
