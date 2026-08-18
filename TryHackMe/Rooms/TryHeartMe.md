# TryHeartMe
#THM #Web #JWT #burp-suite #easy
## Problem:
The shop shows 0 credits, and there's nothing purchasable with them, the hidden item can't be reached through normal browsing or shopping. 
## Goal:
Access the hidden item in the Valentine's gift shop.
## Technique: 
Used Burp Suite to inspect the app's traffic and found it stores user role and credits in a JWT cookie. Decoded the JWT, modified the payload (role → admin, credits → 999999), re-signed it with an arbitrary key, and replaced the browser's cookie with the forged token, the server accepted it without properly verifying the signature.
#### Steps:
###### 1) Analyzing the web application
Tested the application in Burp Suite. The account creation request revealed the app uses a JWT cookie, which was decoded using `jwt.io`.

***Create account request in Burp Suite***

<img width="2446" height="847" alt="image" src="https://github.com/user-attachments/assets/9574c98e-63b5-4d88-a053-88bc6c95178e" />

###### 2) Modifying the jwt cookie

The jwt cookie was modified to set the role as admin and credits as `999999`.

<img width="2307" height="1399" alt="image" src="https://github.com/user-attachments/assets/5e516834-984a-4d4c-9005-5f1c8c672537" />

###### 3) Using the modified cookie

The session cookie is then changed in the application section of Develloper tools in the Firefox browser: 

<img width="2227" height="1423" alt="image" src="https://github.com/user-attachments/assets/5d40508a-f6b3-419a-b5bc-873ed90938c0" />

###### 4) Buying the hidden item

The hidden item appeared in the shop list: 

<img width="2125" height="1398" alt="image" src="https://github.com/user-attachments/assets/fd1f77df-5f3a-4328-94ee-76e30b43f94d" />

<img width="1244" height="439" alt="image" src="https://github.com/user-attachments/assets/30276475-d431-4e7c-8778-3cf26afb2dbe" />

## Root cause: 
The server trusted the JWT payload without properly verifying its signature.
## Flag: 
`THM{<FLAG>}`
## Key Takeaway: 
JWTs are only as secure as their signature verification — if the server doesn't properly check the signature (or accepts weak/no verification), an attacker can freely forge the payload to escalate privileges or fake data.
