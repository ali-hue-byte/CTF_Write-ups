# Natas7
#natas #LFI
## Vulnerability:
Local file inclusion. The application uses user-controlled input to load local files without proper validation or restriction. This allows an attacker to include and read local files, including sensitive system files.
## Technique:
Modified the `page` GET parameter to reference `/etc/natas_webpass/natas8`.
#### Steps:
###### 1) Viewing source code
The source code revealed a hint about the password file location:
```html
<div id="content">
  <a href="index.php?page=home">Home</a>
  <a href="index.php?page=about">About</a>
  <br>
  <br>
  this is the about page
  <!--hint: password for webuser natas8 is in /etc/natas_webpass/natas8-->
</div>
```
###### 2) Modifying the `page` parameter 
Replaced the value of the `page` parameter with `/etc/natas_webpass/natas8`.

<img width="1600" height="541" alt="image" src="https://github.com/user-attachments/assets/c54fa2f5-24f1-4414-9bbb-30f35f47df14" />


## Password found:
[Not disclosed — solve it yourself!]
## Previous Level:
[Natas6](./Natas6.md)
## Next Level:
[Natas8](./Natas8.md)
