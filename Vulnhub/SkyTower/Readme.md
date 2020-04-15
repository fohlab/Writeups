# Skytower

* ### Setup

  Download the virtual machine <a href="https://www.vulnhub.com/entry/brainpan-1,51/">here</a> and import it in either VirtualBox or        VMWare.
  * **Addresses** - This will be important to understand the writeup.
    * My Kali machine: **10.0.2.15**
    * SkyTower machine: **10.0.2.10**
  

  
### Information Gathering
---

* Running **nmap** with `nmap -sC -sV -p- -A -oA skytower 10.0.2.10` reveals **three** ports.

<!-- nmap -->

1. **SSH** appears to be filtered.
2. A <a href="https://en.wikipedia.org/wiki/Squid_(software)">squid proxy</a> is running at port **3128**.
3. And finally a **Web server** 

* Navigating to **http://10.0.2.10**, we find it is a login page asking for credentials.

<!-- login page -->

* Starting up **gobuster** and **nikto** doesn't return any interesting results.

<!-- go -->
<!-- nikto -->

* Navigating to **http://10.0.2.10:3128/** returns a non-useful error.

<!-- squid -->

### Vulnerability Analysis
---
* Trying to login with various default creds **admin/admin**, **admin/password**, **admin/123456** isn't fruitful.

* Let's test for **SQL Injection** with email= **webmaster** and password= **pass'**

  Server returns
  
  <!-- 3 -->
  
  * Looks like it is **Vulnerable!**.
  
* Start up **Burp Suite**, submit a fake login and capture the request and send it to **repeater** for easier testing.

<!-- 4 -->

* Let's try `pass=nopass' OR 1=1 -- -` to bypass the authentication.

<!-- 5 -->

* Our login failed with an SQL error, but we notice something far more important in the respone. Some of our input was **filtered out**.
Specifically, **OR**,**=** and our comment **--**.

* We will circumvent this filter by doing the following **substitutions** that are allowed in **sql syntax**
  1.
  
