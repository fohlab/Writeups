# Skytower

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Kali machine: **10.0.2.15**
    * SkyTower machine: **10.0.2.10**
  

  
### Information Gathering
---

* Running **nmap** with `nmap -sC -sV -p- -A -oA skytower 10.0.2.10` reveals **three** ports.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/1.png">

1. **SSH** appears to be filtered.
2. A <a href="https://en.wikipedia.org/wiki/Squid_(software)">squid proxy</a> is running at port **3128**.
3. And finally a **Web server** 

* Navigating to **10.0.2.10**, we find it is a login page asking for credentials.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/2.png">


* Starting up **gobuster** and **nikto** doesn't return any interesting results and navigating to **10.0.2.10:3128** returns a non-useful error. So this login page appears to be our main attack surface.

### Vulnerability Analysis
---
* Trying to login with various default creds **admin/admin**, **admin/password**, **admin/123456** isn't fruitful.

* Let's test for **SQL Injection** with email= **webmaster** and password= **pass'**

  Server returns
  
  <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/3.png">

  
  * Looks like it is **Vulnerable!**.
  
### Exploitation
---

* Start up **Burp Suite**, submit a fake login and capture the request. Then send it to **repeater** for easier testing.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/4.png">


* Let's try `pass=nopass' OR 1=1 -- -` to bypass the authentication.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/5.png">


* Our login failed with an SQL error, but we notice something far more important in the respone. Some of our input was **filtered out**.
Specifically, **OR**,**=** and our comment **--**.

* We will circumvent this filter by doing the following **substitutions** that are allowed in **sql syntax**  
  * **OR**  &rarr; **||**
  * **=**   &rarr; (blank) because it is not needed since just **OR 1** is a valid **_true_** expression
  * **--**  &rarr; **#**
  
* Let's try again then

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/6.png">


**We succesfully logged in!**

* Typing the same input in the Browser returns this page after login, giving us **ssh credentials** for the user **john**.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/7.png">

* Trying to log in with those credentials doesn;t work, but this is due to the ssh port being **_filtered_**.
One solution is to reach that port through the proxy running at **3128**. So we must set the correct **proxychains** configurations by adding `http 10.0.2.10 3128` at the end of `/etc/proxychains.conf`.

Now run ssh through proxychains with `proxychains ssh john@10.0.2.10`. This works but the connection opens and closes immediately. We can fix this by connecting with  `proxychains ssh john@10.0.2.10 /bin/bash` which will execute **bash** upon entering the connection.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/8.png">


Now we have a shell.

### Privilege Escalation
---

Going back a directory and runnnig `ls`, we discover other **two** users in the system. **sara** and **william**.

<img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/9.png">

  
There are two ways to get their credentials.

* ## First way

  Get the first few lines of code of the login page with `head /var/www/login.php`
  
  <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/10.png">

  
  This way we find out the root password for the database. Note that we also see what kind of syntax filter was implemented :)
  
  If we want to login to the database and execute commands easily, we will have to get a more stable shell. I managed to do that by switching to **/bin/sh** by executing `/bin/sh -i`.
  
  <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/11.png">


  This way we got the credentials for the other users
  
  * ## Second way
  
  We could also leverage the SQL Injection.
  When we gave `' || 1 #` as input, it logged in as the first user found at the `login` mysql table.
  What if we used the mysql the `LIMIT`  and `OFFSET` command?
  
  Going back into burp
  
  <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/12.png">
  
  We can do the same for william with **OFFSET 2**.
  
  * Login as sara and see what you can do as **root** with `sudo -l`.
  
  <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/13.png">
  
  We see that we can run `cat` and `ls` as root with no password fot the directories from `/accounts/*`. But this can easily be bypassed and exploited.
  
 <img src="https://github.com/astasinos/Writeups/blob/master/Vulnhub/SkyTower/images/14.png">
 
 ## Rooted!
 
