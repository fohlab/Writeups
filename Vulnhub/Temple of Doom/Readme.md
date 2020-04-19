# Temple of Doom

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Kali machine: **10.0.2.15**
    * Temple of Doom machine: **10.0.2.13**
  

  
### Information Gathering
---

* Running **nmap** with `nmap -sC -sV -p- -A -oA templeofdoom 10.0.2.13` reveals **two** ports.
<!-- 1 -->  

1. **SSH** on default port **22** .
2. A <a href="https://en.wikipedia.org/wiki/Node.js">NodeJs</a> server running at port **666**.

Let's check the server at port **666**, while also setting **gobuster** and **nikto** running in the background.

<!-- 2 -->

