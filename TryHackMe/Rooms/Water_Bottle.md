# Water Bottle
#THM #OSINT #easy 
## Goal: 
Find the water station name and contact number using OSINT.
## Technique: 
Used google and google maps to find the information. 
#### Steps: 
###### 1) Creating a list of water stations
Searched Google Maps for water stations near Boni Avenue. 
Found that one location previously operated as `<NAME>` (2014) and has since been replaced with `<New>` (2026), this was the only station matching that change pattern.

<img width="1600" height="862" alt="image" src="https://github.com/user-attachments/assets/2af1df0b-c4a0-4c0a-b5fa-3b8c38d2a857" />

###### 2) Searching for contact number 

The contact numbers available in the store banner don't match the number provided (starts with `63922`).

A google search helped to find the correct phone number, which is: `63922_______`

<img width="1362" height="1332" alt="image" src="https://github.com/user-attachments/assets/44c3a5f1-1ff5-4cdc-8741-fbfd4e385404" />

## Flag: 

`THM{<NAME>_63922_______}`
