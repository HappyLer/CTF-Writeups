WriteUp for Troopers20 Student Challenge FishTank
=================================================

After solving the UnicComm challenge, I decided to go for the FishTank challenge next.
Before entering, the description text reads:
>When storing credentials and other secrets in cloud solutions deep trust in the provider is required. At FishBowl, we value security over data collection. Therefore, >we built our 100% free and secure credential storage FishTank to never send any data to our servers. All your secrets stay complete on your machine, protecting your >data against attacks.

The "facts" say:
> FishTank is a web application

> Secrets are stored securely within your browser's localStorage.

> Our engineers implemented the best encryption technology one can get.†

> The scope of this project is limited to the client-side implementation of the FishTank.

> Do not perform any attacks against the server.

Fine. The interesting bit here is, that the secrets are stored in localStorage. After accepting the rules I was presented with two FishTank Safes und the password for one of them.

| FishTank                                                                                                                                                                                                                                                                                                                                                                           |         Passowrd          |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-----------------------: |
| LglVOnp3GQgiYWtLaDZgAtQdltvGThVfpM3Cvh6RAlFVKDFRHxs9NQgzGiMGYwh9X31HJP2sRnEaG+sJ8DsbU25JFi7q8FoNNw==                                                                                                                                                                                                                                                                               | Th1sIs4V3ry$3cur3P4ssw0rd |
| aF8ccGR1QmIDSRJFEwUWLypj43LLGY2sdSPKHo/FGtsTQngfCR1qUyl/MX0yP2Z7Pzlgoz7DCzFZLXssD2R4D8+tE76ltzTnZxQMQ31mFBvIyhjS6cM3rQzIINgBAGYc2/z3fjQ4sgi5DHYLyVSLuTeFoXau6reAtcgxCA2MfaLolHgq7TDggWATegXF6vt75Ll15G8yO0qu9Cu4uutgduspqebrL+c26PE5/IkveggMFokUKxoNKh6K3K5OKcbE6GaVR0ikzG2fa60fXrtltmqWBKUJJ44Xvnc46os9hYFpRZUUpmEfJDhFa+sbiQfGiZKUl9Lg2HTMENws/ZSwYE3hUBCWALpGWwELm+RVl1fSNOdUhF |

The Tanks look like they are Base64 encoded. Trying to decode them gives a seemingly random stream of bytes that is not ASCII printable.

    First tank decoded as Base64
    2e 09 55 3a 7a 77 19 08 22 61 6b 4b 68 36 60 02 d4 1d 96 db c6 4e 15 5f a4 cd c2 be 1e 91 02 51 55 28 31 51 1f 1b 3d 35 08 33 1a 23 06 63 08 7d 5f 7d 47 24 fd ac 46 71 1a 1b eb 09 f0 3b 1b 53 6e 49 16 2e ea f0 5a 0d 37

If you go to the application itself, you are asked to put in the FishTank Data to load it. After giving it the first tank, it asks for the password and we can already see the data being put into localStorage with the key being *"fishtank"*. From here you can also choose to delete the current tank, which also deletes it from localStorage. When entering the password you are greeted by a single Key-Value pair. The stored key is *"flag"* and the corresponding value is "Your flag is in another castle". Now you can delete and add other Key-Value pairs. After adding a pair (a:a), the string stored in localStorage expands, but some parts do not change. 

    Before: LglVOnp3GQgiYWtLaDZgAtQdltvGThVfpM3Cvh6RAlFVKDFRHxs9NQgzGiMGYwh9X31HJP2sRnEaG+sJ8DsbU25JFi7q8FoNNw==
    After:  LglVOnp3GQgiYWtLaDZgAtiK7icSlDwNQ9P2ejj1oq9VKDFRHxs9NQgzGiMGYwh9X31HJP2sRnEaG+sJ8DsbU25JFi7q8FoNZqj0VK5B7xMj

The first part stayed the same. The end also stayed the same but something got appended. If we delete our new pair, the string does not return to its's first value.

    Beginning: LglVOnp3GQgiYWtLaDZgAtQdltvGThVfpM3Cvh6RAlFVKDFRHxs9NQgzGiMGYwh9X31HJP2sRnEaG+sJ8DsbU25JFi7q8FoNNw==
    Now:       LglVOnp3GQgiYWtLaDZgAtJ4DMqeZ6DPXCPeUw6I+4NVKDFRHxs9NQgzGiMGYwh9X31HJP2sRnEaG+sJ8DsbU25JFi7q8FoNNw==

