# Natas11
#natas #crypto
## Vulnerability:
Insecure encryption / weak cookie-based authentication due to a known plaintext attack on XOR encryption
## Technique:
Created a custom cookie that allows access to the password for the next level.
#### Steps:
###### 1) Analyzing the source code 
**Source Code:**
```php
<?
$defaultdata = array("showpassword" => "no", "bgcolor" => "#ffffff");

function xor_encrypt($in) {
    $key = '<censored>';
    $text = $in;
    $outText = '';

    for ($i = 0; $i < strlen($text); $i++) {
        $outText .= $text[$i] ^ $key[$i % strlen($key)];
    }

    return $outText;
}

function loadData($def) {
    global $_COOKIE;

    $mydata = $def;

    if (array_key_exists("data", $_COOKIE)) {

        $tempdata = json_decode(
            xor_encrypt(base64_decode($_COOKIE["data"])),
            true
        );

        if (
            is_array($tempdata) &&
            array_key_exists("showpassword", $tempdata) &&
            array_key_exists("bgcolor", $tempdata)
        ) {
            if (preg_match('/^#(?:[a-f\d]{6})$/i', $tempdata['bgcolor'])) {
                $mydata['showpassword'] = $tempdata['showpassword'];
                $mydata['bgcolor'] = $tempdata['bgcolor'];
            }
        }
    }

    return $mydata;
}

function saveData($d) {
    setcookie(
        "data",
        base64_encode(xor_encrypt(json_encode($d)))
    );
}

$data = loadData($defaultdata);

if (array_key_exists("bgcolor", $_REQUEST)) {
    if (preg_match('/^#(?:[a-f\d]{6})$/i', $_REQUEST['bgcolor'])) {
        $data['bgcolor'] = $_REQUEST['bgcolor'];
    }
}

saveData($data);

?>

<h1>natas11</h1>

<div id="content">

<body style="background: <?= $data['bgcolor'] ?>;">

Cookies are protected with XOR encryption
<br><br>

<?php
if ($data["showpassword"] == "yes") {
    print "The password for natas12 is <censored><br>";
}
?>

<form>
    Background color:
    <input name="bgcolor" value="<?= $data['bgcolor'] ?>">
    <input type="submit" value="Set color">
</form>

<div id="viewsource">
    <a href="index-source.html">View sourcecode</a>
</div>

</div>

</body>
```
This demonstrates that the application uses XOR weak encryption to protect the cookie.
The exact algorithm is the following (for default data):
```
{"showpassword":"no","bgcolor":"#ffffff"}
                   |
                   |
            XOR with the key
                   |
                   |
            Base 64 encoding      
                   |
                   |
                 Cookie   
```
Since the key is repeatedly reused, it is vulnerable to a known-plaintext attack.
###### 2) Finding the XOR key
As we know, XOR is reversible, which means:
```
result = text xor key <-> key = result xor text
```
We can find the result (the cookie) in developer tools. 
And the text is known (`{"showpassword":"no","bgcolor":"#ffffff"}`) as we didn't change the background color in the main page.
Then we can use CyberChef to find the key:

<img width="1976" height="977" alt="image" src="https://github.com/user-attachments/assets/6dce4e10-ad3b-43a6-9fe1-59399716a819" />

CyberChef recovered the repeating XOR key: `kBSw`.

> [!NOTE]
> The URL-encoded characters `%2F` and `%3D` were converted to `/` and `=` before Base64 decoding the cookie.

###### 3) Creating a custom cookie
We'll use the key `kBSw` to encrypt the plaintext `{"showpassword":"yes","bgcolor":"#ffffff"}` and replacing the original cookie in Developer tools with the newly generated one.
**New cookie:**
```
EGAgHwQ1IxYYMSQYGSZxTUk7NgRJbnEVDCE8GwQwcU1JYTURDSQ1EUk/
```
**Result:**

<img width="1399" height="551" alt="image" src="https://github.com/user-attachments/assets/6bb547c2-d5d8-468e-aada-8ff1419076b9" />

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas10](./Natas10.md)
## Next Level:
[Natas12](./Natas12.md)
