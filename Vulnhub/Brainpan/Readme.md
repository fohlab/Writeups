# Brainpan

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  
  
### Information Gathering
---

First of all, we run **nmap** with `nmap -sC -sV -p- -oA brainpan.nmap 10.0.2.8` (replace your target IP)  

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/first.png">

We can see that there are only two ports open and two services running on the machine, one at **9999** and the other at **10000**.

Navigating to **10.0.2.8:9999** with firefox returns   

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/second.png">

The service running there seems to be expecting a password but it doesn't seem to work over http.

Let's try talking to it with **nc** with `nc 10.0.2.8 9999`

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/six.png">


We can now input a password but we don't know it **_yet_**.

Navigating to **10.0.2.8:10000** returns  
<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/third.png">  

Just a page with an image.

At this point we can put **gobuster** (directory fuzzing tool) running in the background with   
`gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.0.2.8:10000/ -t 5 --timeout 50s`

The high timeout was selected because the HTTP Server running is just a python module and can't handle that much traffic at once. Choosing a lower one returns a lot of errors.

After a while, gobuster returns an interesting directory **_/bin_**.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/fourth.png">


Navigating there we see only one file called **brainpan.exe**

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/five.png">


Let's download it.

Running it locally with **wine** reveals it is someking of server expecting for connections at port 9999, just like the service on the box.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/seven.png">


The program now listens on localhost, so let's connect with **nc** at our own machine at port 9999.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/eight.png">


It is clear now that **brainpan.exe** is a copy of the program listening at **9999** at the **remote** box.

Although it is a Windows executable, we can still run `strings` on it.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/nine.png">


This looks interesting.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/ten.png">


This seems to be the correct password but nothing happens when we type it, even at the remote service, so it's pretty useless.

## Vulnerability Analysis
---

We can move **brainpan.exe** to our windows machine for easier analysis.

Start up **Immunity Debugger** and run the program.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/eleven.png">

We will try to crash the application.

Open an ubuntu terminal from the installed ubuntu subsystem on windows, and create a python script like this

```python

print b"A" * 1000
```

Now echo the output to a file `python test.py > crash` , and run `nc 192.168.2.11 9999 <crash` to give the contents of **crash** as input for the password to the program. After running, take a look at **immunity**.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/twelve.png">

We see that not only we crashed the program but we also **overwrote** the instruction pointer **eip**. 
We have a **Buffer Overflow!**

## Exploitation
---

Through a little trial and error, we find that we need to print **524** bytes to start overwriting **eip**.




