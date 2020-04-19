# Temple of Doom

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Kali machine: **10.0.2.15**
    * Temple of Doom machine: **10.0.2.14**
  

  
### Information Gathering
---

* Running **nmap** with `nmap -sC -sV -p- -A -oA templeofdoom 10.0.2.14` reveals **two** ports.
<!-- 1 -->  

1. **SSH** on default port **22** .
2. A <a href="https://en.wikipedia.org/wiki/Node.js">NodeJs</a> server running at port **666**.

Let's check the server at port **666**, while also setting **gobuster** and **nikto** running in the background.

<!-- 2 -->

We see an under construction message. After fuzzing the url a little bit and reloading the page a few times, the application errors out with 

<!-- 3 -->

This error tells us that something went wrong with the serialization process and we also notice the module doing the serialization is **node-serialize**.

* Unfortunately, this module contains a serious **deserialization vulnerability**, allowing **remote code execution**. There is a great analysis of such attacks in nodejs in this pdf https://www.exploit-db.com/docs/english/41289-exploiting-node.js-deserialization-bug-for-remote-code-execution.pdf

### Vulnerability Analysis
---
* First we need to understand what a **deserialization vulnerablity** is. From <a href="https://www.acunetix.com/blog/articles/what-is-insecure-deserialization/">this article</a> by http://acunetix.com, 
> **_Insecure Deserialization_** is a vulnerability which occurs when untrusted data is used to abuse the logic of an application, inflict a denial of service (DoS) attack, or even execute arbitrary code upon it being deserialized. It also occupies the **#8 spot in the OWASP Top 10 2017 list.**
Serialization refers to a process of converting an object into a format which can be persisted to disk (for example saved to a file or a datastore), sent through streams (for example stdout), or sent over a network. JSON and XML are two of the most commonly used serialization formats within web applications.
Deserialization on the other hand, is the opposite of serialization, that is, transforming serialized data coming from a file, stream or network socket into an object.
Web applications make use of serialization and deserialization on a regular basis and most programming languages even provide native features to serialize data. **It’s important to understand that safe deserialization of objects is normal practice in software development. The trouble however, starts when deserializing untrusted user input.**

Based on the paper mentioned in the Information gathering section, we will create a test environment to confirm the vulnerablitiy.

1. First install **npm** with `apt install npm`, then the vulnerable **node-serialize** module with `npm install node-serialize`.
  Note: npm will now warn you the node-serialize was found to contain a serious vulnerability.
  