But again the start and the end are identical with few changes in the middle. After playing around a bit with different inputs we can now lock the Tank with the presented "Lock" Button. The localStorage does not change in this case and we can now re-enter our password or delete the tank. If we try to enter a wrong password we get an error message.

Now it is time to look at the source of the page. The HTML source is pretty standard. A few buttons, inputs etc. Some .js scripts are being loaded:
```html
    <script src="/assets/js/jquery-3.4.1.min.js"></script>
    <script src="/assets/js/popper.min.js"></script>
    <script src="/assets/js/fishtank.js"></script>
    [...HTML...]
    <script src="assets/js/bootstrap.bundle.min.js"></script>
```
There is also some inline Javascript. 2 functions and the listeners for all the  buttons. Most of those show or hide some buttons to represent the current state of the webapp. For the load function which seems to do, what it's name suggests, there is a regex that checks the fishtank Data. 

    /^(?:[A-Za-z0-9+/]{4})*(?:[A-Za-z0-9+/]{2}==|[A-Za-z0-9+/]{3}=)?$/

Looks pretty much like a regex for Base64 encoded data. If the entered data is valid it gets put into localStorage.

    localStorage.setItem('fishtank', data)

The buttonUnlock calls the function "unlockTank". It creates an new tank item and then calls "reloadTank". It calls *tank.open(password)* and fills the content of the HTML with the results.
```javascript
    window.tank.open(password).then(data => {
            $('#divView tbody').empty()
            Object.keys(data).forEach(k => {
                const tr = $('<tr>')
                const key = $('<td>')
                key.text(k)
                const value = $('<td>')
                const pre = $('<pre>')
                pre.text(data[k])
                value.append(pre)
                const controls = $('<td>')
                controls.append($('<button>').addClass('btn btn-danger').text('Delete').on('click', async () => {
                    window.tank.delete(k)
                    localStorage.setItem('fishtank', await window.tank.save())
                    reloadTank()
                }))
                // tr.addClass('bg-primary')
                tr.append(key, value, controls)
                $('#divView tbody').append(tr)
            })
```
Next, we take a look at the embedded .js scripts and see if we can find the *.open()* function. *jquery*, *popper* and *bootstrap* seem like the libraries used to load and update our HTML. So we should take a look at *fishtank.js*. In Firefox's sources tab we can look at the file but it seems like it was minified. Luckily there is a pretty-print function available. Scrolling through the result makes it seem like there is a lot of complicated stuff happening. There are dozens of functions all with minified names. Using the search to find the *open()* function we jump to a bottom part of the script. There we find same functions with readable names, which seem interesting like *open(), add(), save(), key(), encrypt(), decrypt()*. All of those belong to a class named *m*. *open()* gets *t* as a parameter, which is the password. Copying the prettified Code to an Editor and renaming the variables makes the code more readable. 
```javascript
            open(password) {
            return r(this, void 0, void 0, (function* () {
                if (this.isOpen) return this._cache;
                if (void 0 === password) throw m.ERROR_INVALID_PASSWORD;
                try {
                    this._cache = yield this.decrypt(password),
                        this.password = password
                } catch (t) {
                    throw t
                }
                return this._cache
            }))
        }
```
If the tank is already open, it uses the cache and if the password is empty it tries to decrypt the tank using the *decrypt()* function. 
```javascript
    decrypt(password) {
        return r(this, void 0, void 0, (function* () {
            const decodedData = w.from(this.data, 'base64');    // parse Base64 from FishTank data to uint8 Array
            decodedData.fill(32, 0, 32);                        // fill with 32 from position 0 till 32.. weird
            const key = yield m.key(password);
                                                                                        
            for (let i = 0; i < decodedData.length; i++) 
                decodedData[i] ^= i < 32 ? 0 : (key[i % 16] + i - 32) % 256;   //first 32 bytes are set to 0
            try {
                return JSON.parse(decodedData.toString().trim())
            } catch (t) {
                throw m.ERROR_INVALID_PASSWORD
            }
        }))
    }
```
*decrypt()* defines a constant which gets filled with the Base64 decoded data of the tank (so the initial guess with Base64 was right). Afterwards it fills the first 32 bytes with *32* (as an integer). It seems like there might be something interesting about these bytes. Base64 decoding and looking at the first 32 bytes of both tanks shows us, that they are not the same. Then *e* is declared and filled with the output of *key(password)*. Therefore *e* is called key now. Then it starts with what seems like the encryption. In a loop it goes over every byte of the decoded data and XORs it with one of the first 16 bytes of the key, adds the position and substracts 32, which is then moduloed by 256. The first 32 bytes of the data, which were filled with 32 earlier are ignored and set to 0. Afterwards it checks the decrypted data by trying to parse it as JSON after trimming it to get rid of the first 32 bytes which are whitespaces when interpreted as ASCII. Now let's take a look at *key()*.
```javascript
    const r = function(t, r) {
                    if (!(r < 0 || r >= t.length))
                        return t[r]
                }(t, 0);
                if (void 0 === r)
                    throw m.ERROR_INVALID_PASSWORD;
                let e = r;
                "string" == typeof r ? e = w.from(r, "utf8") : Array.isArray(r) && (e = w.from(r));
                const n = w.alloc(32).fill(0);
                let i = 0
                  , o = 1;
                const f = n.length / 2;
```
The assignment of the first constant seems really strange but the FireFox debugger confirms, that it just assigns the password to it. The password is then also assigned to variable *e*. It then checks if the password is a string. If so, it converts *e* to a Uint8Array containing the password in Uint8. This is done via various complicated looking steps but using the debugger easily shows this. If the password is not a string (which should not be possible) it checks if it is an array. If so, it just copies this array into *e*. Next, an empty 32 byte Uint8 array named *n* is filled with 0s. Since *n* is the return value of the function, it is safe to assume, that it is they key used by *decrypt()*. Now two variables are declared. *i* is set to 0 and *o* to 1. Another constant is defined and set to the half of the lengt of *n*, which is always 16. Now a complicated looking for loop is executed.
```javascript
     for (let t = 0; t < e.length; t++) {
                    let r;
                    -1 == t && (r = 1337),                                  // never happens
                    r && r > 0 ? (n[v(i, f)[0]] |= t >= 0 ? e[t] & r : 0, n[v(i++, f)[1]] |= t >= 0 ? e[t] & r : 0, t += r) : 
                    (n[v(i, f)[0]] ^= e[t], n[v(i, f)[1]] = e[t] ^ Math.floor(256 * Math.random()), i += 1),
                    t == e.length - 1 && (t = o++)
                }
```
After fiddling with the debugger and looking at the code I was able to tell, that the first 16 bytes of the key are derived from the password and the last 16 bytes are random. This makes sense, since *decrypt()* only uses the first 16 bytes of the key. *v* is also a function which simply return the modulo of first and second argument and the same again, but adds *r* to it. During my research I examined the function to understand it completely, but since it is really weird an not useful for finding the solution (except the fact that only the first 16 bytes are derived from the password), I will skip this part here. In short: It fills *n* at positions 0-15 with the password content and positions 16-31 with random numbers. Then it loops over *n* many times and does some XOR. 

