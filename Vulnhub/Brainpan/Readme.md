# Brainpan

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  
  
### Information Gathering
---

First of all, we run **nmap** with `nmap -sC -sV -p- -oA brainpan.nmap 10.0.2.8` (replace your target IP)  

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/first.png">

We can see that there are only two ports open and two services running on the machine, one at **9999** and the other at **10000**.

Navigating to **10.0.2.8:9999** with firefox returns
<!-- second -->
The service running there seems to be expecting a password but it doesn't seem to work over http.

Let's try talking to it with **nc** with `nc 10.0.2.8 9999`

<!-- six -->

We can now input a password but we don't know it **_yet_**.

Navigating to **10.0.2.8:10000** returns
<!-- thrid -->
Just a page with an image.

At this point we can put **gobuster** (directory fuzzing tool) running in the background with `gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.2.8:10000/ -t 5 --timeout 50s`

The high timeout was selected because the HTTP Server running is just a python module and can't handle that much traffic at once. Choosing a lower one returns a lot of errors.

After a while, gobuster returns an interesting directory **_/bin_**.

<!-- fourth -->

Navigating there we see only one file called **brainpan.exe**

<!-- five -->

Let's download it.

Running it locally with **wine** reveals it is someking of server expecting for connections at port 9999, just like the service on the box.

<!-- seven -->

The program now listens on localhost, so let's connect with **nc** at our own machine at port 9999.

<!-- eigth -->

It is clear now that **brainpan.exe** is a copy of the program listening at **9999** at the **remote** box.

Although it is a Windows executable, we can still run `strings` on it.

<!-- nine -->

This looks interesting.

<!-- ten -->

This seems to be the correct password but nothing happens when we type it, even at the remote service, so it's pretty useless.

## Vulnerability Analysis
---




