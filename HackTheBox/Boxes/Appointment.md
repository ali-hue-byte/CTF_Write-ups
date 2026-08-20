# Appointment
#HTB #very-easy #sql-injection #Web #databases
## Target: 
*Name:* Appointment

*IP:* 10.129.128.188
## Vulnerability:
SQL injection, unsanitized user input allows authentication bypass.
## Steps: 
#### 1) Reconnaissance
Used nmap to check for open ports.
**Code:** 
```
nmap -sV 10.129.128.188
```
**Result:**
```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-30 05:26 -0400
Nmap scan report for 10.129.128.188
Host is up (0.039s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.38 ((Debian))

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.86 seconds
```
Only http is running on the target machine.
#### 2) Opening the website in the browser

<img width="2559" height="1315" alt="image" src="https://github.com/user-attachments/assets/b3b0ee4f-34fb-4378-bc58-1ebd45a690b0" />

#### 3) Bypassing login page
We can try to login without password using sql injection attack, this can be done using `' OR 1=1 #` in the username field, and anything in password field: 

<img width="2559" height="1362" alt="image" src="https://github.com/user-attachments/assets/6dff8278-2cb5-41b9-8df1-4ca823f8fbcc" />

**Result:**

<img width="1600" height="849" alt="image" src="https://github.com/user-attachments/assets/10cf5a0d-6b36-4501-9114-083281ee158b" />

This works because the server builds a SQL query like:
`SELECT * FROM users WHERE username='INPUT' AND password='INPUT'`
Injecting `' OR 1=1 #` (`SELECT * FROM users WHERE username='' OR 1=1 # AND password='INPUT'`) closes the username string early, adds a condition that's always true (`1=1`), and comments out the rest of the query (including the password check), so the database returns a valid user regardless of credentials.
## Flag: 
[Not disclosed — solve it yourself!]
## Key takeaway:
Login forms that pass user input directly to SQL queries can be bypassed with `' OR 1=1 #`, always test for SQL injection.
