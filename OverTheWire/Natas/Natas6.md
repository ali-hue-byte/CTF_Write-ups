# Natas6
#natas #information-disclosure #misconfiguration
## Vulnerability:
The web server allows direct access to an internal `.inc` file containing sensitive information. Since the file is publicly accessible, an attacker can retrieve the secret by requesting its path directly.
## Technique:
After reviewing the provided source code, the path to the secret file (`/includes/secret.inc`) was identified. By navigating directly to this path in the browser, the server returned the contents of the file, exposing the secret.
```
<?
$secret = "<REDACTED>";
?>
```
Then the password was retrieved using the secret:

<img width="1162" height="422" alt="image" src="https://github.com/user-attachments/assets/82ae6867-7b1a-4f54-aed4-d598d3822085" />

## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas5](./Natas5.md)
## Next Level:
[Natas7](./Natas7.md)
