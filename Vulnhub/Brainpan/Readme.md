# Brainpan

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  
  
### Information Gathering
---

First of all, we run **nmap** with `nmap -sC -sV -p- -oA brainpan.nmap 10.0.2.8` (replace your target IP)  

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/first.png">

We can see that there are only two ports open and two services running on the machine, one at **9999** and the other at **10000**.



