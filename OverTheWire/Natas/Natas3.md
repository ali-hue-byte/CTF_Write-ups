# Natas3
#natas #robots-txt #information-disclosure #directory-listing
## Vulnerability:
`robots.txt` disallowed a hidden directory from search engine crawlers, but the path itself was still publicly accessible to anyone who read the file directly.

*A **search engine crawler** (also called a "bot" or "spider") is an automated program that search engines like Google use to browse the internet and build their index — so when you search something on Google, it can actually return relevant results.*
## Technique:
Used `gobuster` to brute force directories, which revealed `robots.txt`. Then opened `robots.txt` then showed a hidden directory, which was visited directly to find the next password.
##### Steps:
###### 1) Using `gobuster` to brute force directories
**Code:** 
```bash
gobuster dir -u http://natas3:<NATAS2_PASSWORD>@natas3.natas.labs.overthewire.org/ -w /usr/share/wordlists/dirb/common.txt 
```
*Note: Username and password can be included in the URL*
**Result:** 
```bash
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://natas3:<NATAS2_PASSWORD>@natas3.natas.labs.overthewire.org/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 338]
.htaccess            (Status: 403) [Size: 338]
.htpasswd            (Status: 403) [Size: 338]
cgi-bin/             (Status: 403) [Size: 338]
index.html           (Status: 200) [Size: 923]
robots.txt           (Status: 200) [Size: 33]
server-status        (Status: 403) [Size: 338]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```
**Gobuster results breakdown**:
-`.hta (403)` — Apache config file, blocked (expected)  
-`.htaccess (403)` — Apache config file, blocked (expected)  
-`.htpasswd (403)` — Apache password file, blocked (expected)  
-`cgi-bin/ (403)` — legacy CGI directory, blocked  
-`index.html (200)` — homepage, nothing notable  
-`robots.txt (200)` — key file; contents revealed the hidden path  
-`server-status (403)` — Apache status page, blocked 
###### 2) Opening `robots.txt` file in browser

<img width="838" height="257" alt="image" src="https://github.com/user-attachments/assets/e07b9fa5-6892-464c-91bd-71c2704a58de" />

The file exposed the hidden folder `/s3cr3t/`
###### 3) Opening the `/s3cr3t/` folder in browser

**URL:** `http://natas3.natas.labs.overthewire.org/s3cr3t/`

Directory listing was enabled, revealing a `users.txt` file containing the password for natas4.

<img width="2447" height="1155" alt="image" src="https://github.com/user-attachments/assets/02ff75a6-95ac-415a-ae6a-8455649aea8e" />

###### 4) Opening `users.txt` file

<img width="1145" height="266" alt="image" src="https://github.com/user-attachments/assets/3b21f659-43d6-4e19-9ccc-d10c358ac455" />


## Password found: 
[Not disclosed — solve it yourself!]


## Previous Level:
[Natas2](./Natas2.md)
## Next Level:
[Natas4](./Natas4.md)
