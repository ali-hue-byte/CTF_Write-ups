# Natas23
#natas #logic 
## Vulnerability:
**Improper password validation:** The application validates the password using a substring check (`strstr`) and a weak numeric comparison (`> 10`). Because `strstr()` only verifies that the required string exists somewhere and PHP automatically converts numeric strings during comparison, an attacker can provide a value that satisfies both conditions without knowing the actual password.
## Technique:
Bypassed password validation by provided a password that satisfies both conditions.
#### Steps:
###### 1) Analyzing the source code
**Source Code:**
```php
<?php
if (array_key_exists("passwd", $_REQUEST)) {
    if (strstr($_REQUEST["passwd"], "iloveyou") && ($_REQUEST["passwd"] > 10)) {
        echo "<br>The credentials for the next level are:<br>";
        echo "<pre>Username: natas24 Password: <censored></pre>";
    } else {
        echo "<br>Wrong!<br>";
    }
}
// morla / 10111
?>
```
The application validates the password by checking if it contains the string `"iloveyou"`, and if its value is greater than 10. During this comparison, PHP performs type juggling and attempts to convert the string into a numeric value.
> [!NOTE] 
> The developer probably wanted to check if the password's length is greater than 10, but the correct code for this comparison is `strlen($_REQUEST["passwd"]) > 10`.
###### 2) Exploiting the vulnerability
We used a password that satisfies both conditions, for example:
```
11iloveyou
```
`strstr()` function will return true because the string contains `iloveyou`.During the numeric comparison, PHP converts the string to the number `11` because it starts with it. Since `11` is greater than `10`, the second condition is also satisfied, allowing the password validation to be bypassed.
**Result:**

<img width="1496" height="626" alt="image" src="https://github.com/user-attachments/assets/e2428424-7779-4e01-ad09-6f6422070e05" />

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas22](./Natas22.md)
## Next Level:
[Natas24](./Natas24.md)
