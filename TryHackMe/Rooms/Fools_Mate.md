# Fools Mate
#THM #Web #burp-suite #easy #missing-server-validation
## Problem:
Chess engine doesn't let you make the checkmate move, had to bypass it.

<img width="1208" height="1307" alt="image" src="https://github.com/user-attachments/assets/fcf0dd97-e8d2-44ea-a717-1dbfc8ab870e" />

## Goal:
Bypass the restriction blocking the checkmate move.
## Technique:
Used burp suite to send a custom request for the checkmate move.
#### Steps:
###### 1) Analyse the requests made by the application

<img width="2340" height="896" alt="image" src="https://github.com/user-attachments/assets/a0b10063-a2e6-422c-9a8e-38b9ade5d6e7" />

The app requested for a move from a1 to a7. We can modify the request to make the checkmate.
###### 2) Sending custom request using repeater

<img width="1221" height="539" alt="image" src="https://github.com/user-attachments/assets/c67248a9-7dba-49c8-9a14-4dd5e8c2adad" />

After requesting for the move a1 to a8 (which is a checkmate), the server responded with the flag and a status of checkmate.
## Root cause:
The server never checked if the move actually  resulted in a checkmate (move validation happened only in the browser), it just trusted whatever move the client sent.
## Flag:
`THM{<FLAG>}`
## Key takeaway:
Never trust the frontend, always test the actual API with Burp to see if the server enforces the same rules.
