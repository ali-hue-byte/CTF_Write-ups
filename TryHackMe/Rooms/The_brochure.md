# The brochure
#THM #easy #OSINT
## Goal: 
Follow the trail from the photo to the account that posted it, then keep tracing further to find a hidden person the hotel never mentioned.
## Technique: 
Used basic searching to find the resort's social media account.
#### Steps:
###### 1) Analyzing the AI generated image
The image hints "Find us on Instagram or not", confirming the account to look for is on Instagram.
###### 2) Searched Instagram for "Byte Lotus Resort" and found the official account: `@<ACCOUNT>`.
- 2 posts
- 209 followers
- Follows 1 account
###### 3) Checked who `@<ACCOUNT>` follows, led to `@<ACCOUNT2>`. 
- Bio: "Currently working for Byte Lotus Hotel", confirms this is the "<NAME>" mentioned on the image as concierge. 
- The posts contains what looks like an encoded string.

###### 4) Decoding the string using base64decode.org

<img width="1312" height="1101" alt="image" src="https://github.com/user-attachments/assets/7ea23a4c-5022-4fe5-869c-1f6c91c32f91" />

## Flag: 
`THM{<FLAG>}`




