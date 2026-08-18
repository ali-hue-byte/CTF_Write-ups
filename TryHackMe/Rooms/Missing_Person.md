# Missing Person
#THM #OSINT #meta-data #easy
## Goal: 
Track a missing person using two photos and a message, by answering some questions.
## Technique: 
Combined extracting metadata and google searching to find valuable information.
#### Steps:
###### 1) Circuit name — Google search

<img width="1600" height="400" alt="image" src="https://github.com/user-attachments/assets/124be984-d6c6-4e0e-943d-cbee0c63408d" />

*Result:* `Pertamina <.........> Circuit`

###### 2) Event date — Google search

<img width="1600" height="944" alt="image" src="https://github.com/user-attachments/assets/0c0da2ab-1ac1-4ff8-bd96-3d1ecccca69b" />

*Result:* `<DATE>`

###### 3) Restaurant name — image search
<img width="1600" height="1109" alt="image" src="https://github.com/user-attachments/assets/80fd6897-399a-4627-ba1b-ecc1c9a8dd23" />

*Result:* `<NAME>`

###### 4) Time photo was taken — extracted EXIF metadata from the restaurant photo

<img width="677" height="91" alt="image" src="https://github.com/user-attachments/assets/b7b88ff6-f9be-424b-bfc4-e079896a9b7b" />

*Result:* The photo was taken at `<TIME>`
###### 5) Bar name and location — Google search led to a Facebook page

<img width="1600" height="870" alt="image" src="https://github.com/user-attachments/assets/bed3d833-2394-43cd-854c-38011811e130" />

The search revealed the facebook account of the bar named `<NAME>`.
Its location is found using google maps: 

<img width="835" height="981" alt="image" src="https://github.com/user-attachments/assets/9d49a993-a4bc-4211-90c7-a1d30fbbaf23" />

*Result:* `<ADDRESS>`

###### 6) DJ name — found in the same Facebook reel as the bar

<img width="649" height="1152" alt="image" src="https://github.com/user-attachments/assets/c68e57ad-fc99-45c2-81b7-b2f700553b7d" />

*Result:* `<NAME>` 

###### 7) Cave name and DJ's phone number — further searching for "<DJ_NAME>" on Facebook

<img width="1545" height="405" alt="image" src="https://github.com/user-attachments/assets/0ffc87c3-2ee8-45d9-8f1b-4a1f07522531" />

*Result:* 
- *Cave*: `<NAME>`
- *Phone number*: `<NUMBER>`

## Key takeaway:
Small details in photos and social media posts can be chained together to reconstruct someone's movements.





