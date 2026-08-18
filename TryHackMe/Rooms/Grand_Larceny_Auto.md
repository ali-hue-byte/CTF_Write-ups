# Grand Larceny Auto
#THM #games #memory #medium #cheat #cheat-engine #dnSpy
## Problem: 
Developers made an inaccessible vault that no one has opened using normal gameplay.
## Goal: 
Open the vault to retrieve the flag.
## Technique: 
Reverse engineered the game to look for the condition required to open the vault, then used cheat engine to force that condition.
#### Steps:
###### 1) Reverse engineer the game using dnSpy
After opening the game files, GrandLarcenyAuto.dll appears to contain game functions and logic.

<img width="2544" height="879" alt="image" src="https://github.com/user-attachments/assets/7b0a25bf-d981-465e-977e-a08e3570f554" />

This code was found in the SafehouseVault class: 
```c#
public string TryOpen()  
{    
     if (this.player.WantedStars >= 6)
```
Which indicates that we can try to open the vault only when Wanted Stars are >= 6.
The developers also hard coded the exact amount of stars to open the vault: 
```c#
private const int UnlockStars = 6;
```
This confirms that we need 6 stars to open the vault, while the gameplay only allows 5 stars.

<img width="1150" height="646" alt="image" src="https://github.com/user-attachments/assets/09e3b6e9-c429-4504-bd62-7c3ff0ff0489" />

###### 2) Editing memory using cheat engine
After multiple scans, the exact memory address of the WantedStars value was found, then edited to hold the value 6: 
<img width="2282" height="874" alt="image" src="https://github.com/user-attachments/assets/32d6ba26-dd1d-49ec-9ff7-a893e01a53a0" />
###### 3) Opening the vault

<img width="1133" height="536" alt="image" src="https://github.com/user-attachments/assets/fd7c8086-caed-4b49-aca6-b9cdce49ac3a" />

## Flag: 
`THM{<FLAG>}`
## Root cause: 
Developer hardcoded the exact amount of stars to open the vault.
## Key takeaway:
- Hard coding sensitive information can be very dangerous.
- Never use player side data for something critical like an encryption key. 
