# Fools Mate, Revenge
#THM #medium #Web #burp-suite #prototype-pollution
## Problem: 
Server refuses to give the prize even after winning the game. Something is blocking it.
## Goal: 
Bypass the block and claim the prize.
## Technique: 
Used prototype pollution to set `unlocked` variable to true, which unlocked the prize.
#### Steps: 
###### 1) Analyzing the web application behaviour
As described in `Problem` section, the app accepts the checkmate but refuses to give the prize: 

<img width="1260" height="981" alt="image" src="https://github.com/user-attachments/assets/51593620-5261-4e8f-8a07-cd6d86dc7341" />

The server response after requesting the winning move is: 
```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 241
ETag: W/"f1-C9LMe/pSiyoulK1VHiWkWkzJzvY"
Date: Tue, 28 Jul 2026 20:29:52 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{"ok":true,"move":"a1a8","fen":"R5k1/5ppp/8/8/8/8/5PPP/6K1 b - - 1 1","status":"checkmate","turn":"b","winner":"white","locked":true,"message":"Checkmate! No reward for you.","reason":"reward gate closed: session.config.unlocked is not set"}
```
The server gave us the reason of blocking the prize which is `session.config.unlocked is not set`.
We then need to create the `unlocked` variable of config object. 
> [!NOTE]
>`session` and `config` ARE NOT classes, they are objects in JavaScript.
###### 2) Prototype pollution process
Instead of creating the unlocked variable of `config` object, we can create it for every object. 

To do that, we need to create the unlocked variable for the shared parent `Object.prototype`, which is the central base object in JavaScript from which nearly all other objects inherit properties and methods.

All we need to do is to inject this code to the server: 
```js
"constructor": {
    "prototype":{
        "unlocked": true
    }
}
```

> [!NOTE]
> obj.constructor (with obj as an object) refers to `Object` the built-in function that constructs plain objects.

The code is then injected within the settings request using burp suite: 

<img width="1442" height="592" alt="image" src="https://github.com/user-attachments/assets/1af2400a-0756-4a65-8a9b-4f99c4200d79" />

**Why this works for constructor and not for Object?**

`"Object"` as a JSON key is just a plain string, the server would set a harmless property literally named `Object`, with no link to the real, built-in `Object` function.
`"constructor"` is different: every object automatically has a `constructor` property already pointing to the function that created it (`Object`), and that function's `.prototype` is `Object.prototype`, the real shared prototype. So `"constructor": { "prototype": { "unlocked": true } }` walks a real, pre-existing path straight to the shared prototype, while `"Object"` alone leads nowhere.

###### 3) Getting the prize

<img width="1245" height="930" alt="image" src="https://github.com/user-attachments/assets/70a7bda7-ed11-484a-a4f4-69e603cf8ab2" />

## Root cause: 
The server trusted client request, and merged it directly to objects without filtering.
## Flag: 
`THM{<FLAG>}`
## Key takeaway: 
Never trust that user input stays isolated, always test if it can pollute shared objects via `__proto__` or `constructor.prototype`.
