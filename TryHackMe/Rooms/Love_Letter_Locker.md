# Love Letter Locker
#THM #Web #burp-suite #IDOR #easy
## Goal:
Access other users letters.
## Technique:
Used Burp Suite to send custom requests that returns other users letters.
#### Steps: 
###### 1) Analyzing the web application behaviour 
After creating an account, the app shows a tip from Cupid, which reveals that every letter is assigned a unique number in the archive.

**Lead:** Cupid's tip hints that letters are numbered sequentially, likely an IDOR vulnerability, where changing the letter ID in a request could expose other users' letters.

<img width="1254" height="535" alt="image" src="https://github.com/user-attachments/assets/070ae759-aea9-44c4-aac6-d6340389495e" />

Burp is used to analyze the requests made by the application. After creating a letter, the request for viewing it appears to use IDs to get the letters from the server.

<img width="1600" height="500" alt="image" src="https://github.com/user-attachments/assets/02eda670-e4af-43e0-8935-0c52536ef72c" />

###### 2) Using custom request to get other users letters 
The letter ID is then changed to get other's letters, this can be done using burp suite and changing the letter's ID in the request, or directly changing it from the URL and viewing it in the browser.

<img width="1600" height="421" alt="image" src="https://github.com/user-attachments/assets/768f7188-a638-4ba8-b2ea-88314a86a5ba" />

## Root cause: 
The server retrieved letters by ID without verifying that the requesting user actually owned that letter. 
## Flag: 
`THM{<FLAG>}`
## Key takeaway: 
Never trust that a user will only request their own data, the server must always verify ownership on every object access.