After looking at all the different functions and thinking about how to get to the flag with what I found out, I decided to look at the *encrypt()* function. At the very first sight with the pretty printed version of the code done by FireFox I was a bit scarred since it looked like this.
```javascript
    staticencrypt(t, e) {
      return r(this, void 0, void 0, (function * () {
        const r = w.from(JSON.stringify(e), 'utf8'),
        n = yieldm.key(t);
        for (let t = 0; t < r.length; t++) r[t] ^= (n[t % 16] + t) % 256;
        return (0, b[((t, r) =>([] + {
        }) [5] + ([] + {
        }) [1] + r + t[1]) `${(''[0] + '') [1]
      }
      c` + (['42',
      '1337',
      '666'].map(parseInt) [2] + '') [1] + ([] + {
      }
      + {
      }
      - [] + {
      }) [9]]) ([n,
      r]).toString('base64')
    }))
```
Firefox's pretty print function somehow lost the spaces between keywords like static or yield and what comes afterwards. Also the formatting and indents were not really helpful. Also debugging with FireFox was a bit of a mess so I switched to chromium. To my surprise the code was way more readable with chromiums pretty print. 
```javascript
     static encrypt(t, e) {
            return r(this, void 0, void 0, (function*() {
                const r = w.from(JSON.stringify(e), "utf8")
                  , n = yield m.key(t);
                for (let t = 0; t < r.length; t++)
                    r[t] ^= (n[t % 16] + t) % 256;
                return (0,
                b[((t,r)=>([] + {})[5] + ([] + {})[1] + r + t[1])`${(""[0] + "")[1]}c` + (["42", "1337", "666"].map(parseInt)[2] + "")[1] + ([] + {} + {} - [] + {})[9]])([n, r]).toString("base64")
            }
            ))
     }
```
Still, the code is obfuscated. In the original code *t* is the password, *e* the data, *r* the data in an Uint8 Array and *n* the key, so I also renamed those.
```javascript
    static encrypt(password, data) {
            return r(this, void 0, void 0, (function*() {
                const data_Uint = w.from(JSON.stringify(data), "utf8")
                  , key = yield m.key(password);
                for (let t = 0; t < data_Uint.length; t++)
                    data_Uint[t] ^= (key[t % 16] + t) % 256;
                return (0,
                b[((t,r)=>([] + {})[5] + ([] + {})[1] + r + t[1])`${(""[0] + "")[1]}c` + (["42", "1337", "666"].map(parseInt)[2] + "")[1] + ([] + {} + {} - [] + {})[9]])([key, data_Uint]).toString("base64")
            }
            ))
        }
```
The function XORs the data_Uint with the key the same way, *decrypt()* does it. The only difference is, that there is no 32 byte offset. The return statement is the last part we have not looked at and must therefore give us some new insights. Looking at the debugger, the return of this function is the whole FishTank data, base64 encoded. The first, gibberish part seems to be a function call given key and data_Uint as parameters, which is then encoded in a bas64 string (which is the return value). Stepping through the function with the debugger we can see, that the return value builds up to conc and then jumps to a function *o.concat()*. Since concat usually concatenates stuff this might be the solution. It concats key und data_Uint and base64 encodes this. To further verify this, I evaluated the expressions used here to build the function call in the console. `([] + {})[5]` evaluates to the letter 'c'. `([] + {})[1]` to 'o'. *r* is 'n' here. *t* is an array with [0] empty and [1] is 'c'. `${(""[0] + "")[1]}c\` evaluates to 'nc'. `(["42", "1337", "666"].map(parseInt)[2] + "")[1]` evaluates to 'a'. `([] + {} + {} - [] + {})[9]` evaluates to 't'. After finishing my WriteUps I will definitely check out why these expressions evaluate as they do. Now let's check the first 32 bytes of the decoded data and compare it with the key extracted by the debugger when unlocking with the right password or adding an entry and saving it. 

    Base10
    46 9 85 58 122 119 25 8 34 97 107 75 104 54 96 2 9 150 168 89 106 164 176 177 41 120 19 108 158 239 167 35
    46 9 85 58 122 119 25 8 34 97 107 75 104 54 96 2 212 29 150 219 198 78 21 95 164 205 194 190 30 145 2

Here we go. The first 16 bytes needed to decrypt the data match. Now we only need to extract the first 16 bytes of the other FishTank, enter any password when trying to decrypt it and replace the first 16 bytes of the key with the key we found. The key for the FishTank is:

    Base10
    104 95 28 112 100 117 66 98 3 73 18 69 19 5 22 47

Now the FishTank greets us with the flag for this challenge.

    -----BEGIN FISHBOWL FLAG-----
    Lf03BTPEPZVqqiO47JlXNB2rsPmnPjjaivmHpw6P
    xFbq2gYuLdOTExCb/3cnkKn21Xhe/s7Bk4IVp4Qy
    mi9/3L8TsVwuPqX7jZ6IXpokiHiYICDMeFVRdN8z
    bEm+JJnG/hCvgCRzjaZy+sAjg6p6Jjypn89QXFac
    DXwjq764
    -----END FISHBOWL FLAG-----

Even if I took the long route analyzing the JavaScript, I still learned a lot and came up with another idea to crack the tank. I  thought about attacking using the parts I already know about the content of the Tank. Knowing, it is internally stored as JSON and the key value pair probably being "flag:-----BEGIN FISHBOWL FLAG-----[...]" (from the first challenge) I might have been able to guess enough characters to crack the key this way. For this I would have needed the first 16 bytes of the content. By looking at the stringifyed JSON in binary form the flag would be internally stored like this:

    {"flag":"-----BEGIN FISHBOWL FLAG-----[....]

This gives us more than enough bytes to reverse the key. When I tried to do this by hand to verify, there always was an offset between the expected character and what came out of the XOR. This is caused by how the decryption is done since the counter is added to the key before XOR.
```javascript
    (key[counter % 16] + counter - 32) % 256;
```
With this in mind it is easily possible to reverse the key from knowing the input.
