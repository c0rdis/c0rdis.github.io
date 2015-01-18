---
layout: page
title: "OTNote"
comments: false
sharing: false
footer: false
---
<script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/rollups/aes.js"></script>
<script src="http://crypto-js.googlecode.com/svn/tags/3.1.2/build/components/core-min.js"></script>

<script>
    function encrypt(text,key,timestamp){
    key = timestamp+key;
    alert(key);
    var encrypted = CryptoJS.AES.encrypt(text, key);
    return encrypted;
}

    function decrypt(text,key,timestamp){
    key = timestamp+key;
    var decrypted = CryptoJS.AES.decrypt(text, key).toString(CryptoJS.enc.Utf8);
    return decrypted;
}
</script>



###OTN (One-time note) - secure way to transfer data

Always wanted to have my own version of Privnote to be sure of how the data is handled on the server... Finally, here it is.

####Description:

- connection is secured by HTTPS
- note is encrypted on the client side with AES256 and transmitted to the server
- it's up to you which encryption key to set - by default it's your timestamp
- server doesn't know the contents of the note - it is stored as-is with its hash as it's name
- upon successful submission, a URL is generated
- once the URL is accessed, the file is permanently deleted from the server
- source code is available on my <a href="https://github.com/c0rdis/otnote">github page</a>

It is recommended to make your note password protected:<br> 
<input id="encryptionKey" type="password" size="100" name="ekey"><br>

<form id="frm2" action="form_action.asp">
Text to encrypt:<br>
<textarea id="note" name="note" cols="100" rows="6"></textarea>
<input id="timestamp" type="hidden" name="timestamp"><br>
<input id="btnSendNote" type="button" onclick="sendForm()" value="Submit">
</form>

<script>
function sendForm() {
    var txt = document.getElementById('note');
    var key = document.getElementById('encryptionKey');
    var timestamp = document.getElementById('timestamp');
    timestamp.value = Date.now();
    txt.value = encrypt(txt.value,key.value,timestamp.value);
    alert(decrypt(txt.value,key.value,timestamp.value));
    document.getElementById("sendNote").submit();
    
}
</script>