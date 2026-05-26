+++
title = "Breaking E2E Encrypted Web Apps (Client side Code Analysis)"
date = "2025-08-01T01:35:14+05:30"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = "drbl4k"
authorTwitter = "" #do not include @
cover = ""
tags = ["decryption", "cryptography", "client-side", "JavaScript-analysis"]
keywords = ["Burpsuite", ""]
description = "Decrypting requests and responses with the help of client-side JavaScript code."
showFullContent = false
readingTime = true
hideComments = false
+++

We all must have encountered that one particular app which encrypts very single POST/GET request that is being sent. Worst part is that even the response received will be encrypted and cannot be viewed at proxy level (Inside Burp). In today’s blog lets cover how to approach such applications, one cryptographic misconfigurations that took place on the encryption part and how I managed to get the decryption logic and found hidden vulnerabilities present in the encrypted data.

The picture below is what the blog is going to be about. (Just so you don’t wonder whether this is the right blog)

![1.jpeg](/images/E2E-decryption/1.jpeg)

*Encrypted GET request with encrypted response*

***Spoiler***: In this blog’s case I managed to find the static key and iv, proceeded for further decryption. I **used Chat-GPT** to make me a script for decryption. However, applications do encrypt and decrypt on runtime as well. We have to understand the logic of encryption in order to figure out the same .

---

### *Base Logic*

Let’s say we have an application and we are able view the data inside of it on the **browser level**. Now at the **proxy level**, things take a turn, as the request and response become **encrypted** (unreadable). This it self means that, on the browser/front end code, there should be a logic present somewhere, to encrypt and decrypt the request and responses. Having this thing cleared is important in order to know what exactly are we looking for.

***What are we looking for?:***

- The logic of encryption and decryption. (Inside JavaScript files)
- Understand the algorithm and padding type.
- Understand how the key and iv are fetched and trace it back.
- Check if iv re-use takes place or a static iv is present.

Since we know what we are looking for now, let’s dive into the techniques of decryption 

---

### *Techniques*

Setup the Burpsuite and load the application and make a login request. This should have triggered all sorts of files/logic required for encryption and decryption. Now in the HTTP history, search for the following key words,

- .encrypt
- .decrypt
- encfunction
- CryptoJS.AES
- .crypt
- encryptAes

The above keywords should lead you to some sort of encryption logic. *(**Note that some applications might use custom words to name the function. Debug whatever you suspect to be the logic)*** in my case the keywords was present inside the `chunk.js` file.

*Keyword:*

![keyword.png](/images/E2E-decryption/keyword.png)

*Encryption & Decryption Logic:*

![logic.jpeg](/images/E2E-decryption/logic.jpeg)

Based on the number of occurrences you might have to scroll through a lot of code to find the logic. In my case the logic is not obfuscated *(you can flag this as a security issue after successful decryption)* . Now, since we found the logic lets go ahead and hook these functions via developer console. Open the dev console and navigate to “Sources → chunk.js”. Search any particular term from the logic to find the function. hook whatever you feel like is the place of encryption. In my case I hooked at the lines shown in the below image. After hooking try refreshing or making a POST request to trigger the logic. Now you can hover over the key and the iv to get the values.

![2.jpeg](/images/E2E-decryption/2.jpeg)

![3.jpeg](/images/E2E-decryption/3.jpeg)

What do we have till now:

- Got the Logic for both encryption and decryption.
- AES 128 CBC has been used.
- Pkcs5Pad padding has been used.
- The key and the iv are recoverable in Unit8Array format.

Now before proceeding with the decryption part, let’s discuss where the developers had misconfigured. Its critical to understand the misconfiguration to exploit it further.

---

### *The vulnerability*

- The application was using a **static iv and a key** to encrypt and decrypt. Iv is something that should **not be re-used** after a single encryption/decryption takes place. *(This should be flagged upon successful decryption)*
- AES-CBC is an outdated encryption mode. the logic of it is easy to break.
- Even if the key and the iv is present in the form of Unit8Array, it still can be used for decryption.
- There was no obfuscation on the logic whatsoever. In most cases unlike a Unit8Array, you can find the hardcoded 32/128 bit value which can directly be used for decryption.

---

### *Decrypting Everything*

Since we have the Unit8Array of both key and iv, I asked chat GPT to write me a html file with a decryption script for the identified logic. here is what it gave me:

```html
<!DOCTYPE html>
<html>
<head>
  <title>Offline AES-CBC Decryption</title>
  <script src="https://cdn.jsdelivr.net/npm/crypto-js@4.1.1/crypto-js.min.js"></script>
</head>
<body>
  <h2>Decryption Output</h2>
  <pre id="output">Decrypting...</pre>

  <script>
    // Replace with your actual Uint8Array key and IV
    const keyArray = new Uint8Array(["add the array here"]);

    const ivArray = new Uint8Array(["add the array here");

    const ciphertextHex = "acf94ec15f5ef42ec7994ee0e01c92a779682ab797330ee01dacd8a9c2fc5df7 ";

    function decryptAES_CBC(ciphertextHex, keyArray, ivArray) {
      const key = CryptoJS.lib.WordArray.create(keyArray);
      const iv = CryptoJS.lib.WordArray.create(ivArray);
      const ciphertext = CryptoJS.enc.Hex.parse(ciphertextHex);

      const decrypted = CryptoJS.AES.decrypt(
        { ciphertext },
        key,
        {
          iv: iv,
          mode: CryptoJS.mode.CBC,
          padding: CryptoJS.pad.Pkcs7
        }
      );

      return decrypted.toString(CryptoJS.enc.Utf8);
    }

    try {
      const plaintext = decryptAES_CBC(ciphertextHex, keyArray, ivArray);
      document.getElementById("output").innerText = plaintext || "[Decryption failed]";
    } catch (err) {
      document.getElementById("output").innerText = "Error: " + err;
    }
  </script>
</body>
</html>
```

![5.jpeg](/images/E2E-decryption/5.jpeg)

*Decrypted Request.*

![7.jpeg](/images/E2E-decryption/7.jpeg)

*Decrypted response.*

Post running this script on the mentioned *(ciphertextHex line:17)* encrypted text, I was able to get the decrypted output. Now this opens up a lot of possibilities for things like IDOR, SQL Injections etc. I just have to generate another script to encrypt the body with the same key and iv.

---

### *Key Takeaways*

- Always look into the code
- Analyse how keys are used and check whether they are static across multiple accounts/sessions.
- Its not bad to ask for help, (Ask AI).
- Be clear with padding types as they might cause failure of decryption
- encryption should not be the sole line of defence. Comprehensive security testing must be conducted for potential vulnerabilities at multiple levels.

---

### References

- https://www.linkedin.com/pulse/how-i-exploited-idor-encrypted-mobile-api-withfrida-george-joseph-rydjc/?trackingId=dwlmpvH4Qyil87BHWllpYw%3D%3D

---

**Happy Hunting !**
