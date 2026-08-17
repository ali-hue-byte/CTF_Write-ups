#natas #directory-listing #information-disclosure #credential-leak

## Vulnerability:

Source code revealed a `files/` directory: 
![[Pasted image 20260726152421.png]]
## Technique:
Navigated to `http://natas2.natas.labs.overthewire.org/files/` to check for directory listing. `users.txt` file was found and holds the password for the next level.
#### Steps:
###### 1) Navigating to directory `files/`

<img width="2559" height="1244" alt="image" src="https://github.com/user-attachments/assets/f83c927d-4ecc-4939-9bbf-fff49663e96e" />

###### 2) Opening `users.txt` 

<img width="1312" height="590" alt="image" src="https://github.com/user-attachments/assets/62b78cda-956d-4d73-be8c-cca7570c04d3" />

`users.txt` leaked credentials for multiple accounts (alice, bob, charlie, eve, mallory)
## Password found: 
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas1](./Natas1.md)
## Next Level:
[Natas3](./Natas3.md)
